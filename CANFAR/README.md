# CANFAR Deployment

> [!NOTE]
>
> - CANFAR reference:
>   - <https://github.com/opencadc/science-platform/tree/SP-3544/deployment/helm>
> - OIDC TOKEN reference:
>   - <https://confluence.skatelescope.org/display/SRCSC/RED-10+Using+oidc-agent+to+authenticate+to+OpenCADC+services>

## Access check using OIDC

Just for getting temporary access to test each pods - posixmapper, skaha, and so on.

Prepare oidc : running oidc agent

```bash
eval $(oidc-agent)
eval $(oidc-agent-service use) > /dev/null
```

create a new device (time limited)

```bash
oidc-gen --iss=https://ska-iam.stfc.ac.uk --scope max --flow=device <id>
```

check id

```bash
oidc-gen --print <id>
```

when time passed long, it is needed to load again

```bash
oidc-add <id>
```

it is verification with host ID

```bash
curl -s -H "authorization: bearer $SKA_TOKEN" https://ska-iam.stfc.ac.uk/userinfo | jq
```

To test each process, SKA_TOKEN SHOULD BE SET BY BELOW (SKA_TOKEN):

```bash
export SKA_TOKEN=$(oidc-token <id>)
```

## Installation of CANFAR

### Official steps for CANFAR platform

Install helm repository

```bash
helm repo add science-platform https://images.opencadc.org/chartrepo/platform
helm repo update
helm install --values my-base-local-values-file.yaml base science-platform/base
```

at this step, PV, PVC, and StorageClass(SC) should be set

```bash
helm install -n skaha-system --values my-posix-mapper-local-values-file.yaml posixmapper science-platform/posixmapper
helm install -n skaha-system --values my-skaha-local-values-file.yaml skaha science-platform/skaha
helm install -n skaha-system --dependency-update --values my-scienceportal-local-values-file.yaml scienceportal science-platform/scienceportal
helm install -n skaha-system --values my-cavern-local-values-file.yaml cavern science-platform/cavern
helm install -n skaha-system --dependency-update --values my-storage-ui-local-values-file.yaml storage-ui science-platform/storageui
```

### CRT Version

```bash
helm install -n skaha-system --values crt-posix-mapper-local.yaml posixmapper science-platform/posixmapper
helm install -n skaha-system --values crt-skaha-local.yaml skaha science-platform/skaha
helm install -n skaha-system --dependency-update --values crt-scienceportal-local.yaml scienceportal science-platform/scienceportal
helm install -n skaha-system --values crt-cavern-local.yaml cavern science-platform/cavern
helm install -n skaha-system --dependency-update --values crt-ui_local.yaml storage-ui science-platform-client/storageui
```

### Details

BASE is just setting namespaces of `skaha-system` and `skaha-workload`

```bash
helm install --values crt-base-local.yaml base science-platform/base
```

Converting crt & key are needed:

- prepare domain `key.pem` and `cert.pem`
- Actually, pem and crt are same, but 'CR-LF' for pem and 'LF' for crt is different.
- For CANFAR, base64 from crt is necessary.
- Therefore, original pem have to be converted (not copied), and then printed by base64.
- But `openssl` just convert first block of CERTIFICATE. Therefore, last block should be converted and merged them

```bash
cp cert.pem cert1.pem
cp cert.pem cert2.pem
```

at `cert2.pem`, delete first block of CERTIFICATE

```bash
openssl x509 -inform PEM -in cert1.pem -out canfar1.crt
openssl x509 -inform PEM -in cert2.pem -out canfar2.crt

cat canfar1.crt canfar2.crt > canfar_full.crt
cat canfar_full.crt | base64 -w 0
```

--> It will print <base64 coded>
--> It can be used for tls.crt

- In case of key.pem, it just has a one block of CERTIFICATE.
- At first, decode it with a PASSWORD: K....l
- On Linux machine, 'LF' is adopted directly!

```bash
openssl rsa -in key.pem -out key.crt
```

> [!IMPORTANT]
> On the non-linux machine:
>
> ```bash
> openssl x509 -inform PEM -in key.pem -out key.crt
> ```

```bash
cat key.crt | base64 -w 0
```

--> It can be used for tls.key

At all yaml files, tls.crt & tls.key should use upper key values!!!

## Refer to CephFS (1.4) for CephFS Storage setup

After BASE, PV, PVC, and SC have to be set up. Because BASE makes namespace of `skaha-system` and `skaha-workload`.

- `skaha-pvc`: PVC -> SC -> PV `skaha-pv`
- `sksha-workload-cavern-pvc`: PVC -> SC -> PV `skaha-workload-pv`

Basically, SC links 'skaha-pvc' with skaha-pv', at first. And then, not to confuse, 'skaha-workload-cavern-pvc' needs pointing 'skaha-workload-pv'. SC, storageclass uses provisioner of 'rook-ceph.cephfs.csi.ceph.com' (it is made by storageclass_rook-cephfs.yaml)


