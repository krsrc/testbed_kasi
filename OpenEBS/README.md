# Deployment of Kubernetes Storage using OpenEBS

## Install OpenEBS Local Hostpath

Setup helm repository.

```bash
helm repo add openebs https://openebs.github.io/charts
helm repo update
```

Install OpenEBS helm chart with default values.

```bash
helm install openebs --namespace openebs openebs/openebs --create-namespace
```


