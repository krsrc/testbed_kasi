# Preparation of INDIGO IAM deployment

> [!IMPORTANT]
> Majority of shell commands in this document require root privileges.
> Before start, `sudo bash -` should be executed.

## Domain mapping

ask to KASI IT team,  
`krsrc.kasi.re.kr` is mapped to `hanul` server, which is a gateway of KRSRC testbed

## Domain certificate preparation

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

## Configure ufw of gateway

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
