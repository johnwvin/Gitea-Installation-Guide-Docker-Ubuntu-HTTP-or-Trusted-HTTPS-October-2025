# Gitea Installation using Docker on Ubuntu 24

## HTTP-only:

### Install Docker using their guide: 

```
https://docs.docker.com/engine/install/ubuntu/

```
add your user to the Docker group:

```
sudo usermod -aG docker ${USER}

```

### Ready the environment

1. Create the Directory: 

```
mkdir ~/gitea
cd ~/gitea

```
2. Create the compose.yml file:

```
nano ~/gitea/compose.yml

```

#### Paste this into the .yml file:

```
version: "3"

networks:
  gitea-net:
    external: false

services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: always
    networks:
      - gitea-net
    volumes:
      - ./gitea:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      # Port for Gitea web UI
      - "3000:3000"
      # Port for Git via SSH (using 2222 on the host to avoid conflict with system SSH)
      - "2222:22"
    depends_on:
      - db
    environment:
      # These variables pre-configure the database connection
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=db:5432
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=gitea
      - GITEA__database__PASSWD=[CHOOSE A PASSWORD !!!]
 #    - GITEA__webhook__ALLOWED_HOST_LIST=[UNCOMMENT AND REPLACE THIS PLACEHOLDER WITH AN IP ADDRESS TO ALLOW WEBHOOKS WITH A SERVICE SUCH AS JENKINS ON A LOCAL IP !!!]
  
  db:
    image: postgres:15
    container_name: gitea-db
    restart: always
    networks:
      - gitea-net
    volumes:
      # Persists database data on the host machine
      - ./postgres:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=gitea
      - POSTGRES_DB=gitea
      - POSTGRES_PASSWORD=[THE PASSWORD YOU CHOSE IN THE PREVIOUS ENVIRONMENT KEY !!!]

```
### Navigate to `http://[GITEA IP/URL]:3000` to complete the setup.


## TRUSTED HTTPS with Cloudflare installation:

### Install Docker using their guide: 

```
https://docs.docker.com/engine/install/ubuntu/

```
add your user to the Docker group:

```
sudo usermod -aG docker ${USER}

```

### Ready the environment

1. Create the Directory: 

```
mkdir ~/gitea
mkdir ~/gitea/cert
cd ~/gitea

```
2. prepare the certificate for HTTPS:

### install OpenSSL and Certbot with its Cloudflare plugin:

```
sudo apt install openssl
sudo apt install certbot python3-certbot-dns-cloudflare

```

### Get your API key from Cloudflare's website:

My Profile > API Tokens > Create Token > Edit zone DNS template > zone resource = your.domain > continue & create

### store the Cloudflare API key on the Gitea server using this command:

```
echo -e "dns_cloudflare_api_token = [PLACEHOLDER FOR YOUR API TOKEN !!!]" | sudo tee -a ~/gitea/cert/cloudflare.ini

```

### Secure the token:

```
sudo chmod -R 600 ~/gitea/cert

```

### Request the certificate:

```
sudo certbot certonly \
 --dns-cloudflare \
 --dns-cloudflare-credentials ~/gitea/cert/cloudflare.ini \
 -d gitea.[YOUR.DOMAIN !!!] \
 --agree-tos \
 -m [YOUR EMAIL ADDRESS !!!]

```

### Create the usable Cert and key pair:

```
sudo openssl ec -in /etc/letsencrypt/live/gitea.[YOUR.DOMAIN !!!]/privkey.pem -out ~/gitea/cert/privkey.pem

sudo openssl x509 -in /etc/letsencrypt/live/gitea.[YOUR.DOMAIN !!!]/fullchain.pem -out ~/gitea/cert/fullchain.pem

```

### Give yourself ownership of the new pair:

```
sudo chown $USER:$USER ~/gitea/cert/*

```

3. Create the compose.yml file:

```
nano ~/gitea/compose.yml

```

#### Paste this into the .yml file and replace the placeholders with your info:

```
version: "3"

networks:
  gitea-net:
    external: false

services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: always
    networks:
      - gitea-net
    volumes:
      - ./gitea:/data
      - ./cert:/cert:ro
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      # Port for Gitea web UI
      - "443:3000"
      # Port for Git via SSH (using 2222 on the host to avoid conflict with system SSH)
      - "2222:22"
    depends_on:
      - db
    environment:
      # IMPORTANT: Use `id -u` and `id -g` to find your correct values (ON A FRESH SYSTEM THESE VALUES WILL WORK)
      - USER_UID=1000
      - USER_GID=1000
      # These variables pre-configure the database connection
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=db:5432
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=gitea
      - GITEA__database__PASSWD=[CHOOSE A PASSWORD !!!]
 #    - GITEA__webhook__ALLOWED_HOST_LIST=[UNCOMMENT AND REPLACE THIS PLACEHOLDER WITH AN IP ADDRESS TO ALLOW WEBHOOKS WITH A SERVICE SUCH AS JENKINS ON A LOCAL IP!!!]
      - GITEA__server__PROTOCOL=https
      - GITEA__server__ROOT_URL=https://gitea.[YOUR.DOMAIN !!!]
      - GITEA__server__CERT_FILE=/cert/fullchain.pem
      - GITEA__server__KEY_FILE=/cert/privkey.pem
  db:
    image: postgres:15
    container_name: gitea-db
    restart: always
    networks:
      - gitea-net
    volumes:
      # Persists database data on the host machine
      - ./postgres:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=gitea
      - POSTGRES_DB=gitea
      - POSTGRES_PASSWORD=[THE PASSWORD YOU CHOSE IN THE PREVIOUS ENVIRONMENT KEY !!!]


```

### Go to the GUI using your domain to complete setup:
```
https://gitea.[YOUR.DOMAIN]

```
