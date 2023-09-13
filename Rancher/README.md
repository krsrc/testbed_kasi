# Deployment of Rancher and Kubernetes

## Terminology

- **Rancher server**: manages Kubernetes cluster through Rancher's user interface
- **RKE**(Rancher Kubernetes Engine): certified Kubernetes distribution and CLI/library
- **Helm**: Kubernetes package manager 

## Note

- For RKE clusters, **three nodes** are required to achieve a high-availability cluster
  - You should set up a high-availability Kubernetes cluster, then install Rancher on it
 
## Requirements

- Rancher UI works best in Firefox- or Chromium-based browsers
- All supported operating systems are 64-bit x86.
  - Rancher should work with any modern Linux distribution.

## Install

### Setting ssh-key

### Preinstall

```bash
sudo apt update
sudo apt upgrade
sudo apt install vim mlocate curl net-tools
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo apt install ntp
```

### Docker repository registration

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install `containerd`.

> [!NOTE]
> `containerd` is needed, and `docker` is useless.

```bash
sudo apt-get update
sudo apt-get install containerd
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

Check `docker` installation (optional).

```bash
sudo docker version
sudo systemctl status docker
sudo systemctl enable docker
```

### Add bridge for node comm

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```

> [!NOTE]
> How to use `cat <<EOF`: <https://shonm.tistory.com/666>  
> What is `EOF`: <https://ansan-survivor.tistory.com/1301>  
> What is `tee` and how to use it: <https://www.lesstif.com/lpt/linux-tee-89556049.html>

### IP forward

```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

### Swap OFF

> [!NOTE]
> What is 'swap off' and how to do it. <https://seulcode.tistory.com/570>

```bash
sudo swapoff -a
```

Comment out the swap line in `/etc/fstab`

### Install CA package and K8s key

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### Install K8s

```bash
sudo apt list -a kubelet
sudo apt-get install -y kubelet=1.26.7-00 kubeadm=1.26.7-00 kubectl=1.26.7-00
sudo apt-mark hold kubelet kubeadm kubectl
systemctl start kubelet && systemctl enable kubelet
```

> [!NOTE]
> How to prevent package update: <https://ko.linux-console.net/?p=1587#gsc.tab=0>

Should be:

```bash
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

Should be reboot:

```bash
sudo systemctl enable docker 
sudo systemctl daemon-reload 
sudo systemctl restart docker
```

### Run K8s at Master node

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=[ip_master]
```

For exmaple, `ip_master` is `192.168.10.1`

### Start host as regular/root user

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Install add-on

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

> [!NOTE]
> It is possible that the token/join commands are hard to find.
> In that case, use below command:
>
> ```bash
> kubeadm token create --print-join-command
> ```

### Install Helm

