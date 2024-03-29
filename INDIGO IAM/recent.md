# Deploy Indigo IAM for AAI proxy

- where: krsrc06

## Installation

> [!NOTE]
> <https://indigo-iam.github.io/v/v1.8.3/docs/getting-started/>

### Install Nginx

```bash
sudo apt update
sudo apt install nginx
```

for https connection, X.509 certificate is required.

#### Configuration NGINX

Edit default file.

```bash
vi /etc/nginx/sites-available/default
```

Contents of default file for redirecting and reverse proxy.

```nginx configuration file
server {
  listen 80;
  listen [::]:80;
  server_name _;
  return 301 https://krsrc.kasi.re.kr;
}

server {
  listen      443 ssl;
  listen      [::]:443 ssl;
  server_name krsrc.kasi.re.kr;
  access_log  /var/log/nginx/iam.access.log  combined;

#  ssl on;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_certificate      /etc/nginx/cert/cert.pem;
  ssl_certificate_key  /etc/nginx/cert/cert.key;

  ssl_session_cache shared:SSL:1m;
  ssl_session_timeout 5m;

  ssl_ciphers HIGH:MEDIUM:!SSLv2:!PSK:!SRP:!ADH:!AECDH;
  ssl_prefer_server_ciphers on;

  location / {
    proxy_pass              http://krsrc.kasi.re.kr:8080/;
    proxy_set_header        X-Real-IP $remote_addr;
    proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header        X-Forwarded-Proto https;
    proxy_set_header        Host $http_host;
  }
}
```

Include `default` to `nginx.conf` by adding the following sentence within the HTTP block in `nginx.conf`.

```nginx configureation file
include /etc/nginx/sites-available/default;
```

Add host information.

```bash
vi /etc/hosts

127.0.0.1 krsrc.kasi.re.kr
```

> [!NOTE]
> The default location of HTML in Nginx is `/usr/share/nginx/html/`

### Data base configuration

MariaDB and Mysql.

#### Install Maria DB

```bash
sudo apt install mariadb-server
```

#### Change root password after installing MariaDB

One imaportant setting from mysql is here:

```bash
sudo mysql_secure_installation

Normally, root should only be allowed to connect from 'localhost'.
This ensures that someone cannot guess at the root password from the network.
Disallow root login remotely? Y
```

#### Run and Check mariaDB

```bash
service mariadb start
service mariadb status
```

> [!NOTE]
> <https://velog.io/@mini_mouse_/%EB%8D%B0%EC%9D%B4%ED%84%B0%EB%B2%A0%EC%9D%B4%EC%8A%A4-mariadb-setting-s9mbiydb>

#### Create a user for Indigo IAM

```bash
mysql -u root -p changeme
```

```sql
CREATE USER iam_test;
```

> [!NOTE]
> <https://wylee-developer.tistory.com/23>

#### Check the created user

```sql
use mysql;
select host, user from user where user='iam_test';
```

#### Create database and give privileges

```sql
CREATE DATABASE iam_test_db CHARACTER SET latin1 COLLATE latin1_swedish_ci;
GRANT ALL PRIVILEGES on iam_test_db.* to 'iam_test'@'%' identified by 'userpassword';
```

#### reload previleges for table

```sql
flush privileges; 
```

#### Check the created database

```sql
show databases like '%iam_test%';
```

#### Quit mariaDB

```sql
exit
```

#### log-in iam_test_db with iam_test user

```bash
mysql -u iam_test -p iam_test_db
```

```sql
show tables;
```

### Json Web Key configuration

#### Clone json-web-key-generator

```bash
git clone https://github.com/mitreid-connect/json-web-key-generator
```

`Maven` is required as a build tool.
`Maven 3.6.x` or greater supporting `JaVA 11` is required.

```bash
sudo apt-get update && sudo apt-get upgrade
apt install openjdk-11-jre-headless
apt install maven
```

#### Build using maven

```bash
mvn -v
cd json-web-key-generator
mvn pakage
mvn install
mvn compiler:compile
mvn org.apache.maven.plugins:maven-compiler-plugin:compile
mvn org.apache.maven.plugins:maven-compiler-plugin:2.0.2:compile
```

> [!NOTE]
> You will meet build error if you don't move to the directory where pom.xml located.  
> pom.xml error-> <https://doosicee.tistory.com/entry/Maven%EC%9D%98-%EC%84%A4%EC%A0%95%ED%8C%8C%EC%9D%BC-Pomxml%EC%9D%84-%EC%95%8C%EC%95%84%EB%B3%B4%EC%9E%90>
> Lifecycle error -> <http://cwiki.apache.org/confluence/display/MAVEN/LifecyclePhaseNotFoundException>

#### Re-run maven package

```bash
mvn package
```

#### Generate a Web-Key using Json-web-key-generator

```bash
java -jar target/json-web-key-generator-0.9-SNAPSHOT-jar-with-dependencies.jar \
  -t RSA -s 1024 -S -i keyid
```

#### Save the output from the generator

Copy and save the contents from the output message from upper command.
(from { after Full key:)

```bash
vi keystore.jks
```

### Preparation for docker installation

#### Register docker GPG key

```bash
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

#### Repository setting

```bash
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

#### Install docker engine

```bash
apt update
apt install docker-ce docker-ce-cli containerd.io
```

Docker version 24.0.7, build afdd53b

#### Deploy Indigo IAM with docker

```bash
docker pull indigoiam/iam-login-service
```

> [!NOTE]
> Hostname and Port for mariaDB have been chaned for instance.
> hostname=IP address of the system where MariaDB has been installed.
> port={db_port}

## Configurations

### Configuration for Indigo IAM without connecting KAFE

#### Make directories for env file and keystore

For keys

```bash
mkdir /var/lib/indigo/iam-login-service
```

For env file

```bash
mkdir /etc/sysconfig/iam-login-service
```

Move `jks` key to the right path

```bash
mv /home/manager/keystore.jks /var/lib/indigo/iam-login-service
```

#### env file for Indigo IAM without connecting KAFE

```plain text
# Java VM arguments
IAM_JAVA_OPTS=-Dspring.profiles.active=prod,registration -Djava.security.egd=file:///dev/./urandom

# Generic options
IAM_HOST=krsrc.kasi.re.kr
IAM_BASE_URL=https://krsrc.kasi.re.kr
IAM_ISSUER=https://krsrc.kasi.re.kr
IAM_USE_FORWARDED_HEADERS=true
IAM_FORWARD_HEADERS_STRATEGY=native
IAM_KEY_STORE_LOCATION=file:///keys/keystore.jks
IAM_JWK_DEFAULT_KEY_ID=rsa1
IAM_JWT_DEFAULT_PROFILE=wlcg
IAM_ORGANISATION_NAME=KRSRC
IAM_TOP_BAR_TITLE="INDIGO IAM for ${IAM_ORGANISATION_NAME}"

# Test Client configuration
IAM_CLIENT_ID=client
IAM_CLIENT_SECRET=secret
IAM_CLIENT_SCOPES=openid profile email
IAM_CLIENT_FORWARD_HEADERS_STRATEGY=native

# Database connection settings
IAM_DB_HOST={iam_db_host}
IAM_DB_PORT={iam_db_port}
IAM_DB_NAME={iam_db_name}
IAM_DB_USERNAME={iam_db_userid}
IAM_DB_PASSWORD={iam_db_passwd}
IAM_DB_VALIDATION_QUERY=SELECT 1
```

#### Create docker network

```bash
docker network create krSRC_iam
```

#### Run docker

```bash
docker run -d \
  --name iam-login-service \
  --net=krSRC_iam -p 8080:8080 \
  --env-file=/etc/sysconfig/iam-login-service \
  -v /var/lib/indigo/iam-login-service/:/keys/:ro \
  --restart unless-stopped \
  indigoiam/iam-login-service:v1.8.3
```

#### List and check log active dockers

```bash
docker ps
docker logs iam-login-service
```

#### Kill the active docker for re-running

Stop and remove docker

```bash
docker stop containerID/Name
docker rm containerID/Name
```

#### Firewall setting for Indigo IAM  

```bash
ufw allow 8080  # for indigo iam proxy
ufw allow 3306  # for MariaDB default 
ufw allow {db_port}  # for iam_test_db
```

`3306` is a default port for MariaDB

#### Edit MariaDB configuration to connect from the outside

```bash
vi /etc/mysql/mariadb.conf.d/50-server.cnf

# bind-address            = 127.0.0.1
bind-address             = 0.0.0.0
```

### Mail configuration

#### Install and configure `postfix`

Install `postfix` on the indigo iam server (`krsrc06`).

```bash
apt update
apt install postfix
```

Configure `postfix`.

```bash
vi /etc/postfix/main.cf

myhostname = krsrc.kasi.re.kr
mydomain = kasi.re.kr
local_transport = error: this is a null client
myorigin = $myhostname
mynetworks = 127.0.0.0/8 172.0.0.0/8 [::1]/128
relayhost = [{smtp_host}]:{smtp_port}
disable_dns_lookups = yes

smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_generic_maps = hash:/etc/postfix/generic
```
<!-- smtp_use_tls = yes
smtp_sasl_auth_enable = yes
smtp_sasl_security_options = noanonymous

smtp_tls_CApath = /etc/pki/tls/certs
smtp_tls_CAfile = /etc/pki/tls/certs/ca-bundle.crt -->

Set the SMTP SASL credentials.

```bash
vi /etc/postfix/sasl_passwd

[{smtp_host}]:{smtp_port}  {approved_mail_address}:[{mail_password}]

chmod 600 /etc/postfix/sasl_passwd
systemctl restart postfix
postmap /etc/postfix/sasl_passwd
```

Modify the sender mail address.

```bash
vi /etc/postfix/generic

{indigo_iam_admin_mail_address} {approved_mail_address}
```

#### Update Indigo IAM env file

```plain text
...
# Notification settings
IAM_NOTIFICATION_DISABLE=false
IAM_NOTIFICATION_FROM=krsrc@kasi.re.kr
IAM_NOTIFICATION_TASK_DELAY=5000
IAM_NOTIFICATION_ADMIN_ADDRESS=krsrc@kasi.re.kr
IAM_MAIL_HOST={host_ip}
IAM_MAIL_PORT={host_smtp_port}
IAM_MAIL_USERNAME={mail_account_id}
IAM_MAIL_PASSWORD={mail_account_pqssword}
```

### SAML configuration for KAFE integration

#### Generate self-signed key to register in SAML Java key store (JKS)

```bash
openssl req -newkey rsa:2048 -nodes -x509 -days 3650 -keyout self-signed.key.pem \
-out self-signed.cert.pem
```

> [!NOTE]
> KR, DAEJEON, KASI, iam, password

```bash
openssl pkcs12 -export -inkey self-signed.key.pem\
-name iam \
-in self-signed.cert.pem \
-out self-signed.p12
```

#### Federation certificate for KAFE must be registered in SAML Java key store (JKS)

```bash
keytool -importkeystore -destkeystore iam.jks \
-srckeystore self-signed.p12 \
-srcstoretype PKCS12
```

### Check the key list

```bash
keytool -list -keystore -v iam.jks
```

#### Federation certificate for KAFE must be registered in SAML JKS

```bash
cd /var/lib/indigo/iam-login-service/
wget https://fedinfo.kreonet.net/cert/{certificate_name}.crt #**.crt must be provided
openssl x509 -in {certificate_name}.crt -out {certificate_name}.der -outform der
keytool -import -alias {certificate_name} -keystore ./iam.jks -file {certificate_name}.der
```

> [!NOTE]
> Trust this certificate? [no]: yes  
> Set the password for keystore; it is important information to set SAML for Indigo IAM

#### env file for Indigo IAM with connecting KAFE

```plain text
# Java VM arguments
IAM_JAVA_OPTS=-Dspring.profiles.active=prod,saml,registration -Djava.security.egd=file:///dev/./urandom

# Generic options
...

# Test Client configuration
...

# Database connection settings
...

## SAML profile settings
IAM_SAML_ENTITY_ID=https://krsrc.kasi.re.kr/sp/indigo
IAM_SAML_LOGIN_BUTTON_TEXT=Sign in with KAFE
IAM_SAML_KEYSTORE=file:///keys/iam.jks
IAM_SAML_KEYSTORE_PASSWORD=userpassword  # that you set for 'iam.jkr'
IAM_SAML_KEY_ID=userpassword               # for self-signed key
IAM_SAML_KEY_PASSWORD=userpassword   # for self-signed key
IAM_SAML_IDP_METADATA=https://mds.kafe.or.kr/metadata/edugain-idp-signed.xml
IAM_SAML_METADATA_REQUIRE_VALID_SIGNATURE=false
IAM_SAML_MAX_ASSERTION_TIME=3000
IAM_SAML_MAX_AUTHENTICATION_AGE=86400
IAM_SAML_METADATA_LOOKUP_SERVICE_REFRESH_PERIOD_SEC=3600
IAM_SAML_ID_RESOLVERS=eduPersonUniqueId,eduPersonTargetedId,eduPersonPrincipalName
```

#### Metadata Update Scheduling

Using `cron`, schedule metadata update with `wget` at everyday 4 am.

```bash
$ vi /var/lib/indigo/iam-login-service/renew_metadata.sh

cd /var/lib/indigo/iam-login-service
wget https://mds.kafe.or.kr/metadata/edugain-idp-signed.xml >> renew_metadata.log
cp edugain-idp-signed.xml keys/. >> renew_metadata.log
```

Modify permission for execution.

```bash
chmod +x /var/lib/indigo/iam-login-service/renew_metadata.sh
```

Add `cron` schedule.

```bash
$ crontab -e

0 4 * * * /var/lib/indigo/iam-login-service/renew_metadata.sh
```
