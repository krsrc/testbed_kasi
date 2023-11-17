# KRSRC Testbed3 

## Resources

**in KASI**
1. kmtnet-kasi : Master of Kubernetes cluster (x.x.x.1)
   - PC : Dell
   - OS : Ubuntu 22.x
2. traodev2 : Slave 1 (x.x.x.21)
   - PC : HP
   - OS : Ubuntu 18.x
3. traodev3 : Slave 2 (x.x.x.22)
   - PC : HP
   - OS : Ubuntu 22.x

**in UNIST**
1. ska00 (Desktop 1): for AAI
   - system: HDD 13TB, RAM 128GB, 16 cores
   - OS: Ubuntu 22.xx
2. ska01 (Desktop 2): for master of Kubernetes cluster
   - system: HDD 33TB, RAM 256GB, 22 cores
   - OS: Ubuntu 22.xx
3. ska02 (Desktop 3): for slave 1
   - system: HDD 25TB, RAM 256GB,
   - OS: Ubuntu 22.xx
4. ska03 (Desktop 4): for slave 2
   - system: HDD 17GB, RAM 128GB,
   - OS: Ubuntu 22.xx
5. ska04 (Desktop 5): for slave 3
   - system: HDD 17TB, RAM 128GB,
   - OS: Ubuntu 22.xx
6. ska04 (Desktop 5): for slave 4
   - system: HDD, RAM, cores
   - OS : Ubuntu 22.xx
   - 
## Milestones

- [x] Deploy `Kubernetes` (1 master + 2 slaves)
  - [x] Install `Kubernetes` on 210
- [x] Deploy `Rancher`
- [ ] Deploy `JupyterHub` with `Rancher`
- [ ] Deploy `INDIGO IAM`
- [ ] Connect `INDIGO IAM` and KAFE
- [ ] Connect `JupyterHub` and `INDIGO IAM`
- [ ] Deploy `CANFAR`
- [ ] Connect `CANFAR` and `INDIO IAM`

## Links

- Documentation: [Deployment of Rancher K8s cluster](Rancher/README.md)
