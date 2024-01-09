# PoC Test by KRSRC

## References

<https://confluence.skatelescope.org/pages/viewpage.action?pageId=235981012>  
<https://confluence.skatelescope.org/pages/resumedraft.action?draftId=252390056&draftShareId=2069b0b5-62e8-42e0-bd59-65c4778a3722&>  
<https://doc.grid.surfsara.nl/en/latest/Pages/Advanced/softdrive_on_laptop.html#configuring-cvmfs>  
<https://artifacthub.io/packages/helm/sciencebox/cvmfs>  

## Environments

- The PoC has been tested on a container-based Ubuntu 22.04 system running in the K3s cluster deployed on the KRSRC testbed.
- Instead of installing and configuring CVMFS directly on the K3s cluster for all users, it has been installed and configured individually on the Ubuntu container.

## Ubuntu 22 deployment

Making YAML for `ubuntu` deployment: `pod_ubuntu_ubuntu-poc.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-poc
  labels:
    app: ubuntu-poc
spec:
  containers:
  - name: ubuntu
    image: ubuntu:latest
    command: ["/bin/sleep", "3650d"]
    securityContext:
      privileged: true
      capabilities:
        add: ["SYS_ADMIN"]
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
```

Open terminal of `ubuntu-poc` pod.

```bash
kubectl exec --stdin --tty ubuntu-poc -- /bin/bash
```

Set root passwd

```bash
passwd
```

Install required apps.

```bash
apt update
apt install net-tools sudo vim
```

Add user for ssh connection.

```bash
adduser {username}
usermod -aG sudo {username}
groups {username}
```

Install ssh server.

```bash
apt update
apt install openssh-server
```

Modify ssh port.

```bash
vi /etc/ssh/sshd_config

Port {sshd_port}
```

Create LoadBalancer service.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ubuntu-ssh
spec:
  type: LoadBalancer
  externalIPs:
  - {master_ip}
  ports:
  - name: ubuntu-poc
    port: {sshd_external_port}
    protocal: TCP
    targetPort: {sshd_port}
  selector:
    app: ubuntu-poc
status:
  loadBalancer:
    ingress:
    - ip: {master_ip}
```

Test to connection.

```bash
ssh -X -p {sshd_external_port} manager@{master_ip}
```

## FUSE installation

Start with root account.

```bash
sudo bash -
```

```bash
apt install meson pkg-config python3-pip udev
wget https://github.com/libfuse/libfuse/releases/download/fuse-3.10.5/fuse-3.10.5.tar.xz
tar xfJ fuse-3.10.5.tar.xz
cd fuse-3.10.5/
mkdir build
cd build
meson ..
meson configure
meson configure -D disable-mtab=true
ninja
ninja install
rm /usr/lib/x86_64-linux-gnu/libfuse3.so.3
ln -s /usr/local/lib/x86_64-linux-gnu/libfuse3.so.3 /usr/lib/x86_64-linux-gnu/libfuse3.so.3
```

Configure `FUSE` (uncomment).

```bash
vi /etc/fuse.conf

user_allow_others
```

Install `squid` and check http proxy port.

```bash
apt install squid
cat /etc/squid/squid.conf | grep http_port
service squid start
```

## CVMFS installation

Add the CVMFS repository and install CFMFS.

```bash
wget https://ecsft.cern.ch/dist/cvmfs/cvmfs-release/cvmfs-release-latest_all.deb
dpkg -i cvmfs-release-latest_all.deb
rm -f cvmfs-release-latest_all.deb
apt update
apt install cvmfs
```

```bash
cat /etc/auto.master.d/cvmfs.autofs
service austofs restart
```

Create the configuration file.

```bash
vi /etc/cvmfs/default.local

CVMFS_NFILES=32768
CVMFS_REPOSITORIES=softdrive.nl
CVMFS_QUOTA_LIMIT=2000
CVMFS_HTTP_PROXY="http://localhost:{squid_http_port}"

vi /etc/cvmfs/config.d/softdrive.nl.conf

CVMFS_SERVER_URL=http://cvmfs01.nikhef.nl/cvmfs/@fqrn@
CVMFS_PUBLIC_KEY=/etc/cvmfs/keys/softdrive.nl.pub

vi /etc/cvmfs/keys/softdrive.nl.pub

-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA481/kCXbrVtLuzcFZ2uO
EmiAKx28qXIkonPwr/gSmqQ8k1zQA7dKK5YZwZSbVwgYqvhvW6i3vKWLGVDj+elH
1u8uumPzzlAJHrS1XoR8rY4xUULjQBvV9HuJxE6OK4ZEZPvQmeGmjXd446c8J5cv
BQFtaonRnrxAbtO+Z0KtzsNOzBNFegu9z+lT7/fxV17Qh10w5IKQjm/v6jPdj1ME
CrG4QW2S9+Y+7YzbRP5QYaE4cl5cBI3Yb048ufgLJMfX3++uqwGM+rqNs/CzHvsW
dO6Jznr9EbzqbIrTsFeUThNmsGPObxOT3VmB0BTTjrZSYjgf8oEE4hdhgNQgh7vs
OwIDAQAB
-----END PUBLIC KEY-----
```

```bash
cvmfs_config chksetup
OK
```

```bash
sudo mkdir /cvmfs/softdrive.nl
```
