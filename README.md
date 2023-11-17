# KRSRC Testbed3 

## Resources

__ in KASI __
1. Master of Kubernetes cluster
  - kmtnet-kasi (x.x.x.1)
  - Dell
  - Ubuntu 22.x
2. Slave 1
  - traodev2 (x.x.x.21)
  - HP
  - Ubuntu 18.x
3. xxx: Slave 2
  - traodev3 (x.x.x.22)
  - HP
  - Ubuntu 22.x

** in UNIST **
1. ska00 (Desktop 1): for AAI
   - info: HDD 13TB, RAM 128GB, 16 cores
   - system : Ubuntu 22.04
2. ska01 (Desktop 2): for master of Kubernetes cluster
   - info: HDD 33TB, RAM 256GB, 22 cores
   - system : Ubuntu 22.04
3. ska02 (Desktop 3): for slave 1
   - info: HDD 25TB, RAM 256GB,
   - system : Ubuntu 22.04
4. ska03 (Desktop 4): for slave 2
   - info: HDD 17GB, RAM 128GB,
   - system : Ubuntu 22.04
5. ska04 (Desktop 5): for slave 3
   - info: HDD 17TB, RAM 128GB,
   - system : Ubuntu 22.04
6. ska04 (Desktop 5): for slave 4
   - info: HDD, RAM, cores
   - system : Ubuntu 22.04
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
