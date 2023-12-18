# Outline

- where: krsrc06

# Installation

> [!NOTE]
> https://indigo-iam.github.io/v/v1.8.3/docs/getting-started/

```bash
sudo apt update
sudo apt install nginx
```

## X.509 certificate

> [!NOTE]
> https://letsencrypt.org/ko/getting-started/
> https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal

### Install snap

> [!NOTE]
> https://snapcraft.io/docs/installing-snap-on-ubuntu

```bash
sudo apt install snapd
```
<!-- sudo apt install snapd
Reading package lists... Done
Building dependency tree
Reading state information... Done
snapd is already the newest version (2.58+18.04.1).
snapd set to manually installed.
The following packages were automatically installed and are no longer required:
  aufs-tools bridge-utils cgroupfs-mount docker-buildx-plugin docker-ce-rootless-extras docker-compose-plugin pigz ubuntu-fan
Use 'sudo apt autoremove' to remove them.
0 upgraded, 0 newly installed, 0 to remove and 4 not upgraded. 
-->

> [!IMPORTANT]
> in Ubuntu 22.04, snapd is already installed.

### Remove Certbot-auto, Install Certbot

```bash
sudo apt-get remove certbot
sudo snap install --classic certbot
```

<!-- sudo apt-get remove certbot 
Reading package lists... Done
Building dependency tree
Reading state information... Done
Package 'certbot' is not installed, so not removed
The following packages were automatically installed and are no longer required:
  aufs-tools bridge-utils cgroupfs-mount docker-buildx-plugin docker-ce-rootless-extras docker-compose-plugin pigz ubuntu-fan
Use 'sudo apt autoremove' to remove them.
0 upgraded, 0 newly installed, 0 to remove and 4 not upgraded.
-->

<!-- sudo snap install --classic certbot 
certbot 2.8.0 from Certbot Project (certbot-eff✓) installed
-->

### Prepare the Certbot command

```bash
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

### Choose how you'd like to run Certbot

Either get and install your certificates...

```bash
sudo certbot --nginx
```

> [!WARNING]
> unexpected error occured.

## Configuration NGINX

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