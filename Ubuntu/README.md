# Initial setting

## Basic libs installation

```bash
sudo apt install build-essential libncurses5 libncurses5-dev bin86 libssl-dev bison flex libelf-dev vim mlocate libpython3-dev python2.7-dev meld autoconf libtool pkgconf net-tools git cutecom linux-source xterm
```

### alias

```bash
alias vi="vim"
```

## SSL for KASI

download ePrism.crt for linux from kms.kasi.re.kr

make extra directory

```bash
sudo mkdir /usr/share/ca-certificates/extra
```

copy crt to the extra directory

```bash
sudo cp ~/Downloads/ePrism.crt /usr/share/ca-certificates/extra/.
```

reconfigure certificates

```bash
sudo dpkg-reconfigure ca-certificates
```

enable ePrism.crt

### for browser

Settings > Privacy & Security > Security > Certificates > View Certificates... > Authorities > Import...

## SSH

install `openssh-server`

```bash
sudo apt install openssh-server
```

set port

```bash
sudo vi /etc/ssh/sshd_config

Port 1234
MaxAuthTries 5
```

restart ssh service

```bash
sudo service ssh restart
```

## Chrome

> [!NOTE]
> https://www.xda-developers.com/how-install-chrome-ubuntu/

download chrome pkg

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
```

install chrome

```bash
sudo dpkg -i google-chrome-stable_current_amd64.deb
```



