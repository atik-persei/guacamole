# Guacamole ![Endpoint Badge](https://img.shields.io/endpoint?url=https%3A%2F%2Ffirebasestorage.googleapis.com%2Fv0%2Fb%2Fatik-persei.appspot.com%2Fo%2Fguacamole.html%3Falt%3Dmedia%26token%3Dc1d79320-3e13-4f95-b7de-797fa422c229)
The repository provides documentation on how to conveniently configure the Apache Guacamole environment. It uses the default values of ports 80 and 443, automatically redirecting from HTTP to HTTPS. Additionally, it offers an automated certificate issuance container for easy deployment, particularly for production setups.

<br><br>

## 📃 Environment Configuration
### Requirements
The following items are required for environment configuration.
- docker
- docker-compose
<br><br>


### Quick Start
**Step 1 - Clone Repository and Installation**<br>
Clone the repository, configure environment variables, and install Guacamole.
```bash
git clone https://github.com/atik-persei/guacamole
cd guacamole

echo 'PGDATA=/var/lib/postgresql/data/guacamole
GUACD_HOSTNAME=guacamole-daemon
POSTGRES_HOSTNAME=guacamole-db
POSTGRES_DATABASE=guacamole_db
POSTGRES_DB=guacamole_db
POSTGRES_USER=guacamole_user
POSTGRES_PASSWORD=<Your Password>' > .env

docker-compose up -d
```

<br>

**Step 2 - Encryption Configuration**<br>
Set up SSL for encrypted communication.
```bash
./ssl_self.sh
```

<br>

**Step 3 - Using Guacamole**<br>
Now you can start using the Guacamole server.
```
https://server ip or your ip

ID : guacadmin
PW : guacadmin
```

<br>
<br>

## Details
### Environment Variables
A collection of data used for Guacamole environment configuration. Stored in the .env file, it includes configuration for data storage paths, internal domain for communication, database name, and user information.
```
PGDATA=/var/lib/postgresql/data/guacamole
GUACD_HOSTNAME=guacamole-daemon
POSTGRES_HOSTNAME=guacamole-db
POSTGRES_DATABASE=guacamole_db
POSTGRES_DB=guacamole_db
POSTGRES_USER=guacamole_user
POSTGRES_PASSWORD=<Your Password>
```

<br>

### Service<br>
**Init Bot**<br>
This container handles the generation of SQL and SSL certificates necessary for Guacamole environment configuration. It skips the process if the SQL file already exists.
<br>
The issued SSL certificate is not suitable for production deployment and is intended for testing purposes only. Please refer to the [Certificates](#Certificates) section for more information.

```yaml
guacamole-initbot:
    image: guacamole/guacamole
    container_name: guacamole-initbot
    user: root
    command: >
        /bin/bash -c "test -e /templates/database/initdb.sql && echo 'init file already exists' || /opt/guacamole/bin/initdb.sh --postgresql > /templates/database/initdb.sql; openssl req -nodes -newkey rsa:2048 -new -x509 -keyout /templates/ssl/self-ssl.key -out /templates/ssl/self.cert -subj '/C=DE/ST=BY/L=Hintertupfing/O=Dorfwirt/OU=Theke/CN=www.createyourown.domain/emailAddress=docker@createyourown.domain'"
    volumes:
        - ./templates/database:/templates/database:rw
        - ./templates/ssl:/templates/ssl:rw
```

<br>

**Daemon**<br>
This container performs the core functions of Guacamole. It communicates with the Guacamole web service to carry out protocol handling, remote connections, authentication, and more.
```yaml
guacamole-daemon:
    image: guacamole/guacd
    container_name: guacamole-daemon
    volumes:
        - ./Data/Daemon/drive:/drive:rw
        - ./Data/Daemon/record:/record:rw
    networks:
        - guacamole_network
    restart: unless-stopped
```

<br>

**Guacamole**<br>
This container is responsible for Guacamole's web interface. It handles displaying the user interface and operates on the internal port 8080 for configuration.
```yaml
guacamole:
    image: guacamole/guacamole
    container_name: guacamole
    networks:
        - guacamole_network
    ports:
        - 8080/tcp
    depends_on:
        - guacamole-initbot
        - guacamole-daemon
        - guacamole-db
    restart: unless-stopped
    env_file:
        - .env
```

<br>

**Database**<br>
This container handles user authentication and remote session logging for Guacamole usage. Guacamole supports MySQL and PostgreSQL databases, and this project utilizes PostgreSQL version 15.
```yaml
guacamole-db:
    image: postgres:15.2-alpine
    container_name: guacamole-db
    volumes:
        - ./templates/database:/docker-entrypoint-initdb.d:ro
        - ./Data/Database/data:/var/lib/postgresql/data:rw
    env_file:
        - .env
    networks:
        - guacamole_network
    depends_on:
        - guacamole-initbot
    restart: unless-stopped
```

<br>

**Nginx**<br>
The web proxy container utilizes Nginx to perform port mapping from external ports 80 and 443 to internal port 8080, directing traffic to the internal server.
```yaml
guacamole-wps:
    image: nginx:1.21.6
    container_name: guacamole-wps
    volumes:
        - ./templates/nginx:/etc/nginx/templates:ro
        - ./templates/ssl:/etc/nginx/ssl:ro
        - ./Data/SSL/certbot/letsencrypt:/etc/letsencrypt:ro
        - ./Data/WEBROOT:/usr/share/nginx/html:rw
    networks:
        - guacamole_network
    ports:
        - 80:80
        - 443:443
    restart: unless-stopped
```

<br>

**Cert Bot**<br>
This container is used for issuing valid CA certificates for production deployment. The issued certificates are stored in /Data/SSL/certbot/letsencrypt.
```yaml
guacamole-certbot:
    container_name: guacamole-certbot
    image: certbot/certbot:v1.27.0
    volumes:
        - ./Data/SSL/certbot/letsencrypt:/etc/letsencrypt:rw
        - ./Data/SSL/certbot/log:/var/log/letsencrypt:rw
        - ./Data/WEBROOT:/var/www/certbot:rw
    command: certonly --webroot --webroot-path=/var/www/certbot --email <Your Email> --agree-tos --no-eff-email --force-renewal -d <Your Domain>
    restart: 'no'
```

<br><br>


## 📃 Reference
### Certificates
SSL certificates can be configured to use either self-signed certificates or certificates issued by Let's Encrypt.
- Self-signed certificate path : /templates/ssl:/templates/ssl
- CA certificate path : /Data/SSL/certbot/letsencrypt:/etc/letsencrypt

<br>

**Configuration Item**<br>
In the case of using a CA certificate, you need to modify the following items.<br>
Update `<Your Email>` and `<Your Domain>` in the certificate issuance section of the `/docker-compose.yml` file.

```yaml
guacamole-certbot:
    container_name: guacamole-certbot
    image: certbot/certbot:v1.27.0
    volumes:
        - ./Data/SSL/certbot/letsencrypt:/etc/letsencrypt:rw
        - ./Data/SSL/certbot/log:/var/log/letsencrypt:rw
        - ./Data/WEBROOT:/var/www/certbot:rw
    command: certonly --webroot --webroot-path=/var/www/certbot --email <Your Email> --agree-tos --no-eff-email --force-renewal -d <Your Domain>
    restart: 'no'
```

<br>

Modify `<Your Domain>` in the `/templates/nginx/default_ssl_certbot.conf` file.

```conf
server {
    listen 80 ssl;
    listen 443 ssl http2;
    server_name <Your Domain>;

    ssl on;
    ssl_certificate     /etc/letsencrypt/live/<Your Domain>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<Your Domain>/privkey.pem;
```

<br>

CA certificates need to be renewed periodically every 3 months.
<br>
Certificate renewal can be achieved by rebooting the Guacamole-certbot container.

```bash
docker restart guacamole-certbot
```

For Windows, you can use the Task Scheduler, and for Linux, you can use the crontab scheduler to achieve periodic certificate renewal.



<br>

### Relevant Book
- [Apache Guacamole](https://guacamole.apache.org/)
- [Docker](https://www.docker.com/)
- [Docker Compose](https://docs.docker.com/compose/)

<br><br>

## 📃 Multilingual Document
We are translating documents into various languages for users. Currently, we support only two languages.
<p align="center">
    <a href="https://github.com/atik-persei/guacamole">English</a>
    · 
    <a href="/docs/README_kr.md">한국어</a>
</p>
