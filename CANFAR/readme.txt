############################################################################
#                             CANFAR installation                          #
############################################################################
# Based on
(CANFAR)
https://github.com/opencadc/science-platform/tree/SP-3544/deployment/helm
(ROOK for CephFS)
https://rook.io/docs/rook/latest-release/Storage-Configuration/Shared-Filesystem-CephFS/filesystem-storage/#quotas
https://github.com/rook/rook/tree/release-1.13/deploy/examples/csi
(oidc token)
https://confluence.skatelescope.org/display/SRCSC/RED-10+Using+oidc-agent+to+authenticate+to+OpenCADC+services
############################################################################
############################################################################
0.1 oidc : Just for getting temporary access to test each pods - posixmapper, skaha, and so on
############################################################################
#Prepare oidc : running oidc agent
eval $(oidc-agent)
eval $(oidc-agent-service use) > /dev/null
#create a new device (time limited)
oidc-gen --iss=https://ska-iam.stfc.ac.uk --scope max --flow=device <id>
oidc-gen --iss=https://krsrc.kasi.re.kr --scope max --flow=device <id>
#check id
oidc-gen --print <id>
#when time passed long, it is needed to load again
oidc-add <id>
#it is verification with host ID
curl -s -H "authorization: bearer $SKA_TOKEN" https://ska-iam.stfc.ac.uk/userinfo | jq
curl -s -H "authorization: bearer $SKA_TOKEN" https://krsrc.kasi.re.kr/userinfo | jq

#To test each process, SKA_TOKEN SHOULD BE SET BY BELOW (SKA_TOKEN):
export SKA_TOKEN=$(oidc-token <id>)

############################################################################
# 1. Installation of CANFAR
############################################################################
1.1 Official steps for CANFAR platform
#Install helm repository
helm repo add science-platform https://images.opencadc.org/chartrepo/platform
helm repo update

helm install --values my-base-local-values-file.yaml base science-platform/base
#--at this step, PV, PVC, and StorageClass(SC) should be set
helm install -n skaha-system --values my-posix-mapper-local-values-file.yaml posixmapper science-platform/posixmapper
helm install -n skaha-system --values my-skaha-local-values-file.yaml skaha science-platform/skaha
helm install -n skaha-system --dependency-update --values my-scienceportal-local-values-file.yaml scienceportal science-platform/scienceportal
helm install -n skaha-system --values my-cavern-local-values-file.yaml cavern science-platform/cavern
helm install -n skaha-system --dependency-update --values my-storage-ui-local-values-file.yaml storage-ui science-platform/storageui

############################################################################
1.2 CRT version
helm install --values crt-base-local.yaml base science-platform/base
#--at this step, PV, PVC, and StorageClass(SC) should be set
helm install -n skaha-system --values crt-posix-mapper-local.yaml posixmapper science-platform/posixmapper
helm install -n skaha-system --values crt-skaha-local.yaml skaha science-platform/skaha
helm install -n skaha-system --dependency-update --values crt-scienceportal-local.yaml scienceportal science-platform/scienceportal
helm install -n skaha-system --values crt-cavern-local.yaml cavern science-platform/cavern
helm install -n skaha-system --dependency-update --values crt-ui_local.yaml storage-ui science-platform-client/storageui

############################################################################
1.3 Details
############################################################################
# BASE is just setting namespaces of 'skaha-system' and 'skaha-workload'
helm install --values crt-base-local.yaml base science-platform/base
Converting crt & key are needed:

- prepare domain key.pem and cert.pem
- Actually, pem and crt are same, but 'CR-LF' for pem and 'LF' for crt is different.
- For CANFAR, base64 from crt is necessary.
- Therefore, original pem have to be converted (not copied), and then printed by base64.
- But 'openssl' just convert first block of CERTIFICATE. Therefore, last block should be converted and merged them

cp cert.pem cert1.pem
cp cert.pem cert2.pem

-at cert2.pem, delete first block of CERTIFICATE

openssl x509 -inform PEM -in cert1.pem -out canfar1.crt
openssl x509 -inform PEM -in cert2.pem -out canfar2.crt

cat canfar1.crt canfar2.crt > canfar_full.crt
cat canfar_full.crt | base64 -w 0
--> It will print <base64 coded>
--> It can be used for tls.crt

- In case of key.pem, it just has a one block of CERTIFICATE.
- At first, decode it with a PASSWORD: K....l
- On Linux machine, 'LF' is adopted directly!
openssl rsa -in key.pem -out key.crt
//<no needed on LInux>openssl x509 -inform PEM -in key.pem -out key.crt
cat key.crt | base64 -w 0
--> It can be used for tls.key

At all yaml files, tls.crt & tls.key should use upper key values!!!

################################################
# Refer to CephFS (1.4) for CephFS Storage setup.
################################################
# After BASE, PV, PVC, and SC have to be set up. Because BASE makes namespace of 'skaha-system' and 'skaha-workload'.

skaha-pvc                 : PVC -> SC -> PV : skaha-pv
sksha-workload-cavern-pvc : PVC -> SC -> PV : skaha-workload-pv

Basically, SC links 'skaha-pvc' with skaha-pv', at first. And then, not to confuse, 'skaha-workload-cavern-pvc' needs pointing 'skaha-workload-pv'. SC, storageclass uses provisioner of 'rook-ceph.cephfs.csi.ceph.com' (it is made by storageclass_rook-cephfs.yaml)

- Set up SC : Needed Once!
kubectl apply -f storageclass_rook-cephfs.taml
- Set up PVs and PVCs : Needed when BASE is reloaded only! At that time (BASE reloaded), old PVs and PVCs (released) may need deleted.
kubectl apply -f pv_cephfs_1.yaml -f pv_cephfs_2.yaml -f pvc_skaha.yaml -f pvc_skaha-workload.yaml 

################################################
# Refer to CephFS (1.4) for CephFS Storage setup.
################################################
# It may be that I'm faking. But if CephFS and others are well. It is just simple followed:
helm install -n skaha-system --values crt-posix-mapper-local.yaml posixmapper science-platform/posixmapper
helm install -n skaha-system --values crt-skaha-local.yaml skaha science-platform/skaha
helm install -n skaha-system --dependency-update --values crt-scienceportal-local.yaml scienceportal science-platform/scienceportal
helm install -n skaha-system --values crt-cavern-local.yaml cavern science-platform/cavern
helm install -n skaha-system --dependency-update --values crt-ui-local.yaml storage-ui science-platform-client/storageui

#### BUT I found (a) bug(s) ####
#storage UI can't open -> cavern-tomcat pod :
  Warning  FailedMount             2m35s (x13 over 50m)  kubelet                  MountVolume.SetUp failed for volume "pv-cephfs" : rpc error: code = DeadlineExceeded desc = context deadline exceeded

While installing, if you test at posix-mapper, orionkhw_uk may have other UID & GID. Check it and apply it on crt-cavern-local.yaml
    filesystem:
      dataDir: /data
      subPath: cavern
      rootOwner:
        adminUsername: orionkhw_uk
        username: orionkhw_uk
        uid: 10001
        gid: 10001


################################################
# Test for each step (SKA_TOKEN is necessary)
################################################
# At posix-mapper step:
curl -SsL --header "authorization: Bearer ${SKA_TOKEN}" https://canfar.kasi.re.kr/posix-mapper/uid
curl -SsL --header "authorization: Bearer ${SKA_TOKEN}" "https://canfar.kasi.re.kr/posix-mapper/uid?user=mynewuser"

# At skaha step:
curl -SsL --header "authorization: Bearer ${SKA_TOKEN}" https://canfar.kasi.re.kr/skaha/v0/session
curl -SsL --header "authorization: Bearer ${SKA_TOKEN}" -d "ram=1" -d "cores=1" -d "image=images.canfar.net/canucs/canucs:1.2.5" -d "name=myjupyternotebook" "https://canfar.kasi.re.kr/skaha/v0/session"

############################################################################
1.4 CephFS
############################################################################
# There are other storage solution - longhorn, local, hostpath,... But only one solution of multiple connection to one storage is CephFS only. Even NFS is unstable for CANFAR (At Slack...)

# Rook is simple(?) solution to set up via yaml. There is another solution with CLI. KRSRC just uses ROOK.
https://rook.io/docs/rook/latest-release/Storage-Configuration/Shared-Filesystem-CephFS/filesystem-storage/#quotas

################################################
1.4.1. Preparation
: Raw sdd or hdd (minimum 3 recommanded) are necessary with attached some nodes.
: if hdd used, zapping device have to be:
https://rook.io/docs/rook/latest-release/Getting-Started/ceph-teardown/#delete-the-cephcluster-crd

: CephFS is automatic system to detect raw disk (OSD). While initializing CephFS, it will be set up.

- At a K3S cluster. initial set up is required: crds, common, and, operator
- Because K3S already set up some solution, when 'create' is requested, there will be so many errors.
- cluster.yaml is followed to install basic cluster.

kubectl apply -f crds.yaml -f common.yaml -f operator.yaml
kubectl apply -f cluster.yaml

- Wait some times for pods running
- Rancher : At Service Discovery -> Services -0> rook-ceph-mgr-dashboard, password is:
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo

###### From here, Ceph Usage is needed ######
At Dashboard, check health and reason, or
Goto ceph-tool:
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
- if toolbox is absent: (I already pulled rook Git)
kubectl create -f rook/deploy/examples/toolbox.yaml 
kubectl -n rook-ceph rollout status deploy/rook-ceph-tools

-and:
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

[Ceph command] : to find what problem is
ceph health detail
------------------------------------------------
root@kmtnet-kasi:/home/kmtkasi/CANFAR/Ceph# kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
bash-4.4$ ceph health detail
HEALTH_WARN 1 pool(s) do not have an application enabled
[WRN] POOL_APP_NOT_ENABLED: 1 pool(s) do not have an application enabled
    application not enabled on pool '.mgr'
    use 'ceph osd pool application enable <pool-name> <app-name>', where <app-name> is 'cephfs', 'rbd', 'rgw', or freeform for custom applications.
------------------------------------------------
# In this case, mgr is not enabled. Solution is:
ceph osd pool application enable .mgr mgr

[Ceph command] : to enable orchestra by rook
------------------------------------------------
bash-4.4$ ceph mgr module enable rook
bash-4.4$ ceph orch set backend rook
bash-4.4$ ceph orch status
------------------------------------------------
################################################
1.4.2. Make CephFS and mount it to K3S
# It is recommended to create CephFS with yaml
a) kubectl apply -f filesystem.yaml
------------------------------------------------
root@kmtnet-kasi:/home/kmtkasi/CANFAR/Ceph# kubectl apply -f filesystem.yaml 
cephfilesystem.ceph.rook.io/canfar-cephfs created
cephfilesystemsubvolumegroup.ceph.rook.io/canfar-cephfs-csi created
------------------------------------------------
#
b) Make storage class to link with CephFS to K3S
------------------------------------------------
root@kmtnet-kasi:/home/kmtkasi/CANFAR/Ceph# kubectl apply -f storageclass_rook-cephfs.yaml 
------------------------------------------------

c) Make subvolume on Ceph GUI
UI:(password is at upper!) -> FileSystems -> click a file system of 'canfar-cephfs' -> click a Groups of 'csi' -> click '+Create'
-> subvolume name : subvolume, size : 10GiB and 'create'
# You SHOULD copy 'Path'
/volumes/csi/subvolume/919735b7-450f-4114-a3fa-3ab41d24856d

d) Edit pv_cephfs_1.yaml and pv_cephfs_2.yaml
- pv_cephfs_1.yaml
at spec.csi:
      subvolumeName: subvolume
      subvolumePath: >-
        /volumes/csi/subvolume/919735b7-450f-4114-a3fa-3ab41d24856d
      staticVolume: "true"
      rootPath: /volumes/csi/subvolume/919735b7-450f-4114-a3fa-3ab41d24856d
- pv_cephfs_2.yaml
at spec.csi:
      subvolumeName: subvolume
      subvolumePath: >-
        /volumes/csi/subvolume/919735b7-450f-4114-a3fa-3ab41d24856d
      rootPath: >-
        /volumes/csi/subvolume/919735b7-450f-4114-a3fa-3ab41d24856d
      staticVolume: "true"
      rootPath: /volumes/csi/subvolume/919735b7-450f-4114-a3fa-3ab41d24856d

################################################
1.4.3. Make K3S to link with CephFS
# At Rancher: (from rook site:)
https://rook.io/docs/rook/latest-release/Storage-Configuration/Shared-Filesystem-CephFS/filesystem-storage/#consume-the-shared-filesystem-toolbox

a) Storage->Secrets->rook-csi-cephfs-node
   Clone it and rename clond secret to 'rook-csi-cephfs-node-user'
b) Edit adminID -> userID, adminKey -> userKey

################################################
1.4.4. Create PVs and PVCs
-----------------------------------------------
root@kmtnet-kasi:/home/kmtkasi/CANFAR/Ceph# kubectl apply -f pv_cephfs_1.yaml -f pv_cephfs_2.yaml -f pvc_skaha.yaml -f pvc_skaha-workload.yaml 
persistentvolume/pv-cephfs created
persistentvolume/pv-cephfs-workload created
persistentvolumeclaim/skaha-pvc created
persistentvolumeclaim/skaha-workload-cavern-pvc created
-----------------------------------------------
########        CephFS FINISHED       ##########


################################################
################################################
#######          Garbage but if       ##########
################################################
################################################
#How to make ca.key & ca.crt for canfar.kasi.re.kr
#making ca.key
openssl genrsa -out ca.key 2048
#making ca.crt
openssl req -x509 -new -nodes -key ca.key -sha256 -subj "/CN=canfar.kasi.re.kr" -days 1024 -out ca.crt -extensions san -config <(
echo '[req]';
echo 'distinguished_name=req';
echo '[san]';
echo 'subjectAltName=DNS:canfar.kasi.re.kr')

#making base64 encoded value
cat ca.key | base64 -w 0
cat ca.crt | base64 -w 0

#But it doesn't support 'fullchain'. 
#########################

# find cadc-registry.properties (docker)
# add (Rancher->Local->Storage->ConfigMaps->posix-mapper-config) posix-mapper-config
ivo://canfar.krsrc.kr/posix-mapper = https://canfar.kasi.re.kr/posix-mapper/capabilities

# find cadc-registry.properties (docker)
# add (Rancher->Local->Storage->ConfigMaps->skaha-config) skaha-config
ivo://canfar.org/posix-mapper = https://canfar.kasi.re.kr/posix-mapper/capabilities
#########################
#if an error:
unexpected exception: ca.nrc.cadc.util.InvalidConfigException: failed to load configured IdentityManager: org.opencadc.auth.StandardIdentityManager
-> device is not initialized well.
:Check IAM logged, and do below:
curl -s -H "authorization: bearer $SKA_TOKEN" https://ska-iam.stfc.ac.uk/userinfo | jq

#DO SET SKA_TOKEN MUST BY : 
SKA_TOKEN=$(oidc-token example-client)
SKA_TOKEN=$(oidc-token canfar)
#########################

#########################
#when cert changed, reload it to Rancher
kubectl -n cattle-system create secret tls tls-rancher-ingress \
  --cert=tls.crt \
  --key=tls.key \
  --dry-run --save-config -o yaml | kubectl apply -f -


kubectl get validatingwebhookconfiguration rancher.cattle.io -o yaml > vwc-rancher.cattle.io.yaml
kubectl get mutatingwebhookconfiguration rancher.cattle.io -o yaml > mwc-rancher.cattle.io.yaml
kubectl delete validatingwebhookconfiguration rancher.cattle.io
kubectl delete mutatingwebhookconfiguration rancher.cattle.io

kubectl apply -f vwc-rancher.cattle.io.yaml
kubectl apply -f mwc-rancher.cattle.io.yaml

#########################

#(UFW) Mounting at Ubuntu 22.04
sudo ufw allow from 192.168.10.0/24 to any port 2049
sudo ufw allow from 192.168.10.0/24 to any port 111

# At Rancher, PersistentVlume form should be:
Path: /home/kmtkasi/CANFAR <only dir>
Server: 192.168.10.1 <only IP>

#########################
#About PVC,
It should be RW enabled by any user. (nobody, nogroup)
For example, ../CANFAR mode is 777

#########################
#pin out configmap
kubectl -n skaha-system get configmap -o yaml > get_configmap_on_posixmapper.yaml
#########################

#########################
#upgrade pods
helm -n skaha-system upgrade --install --values my-posix-mapper-local-values-file.yaml posixmapper science-platform/posixmapper
helm -n skaha-system upgrade --install --values my-skaha-local-values-file.yaml skaha science-platform/skaha
helm -n skaha-system upgrade --install --dependency-update --values my-scienceportal-local-values-file.yaml scienceportal science-platform/scienceportal
helm -n skaha-system upgrade --install --dependency-update --values <my-storage-ui-values-file> storage-ui science-platform-client/storage-ui
#########################

#########################
#uninstall all
helm uninstall -n skaha-system storageui
helm uninstall cavern
helm uninstall -n skaha-system scienceportal
helm uninstall -n skaha-system skaha
helm uninstall -n skaha-system posixmapper
helm uninstall base

#########################
#oidc commands
oidc-add example-client
oidc-gen -d <id>   # to delete previous account

#########################
# Ceph commands
#########################
# Password for Ceph Dashboard
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo

# How to delete Pools
ceph osd pool delete {pool-name} [{pool-name} --yes-i-really-really-mean-it]

##At recent_mgr_module_crash
ceph crash archive-all

## How to run CEPH TOOLBOX on CLI
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

ceph-secret:
AQDcS+FliC7FNBAApzQ8yLoHTie9APnDilNHUg==
ceph-username:
client.admin
fsid:
4bba1e8f-f2d5-4e14-beb6-b8ce67acc9b7
mon-secret:
AQDcS+FlaW1SMxAAJ17yS5ItxVHTpdTIW89DNw==


###########################
# Kubernetes Tips
###########################
# How to erase un-terminated things (pod/secret/namespace...)
a) check what is not terminated (IMPORTANT STEP)
kubectl api-resources --verbs=list --namespaced -o name   | xargs -n 1 kubectl get --show-kind --ignore-not-found -n <namespace>

b) make a un-terminated thing(pod....) that 'finalizer' is disabled:
- in case of namespace 'rook-ceph'
kubectl get ns/rook-ceph -o yaml > rook-ceph_exit.yaml
- in case of secret 'rook-ceph-mon'
kubectl secret rook-ceph-mon -n rook-ceph -o yaml > rook-ceph-secret-exit.yaml

- then erase 'finalizer' value, and apply
kubectl apply -f ...

- kill on force mode
kubectl -n rook-ceph delete secret --grace-period=0 --force rook-ceph-mon

######### CephFS Tear Down : How To ########
https://rook.io/docs/rook/v1.0/ceph-teardown.html#troubleshooting
-about filesystem:
kubectl patch cephfilesystem.ceph.rook.io canfar-cephfs -n rook-ceph -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl patch cephfilesystemsubvolumegroup.ceph.rook.io canfar-cephfs-csi -n rook-ceph -p '{"metadata":{"finalizers":[]}}' --type=merge

######### CephFS : RECENT_MGR_MODULE_CRASH ########
when health report it, solve at ceph-tool by:
ceph crash archive-all

######## CephFS Dash board #########
- at ceph-tool
cd /tmp
echo -n "password" > password.txt
ceph dashboard ac-user-create orionkhw -i password.txt  administrator
ceph dashboard ac-user-show

