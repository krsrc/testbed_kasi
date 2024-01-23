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
Edit default file 
```
vi /etc/nginx/sites-available/default
```
Contents of default file for redirecting and reverse proxy
```
server {
  listen 80;
  listen [::]:80;
  server_name _;
  return 301 https://krsrc.kasi.re.kr;
}

server {
  listen        443 ssl;
  listen        [::]:443 ssl;
  server_name   krsrc.kasi.re.kr;
  access_log   /var/log/nginx/iam.access.log  combined;

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
Include 'default' to 'nginx.conf' by adding the following sentence within the HTTP block in 'nginx.conf'
```
include /etc/nginx/sites-available/default;

```


> [!NOTE]
> The default location of HTML in Nginx is `/usr/share/nginx/html/`

### Data base configuration
MariaDB and Mysql 

### Install Maria DB
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
GRANT ALL PRIVILEGES on iam_test_db.* to 'iam_test'@'%' identified by 'userpassword';
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
cd json-web-key-generator
mvn pakage
mvn install
mvn compiler:compile
mvn org.apache.maven.plugins:maven-compiler-plugin:compile
mvn org.apache.maven.plugins:maven-compiler-plugin:2.0.2:compile
```

> [!Try and Errors]
> You will meet build error if you don't move to the directory where pom.xml located.
> pom.xml error-> https://doosicee.tistory.com/entry/Maven%EC%9D%98-%EC%84%A4%EC%A0%95%ED%8C%8C%EC%9D%BC-Pomxml%EC%9D%84-%EC%95%8C%EC%95%84%EB%B3%B4%EC%9E%90
> Lifecycle error -> http://cwiki.apache.org/confluence/display/MAVEN/LifecyclePhaseNotFoundException

### Re-run maven package
```
mvn package
```

### Generate a Web-Key using Json-web-key-generator
```
java -jar target/json-web-key-generator-0.9-SNAPSHOT-jar-with-dependencies.jar \
  -t RSA -s 1024 -S -i keyid
```



### Save the output from the generator
Copy and save the contents from the output message from upper command.
(from { after Full key:)
```
vi keystore.jks
```
### Preparation for docker installation

### Register docker GPG key
```
mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

### Repository setting
```
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Install docker engine
```apt-get update
apt-get install docker-ce docker-ce-cli containerd.io
```
Docker version 24.0.7, build afdd53b

### Deploy Indigo IAM with docker
```
docker pull indigoiam/iam-login-service
```

> [!NOTE]
> Hostname and Port for mariaDB have been chaned for instance. 
> hostname=IP address of the system where MariaDB has been installed. 
> port=4567

### Configuration for Indigo IAM without connecting KAFE
### Make directories for env file and keystore

 - for keys
```
mkdir /var/lib/indigo/iam-login-service
```
 - for env file
```
mkdir /etc/sysconfig/iam-login-service
```
 - move jks key to the right path
```
mv /home/manager/keystore.jks /var/lib/indigo/iam-login-service
```

### env file for Indigo IAM without connecting KAFE
```
IAM_JAVA_OPTS=-Dspring.profiles.active=prod,registration -Djava.security.egd=file:///dev/./urandom \
IAM_HOST=krsrc.kasi.re.kr \
IAM_BASE_URL=https://krsrc.kasi.re.kr
IAM_ISSUER=https://krsrc.kasi.re.kr
IAM_USE_FORWARDED_HEADERS=true
IAM_FORWARD_HEADERS_STRATEGY=native
IAM_KEY_STORE_LOCATION=file:///keystore.jks
IAM_JWK_DEFAULT_KEY_ID=keyid
IAM_DB_HOST=IP address of the system where MariaDB has been installed.
IAM_DB_NAME=iam_test_db
IAM_DB_PORT=4567
IAM_DB_USERNAME=iam_test
IAM_DB_PASSWORD=userpassword
IAM_DB_VALIDATION_QUERY=SELECT 1
IAM_ORGANISATION_NAME= KRSRC
IAM_TOP_BAR_TITLE="INDIGO IAM for ${IAM_ORGANISATION_NAME}"
```
### Run docker
```
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
```

### Re-run docker after kill the previous one
- list ative dockers
```
docker ps
```
- stop and remove docker
```
docker stop containerID/Name
docker rm containerID/Name
```
### Firewall setting for Indigo IAM  
```
ufw allow 8080 ; for proxy?
ufw allow 3306 ; for MariaDB 
ufw allow 4567 ; for iam_test_db
```
3306 is a default port for MariaDB

### Edit MariaDB configuration to connect from the outside
```
vi /etc/mysql/mariadb.conf.d/50-server.cnf
```

> #bind-address            = 127.0.0.1
> bind-address             = 0.0.0.0

