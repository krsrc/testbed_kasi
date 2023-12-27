# KRSRC Testbed3

## Resources

| No. | Host name   | Role       | Ubuntu  | IP1   | IP2   | RAM   | CPU         | Clock | Ncore |
| --- | ----------- | ---------- | ------- | ----- | ----- | ----- | ----------- | ----- | ----- |
| 01  | kmtnet-kasi | K3s master | 22.04.3 | 0.210 | 10.1  | 32 GB | Xeon E-2234 | 3.6   | 8     |
| 02  | traodev2    | K3s worker | 18.04.6 | 0.21  | 10.11 | 16 GB | Xeon E-2236 | 3.4   | 12    |
| 03  | traodev3    | K3s worker | 22.04.3 | 0.22  | 10.12 | 16 GB | Xeon E-2236 | 3.4   | 12    |
| 04  | krsrc04     | K3s worker | 22.04.3 | n/a   | 10.14 | 16 GB | i5-4670     | 3.4   | 4     |
| 05  | krsrc05     | K3s worker | 22.04.3 | n/a   | 10.15 | 16 GB | i5-4690     | 3.5   | 4     |
| 06  | krsrc06     | AAI        | 22.04.3 | 0.206 | n/a   | 32 GB | i7-6700     | 3.4   | 8     |

<!-- **in UNIST**
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
   - OS : Ubuntu 22.xx -->

## Milestones

- [x] Deploy `Kubernetes` (1 master + 2 slaves)
  - [x] Install `Kubernetes` on 210
- [x] Deploy `Rancher`
- [x] Deploy `JupyterHub` with `Rancher`
- [ ] Deploy `INDIGO IAM`
- [ ] Connect `INDIGO IAM` and KAFE
- [ ] Connect `JupyterHub` and `INDIGO IAM`
- [ ] Deploy `CANFAR`
- [ ] Connect `CANFAR` and `INDIO IAM`

## Links

- Documentation: [Deployment of Rancher K8s cluster](Rancher/README.md)
