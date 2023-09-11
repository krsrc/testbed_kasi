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

> [!NOTE]
> [Setting up a High-availability RKE Kubernetes Cluster](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-cluster-setup/rke1-for-rancher)

### Installing Kubernetes

Install `kubectl`, a Kubernetes command-line tool.

> [!NOTE]
> [Install and Set Up kubectl on Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

**Install `kubectl` on Linux**

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

> [!NOTE]
> To download a specific version
> ```
> curl -LO "https://dl.k8s.io/release/v1.28.1/bin/linux/amd64/kubectl
> ```

Validate the binary

```
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
```

Install kubectl
```
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Test to ensure the version you installed is up-to-date
```
kubectl version --client
```

(optional) for detailed view of version
```
kubectl version --client --output=yaml
```

Install `RKE`, the Rancher Kubernetes Engine, a Kubernetes distribution and command-line tool.

**RKE Kubernetes Installation**

Download the RKE binary

[Latest available RKE release](https://github.com/rancher/rke/#latest-release)

[Release v1.4.8](https://github.com/rancher/rke/releases/tag/v1.4.8)

Rename it `rke` and make executable

```
mv rke_linux-amd64 rke
chmod +x rke
```

Confirm that RKE is now executable

```
rke --version
```

Creating the cluster configuratio file

```
rke config --name cluster.yml
```

> [!IMPORTANT]
> **High Availability**
> RKE is HA ready, you can specify more than one controlplane node in the cluster.yml file. RKE will deploy master components on all of these nodes and the kubelets are configured to connect to 127.0.0.1:6443 by default which is the address of nginx-proxy service that proxy requests to all master nodes.
> To create an HA cluster, specify more than one host with role controlplane.







<!-- 
This document includes items on UNIST 
# 2023.8.31
-->

## How to install Rancher 

### Preinstall
```
sudo apt update
sudo apt upgrade
sudo apt install vim mlocate curl net-tools
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo apt install ntp
```

### Install `Kubernetes` latest version
> [!Note]
> There is Korean version of [k8s installation document](https://kubernetes.io/ko/docs/tasks/tools/install-kubectl-linux/)
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

verify it
```
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
```

install k8s
```
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

check k8s
```
kubectl version --client
```

### Host node
Swap off
```
sudo swapoff -a
#at /etc/fstab : comment /swapfile
#/swapfile                                 none            swap    sw              0       0
```

### Install Docker

패키지 관리 도구 업데이트
```
sudo apt update
sudo apt-get update
```

기존 docker 설치된 리소스 확인 후 발견되면 삭제
```
sudo apt-get remove docker docker-engine docker.io
```

docker를 설치하기 위한 각종 라이브러리 설치
```
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common -y
```

curl 명령어를 통해 gpg key 내려받기
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

키를 잘 내려받았는지 확인
```
sudo apt-key fingerprint 0EBFCD88
```

패키지 관리 도구에 도커 다운로드 링크 추가
```
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

패키지 관리 도구 업데이트
```
sudo apt-get update
```

docker-ce의 버젼을 최신으로 사용하는게 아니라 18.06.2~3의 버젼을 사용하는 이유는 
kubernetes에서 권장하는 버젼의 범위가 최대 v18.09 이기 때문이다.
```
sudo apt-get install docker-ce=18.06.2~ce~3-0~ubuntu -y
```

Firstly, latest version is installed (31.Aug.23)
```
sudo apt-get install docker-ce -y
```

Docker 설치 완료 후 테스트로 hello-world 컨테이너 구동
```
sudo docker run hello-world
```

Change systemd for Docker (https://docs.docker.com/config/daemon/systemd/)
created and edited manually (not EOF)
```
sudo cat > /etc/docker/daemon.json << EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
sudo mkdir -p /etc/systemd/system/docker.service.d
```

Create a http-proxy.conf file:
```
/etc/systemd/system/docker.service.d/http-proxy.conf
--- includes ---
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:3128"
Environment="HTTPS_PROXY=https://proxy.example.com:3129"
----------------
```

mark CRI at /etc/containerd/config.toml
```
sudo vi /etc/containerd/config.toml 

sudo systemctl restart containerd
sudo systemctl daemon-reload
sudo systemctl restart docker
```

verification:
```
sudo systemctl show --property=Environment docker
```

install important file of kubeadm & kubelet (kubectl is already installed)
```
##curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
##sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
##echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys <PUBKEY>
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```

패키지가 자동으로 설치, 업그레이드, 제거되지 않도록 hold함.
```
sudo apt-mark hold kubelet kubeadm kubectl
```

설치 완료 확인
```
kubeadm version
kubelet --version
kubectl version
```

CRI installation (remove container??) - https://yooloo.tistory.com/229
```
sudo apt-get install containerd
lsmod | grep br_netfilter 
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

ip forward
```
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

install Add-on
```
wget https://raw.githubusercontent.com/flannel-io/flannel/v0.20.2/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml

export KUBECONFIG=/etc/kubernetes/admin.conf
```

set up host node
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=10.70.16.113 
```

Start host as regular user:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Useful links
```
#when apt-key tangled:
https://askubuntu.com/questions/1459005/cant-add-a-public-key-to-ubuntu-22-04

gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 871920D1991BC93C
gpg --export 871920D1991BC93C | sudo tee /etc/apt/trusted.gpg.d/ubuntu.lafibre.info.gpg
```


### Q&A

Error:
```
couldn't get current server API group list  : admin.conf is not loaded well.
```

When rebooted, kubeadm init error with ip_forward
```
edit /etc/sysctl.conf
activate net.ipv4.ip_forward = 1

sudo sysctl -p
```

additional work for containerd with k8s
```
# add below in /etc/containerd/config.toml
version = 2
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

kubectl get nodes (by normal user)
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl edit cm coredns -n kube-system (loop 24 lines)
```

by following this, CLI seems to be solved.
https://ko.linux-console.net/?p=3490#gsc.tab=0

### HELM!! Helm
```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

Rancher from Stable site:
```
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
```

create a namespace for rancher
```
kubectl create namespace cattle-system
```

install SSL configuration (at this time, cert-manager used)
If you have installed the CRDs manually instead of with the `--set installCRDs=true` option added to your Helm install command, you should upgrade your CRD resources before upgrading the Helm chart:
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.3/cert-manager.crds.yaml
```

Add the Jetstack Helm repository
```
helm repo add jetstack https://charts.jetstack.io
```

Update your local Helm chart repository cache
```
helm repo update
```

Install the cert-manager Helm chart
```
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.11.0

helm install rancher rancher-latest/rancher   --namespace cattle-system   --set hostname=10.70.16.113.sslip.io
```

[useful commands](https://platform9.com/docs/PEC/troubleshooting--useful-kubernetes-commands)

<!--
### CA
https://www.kubeworks.net/post/cert-manager%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%A7%80-%EC%95%8A%EA%B3%A0-open-ssl%EC%9D%84-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-%EC%83%9D%EC%84%B1%ED%95%9C-ca-%ED%82%A4-%EC%9D%B8%EC%A6%9D%EC%84%9C-%EC%84%9C%EB%B2%84-%ED%82%A4-%EC%9D%B8%EC%A6%9D%EC%84%9C%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%98%EC%97%AC-rancher%EB%A5%BC-%EC%84%A4%EC%B9%98-%EA%B5%AC%EB%8F%99%ED%95%9C%EB%8B%A4-%EA%B8%B0%EB%B3%B8%EA%B0%92%EC%9D%80-cert
-->

