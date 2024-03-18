# Jupyterhub installation on Rancher K3s cluster

> [!NOTE]  
> Installing JupyterHub.  
> <https://z2jh.jupyter.org/en/latest/jupyterhub/installation.html>  
>
> Chart config reference:  
> <https://zero-to-jupyterhub.readthedocs.io/en/stable/resources/reference.html>  
>
> Chart default values:  
> <https://github.com/jupyterhub/zero-to-jupyterhub-k8s/blob/HEAD/jupyterhub/values.yaml>

## Initialize a Helm chart configuration file

Just create an empty `jupyterhub-config.yaml` file with some helpful comments.

```yaml
# This file can update the JupyterHub Helm chart's default configuration values.
#
# For reference see the configuration reference and default values, but make
# sure to refer to the Helm chart version of interest to you!
#
# Introduction to YAML:     https://www.youtube.com/watch?v=cdLNKUoMc6c
# Chart config reference:   https://zero-to-jupyterhub.readthedocs.io/en/stable/resources/reference.html
# Chart default values:     https://github.com/jupyterhub/zero-to-jupyterhub-k8s/blob/HEAD/jupyterhub/values.yaml
# Available chart versions: https://hub.jupyter.org/helm-chart/
#
```

## Install JupyterHub

Make Helm aware of the JupyterHub Helm chart repository.

```bash
helm repo add jupyterhub https://hub.jupyter.org/helm-chart/
helm repo update
```

Reference.

```bash
helm upgrade --cleanup-on-fail \
  --install [helm-release-name] jupyterhub/jupyterhub \
  --namespace [k8s-namespace] \
  --create-namespace \
  --version=[chart-version] \
  --values config.yaml
```

Install the chart configured by your `jupyterhub-config.yaml`.

```bash
helm upgrade --cleanup-on-fail \
  --install jupyterhub jupyterhub/jupyterhub \
  --namespace jhub \
  --create-namespace \
  --values jupyterhub-config.yaml
```

Check running pods.

```bash
kubectl get pod -n jupyterhub

NAME                              READY   STATUS    RESTARTS   AGE
continuous-image-puller-7fwtd     1/1     Running   0          4d19h
continuous-image-puller-7hxwd     1/1     Running   0          4d19h
hub-dfcc6bc4b-v5ntt               0/1     Pending   0          4d19h
proxy-6b6f55f988-v2549            1/1     Running   0          4d19h
user-scheduler-7697c9b4ff-b66vd   1/1     Running   0          4d19h
user-scheduler-7697c9b4ff-wbhq2   1/1     Running   0          4d19h
```

I found the pending status of the `hub` pod.

## Set external IP and port

Check running svcs.

```bash
kubectl get svc -n jupyterhub

NAME           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
hub            ClusterIP      10.109.74.72     <none>        8081/TCP       3d20h
proxy-api      ClusterIP      10.97.229.35     <none>        8001/TCP       3d20h
proxy-public   LoadBalancer   10.105.111.229   <pending>     80:32246/TCP   3d20h
```

`EXTERNAL-IP` of the `proxy-public` svc is also pending.

Get `yaml` of the `proxy-public` svc.  

```bash
kubectl get svc proxy-public -n jupyterhub -o yaml > jupyterhub-proxy-public.yaml
```

Add `externalIPs` and change `port` in the `jupyterhub-proxy-public.yaml`

```yaml
spec:
  ports:
  - name: http
    nodeport: 32246
    port: {jhub_port}
  externalIPs:
  - {master_ip}
```

Delete `proxy-public` and apply `jupyterhub-proxy-public.yaml`

```bash
kubectl delete svc proxy-public -n jupyterhub
kubectl apply -f jupyterhub-proxy-public.yaml
```

Check running svcs.

```bash
kubectl get svc -n jupyterhub
```

> [!NOTE]
> Using the Rancher dashboard, modifying port and adding external IP are too simple.  
> Rancher > local > Service Discovery > Services > jhub:proxy-public > Edit Config

## Testing installation

Open JupyterHub on the browser.

```bash
http://{master_ip}:{jhub_port}
```

Default account is id: `jovyan` and password: `jupyter`.
