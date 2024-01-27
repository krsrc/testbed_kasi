# K3s installation

- Using 5 physical server for test
  - `krsrc01` :  control-plane
  - `krsrc02:05` :  worker

## K3s installation using script for master

> [!CAUTION]
> `sudo` has been omitted from all scripts. Run `sudo bash -` and proceed from the root account.

```bash
curl -sfL https://get.k3s.io | sh -
```

Make `kubectl` able to control installed `k3s`.

```bash
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
chmod 600 ~/.kube/config
export KUBECONFIG=~/.kube/config
```

Set external IP for the control-plane.

```bash
vi /etc/systemd/system/k3s.service

ExecStart=/usr/local/bin/k3s \
    server --node-external-ip {master_ip} \

systemctl daemon-reload
systemctl restart k3s
```

## K3s installation using script for worker nodes

Check the token of the master node.

```bash
cat /var/lib/rancher/k3s/server/node-token
```

Install K3s on worker nodes.

> [!IMPORTANT]
> This command must be run on each worker node.

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://{master_ip}:6443 K3S_TOKEN={master_token} sh -
```

Label worker nodes.

```bash
kubectl label node traodev2 node-role.kubernetes.io/worker=worker
kubectl label node traodev3 node-role.kubernetes.io/worker=worker
kubectl label node krsrc04 node-role.kubernetes.io/worker=worker
kubectl label node krsrc05 node-role.kubernetes.io/worker=worker
```

Check the status of the K3s cluster.

```bash
kubectl get nodes
```

## Rancher installation

Add helm chart and make namespace.

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
kubectl create namespace cattle-system
```

Install `cert-manager` for k3s certificate

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.1/cert-manager.crds.yaml
helm repo add jetstack https://charts.jetstack.io
helm repo update
kubectl create namespace cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.13.1
kubectl get pods --namespace cert-manager
```

Download `Rancher` and modify the version requirement.

> [!NOTE]
> Latest k3s version is higher than what Rancher supports.

```bash
helm pull rancher-stable/rancher
tar zxvf rancher-2.7.9.tgz
vi rancher/Chart.yaml

kubeVersion: < 1.28.5
```

Install or upgrade `Rancher`.

```bash
helm install rancher ~/rancher \
  --namespace cattle-system \
  --set hostname={master_ip}.nip.io \
  --set bootstrapPassword={admin_password}

helm upgrade rancher ~/rancher \
  --namespace cattle-system \
  --set hostname={master_ip}.nip.io \
  --set bootstrapPassword={admin_password}
```

Initialize `Rancher`

```bash
echo https://{master_ip}.nip.io/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')
```

> [!NOTE]
> Reset `Rancher` password.
> ```bash
> kubectl -n cattle-system exec $(kubectl -n cattle-system get pods -l app=rancher --no-headers | head -1 | awk '{ print $1 }') -c rancher -- reset-password
> ```

> [!IMPORTANT]
> Reboot is required!!!

Open `Rancher` dashboard on browser.

```bash
https://{master_ip}.nip.io
```

## Tips

### Renewal Rancher CA

```bash
kubectl create secret tls tls-rancher-ingress \
  --cert=tls.crt \
  --key=tls.key \
  --dry-run --save-config -o yaml | kubectl apply -f -
```


