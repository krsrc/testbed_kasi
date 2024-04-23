#krsrc06
mkdir GIT
cd GIT
 
# Site of SRC:
https://gitlab.com/ska-telescope/src/group-membership-service

git clone https://gitlab.com/ska-telescope/src/group-membership-service.git
cd group-membership-service
git checkout development
git pull

###########################################
2024.3.20
Edited Dockerfile
pom.xml
###########################################
cd group-membership-service/
#Dockerfile:
add DEBUG option of '-X' at 'RUN mvn package -DskipTests'
FROM openjdk:17-jdk

#pom.xml:
Change lombok version from 1.18.27 to 1.18.30

#build docker
docker build -t gms/gms:latest .

#run with krsrc env path: (/root/iam_gms_start.sh)
   docker run -d \
   --name gms \
   --net=krSRC_iam -p 444:443 \
   -v /root/GIT/group-membership-service/:/keys/:ro \
   --env-file=/root/GIT/group-membership-service/iam-gms.env \
   --restart unless-stopped \
   gms/gms:latest

   docker run -it \
   --name gms \
   --net=krSRC_iam -p 444:443 \
   -v /root/GIT/group-membership-service/:/keys/:ro \
   --env-file=/root/GIT/group-membership-service/iam-gms.env \
   gms/gms:latest \
   /bin/bash

###########################################3
#2024.3.19. 
# krsrc06 has jdk-11, and it has error of:
 error: invalid target release: 17
# therefore, other version of 'openjdk-17-jdk' is installed.
# (when uninstall :  apt remove openjdk-17-jdk

sudo apt install openjdk-17-jdk

java -version

# By using mven (Using JAR)
mvn clean verify -DskipTests

openssl pkcs12 -export -out krsrc-cert.p12 -inkey /etc/nginx/cert/key.pem -in /etc/nginx/cert/cert.pem 
Enter pass phrase for /etc/nginx/cert/key.pem:
Enter Export Password:
Verifying - Enter Export Password:

### From
openssl pkcs12 -export -out iam-pkcs.p12 -inkey edc.key -in edc.crt -name spring -noiter -nomaciter
keytool -importkeystore -destkeystore iam-keystoye.jks -srcstoretype -PKCS12 -srckeystore iam-pkcs.p12

openssl pkcs12 -export -out krsrc-cert.p12 -inkey /etc/nginx/cert/key.pem -in /etc/nginx/cert/cert.pem -name spring -noiter -nomaciter
keytool -importkeystore -destkeystore iam-keystoye.jks -srcstoretype -PKCS12 -srckeystore gms-cert.p12

#password Kasi-ssl
# Modify iam-login-service

export $(xargs < /etc/sysconfig/iam-login-service) 
java -jar target/group-management-service-0.0.1.jar





#2024.4.10.
#Make local GMS docker image : gms-krsrc-2
docker exec -it gms-app /bin/bash
cp keys/prism.crt /usr/local/share/ca-certifictes
update-ca-certificates
./__cacert_entrypoint.sh 
trust extract --overwrite --format=java-cacerts --filter=ca-anchors --purpose=server-auth "/opt/java/openjdk/lib/security/cacerts

exit

#update docker
docker commit gms-app gms-krsrc-2
docker build . -t gms/gms-krsrc-2
#edit gms-start.sh 
GMS_IMAGE_NAME=gms-krsrc-2

#re-run GMS



