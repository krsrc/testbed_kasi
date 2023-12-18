# Deployment of Kubernetes Storage using OpenEBS

## Install OpenEBS Local Hostpath

Make namespace.

```bash
kubectl create namespace openebs
```

Install `OpenEBS` from url.

```bash
kubectl apply -f https://openebs.github.io/charts/openebs-operator-lite.yaml
kubectl apply -f https://openebs.github.io/charts/openebs-lite-sc.yaml
```


