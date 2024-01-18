# Deploy Indigo IAM for AAI proxy

- where: krsrc06

## Preparation

> [!IMPORTANT]
> Since the terminal commands in this document require root privileges, it is assumed that `sudo bash -` has been executed.

### Domain mapping

ask to KASI IT team,  
`krsrc.kasi.re.kr` is mapped to `hanul` server, which is a gateway of KRSRC testbed

### Domain certificate preparation

Request a domain certificate for `*.kasi.re.kr` from the KASI IT team.  
The received certificate consists of `cert.pem` and `key.pem`.  
Remove password from `key.pem`.

```bash
openssh rsa -in key.pem -out cert.key
```

When configure nginx,

```bash
ssl_certificate     cert.pem;
ssl_certificate_key cert.key;
```

### Configure ufw of gateway

> [!NOTE]
> <https://www.cyberciti.biz/faq/how-to-configure-ufw-to-forward-port-80443-to-internal-server-hosted-on-lan/>

add rules of ufw.

```bash
vi /etc/ufw/before.rules

# NAT table rules
*nat
:PREROUTING ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-F
-A PREROUTING -i em1 -d ..27.27 -p tcp --dport 80 -j DNAT --to-destination ..0.206:80
-A PREROUTING -i em1 -d ..27.27 -p tcp --dport 443 -j DNAT --to-destination ..0.206:443
-A POSTROUTING -o em1 -j MASQUERADE
COMMIT
```

enable port forwading.

```bash
vi /etc/sysctl.conf

net.ipv4.ip_forward=1

sysctl -p
```

restart the firewall.

```bash
systemctl restart ufw

ufw allow proto tcp from any to ..27.27 port 80
ufw allow proto tcp from any to ..27.27 port 443
```

verify new setting.

```bash
ufw status
iptables -t nat -L -n -v
```

## Installation

> [!NOTE]
> https://indigo-iam.github.io/v/v1.8.3/docs/getting-started/

### Install Nginx

```bash
sudo apt update
sudo apt install nginx
```

for https connection, X.509 certificate is required.

### Configuration NGINX

```
server {
  listen 80;
  listen [::]:80;
  server_name _;
  return 301 https://$host$request_uri;
}

server {
  listen        443 ssl;
  server_name   YOUR_HOSTNAME_HERE;
  access_log   /var/log/nginx/iam.access.log  combined;

  ssl on;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_certificate      /path/to/your/ssl/cert.pem;
  ssl_certificate_key  /path/to/your/ssl/key.pem;

  location / {
    proxy_pass              http://THE_IAM_APP_HOSTNAME_HERE:8080;
    proxy_set_header        X-Real-IP $remote_addr;
    proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header        X-Forwarded-Proto https;
    proxy_set_header        Host $http_host;
  }
}
```




> [!NOTE]
> The default location of HTML in Nginx is `/usr/share/nginx/html/`

### Data base configuration
MariaDB and Mysql 

### install Maria DB
```
sudo apt install mariadb-server
```

### Change root password after installing MariaDB
```
sudo mysql_secure_installation
```
 One imaportant setting from mysql is here:
```
Normally, root should only be allowed to connect from 'localhost'.
This ensures that someone cannot guess at the root password from the network.
Disallow root login remotely? Y
```

### Run and Check mariaDB
```
service mariadb start
service mariadb status
```

> [!NOTE]
> https://velog.io/@mini_mouse_/%EB%8D%B0%EC%9D%B4%ED%84%B0%EB%B2%A0%EC%9D%B4%EC%8A%A4-mariadb-setting-s9mbiydb

### Create a user for Indigo IAM 
```
mysql -u root -p changeme
CREATE USER iam_test;
```

> [!NOTE]
> https://wylee-developer.tistory.com/23

### Check the created user
```
use mysql;
select host, user from user where user='iam_test';
```

### Create database and give privileges

```
CREATE DATABASE iam_test_db CHARACTER SET latin1 COLLATE latin1_swedish_ci;
GRANT ALL PRIVILEGES on iam_test_db.* to 'iam_test'@'%' identified by 'aaitest#AAI';
```
### reload previleges for table
```
flush privileges; 
```

### Check the created database
```
show databases like '%iam_test%';
```

### Quit mariaDB
```
exit
```

### log-in iam_test_db with iam_test user
```
mysql -u iam_test -p iam_test_db
show tables;
```

### Json Web Key configuration
### Clone json-web-key-generator
```
git clone https://github.com/mitreid-connect/json-web-key-generator
```

Maven is required as a build tool.

Maven 3.6.x or greater supporting JaVA 11 is required.
```
sudo apt-get update && sudo apt-get upgrade
apt install openjdk-11-jre-headless
apt install maven
```

### Build using maven
```
mvn -v
```

#move to directory json-web-key-generator
#You will meet build error if you don't move to the directory where pom.xml located. 
#pom.xml error-> https://doosicee.tistory.com/entry/Maven%EC%9D%98-%EC%84%A4%EC%A0%95%ED%8C%8C%EC%9D%BC-Pomxml%EC%9D%84-%EC%95%8C%EC%95%84%EB%B3%B4%EC%9E%90
cd json-web-key-generator
mvn pakage

#Lifecycle error -> http://cwiki.apache.org/confluence/display/MAVEN/LifecyclePhaseNotFoundException
mvn install
mvn compiler:compile
mvn org.apache.maven.plugins:maven-compiler-plugin:compile
mvn org.apache.maven.plugins:maven-compiler-plugin:2.0.2:compile

#re-run mvn package
mvn package

#generate key using JWK
java -jar target/json-web-key-generator-0.9-SNAPSHOT-jar-with-dependencies.jar \
  -t RSA -s 1024 -S -i rsa1

#save the output from the generator; full key (from { after Full key:)
vi keystore.jks

#preparation for docker installation
apt update

#register docker GPG key
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

#repository setting
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

#install docker engine
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io
#Docker version 24.0.7, build afdd53b

#deployment with docker
docker pull indigoiam/iam-login-service

#hostname and port has been changed for mariaDB 
hostname=IP address of the system where MariaDB has been installed. port=4567

#configuration for Indigo IAM without connecting KAFE
#make directories for env. file and keys
 - for keys
mkdir /var/lib/indigo/iam-login-service

 - for env file
mkdir /etc/sysconfig/iam-login-service

 - move jks key to upper location
mv /home/manager/keystore.jks /var/lib/indigo/iam-login-service

#env file for Indigo IAM without connecting KAFE
IAM_JAVA_OPTS=-Dspring.profiles.active=prod,registration -Djava.security.egd=file:///dev/./urandom \
IAM_HOST=krsrc.kasi.re.kr \
IAM_BASE_URL=https://krsrc.kasi.re.kr
IAM_ISSUER=https://krsrc.kasi.re.kr
IAM_USE_FORWARDED_HEADERS=true
IAM_FORWARD_HEADERS_STRATEGY=native
IAM_KEY_STORE_LOCATION=file:///keystore.jks
IAM_JWK_DEFAULT_KEY_ID=rsa1
IAM_DB_HOST=IP address of the system where MariaDB has been installed.
IAM_DB_NAME=iam_test_db
IAM_DB_PORT=4567
IAM_DB_USERNAME=iam_test
IAM_DB_PASSWORD=aaitest#AAI
IAM_DB_VALIDATION_QUERY=SELECT 1
IAM_ORGANISATION_NAME= krSRC
IAM_TOP_BAR_TITLE="INDIGO IAM for ${IAM_ORGANISATION_NAME}"

#run docker
docker network create krSRC_iam

docker stop iam-login-service
docker rm iam-login-service
docker run -d \
  --name iam-login-service \
  --net=krSRC_iam -p 8080:8080 \
  --env-file=/etc/sysconfig/iam-login-service \
  -v /var/lib/indigo/iam-login-service/keystore.jks:/keystore.jks:ro \
  --restart unless-stopped \
  indigoiam/iam-login-service:v1.8.3
docker ps
docker logs iam-login-service

#Federation certificate for KAFE (kafe-fed.crt) must be registered in SAML Java key store(JKS).
cd /var/lib/indigo/iam-login-service/
wget https://fedinfo.kreonet.net/cert/kafe-fed.crt
openssl x509 -in kafe-fed.crt -out kafe-fed.der -outform der
keytool -import -alias kafe-fed -keystore ./iam.jks -file kafe-fed.der
# Trust this certificate? [no]: 에서 yes
keytool -list -keystore ./iam.jks

#re-run docker after kill the previous one
- list ative dockers
docker ps
- stop and remove docker
docker stop ****
docker rm ****

#ufw port 
ufw allow 8080 ; for proxy?
ufw allow 3306 ; for MariaDB

#edit configuration for mariaDB to connect from 0.0.0.0
/etc/mysql/mariadb.conf.d/50-server.cnf

# Instead of skip-networking the default is now to listen only on
# localhost which is more compatible and is not less secure.
#bind-address            = 127.0.0.1
bind-address             = 0.0.0.0


