# 🚀 AIML Platform — Proxmox K8s Cluster | Daily Progress Log

> **Project:** AIML Platform Infrastructure on Proxmox VE 9.1.1  
> **Environment:** Staging (Proxmox Cluster: controller-1 + controller-2 + director)  
> **Date:** 06 May 2026  
> **Author:** Surajeet Banerjee  
> **Status:** 🟡 In Progress

***

## 📋 Overview

This document tracks the daily infrastructure work done on 6 May 2026 for the AIML Platform deployment on a self-hosted Proxmox VE cluster. The goal is to build a production-grade HA Kubernetes cluster (3 control plane + 3 worker nodes) running on Proxmox local-zfs storage with VLAN-segmented networking.

***

## 🗺️ Architecture

```
Proxmox Cluster
├── controller-1  (172.20.10.101)
│   ├── VM 113  — Product-Keycloak
│   ├── VM 115  — Product-Quay       [running]
│   ├── VM 120  — Product-Jenkins    [running]
│   ├── VM 126  — Product-JumpServer [running] ← Bastion / Admin host
│   ├── VM 135  — AIML-Stg-Master-1  [running] ✅ OS installed
│   └── VM 138  — AIML-Stg-Worker-1  [running] ✅ OS installed
│
├── controller-2
│   ├── (Master-2, Master-3, Worker-2, Worker-3 — pending OS install)
│
└── director
```

**K8s Target Topology (Staging):**
```
HAProxy + Keepalived VIP (172.20.10.148)
        ↓
┌──────────────┬──────────────┬──────────────┐
│  Master-1    │  Master-2    │  Master-3    │
│ 172.20.10.141│ 172.20.10.142│ 172.20.10.143│
└──────────────┴──────────────┴──────────────┘
        ↓           ↓           ↓
┌──────────────┬──────────────┬──────────────┐
│  Worker-1    │  Worker-2    │  Worker-3    │
│ 172.20.10.144│ 172.20.10.145│ 172.20.10.146│
└──────────────┴──────────────┴──────────────┘
```

***

## 📦 VM Inventory

| VMID | VM Name | Role | IP | vCPU | RAM | Disk | Host | Status |
|------|---------|------|----|------|-----|------|------|--------|
| 126 | Product-JumpServer | Bastion / Admin | 172.20.10.137 | 2 | 4 GiB | 40 GiB | controller-1 | ✅ Running |
| 135 | AIML-Stg-Master-1 | K8s Control Plane | 172.20.10.141 | 4 | 8 GiB | 40 GiB | controller-1 | ✅ OS Ready |
| 136 | AIML-Stg-Master-2 | K8s Control Plane | 172.20.10.142 | 4 | 8 GiB | 40 GiB | controller-1 | 🔲 Pending |
| 137 | AIML-Stg-Master-3 | K8s Control Plane | 172.20.10.143 | 4 | 8 GiB | 40 GiB | controller-2 | 🔲 Pending |
| 138 | AIML-Stg-Worker-1 | K8s Worker | 172.20.10.144 | 8 | 16 GiB | 40 GiB | controller-1 | ✅ OS Ready |
| 139 | AIML-Stg-Worker-2 | K8s Worker | 172.20.10.145 | 8 | 16 GiB | 40 GiB | controller-2 | 🔲 Pending |
| 140 | AIML-Stg-Worker-3 | K8s Worker | 172.20.10.146 | 8 | 16 GiB | 40 GiB | controller-2 | 🔲 Pending |

**IP Range Allocated:** `172.20.10.135 – 172.20.10.149`  
**VIP (Keepalived, no VM):** `172.20.10.148`

***

## ✅ Completed Today (06 May 2026)

### Infrastructure Setup
- [x] Proxmox VE 9.1.1 cluster confirmed healthy — controller-1, controller-2, director all green
- [x] Existing platform VMs confirmed running: Keycloak (113), Quay (115), Jenkins (120)
- [x] VM 126 designated as **JumpServer / Bastion** — Ubuntu installed, networking verified
- [x] Confirmed internet access from jump host (`ping 8.8.8.8` ✅, `curl ifconfig.me` ✅)

### VM Provisioning
- [x] VM 135 (`AIML-Stg-Master-1`) created on controller-1
  - Ubuntu 26.04 installed via ISO
  - Static IP: `172.20.10.141/24`
  - Interface: `ens18` on `vmbr0` with VLAN tag `10`
  - Internet connectivity verified: `ping -c 2 8.8.8.8` → 0% packet loss, ~1.7ms RTT
- [x] VM 138 (`AIML-Stg-Worker-1`) created on controller-1
  - Ubuntu 26.04 installed via ISO
  - Static IP: `172.20.10.144/24`
  - Interface: `ens18` on `vmbr0` with VLAN tag `10`
  - Internet connectivity verified: `ping -c 2 8.8.8.8` → 0% packet loss, ~1.7ms RTT

### Networking Validation
- [x] VLAN tag `10` confirmed working on `vmbr0` for both Master-1 and Worker-1
- [x] Static IPs confirmed persistent (set via Ubuntu installer)
- [x] Gateway reachability confirmed: `172.20.10.1`
- [x] Root cause identified for previous VM failures (Arpit, Mooaz, Surajeet's test VMs):
  - Missing VLAN tag `10` on network adapter = no gateway access
  - Missing bridge adapter = VM isolated from network

### Runbook & Automation
- [x] Complete 8-script automation bundle created and documented
- [x] Scripts cover: VM creation, sysprep, kubeadm init, master join, worker join, HAProxy+Keepalived, clone, post-clone fixup
- [x] README with full IP table and execution order published

***

## 🔧 Technical Decisions Made

| Decision | Chosen Option | Reason |
|----------|--------------|--------|
| Network bridge | `vmbr0` with VLAN tag `10` | Only config that allows internet access in this Proxmox setup |
| Storage | `local-zfs` | Mandated by infra rules; ZFS gives snapshot/rollback capability |
| OS | Ubuntu 26.04 LTS | Already available as ISO in Proxmox; LTS for stability |
| CNI | Calico v3.28 | BGP-based pod routing; production standard; matches prior cluster work |
| K8s version | v1.31 (stable) | Latest stable from `pkgs.k8s.io` |
| Containerd cgroup | `SystemdCgroup = true` | Required on Ubuntu 24/26 — kubelet crashes without this |
| Control plane endpoint | HAProxy VIP `172.20.10.148` | Allows HA — kubeadm init uses VIP so Master-2/3 join works |
| Pool | `PRODUCT-` | Organisation policy |
| VM naming | `AIML-{VMName}` | Organisation policy |

***

## ⚠️ Issues Encountered & Resolutions

### Issue 1 — Jump Host Polluted with Wrong Packages
**What happened:** `haproxy`, `keepalived`, and `qemu-11.0.0` source compilation were run on the jump host (VM 126).  
**Impact:** Wasted disk space; wrong architecture — jump host is bastion only.  
**Resolution:** Remove all three packages; jump host role locked to SSH tunneling + kubectl access only.  
```bash
apt-get remove --purge -y haproxy keepalived
apt-get autoremove -y
rm -rf /root/qemu-11.0.0 /root/qemu-11.0.0.tar.xz
```

### Issue 2 — Previous VM Network Failures (Arpit / Mooaz / Initial Test VMs)
**What happened:** Three separate team members failed to create working VMs with network access.  
**Root cause:** All three VMs lacked `VLAN tag 10` on `vmbr0`. Without the tag, VMs land on an untagged VLAN with no gateway route.  
**Resolution:** All new VMs created with explicit `tag=10` on `vmbr0`. Confirmed by ping test on Master-1 and Worker-1.

### Issue 3 — `ping -c 8.8.8.8` typo on Master-1
**What happened:** Operator typed `ping -c 8.8.8.8` (no count value) — `8.8.8.8` was interpreted as the count.  
**Resolution:** Corrected to `ping -c 2 8.8.8.8`. Trivial but documented to avoid confusion.

***

## 🔲 Pending (Next Steps)

### Immediate (Next Session)
- [ ] Run sysprep script on Master-1 (`01-vm-sysprep.sh aiml-stg-master-1 172.20.10.141`)
- [ ] Run sysprep script on Worker-1 (`01-vm-sysprep.sh aiml-stg-worker-1 172.20.10.144`)
- [ ] Create + install Ubuntu on VM 136 (Master-2) — controller-1
- [ ] Create + install Ubuntu on VM 137 (Master-3) — controller-2
- [ ] Create + install Ubuntu on VM 139 (Worker-2) — controller-2
- [ ] Create + install Ubuntu on VM 140 (Worker-3) — controller-2

### After All 6 Nodes Are Sysprep'd
- [ ] Initialize K8s control plane on Master-1 (`02-init-master.sh`)
- [ ] Join Master-2 and Master-3 as control plane nodes
- [ ] Join Worker-1, Worker-2, Worker-3
- [ ] Verify `kubectl get nodes` shows all 6 as `Ready`
- [ ] Deploy test workload (nginx pod) to validate cluster

### After Cluster is Healthy
- [ ] Set up HAProxy + Keepalived on dedicated VM (VIP: 172.20.10.148)
- [ ] Configure Calico network policies
- [ ] Install MetalLB for LoadBalancer service support
- [ ] Install NGINX Ingress Controller
- [ ] Connect to existing Quay registry (172.20.10.135)
- [ ] Connect to existing Keycloak for auth (172.20.10.136)

***

## 📁 Runbook Scripts

All scripts are in `aiml-k8s-scripts-bundle.zip`. Execution order:

```
00-proxmox-create-vms.sh      ← Run on Proxmox shell
01-vm-sysprep.sh              ← Run inside each VM after OS install
02-init-master.sh             ← Run on Master-1 only
03-join-master.sh             ← Reference: join Master-2 and Master-3
04-join-worker.sh             ← Reference: join workers
05-haproxy-keepalived.sh      ← Run on dedicated HAProxy VM
06-proxmox-clone-from-master.sh  ← Alternative: clone instead of fresh install
07-post-clone-fixup.sh        ← Run inside cloned VM to fix machine-id + IP
```

***

## 🔒 Infra Rules (Never Violate)

```
Network  : vmbr0 + VLAN tag 10   ← without this, no internet
Storage  : local-zfs only
Pool     : PRODUCT-
Prefix   : AIML-{VMName}
Max Disk : 40 GiB (testing phase)
Swap     : MUST be OFF (K8s requirement)
Containerd: SystemdCgroup = true (Ubuntu 24/26 requirement)
Machine-ID: MUST be unique on every cloned VM
```

***

## 📊 Progress Summary

| Category | Total | Done | Remaining |
|----------|-------|------|-----------|
| VMs with OS installed | 6 needed | 2 | 4 |
| Nodes sysprep'd | 6 needed | 0 | 6 |
| K8s cluster initialized | 1 needed | 0 | 1 |
| Nodes joined to cluster | 5 needed | 0 | 5 |
| Post-cluster services | 5 needed | 0 | 5 |

***

*Last updated: 06 May 2026 23:18 IST*

---



# 🚀 AIML Platform — Kubernetes + Rancher on Proxmox VE | Progress Log

> **Project:** AIML Platform — Staging K8s Cluster with Rancher UI  
> **Cluster Name:** `host360-staging`  
> **Environment:** Proxmox VE 9.1.1 (controller-1 + controller-2 + director)  
> **Date:** 07 May 2026  
> **Author:** Surajeet Banerjee  
> **Status:** 🟡 In Progress — Master-1 init done, CNI pending, workers pending

---

## 📐 Final Architecture Decision

After evaluating two approaches (kubeadm vs RKE2), the team is proceeding with:

```text
┌─────────────────────────────────────────────────────────────────┐
│  AIML-Stg-JumpServer (172.20.10.137) — Bastion / kubectl host  │
└───────────────────────────┬─────────────────────────────────────┘
                            │ manages
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
  AIML-Stg-Master-1  AIML-Stg-Master-2  AIML-Stg-Master-3
  172.20.10.141      172.20.10.142      172.20.10.143
  (kubeadm init)     (join CP)          (join CP)
          └─────────────────┼─────────────────┘
                            │
          ┌─────────────────┼─────────────────┐
          ▼                 ▼                 ▼
  AIML-Stg-Worker-1  AIML-Stg-Worker-2  AIML-Stg-Worker-3
  172.20.10.144      172.20.10.145      172.20.10.146

  ┌──────────────────────────────────────────────────────────┐
  │  Rancher UI — installed via Helm AFTER cluster is Ready │
  │  Access: https://rancher.172.20.10.141.nip.io           │
  └──────────────────────────────────────────────────────────┘
```

---

## 📦 VM Inventory

| VMID | VM Name | Role | IP | vCPU | RAM | Disk | Host | Status |
|------|---------|------|----|------|-----|------|------|--------|
| 126 | Product-JumpServer | Bastion / kubectl | 172.20.10.137 | 2 | 4 GiB | 40 GiB | controller-1 | ✅ Running |
| 135 | AIML-Stg-Master-1 | K8s Control Plane | 172.20.10.141 | 4 | 8 GiB | 40 GiB | controller-1 | ✅ kubeadm init done |
| 136 | AIML-Stg-Master-2 | K8s Control Plane | 172.20.10.142 | 4 | 8 GiB | 40 GiB | controller-1 | 🔲 Pending |
| 137 | AIML-Stg-Master-3 | K8s Control Plane | 172.20.10.143 | 4 | 8 GiB | 40 GiB | controller-2 | 🔲 Pending |
| 138 | AIML-Stg-Worker-1 | K8s Worker | 172.20.10.144 | 8 | 16 GiB | 40 GiB | controller-1 | ✅ OS Ready |
| 139 | AIML-Stg-Worker-2 | K8s Worker | 172.20.10.145 | 8 | 16 GiB | 40 GiB | controller-2 | 🔲 Pending |
| 140 | AIML-Stg-Worker-3 | K8s Worker | 172.20.10.146 | 8 | 16 GiB | 40 GiB | controller-2 | 🔲 Pending |

**IP Range:** `172.20.10.135 – 172.20.10.149`  
**Network:** `vmbr0` | VLAN tag `10` | Gateway `172.20.10.1`  
**Storage:** `local-zfs` on all VMs  
**Pool:** `PRODUCT-`

---

## ✅ Completed — 07 May 2026

### Proxmox Infrastructure
- [x] All Proxmox nodes healthy — controller-1, controller-2, director confirmed green
- [x] VLAN `10` confirmed as required tag on `vmbr0` for internet access — root cause of all previous failures
- [x] VM 126 locked as JumpServer / Bastion role only
- [x] VM 138 (Worker-1) Ubuntu installed, static IP `172.20.10.144`, internet verified

### Master-1 — kubeadm Init (K8s v1.35.0)

**sysprep checklist completed:**
- [x] Swap disabled (`swapoff -a` + fstab commented)
- [x] `overlay` + `br_netfilter` kernel modules loaded
- [x] sysctl params applied (`bridge-nf-call-iptables`, `ip_forward`)
- [x] `containerd` installed + `SystemdCgroup = true` set
- [x] K8s v1.35 packages installed: `kubelet`, `kubeadm`, `kubectl`
- [x] `qemu-guest-agent` installed and running

**kubeadm init:**
- [x] `kubeadm-config.yaml` written with correct cluster config
- [x] `kubeadm init` completed successfully
- [x] All certs generated (ca, apiserver, etcd, front-proxy, sa)
- [x] `etcd`, `kube-apiserver`, `kube-controller-manager`, `kube-scheduler` all healthy
- [x] `CoreDNS` + `kube-proxy` addons applied
- [x] `KUBECONFIG=/etc/kubernetes/admin.conf` exported

**kubeadm-config.yaml used:**
```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
clusterName: host360-staging
kubernetesVersion: v1.35.0
controlPlaneEndpoint: "172.20.10.141:6443"
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
***
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 172.20.10.141
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
```

**kubeadm init output (key lines):**
```text
[certs] Generating "ca" certificate and key
[certs] apiserver serving cert is signed for DNS names [aiml-stg-master-1 kubernetes ...] and IPs [10.96.0.1 172.20.10.141]
[kubelet-check] The kubelet is healthy after 1.001255495s
[control-plane-check] kube-apiserver is healthy after 4.001976792s
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy
Your Kubernetes control-plane has initialized successfully!
```

**Bootstrap token generated:**
```text
Token:    drz8ek.5si2geombz458uyd
CA Hash:  sha256:41d058990f6af02d40e326cac6b36fa58562deab9642619cb60750948ba3c069
```

> ⚠️ Token expires in 24 hours. Regenerate if expired:
```bash
kubeadm token create --print-join-command
```

---

## ⚠️ Issues Found & Resolved

### Issue 1 — `kubeadm-config.yaml` was appended multiple times
**What happened:** `cat >>` was used instead of `cat >`, first with wrong IP `172.237.44.177`, later with correct IP.  
**Impact:** File became dirty/corrupt, but kubeadm used the last valid block so init still succeeded.  
**Resolution:** Rewrote config cleanly using `cat >` with correct `v1beta4` API.

### Issue 2 — `--upload-certs` skipped
**What happened:** kubeadm init ran without `--upload-certs`.  
**Impact:** Master-2 and Master-3 cannot join as control plane nodes until cert key is uploaded.  
**Resolution:**
```bash
kubeadm init phase upload-certs --upload-certs 2>&1 | tee /root/cert-key.txt
```

> ⚠️ Cert key expires in 2 hours. Re-run before Master-2/3 join if needed.

### Issue 3 — Rogue Docker + Rancher container on Master-1
**What happened:** `docker run rancher/rancher` was executed directly on the kubeadm control-plane node.  
**Impact:** Trash architecture. Docker on master node creates runtime confusion and Rancher should not be run like this for HA setup.  
**Resolution:**
```bash
docker stop priceless_grothendieck && docker rm priceless_grothendieck
apt-get remove --purge -y docker.io && apt-get autoremove -y
```

### Issue 4 — QEMU compiled from source on JumpServer
**What happened:** QEMU source build was attempted on jump host.  
**Impact:** Wrong host, wrong objective, wasted time. `qm` exists only on Proxmox nodes, not inside guest VMs.  
**Resolution:** Removed unnecessary QEMU source/build activity. JumpServer restricted to bastion + kubectl.

### Issue 5 — Previous team VM failures
**Root cause:** Missing VLAN tag `10` on `vmbr0`.  
**Impact:** No gateway access, no internet, broken provisioning.  
**Resolution:** All new VMs now explicitly use `vmbr0` + VLAN `10`.

---

## 🔑 Key Technical Decisions

| Decision | Choice | Reason |
|----------|--------|--------|
| K8s bootstrap tool | `kubeadm` | Already initialized on Master-1; practical to continue |
| Kubernetes version | `v1.35.0` | Latest stable used in current setup |
| Container runtime | `containerd` | Docker is obsolete for modern K8s node runtime |
| Cgroup driver | `SystemdCgroup = true` | Mandatory for Ubuntu with systemd cgroups |
| CNI | `Flannel` | Matches configured `10.244.0.0/16` pod subnet |
| Rancher install method | Helm on top of cluster | Correct HA-capable pattern |
| API endpoint | `172.20.10.141:6443` | Temporary direct endpoint before VIP |
| Future VIP | `172.20.10.148` | Planned HAProxy + Keepalived endpoint |
| Network | `vmbr0` + VLAN `10` | Mandatory in this environment |
| Storage | `local-zfs` | Infra rule + snapshot-friendly |

---

## 📋 Rancher Clarification

**Rancher does NOT need Docker on the nodes.**

```text
OLD (wrong for modern K8s):
kubelet → dockershim → Docker → containerd → runc

CURRENT (correct):
kubelet → CRI → containerd → runc
```

**Rancher runs as pods after Helm install:**
```bash
kubectl get pods -n cattle-system
# rancher-xxxx Running
```

**Rancher install command (after cluster is healthy):**
```bash
helm install rancher rancher-stable/rancher \
  --namespace cattle-system --create-namespace \
  --set hostname=rancher.172.20.10.141.nip.io \
  --set bootstrapPassword=ChangeMeStrongPassword \
  --set replicas=3
```

---

## 🔲 Pending Next Steps

### Immediate
- [ ] Run `kubectl get nodes` and confirm Master-1 is currently `NotReady` before CNI
- [ ] Install Flannel CNI
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
- [ ] Confirm Master-1 flips to `Ready`
- [ ] Run upload-certs phase
- [ ] Save `worker-join.sh` and `master-join.sh`

### VM Provisioning
- [ ] Create VM 136 (Master-2) on controller-1
- [ ] Create VM 137 (Master-3) on controller-2
- [ ] Create VM 139 (Worker-2) on controller-2
- [ ] Create VM 140 (Worker-3) on controller-2

### Cluster Completion
- [ ] Sysprep Master-2 and Master-3
- [ ] Join Master-2 and Master-3 as control plane nodes
- [ ] Sysprep Worker-1, Worker-2, Worker-3
- [ ] Join all workers
- [ ] Verify all 6 nodes are `Ready`

### After Cluster Is Healthy
- [ ] Install HAProxy + Keepalived for VIP `172.20.10.148`
- [ ] Install cert-manager
- [ ] Install Rancher via Helm
- [ ] Integrate Quay registry
- [ ] Integrate Keycloak auth

---

## 🔒 Infra Rules — Never Violate

```text
Network       : vmbr0 + VLAN tag 10
Storage       : local-zfs only
Resource Pool : PRODUCT-
VM Prefix     : AIML-{VMName}
Max Disk      : 40 GiB
Swap          : MUST be OFF
Containerd    : SystemdCgroup = true
machine-id    : MUST be unique on cloned VMs
SSH host keys : MUST be regenerated on cloned VMs
```

---

## 📊 Progress Summary

| Category | Target | Done | Remaining |
|----------|--------|------|-----------|
| VMs with OS installed | 6 | 2 | 4 |
| Nodes sysprep'd | 6 | 1 | 5 |
| K8s cluster initialized | 1 | 1 | 0 |
| CNI installed | 1 | 0 | 1 |
| Nodes joined | 5 | 0 | 5 |
| Rancher installed | 1 | 0 | 1 |

---

## 📁 Reference Scripts

| Script | Purpose | Run Where |
|--------|---------|-----------|
| `00-proxmox-create-vms.sh` | Create all VMs | Proxmox shell |
| `01-vm-sysprep.sh` | Node prep | Inside each VM |
| `02-init-master.sh` | kubeadm init + CNI + join scripts | Master-1 |
| `03-join-master.sh` | Master join reference | Master-2/3 |
| `04-join-worker.sh` | Worker join reference | Workers |
| `05-haproxy-keepalived.sh` | HA VIP setup | HAProxy VM |
| `06-proxmox-clone-from-master.sh` | Clone masters | Proxmox shell |
| `07-post-clone-fixup.sh` | Reset identity after clone | Cloned VM |

---

*Last updated: 07 May 2026 22:03 IST*  
*Next milestone: all 6 nodes Ready + Rancher UI accessible*

---

<!-- Revision Required ... -->

## [2026-05-08] - Phase 4: High Availability & Observability Implementation

### 🚀 Objectives Achieved
- **HA Cluster Provisioning:** Successfully expanded the RKE2 bare-metal cluster to a production-grade High Availability (HA) architecture.
- **Observability Bootstrapping:** Deployed the Rancher Monitoring suite (Prometheus, Grafana, Alertmanager) with persistent storage.
- **Data Retention Hardening:** Secured monitoring data against accidental namespace or helm chart deletion by enforcing strict PV retention policies.

### 🏗️ Infrastructure State
- **Total Nodes:** 6 (Active & Ready)
  - 3x Control Plane / etcd nodes (`product-cmp-prod-cp-1`, `cp-2`, `cp-3`)
  - 3x Worker nodes (`product-cmp-prod-wrk-1`, `wrk-2`, `wrk-3`)
- **Storage:** `longhorn` CSI provisioner deployed and verified as the active storage class.

### 👁️ Observability Stack (`cattle-monitoring-system`)
- **Prometheus:** Active (`3/3` ready). PVC Bound (`10Gi` via Longhorn).
- **Grafana:** Active (`3/3` ready). PVC Bound (`5Gi` via Longhorn).
- **Node Exporters:** DaemonSet deployed successfully across all 6 nodes for hardware-level metric scraping.
- **Access:** Verified UI access via `kubectl port-forward` to the Grafana service on port `3000:80`.

### 🛡️ Security & Hardening Applied
- **OS Hardening:** Swap disabled and SSH root login blocked across all 6 nodes via automated deployment script.
- **PV Reclaim Policy Patch:** By default, dynamic PVs provisioned via Helm are set to `Delete`. Successfully patched the underlying Persistent Volumes for Prometheus (`pvc-7ada84f7...`) and Grafana (`pvc-92a9fe52...`) to `Retain`.
  - *Impact:* If the monitoring helm chart is uninstalled or crashes, the Longhorn block storage volumes will survive, preventing catastrophic historical metric data loss.

### 🚧 Next Steps
- Deploy Loki & Promtail (Logging Stack) to the isolated `logging` namespace.
- Configure an Ingress Route (Traefik) to expose Grafana securely, removing the dependency on local port-forwarding.

***

This README update reflects the technical grind of Agenda Item 1 and the architectural setup of Agenda Items 2 & 3. It captures the transition from a broken Loki installation to a fully functional observability pipeline and a high-availability network entry point.

---

## Daily Progress Update: May 11, 2026

### Log Aggregation & Observability (Agenda Item 1)

* **Loki Stack Deployment:** Successfully deployed Grafana Loki in `SingleBinary` mode using the official Helm chart. Resolved a critical installation error where conflicting deployment targets were active simultaneously.


* **Storage Remediation:** Identified and resolved a Longhorn "insufficient storage" failure. Discovered that 10Gi volumes were being blocked due to root disk pressure (51% used) on worker nodes. Successfully implemented a tactical bypass by resizing the volume to 2Gi to fit current disk headroom.


* **Gateway Stabilization:** Fixed `loki-gateway` CrashLoopBackOff by correcting the Nginx DNS resolver. Transitioned from the default `kube-dns` to the RKE2-specific `rke2-coredns-rke2-coredns.kube-system.svc.cluster.local` address.


* **Agent Integration:** Deployed **Promtail** as a DaemonSet across all 6 nodes to scrape container logs from `/var/log/containers/`.


* **End-to-End Validation:** Verified the pipeline by deploying a demo Nginx application and a traffic generator. Confirmed live log streaming in Grafana using LogQL: `{namespace="test-apps", app="demo-nginx"}`.



### Network Infrastructure & High Availability (Agenda Items 2 & 3)

* **HAProxy & Keepalived VIP:** Initiated the setup of a dual-node Load Balancer stack to act as the primary entry point for the cluster.


* **VIP Configuration:** Configured Keepalived with VRRP to manage a floating Virtual IP (VIP: `172.20.4.100`) for Master-Backup failover.


* **Traffic Routing Logic:**
* **Port 6443:** Layer 4 TCP balancing across 3 RKE2 Control Plane nodes.


* **Ports 80/443:** Routing external HTTP/S traffic to the internal Traefik Ingress Controller running on Worker nodes.




* **CSI Driver Verification:** Confirmed `driver.longhorn.io` is correctly registered and serving as the primary storage backend.



### Technical Documentation & Sourcing

* Referenced official documentation from **Grafana (Loki v7.x)**, **Longhorn CSI**, and **RKE2** to ensure implementation follows industry standards for production environments.


---

**Next Objectives:**

1. Verify HAProxy failover by simulating node termination.
2. Expand node root disks or add dedicated data disks for Longhorn to support larger logging volumes.
3. Set up ResourceQuotas for the `logging` namespace to prevent memory exhaustion by memcached.

***
***


### Date: 2026-05-12
**Focus:** Observability Validation & Massive Storage Architecture Pivot

#### 🏆 Milestones Achieved
* **Bypassed Corporate DNS Lockdown:** Successfully routed traffic to the HAProxy VIP using Magic DNS (`sslip.io`), allowing Traefik Ingress to route to Grafana and Prometheus without modifying the locked-down Windows `hosts` file.
* **Observability Smoke Tests:** Verified the monitoring stack network layer. Confirmed `node-exporter` telemetry is actively populating the pre-built Kubernetes/Compute dashboards in Grafana.
* **Architectural Storage Pivot:** Recognized redundant overhead and abandoned Longhorn (Software-Defined Storage) in favor of the hypervisor's native physical Ceph/ZFS cluster.
* **The Great Purge:** Completely eradicated Longhorn from the cluster. Manually stripped finalizers from stuck `Terminating` PVCs/PVs and deleted orphaned mutating/validating webhooks to unlock the cluster's API.
* **Proxmox CSI Implementation:** * Built and deployed the `proxmox-csi-plugin` via Helm CLI.
  * Corrected API authentication boot-loops by explicitly mapping the `config.yaml` secret to the Proxmox API token.
  * Installed missing host-level dependencies (`open-iscsi`, `sg3-utils`, `ceph-common`) directly on the Ubuntu worker VMs, enabling the OS to physically mount the hypervisor's block devices.
* **Production StorageClass Configuration:** Deployed a new CSI-backed StorageClass (`proxmox-local-zfs`). Crucially, enforced the `Retain` Reclaim Policy to ensure Prometheus, Grafana, and Loki databases survive accidental pod or namespace deletions.
* **Final Binding:** Successfully provisioned and bound persistent volumes for the entire observability stack via the Rancher UI using the new hypervisor-backed storage.

#### ⚠️ Lessons Learned (The Hard Way)
* **Strict Secret Mapping:** A container hardcoded to look for a specific file (`config.yaml`) will continuously crash loop if the Kubernetes secret injects it under a different key name, even if the payload data is 100% correct.
* **Kubernetes Finalizers:** Deleting a storage controller (like Longhorn) before deleting its provisioned PVCs will leave those resources trapped in a `Terminating` state forever. Finalizers must be manually patched to `null` to force the purge.
* **Reclaim Policies:** Defaulting a StorageClass to `Delete` is a catastrophic risk for databases. Production persistent data (like logs and metrics) must always utilize the `Retain` policy.
* **Host vs. Container Dependencies:** A CSI node agent container cannot magically mount a storage protocol (like Ceph or iSCSI) if the underlying hypervisor OS (Ubuntu worker node) lacks the necessary client binaries.
