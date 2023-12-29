# Deploy Indigo IAM for AAI proxy

- where: krsrc06

## Preparation

> [!IMPORTANT]
> Since the terminal commands in this document require root privileges, it is assumed that `sudo bash -` has been executed.

### Domain mapping

ask to KASI IT team,  
`krsrc.kasi.re.kr` is mapped to `hanul` server, which is a gateway of krsrs testbed

### Domain certificate preparation

Request a domain certificate for `kasi.re.kr` from the KASI IT team.
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




> [!NOTE]
> The default location of HTML in Nginx is `/usr/share/nginx/html/`
