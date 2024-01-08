# PoC

<https://confluence.skatelescope.org/pages/viewpage.action?pageId=235981012>  
<https://confluence.skatelescope.org/pages/resumedraft.action?draftId=252390056&draftShareId=2069b0b5-62e8-42e0-bd59-65c4778a3722&>  
<https://doc.grid.surfsara.nl/en/latest/Pages/Advanced/softdrive_on_laptop.html#configuring-cvmfs>  
<https://artifacthub.io/packages/helm/sciencebox/cvmfs>  

## Ubuntu 22 deployment

Making YAML for `ubuntu` deployment

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-poc
  labels:
    app: ubuntu
spec:
  containers:
  - name: ubuntu
    image: ubuntu:jammy
    command: ["/bin/sleep", "3650d"]
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
```
