### Change Hostname

```bash
hostname {new_hostname}

vi /etc/hostname
{new_hostname}

vi /etc/hosts
127.0.1.1  {new_hostname}
```

### Install Library

```bash
apt update
apt install build-essential libncurses5 libncurses5-dev \
bin86 libssl-dev bison flex libelf-dev vim mlocate \
libpython3-dev python2.7-dev meld autoconf libtool \
pkgconf net-tools git cutecom linux-source xterm \
openssh-server curl
```

### Configure UFW

```bash
ufw enable
ufw allow 443
ufw allow {ssh_port}
ufw allow 6443
ufw allow 10250
```

### Configure SSH

```bash
vi /etc/ssh/sshd_config
port {ssh_port}
MaxAuthTries 5
```

at krsrc-main

```bash
ssh-copy-id username@ip
```

### Install K3s Worker Node

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://{master_ip}:6443 K3S_TOKEN={master_token} sh -
```

at krsrc-main

```bash
vi /root/k3s_worker.txt
```

