# Deployment of Rancher and Kubernetes

## Terminology

- **Rancher server**: manages Kubernetes cluster through Rancher's user interface
- **RKE**(Rancher Kubernetes Engine): certified Kubernetes distribution and CLI/library
- **Helm**: Kubernetes package manager 
- **kubeadm**: command tool to deploy Kubernetes cluster
- **kubelet**: daemon process, watching creation/deletion/status of containers and pods
- **kubectl**: command tool for users to operate the Kubernetes cluster

## Note

- For RKE clusters, **three nodes** are required to achieve a high-availability cluster
  - You should set up a high-availability Kubernetes cluster, then install Rancher on it
 
## Requirements

- Rancher UI works best in Firefox- or Chromium-based browsers
- All supported operating systems are 64-bit x86.
  - Rancher should work with any modern Linux distribution.

## Install

### Setting ssh-key

> [!NOTE]
> SSH connection with ssh-key: <https://psychoria.tistory.com/749>

Make SSH key:

```bash
cd ~/.ssh
ssh-keygen -t rsa -b 4096
```

Check SSH key pair:

```bash
ls ~/.ssh/id_rsa*
```

Send SSH key to server:

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub userid@severip
```

Confirm SSH connection without input password:

```bash
ssh -p port userid@serverip
```

### Preinstall

```bash
sudo apt update
sudo apt upgrade
sudo apt install vim mlocate net-tools
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
<!-- master and slave -->

`Kubernetes` uses `iptables` to enable communication between pods.  
To ensure that `iptables` operates properly, configure as:

> [!NOTE]
> Master and Woker nodes preparation: <https://andrewpage.tistory.com/234>

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

Comment out the filesystem item related to swap in `/etc/fstab`, or:

```bash
sudo sed -i '/swqp/d' /etc/fstab
```

Check swap off:

```bash
free -h
```

### Install K8s CA key

> [!NOTE]
> Before Ubuntu 22.04, `/etc/apt/keyrings` does not exist in default.
> If necessary, make it in advance with `755` permission.

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

For exmaple, `ip_master` is `192.168.x.x`

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

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | \
  sudo tee /usr/share/keyrings/helm.gpg > /dev/null

sudo apt-get install apt-transport-https --yes

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] \
  https://baltocdn.com/helm/stable/debian/ all main" | \
  sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

sudo apt-get update
sudo apt-get install helm

helm repo add jetstack https://charts.jetstack.io
```

Two options: `stable` or `latest`

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
```

```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
```

```bash
helm repo update
```

### Install Rancher

```bash
kubectl create namespace cattle-system
kubectl create namespace cert-manager

kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.4/cert-manager.crds.yaml

helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.0.4
```

> [!WARNING]
> At Ubuntu 22.04, same error happens that time-out. 
> In this case, it is needed to upgrade to continue installation.
>
> ```bash
> helm upgrade -i cert-manager jetstack/cert-manager --namespace cert-manager --version v1.0.4
> ```

Validate installation of `cert-manager`.

```bash
kubectl get pods --namespace cert-manager
kubectl describe pods -n cert-manager
```

Install `Rancher`.

```bash
helm install rancher rancher-latest/rancher --namespace cattle-system --set hostname=rancher.my.org
```

> [!WARNING]
> At `Ubuntu 22.04` with latest versions, `rancher` argues downgrade. 
> It is recommended that `rancher` needs to be downloaded and `Chart.yaml` is edited.
> 
> ```bash
> helm pull rancher-latest/rancher
>
> tar zxvf rancher-x.x.x.tgz
> vi rancher/Chart.yaml
>
> kubeVersion: < 1.xx.0-0
> ```

```bash
helm install rancher ~/rancher --namespace cattle-system --set hostname=rancher.my.org
```

> [!NOTE]
> It is not sure but after installing K8s dashboard, Rancher dashboard also installed.

```bash
kubectl create deployment nginx-deployment --image nginx --replicas 2
kubectl get deployment nginx-deployment
kubectl get pods
kubectl expose deployment nginx-deployment --type NodePort --port 80
kubectl get svc nginx-deployment
```

### When restart only

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.10.1

kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

kubectl reate namespace cattle-system
```

For CAs only

```bash
kubectl create namespace cert-manager
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.12.0/cert-manager.crds.yaml
helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.12.0
kubectl get pods --namespace cert-manager
kubectl describe pods -n cert-manager
```

For Rancher-CA

```bash
helm install rancher rancher-stable/rancher --namespace cattle-system \
  --set hostname=192.168.10.1.nip.io --set bootstrapPassword=password
```

For self-CA

```bash
helm install rancher rancher-stable/rancher --namespace cattle-system \
  --set hostname=192.168.10.1.nip.io --set bootstrapPassword=password --set ingress.tls.source=secret
```

After CA install

```bash
kubectl get pods -n cattle-system
kubectl describe pods -n cattle-system
```

Nginx controller
```bash
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.0/deploy/static/provider/cloud/deploy.yaml
kubectl create namespace ingress-nginx
cd ./ingress-nginx/deploy/static/provider/baremetal
kubectl apply -f deploy.yaml
kubectl get deploy -n ingress-nginx
kubectl get svc -n ingress-nginx
```

> [!NOTE]
> Useful commands:
>
> ```bash
> kubectl apply -f (name).yaml
> kubectl get (pods/svc/ingress) (its-name) -o yaml > (wanted-name).yaml
> ```

### Setup ingress

By default, K8s doesn't make nginx enabled
delete current LoadBalancer first

```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx -o yaml > ingress-nginx-controller.yaml
kubectl delete svc ingress-nginx-controller -n ingress-nginx
```

Edit its yaml (adding externalIP), and run it

```bash
kubectl apply -f ingress-nginx-controller.yaml
```

Also, edit rancher ingress at cattle-system

```bash
kubectl get ingress rancher -n cattle-system -o yaml > ingress-rancher-org.yaml
kubectl delete ingress rancher -n cattle-system
```

Edit ingress-rancher-org.yaml (adding ClassName nginx), and run it

```bash
kubectl apply -f ingress-rancher-org.yaml
kubectl get ingress -A
```

Wait until it has ADDRESS, and open browser.

`https://192.168.10.1.nip.io`

### K3s

Master node.

```bash
curl -sfL https://get.k3s.io | sh -
mkdir ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config && sudo chown $USER ~/.kube/config
sudo chmod 600 ~/.kube/config && export KUBECONFIG=~/.kube/config
sudo vi /etc/systemd/system/k3s.service

ExecStart=/usr/local/bin/k3s \
    server --node-external-ip 192.168.10.1 \
```

Worker node.

To get a TOKEN.

```bash
cat /var/lib/rancher/k3s/server/node-token
```

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.10.1:6443 K3S_TOKEN="<TOKEN>" sh -
```

Nginx controller.

```bash
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.0/deploy/static/provider/cloud/deploy.yaml
kubectl create namespace ingress-nginx
cd ./ingress-nginx/deploy/static/provider/baremetal
kubectl apply -f deploy.yaml
kubectl get deploy -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### When reboot processes needed

For K3s.

```bash
/usr/local/bin/k3s-killall.sh
/usr/local/bin/k3s-uninstall.sh
/usr/local/bin/k3s-agent-uninstall.sh
```

For K8s.

```bash
kubectl delete -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
kubectl delete -f https://github.com/jetstack/cert-manager/releases/download/v1.12.0/cert-manager.crds.yaml --force
kubeadm reset
systemctl stop containerd
systemctl stop kubelet
systemctl daemon-reload
rm -rf /var/lib/cni/
rm -rf /var/lib/kubelet/*
rm -rf /etc/cni/
rm -rf /run/flannel
rm -rf ~/.kube/cache
ifconfig cni0 down
brctl delbr cni0
ifconfig flannel.1 down
ifconfig docker0 down
ip link delete cni0 type bridge
ip link delete cni0
ip link delete flannel.1
systemctl start kubelet
systemctl start containerd
```

UFW in Master.

```bash
$ sudo ufw enable
$ sudo ufw allow 6443/tcp
$ sudo ufw allow 2379:2380/tcp
$ sudo ufw allow 10250/tcp
$ sudo ufw allow 10251/tcp
$ sudo ufw allow 10252/tcp
$ sudo ufw status
```

How to reset Rancher password.

```bash
kubectl -n cattle-system exec $(kubectl --kubeconfig $KUBECONFIG -n cattle-system get pods \
  -l app=rancher --no-headers | head -1 | awk '{ print $1 }') -c rancher --reset-password
```
