# Deployment of Jupyterhub on the K8s cluster

> [!NOTE]  
> Installing JupyterHub.  
> <https://z2jh.jupyter.org/en/latest/jupyterhub/installation.html>  
>
> Chart config reference:  
> <https://zero-to-jupyterhub.readthedocs.io/en/stable/resources/reference.html>  
>
> Chart default values:  
> <https://github.com/jupyterhub/zero-to-jupyterhub-k8s/blob/HEAD/jupyterhub/values.yaml>

## Installing JupyterHub

### Initialize a Helm chart configuration file

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

### Install JupyterHub

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
  --namespace jupyterhub \
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
    port: 4321
  externalIPs:
  - 192.168.10.1
```

Delete `proxy-public` and apply `jupyterhub-proxy-public.yaml`

```bash
kubectl delete svc proxy-public -n jupyterhub
kubectl apply -f jupyterhub-proxy-public.yaml
```

Check running svcs.

```bash
kubectl get svc -n jupyterhub

NAME           TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)          AGE
hub            ClusterIP      10.109.74.72     <none>         8081/TCP         3d20h
proxy-api      ClusterIP      10.97.229.35     <none>         8001/TCP         3d20h
proxy-public   LoadBalancer   10.105.111.229   192.168.10.1   4321:32246/TCP   4m46s
```

`EXTERNAL-IP` is updated and `PORT(S)` is changed to `4321`.  
However, still the `hub` pod is pending.

I got two events for `jupyterhub`.

```bash
kubectl events -n jupyterhub

LAST SEEN                   TYPE      REASON             OBJECT                             MESSAGE
4m5s (x1395 over 4d20h)     Warning   FailedScheduling   Pod/hub-dfcc6bc4b-v5ntt            0/3 nodes are available: pod has unbound immediate PersistentVolumeClaims. preemption: 0/3 nodes are available: 3 Preemption is not helpful for scheduling..
2m12s (x27845 over 4d20h)   Normal    FailedBinding      PersistentVolumeClaim/hub-db-dir   no persistent volumes available for this claim and no storage class is set
```

To resolve the `0/3 nodes are available` error, check the nodes status.

```bash
kubectl get node

NAME          STATUS   ROLES           AGE   VERSION
kmtnet-kasi   Ready    control-plane   42d   v1.26.7
traodev2      Ready    <none>          42d   v1.26.7
traodev3      Ready    <none>          42d   v1.26.7
```

We found there was no worker.  
Change `ROLES` of two slave nodes to `worker`.

```bash
kubectl label node traodev2 node-role.kubernetes.io/worker=worker
kubectl label node traodev3 node-role.kubernetes.io/worker=worker
```

Check the nodes status.

```bash
kubectl get node

NAME          STATUS   ROLES           AGE   VERSION
kmtnet-kasi   Ready    control-plane   42d   v1.26.7
traodev2      Ready    worker          42d   v1.26.7
traodev3      Ready    worker          42d   v1.26.7
```

Nov 7, 1 pm: The `hub` pod is still pending, and the same events are seen.

