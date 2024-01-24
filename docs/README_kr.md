# Guacamole ![Endpoint Badge](https://img.shields.io/endpoint?url=https%3A%2F%2Ffirebasestorage.googleapis.com%2Fv0%2Fb%2Fatik-persei.appspot.com%2Fo%2Fguacamole.html%3Falt%3Dmedia%26token%3Dc1d79320-3e13-4f95-b7de-797fa422c229)
해당 레포지토리는 Apache Guacamole 환경을 편리하게 구성하는 방법에 대해 제공합니다. 80, 443 포트를 기본 값으로 사용하며, HTTP에서 HTTPS로의 자동 리다이렉션을 제공합니다. 더불어 프로덕션 설정에 대한 쉬운 배포를 위해 자동 인증서 발급 컨테이너를 제공합니다. 

<br><br>

## 📃 Environment Configuration
### Requirements
환경 구성에는 다음과 같은 패키지가 필요합니다.
- docker
- docker-compose
<br><br>


### 빠른 시작
**Step 1 - 저장소 복제 및 설치**<br>
저장소를 복제하고, 환경변수를 구성한 후 Guacamole를 설치하세요.
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

**Step 2 - 암호화 구성**<br>
암호화 통신을 위해 SSL을 설정하세요.
```bash
./ssl_self.sh
```

<br>

**Step 3 - Guacamole 사용**<br>
이제 Guacamole를 다음의 계정으로 사용하시면 됩니다.
```
https://server ip or your ip

ID : guacadmin
PW : guacadmin
```

<br>
<br>

## Details
### Environment Variables
다음은 Guacamole 환경 구성에 사용되는 환경변수입니다. .env 파일에 저장되며, 데이터 저장 경로, 통신을 위한 내부 도메인, 데이터베이스 이름 및 사용자 정보를 구성하는데 사용됩니다.
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
Guacamole 환경 구성에 필요한 SQL 및 SSL 인증서를 생성합니다. 이미 SQL 파일이 존재하는 경우 해당 프로세스를 건너뜁니다.
<br>
발급된 SSL 인증서는 프로덕션 배포에 적합하지 않으며, 테스트 목적으로 사용되는 인증서입니다. 자세한 내용은 [Certificates](#Certificates) 섹션을 참고하세요.

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
Guacamole 핵심 기능을 수행하는 컨테이너입니다. 프로토컬 처리, 원격 연결, 인증 등을 수행하기 위해 Guacamole 웹 서비스와 통신합니다.
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
Guacamole의 웹 인터페이스를 담당합니다. 사용자 인터페이스를 표시하고, 8080 내부 포트를 통해 동작합니다.
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
Guacamole 사용을 위한 사용자 인증, 원격 세션 로깅을 처리합니다. Guacamole가 지원하는 데이터베이스는 MySQL과 PostgresSQL입니다. 해당 프로젝트에서는 PostrgresSQL 15 버전을 사용합니다.
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
웹 프록시 역할을 수행하며, Nginx를 활용해 80 및 443 외부 포트와 8080 내부 포트의 포트 매핑을 수행하여 트래픽을 내부 서버로 전달합니다.
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
프로덕션 배포를 위해 유효한 CA 인증서를 발급하는데 사용됩니다. 발급된 인증서는 /Data/SSL/certbot/letsencrypt에 저장됩니다.
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
SSL 인증서는 자체 서명된 인증서 또는 Let's Encrypt에서 발급한 인증서 중 하나를 선택하여 구성할 수 있습니다.
- Self-signed 인증서 경로 : /templates/ssl:/templates/ssl
- CA 인증서 경로 : /Data/SSL/certbot/letsencrypt:/etc/letsencrypt

<br>

**Configuration Item**<br>
CA 인증서를 사용하는 경우 다음의 항목을 수정해야 합니다.<br>
`/docker-compose.yml` 파일의 인증서 발급 부분에서 `<Your Email>` 및 `<Your Domain>` 부분을 업데이트하세요.

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

그 다음 `/templates/nginx/default_ssl_certbot.conf` 파일에서  `<Your Domain>`을 수정하세요.

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

CA 인증서는 3개월마다 주기적으로 갱신해야합니다.
<br>
Guacamole-certbot 컨테이너를 재시작함으로써 인증서 갱신이 가능합니다.

```bash
docker restart guacamole-certbot
```

Windows에서는 작업 스케줄러를, Linux에서는 crontab 스케줄러를 사용해 주기적으로 인증서 갱신이 가능합니다.



<br>

### Relevant Book
- [Apache Guacamole](https://guacamole.apache.org/)
- [Docker](https://www.docker.com/)
- [Docker Compose](https://docs.docker.com/compose/)

<br><br>

## 📃 Multilingual Document
현재 사용자를 위해 문서를 다양한 언어로 번역하고 있습니다. 현재로써는 두 가지 언어만 지원하고 있습니다.
<p align="center">
    <a href="https://github.com/atik-persei/guacamole">English</a>
    · 
    <a href="/docs/README_kr.md">한국어</a>
</p>
