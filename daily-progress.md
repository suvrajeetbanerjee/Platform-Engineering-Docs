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

***
***



# 🚀 Daily Progress - Proxmox CSI Driver Debugging & Rancher Monitoring Persistence

## 📅 Date
13 May 2026

---

# 🎯 Objective

Configured and debugged dynamic persistent storage provisioning for a Rancher-managed RKE2 Kubernetes cluster using the Proxmox CSI Plugin.

Goal was to enable persistent volumes for:
- Prometheus
- Grafana
- Loki
- Rancher Monitoring Stack

using Proxmox-backed storage inside Kubernetes.

---

# 🏗️ Infrastructure Context

## Kubernetes Platform
- Rancher Managed RKE2 Cluster
- HA Control Plane
- Dedicated Worker Nodes

## Storage Backend
- Proxmox VE
- ZFS-backed local storage
- Storage Pool:
  
```text
local-zfs


## CSI Driver

* Proxmox CSI Plugin
* Provisioner:

```text
csi.proxmox.sinextra.dev
```

---

# 🔥 Initial Problem Observed

Persistent Volume Claims remained stuck in:

```text
Pending
```

Affected workloads:

* Prometheus
* Grafana
* Test PVCs
* Monitoring stack components

PVCs were unable to dynamically provision volumes.

---

# 🔍 Investigation & Root Cause Analysis

## 1️⃣ Verified PVC Status

Used:

```bash
kubectl get pvc -A
```

Observed all PVCs in pending state.

---

## 2️⃣ Inspected PVC Events

Used:

```bash
kubectl describe pvc <pvc-name> -n <namespace>
```

Initial error discovered:

```text
dial tcp 172.20.10.10:8006: connect: no route to host
```

---

# 🌐 Networking Issue Identified

## Problem

CSI plugin was attempting to communicate with incorrect Proxmox API endpoint:

```text
172.20.10.10
```

while actual reachable Proxmox endpoint was:

```text
172.20.10.101
```

---

## Validation Performed

Successful connectivity test:

```bash
curl -k https://172.20.10.101:8006/api2/json
```

Confirmed:

* Proxmox API reachable
* Kubernetes node networking functional

---

# ⚙️ CSI Secret Reconfiguration

## Extracted Existing CSI Configuration

```bash
kubectl get secret proxmox-csi-plugin -n csi-proxmox -o jsonpath='{.data.config\.yaml}' | base64 -d
```

Observed incorrect API URL.

---

## Updated CSI Config

Modified:

```yaml
url: https://172.20.10.101:8006/api2/json
```

---

## Recreated Kubernetes Secret

```bash
kubectl delete secret proxmox-csi-plugin -n csi-proxmox
```

```bash
kubectl create secret generic proxmox-csi-plugin \
  --from-file=config.yaml \
  -n csi-proxmox
```

---

# 🔄 CSI Driver Restart

Restarted:

* CSI Controller Deployment
* CSI Node DaemonSet

Commands used:

```bash
kubectl rollout restart deployment proxmox-csi-plugin-controller -n csi-proxmox
```

```bash
kubectl rollout restart daemonset proxmox-csi-plugin-node -n csi-proxmox
```

---

# 📊 Post-Fix Behavior Change

## Previous Error

```text
connect: no route to host
```

## New Error

```text
failed to get proxmox storage config: not found
```

This confirmed:

* networking issue resolved
* CSI successfully reaching Proxmox API

---

# 🧠 Deep-Dive Into CSI Provisioning Flow

Learned the internal provisioning sequence of CSI-based storage orchestration:

```text
Kubernetes PVC
    ↓
CSI Controller
    ↓
Proxmox API
    ↓
Cluster Storage Discovery
    ↓
Topology Validation
    ↓
Volume Creation
    ↓
Disk Attachment
```

---

# 🏷️ StorageClass Validation

Verified Kubernetes StorageClass configuration:

```yaml
parameters:
  storage: local-zfs
  fstype: ext4
```

Validated against actual Proxmox storage backend.

---

# 🌍 Kubernetes Topology Validation

Verified topology labels on worker nodes:

```text
topology.kubernetes.io/region=cmp-prod-v1
topology.kubernetes.io/zone=controller-1
```

Corrected malformed topology label on one worker node.

---

# 🔥 Major Discovery — Proxmox API Permission Issue

Performed direct API tests against Proxmox.

## Command Used

```bash
curl -k \
-H 'Authorization: PVEAPIToken=sameer@pve!proxmox-csi=<TOKEN>' \
'https://172.20.10.101:8006/api2/json/cluster/resources?type=storage'
```

---

## Critical Result

```json
{"data":[]}
```

This revealed:

* authentication successful
* networking successful
* API accessible
* BUT API token had zero visibility into storage resources

---

# 🚨 Root Cause Identified

Primary blocker isolated to:

## Proxmox RBAC / API Token Permissions

CSI driver could not discover:

```text
local-zfs
```

because the API token lacked sufficient permissions for:

* cluster storage visibility
* datastore auditing
* storage resource enumeration

---

# 🧠 Key Concepts Learned

## Kubernetes Storage Concepts

* PVC Lifecycle
* CSI Architecture
* Dynamic Provisioning
* StorageClasses
* Volume Binding Modes
* Persistent Volume orchestration

---

## Infrastructure & Networking Concepts

* Proxmox API communication
* CSI Controller workflows
* Cluster topology awareness
* Storage backend mapping
* API-driven infrastructure orchestration

---

## Debugging Concepts

* Event-driven troubleshooting
* CSI log analysis
* Storage topology tracing
* Kubernetes-Proxmox integration debugging
* Infrastructure RBAC validation

---

# 🛠️ Core Commands Used

## PVC Debugging

```bash
kubectl get pvc -A
```

```bash
kubectl describe pvc <pvc-name> -n <namespace>
```

---

## CSI Debugging

```bash
kubectl logs -n csi-proxmox deploy/proxmox-csi-plugin-controller -f
```

---

## StorageClass Inspection

```bash
kubectl get sc proxmox-local-zfs -o yaml
```

---

## Node Topology Verification

```bash
kubectl get nodes --show-labels
```

---

## Proxmox API Validation

```bash
curl -k \
-H 'Authorization: PVEAPIToken=<TOKEN>' \
'https://172.20.10.101:8006/api2/json/cluster/resources?type=storage'
```

---

# 📌 Current Status

## Completed

* CSI deployment functional
* Kubernetes connectivity functional
* Proxmox API connectivity functional
* StorageClass configured
* Topology labels configured
* Root cause isolated

## Pending

* Proxmox API token RBAC correction
* Successful dynamic volume provisioning
* Monitoring stack PVC binding
* Persistent monitoring workloads

---

# 🚀 Key Takeaway

This debugging session demonstrated that modern platform engineering failures are rarely caused by a single component.

A functioning CSI architecture depends on:

* networking
* storage topology
* RBAC
* API visibility
* orchestration consistency
* infrastructure identity mapping

A single misconfigured API token permission was enough to break the entire Kubernetes storage provisioning workflow even though all other components appeared healthy.

```
```


***
***


# 🚀 Daily Progress Report — Rancher Monitoring + Proxmox CSI Storage Debugging

> Deep dive into Kubernetes CSI provisioning, Proxmox API integration, StatefulSet persistence, PVC lifecycle debugging, and monitoring stack recovery.

Reference Notes: :contentReference[oaicite:0]{index=0}

---

# 📅 Date

14-05-2026

---

# 🎯 Objective

Enable persistent storage for:

- Prometheus
- Grafana
- Loki
- Rancher Monitoring Stack

inside a Rancher-managed RKE2 Kubernetes cluster using the Proxmox CSI Plugin.

---

# 🏗️ Infrastructure Overview

## Kubernetes Platform
- Rancher Managed RKE2 Cluster
- HA Control Plane
- Dedicated Worker Nodes

## Storage Backend
- Proxmox VE
- ZFS/LVM-backed storage
- CSI Driver:

```text
csi.proxmox.sinextra.dev
```

## StorageClasses Used

### Initial StorageClass

```text
proxmox-local-zfs
```

### Final Working StorageClass

```text
proxmox-primera
```

---

# 🔥 Initial Problem

All monitoring workloads failed to initialize because Persistent Volume Claims remained stuck in:

```text
Pending
```

Affected workloads:
- Prometheus
- Grafana
- Loki
- Rancher Monitoring

---

# 🔍 PVC Investigation

## Commands Used

```bash
kubectl get pvc -A
```

```bash
kubectl describe pvc <pvc-name> -n <namespace>
```

---

# 🚨 Root Cause #1 — Incorrect Proxmox API Endpoint

CSI logs revealed:

```text
dial tcp 172.20.10.10:8006: connect: no route to host
```

---

# 🌐 Network Validation

Confirmed actual reachable Proxmox API endpoint:

```bash
curl -k https://172.20.10.101:8006/api2/json
```

---

# ⚙️ CSI Secret Reconfiguration

## Extracted Existing Config

```bash
kubectl get secret proxmox-csi-plugin -n csi-proxmox \
-o jsonpath='{.data.config\.yaml}' | base64 -d
```

---

## Corrected API Endpoint

Updated:

```yaml
url: https://172.20.10.101:8006/api2/json
```

---

## Recreated Kubernetes Secret

```bash
kubectl delete secret proxmox-csi-plugin -n csi-proxmox
```

```bash
kubectl create secret generic proxmox-csi-plugin \
  --from-file=config.yaml \
  -n csi-proxmox
```

---

# 🔄 CSI Driver Restart

Restarted:
- CSI Controller
- CSI Node DaemonSet

```bash
kubectl rollout restart deployment proxmox-csi-plugin-controller -n csi-proxmox
```

```bash
kubectl rollout restart daemonset proxmox-csi-plugin-node -n csi-proxmox
```

---

# 📊 Post-Network Fix Result

Old Error:

```text
no route to host
```

New Error:

```text
failed to get proxmox storage config: not found
```

This confirmed:
- Kubernetes networking fixed
- CSI reaching Proxmox successfully
- Next blocker moved to storage/resource visibility

---

# 🔥 Root Cause #2 — Proxmox API Permission Failure

Direct API testing showed:

```bash
curl -k \
-H 'Authorization: PVEAPIToken=sameer@pve!proxmox-csi=<TOKEN>' \
'https://172.20.10.101:8006/api2/json/cluster/resources?type=storage'
```

Response:

```json
{"data":[]}
```

Meaning:
- API authentication succeeded
- BUT token had zero visibility into storage resources

---

# 🛠️ Proxmox RBAC Troubleshooting

Identified missing permissions:
- Datastore.Audit
- Datastore.AllocateSpace
- Sys.Audit
- Storage visibility propagation

Required fixes:
- Administrator role assignment
- Propagate checkbox enabled

---

# 🧠 Kubernetes Storage Concepts Explored

## CSI Workflow

```text
PVC
 ↓
CSI Controller
 ↓
Proxmox API
 ↓
Storage Discovery
 ↓
Volume Creation
 ↓
Volume Attachment
 ↓
Mount into Pod
```

---

# 🔥 StorageClass Evolution

## Original StorageClass

```text
proxmox-local-zfs
```

Eventually replaced with:

```text
proxmox-primera
```

---

# 📦 Final Working StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: proxmox-primera
provisioner: csi.proxmox.sinextra.dev
parameters:
  storage: primera-lvm
  cache: none
  ssd: "true"
  csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

---

# 🧪 PVC Validation Testing

Created test PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-proxmox-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: proxmox-primera
  resources:
    requests:
      storage: 2Gi
```

---

# 🚨 Root Cause #3 — StorageClass Mismatch

Discovered monitoring workloads were still requesting:

```text
proxmox-local-zfs
```

while cluster only contained:

```text
proxmox-primera
```

This caused:

```text
storageclass.storage.k8s.io "proxmox-local-zfs" not found
```

---

# 🔄 Monitoring Stack Reconfiguration

Updated Rancher Monitoring Helm values:

```yaml
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: proxmox-primera

grafana:
  persistence:
    enabled: true
    storageClassName: proxmox-primera
```

---

# 🚀 Helm Upgrade Performed

```bash
helm upgrade rancher-monitoring rancher-charts/rancher-monitoring \
  -n cattle-monitoring-system \
  --reuse-values \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.storageClassName=proxmox-primera \
  --set grafana.persistence.size=20Gi
```

---

# ✅ Successful PVC Provisioning

Final successful state:

```text
prometheus-rancher-monitoring-prometheus-db -> Bound
rancher-monitoring-grafana -> Bound
```

This confirmed:
- CSI provisioning operational
- Proxmox disk creation operational
- Kubernetes storage orchestration operational

---

# 🔥 StatefulSet Recovery Work

Force restarted stuck Prometheus pod:

```bash
kubectl delete pod prometheus-rancher-monitoring-prometheus-0 \
-n cattle-monitoring-system \
--force --grace-period=0
```

---

# 🚨 Final Blocking Issue Identified

Prometheus pod remained stuck in:

```text
Init:0/1
```

---

# 🔍 Deep Pod Investigation

Used:

```bash
kubectl describe pod prometheus-rancher-monitoring-prometheus-0 \
-n cattle-monitoring-system
```

Critical discovery:

```text
FailedAttachVolume:
rpc error: code = Internal desc = instance not found
```

---

# 🧠 Key Technical Learnings

## Kubernetes Concepts
- PVC lifecycle
- StatefulSets
- Init containers
- Volume attachments
- CSI architecture
- VolumeBindingMode
- Reclaim policies
- Pod scheduling

---

## Proxmox Concepts
- API token RBAC
- Storage visibility
- CSI integration
- Virtual disk orchestration
- VM disk attachment workflows

---

## Troubleshooting Concepts
- Event-driven debugging
- CSI log tracing
- Stateful workload recovery
- Persistent storage migration
- Zombie PVC cleanup
- Finalizer removal
- StorageClass migration strategy

---

# 📌 Current Cluster Status

| Component | Status |
|---|---|
| Proxmox API Connectivity | ✅ Working |
| CSI Controller | ✅ Working |
| CSI Node Plugin | ✅ Working |
| Dynamic PVC Provisioning | ✅ Working |
| Grafana PVC | ✅ Bound |
| Prometheus PVC | ✅ Bound |
| Grafana Pod | ⚠️ Pending Verification |
| Prometheus Pod | ❌ FailedAttachVolume |
| Loki Persistence | ⚠️ Pending |
| Volume Attachment Layer | ❌ Current Blocker |

---

# 🔥 Major Takeaway

This debugging session demonstrated how Kubernetes persistent storage failures can originate from multiple infrastructure layers simultaneously:

- Networking
- API identity
- RBAC permissions
- CSI orchestration
- Storage topology
- Stateful workload lifecycle
- Volume attachment mechanics

A cluster may successfully provision Persistent Volumes while still failing at the final attachment stage between Kubernetes worker nodes and Proxmox virtual machine infrastructure.

---

# 🚀 Skills Strengthened

- Kubernetes Storage Internals
- Proxmox CSI Architecture
- Rancher Monitoring Stack Management
- StatefulSet Recovery
- CSI Driver Debugging
- PVC/PV Lifecycle Analysis
- Proxmox API Debugging
- Infrastructure Root Cause Analysis
- Production Monitoring Persistence Design

***
***

# Platform Engineering Daily Progress - May 15, 2026

## Objective: Standardizing Observability & Persistent Storage Handshake

Today was a high-stakes troubleshooting day that resulted in a "bulletproof" observability foundation for the `cmp-prod-v1` cluster. The primary focus was resolving persistent storage deadlocks and wiring the Monitoring (Prometheus/Grafana) and Logging (Loki) stacks correctly after a complete infrastructure "nuke and rebuild."

---

## 🛠 Tasks Executed

### 1. Infrastructure Nuke & Namespace Purification
* **The Problem:** Stale Helm releases and "zombie" pods were holding onto orphaned Proxmox disk sessions, causing session errors in the Rancher UI.
* **Action:**
    * Uninstalled `rancher-monitoring` and `loki-stack` via Helm.
    * Purged all Persistent Volume Claims (PVCs) in `cattle-monitoring-system` and `logging` namespaces.
    * Verified the **Proxmox CSI Driver** successfully deleted physical ZFS datasets on the `primera-lvm` backend.
    * Deleted and recreated namespaces to ensure a clean metadata slate.

### 2. Standardized Storage Handshake (Proxmox + ZFS)
* **The Achievement:** Achieved a **2-second Bound time** for production volumes.
* **Provisioned Resources:**
    * **Prometheus:** 50Gi Persistent Volume via `proxmox-primera` StorageClass.
    * **Grafana:** 20Gi Persistent Volume via `proxmox-primera` StorageClass.
    * **Loki:** 50Gi Persistent Volume via `proxmox-primera` StorageClass.
* **Key Learning:** Validated that lower-case node naming convention in Proxmox is mandatory for CSI volume attachment reliability.

### 3. Monitoring Stack Reconstruction (Rancher UI Flow)
* **Action:** Reinstalled Rancher Monitoring strictly through the Rancher Apps UI to restore the **Rancher Auth Proxy** session.
* **Result:** Successfully restored the "Monitoring" shortcut link in Rancher Dashboard, resolving the `failed to find Session for client stv-cluster-api` error.

### 4. Centralized Logging (Loki-Stack) Integration
* **Agent Deployment:** Deployed `promtail` as a DaemonSet across all worker nodes to tail container logs.
* **Data Source Wiring:** Manually wired Loki to Grafana using internal K8s DNS: `http://loki.logging.svc.cluster.local:3100`.
* **Chaos Testing:** Deployed a `log-generator` application spewing randomized INFO/WARN/ERROR logs to verify real-time ingestion.

---

## 📈 Visual Verification
* **Grafana URL:** Managed through internal ClusterIP (Safe) and accessible via Rancher Proxy.
* **Dashboards:** Successfully imported **Node Exporter Full (1860)** and **Loki Kubernetes Logs (15141)**.
* **Status:** All pods in `cattle-monitoring-system` and `logging` namespaces are **3/3 Running**.

---

## ⚠️ Ruthless Audit & Cleanups
* **Zombie Pods:** Terminated `tmp-shell` (interactive debug container) which had been idling for 159 minutes.
* **Network Optimization:** Nuked redundant Ingress objects to prevent URL redirection loops.
* **Security:** Reverted manual `NodePort` exposures back to `ClusterIP` to ensure traffic flows through the authenticated Traefik/Rancher gateway.

---

## 🚀 Next Steps (Phase 4: Performance & Performance)
1.  **etcd Performance Tuning:** Migrate the etcd data directory to dedicated 20GB high-speed VirtIO disks on `cp-1`, `cp-2`, and `cp-3`.
2.  **Loki Retention Policy:** Configure the Compactor to prevent ZFS pool exhaustion.
3.  **Alerting Rules:** Define Alertmanager configs for critical pod failures.

---
*Last Updated: 2026-05-15 | Platform Engineer: Suvrajeet Banerjee*

***
***

## 18-MAY-26

K8s cluster tasks
- check pvs created in which node
- check the volume/size of data generated from the test apps deployed and deleted (3x)
- size of pvs filled

***
***

## 19-May-26
- K8s cluster dissection
- k8s cluster inside out
  
***
***

## 20-May-26
- pv checked for loki
- size & location confirmed
- working on bugs (checks) in portal
- crm system

***
***

## 22-May-26
- codebase study
- setting up frontend & backend
- connection establishment b/w containers for testing
- database in containers for testing

***
***

## 25-May-26
- Rancher k8s cluster showing main k8s cluster down
        - probably due to one of the nodes auto restarted or someone shut it down on purpose
        - issue = stopped rke2 ; restarted it - issue fiksed
- steps to be taken so that this never occurs in future
- automation plan : use signal/telegrram with a dashboard showing the status of each and every node and rancher admin as well so that in case of any downtime, necessary actions can be reviewed
- further plan scope
        - use local llm to fiks these small issues without the need for HITL.

***
***

## 27-May-26
- deployed the application on the k8s cluster
- 2 frontends + 2 backends (1 being the cronjob) + databases (mariadb galera custer + mongodb + redis) 
- fiksed backend not connecting with frontend and database connection errors
- resolved pods starting but not ready
- 2fa authentication added using authenticator
- assigned public ip to the frontend portal and gitlab server routed via HAproxy

***
***

## 29-May-26
- *RKE2 Cluster Recovery*
  - The RKE2 control plane (3-node etcd cluster) had gone into a no-quorum state due to simultaneous node crashes.
  - Identified the root cause as zombie containerd-shim processes blocking etcd leader election.
  - Cleared stale processes across all control plane nodes; cluster self-recovered without requiring a --cluster-reset.
  - All 6 nodes (3 CP + 3 workers) are back to Running state as confirmed in Rancher.

- *Public DNS Mapping*
  - Configured DNS A records for console.host360.ai and product-gitlab.host360.ai pointing to public IP ***.***.***.***.
  - Both domains are now resolving correctly via public DNS (verified via nslookup against 8.8.8.8).

- *HAProxy Routing Validated*
  - Reviewed and confirmed the existing HAProxy config on VIP 172.20.4.80.
  - HTTP/HTTPS traffic for console.host360.ai correctly routes to Traefik NodePort (30080/30443) on worker nodes, and product-gitlab.host360.ai routes directly to the GitLab VM at 172.20.4.60.
  - HAProxy and Keepalived both confirmed active.

- *GitLab HTTPS Enabled*
  - Configured Let's Encrypt TLS on the GitLab VM by updating external_url to https://product-gitlab.host360.ai in gitlab.rb and enabling the built-in Let's Encrypt integration.
  - Ran gitlab-ctl reconfigure — GitLab is now accessible over HTTPS at product-gitlab.host360.ai with a valid certificate.

- *Pending*
  - Traefik IngressRoute for console.host360.ai — Traefik is running on the cluster (NodePort 30080/30443), app services are healthy in host360-prod namespace.
  - IngressRoute creation and HTTPS via cert-manager for console.host360.ai is in progress as the next step.

***
***

## 01-Jun-26

- ingress - nodeport (earlier)
- changed to metal-lb
- https ingress reapplied (already installed earlier)
- haproksy config (metal-lb ip added)
- fiksedd wrong endpoint call
- lets enncrypt also reapplied
- updated servers-ip sheet with vms passwords
- created user named soniya in rancher admin with admin privileges
- updated gitlab repo with updated codebase


***
***

## 04-Jun-26
- moved from promeheus grafana loki to ELK stacck
- configured
  - Elastic Search
  - Logstash
  - Kibana 


***
***

## 05-Jun-26



## ELK Stack Setup Progress

## Current Status
The ELK stack setup has reached a working baseline in production-style mode.

- **Elasticsearch HA cluster** is up with 3 nodes and healthy shard allocation
- **Kibana** is reachable and connected to the cluster
- **Logstash** is receiving events on Beats input and forwarding to Elasticsearch
- **Filebeat** is installed and forwarding Linux logs from the source VM
- **Linux syslog and auth logs** are ingested successfully
- **Grok parsing** is working and extracting structured fields
- **Data streams** are working with `logs-linux-prod`
- **Kibana Discover** is able to visualize the ingested logs

---

## End-to-End Flow Implemented


Linux VM logs
   ↓
Filebeat
   ↓
Logstash (Beats input + Grok filter)
   ↓
Elasticsearch HA cluster
   ↓
Kibana Discover / Data Views


---

## Components Built

### Elasticsearch Cluster

* 3-node cluster:

  * `elk-es-01` → `172.20.4.38`
  * `elk-es-02` → `172.20.4.39`
  * `elk-es-03` → `172.20.4.40`
* Cluster health verified as **green**
* Shards and replicas distributed across nodes
* Master election and node visibility verified
* Current master was validated from the cluster outputs

### Logstash

* Installed on `elk-logstash` → `172.20.4.41`
* Beats input listening on **5044**
* Elasticsearch output configured to use all 3 ES nodes
* Data stream output enabled:

  * `data_stream => true`
  * `data_stream_type => "logs"`
  * `data_stream_dataset => "linux"`
  * `data_stream_namespace => "prod"`
* Grok filter added for structured parsing of syslog-style logs

### Filebeat

* Installed on `elk-logstash`
* Configured to collect:

  * `/var/log/syslog`
  * `/var/log/auth.log`
* Filebeat output initially pointed to Logstash
* Filebeat `system` module enabled
* Module loader configured correctly
* `system` module parsing active for:

  * `syslog`
  * `auth`

### Kibana

* Running on `elk-kibana` → `172.20.4.42`
* Connected to the ES cluster
* Data views created and validated
* Discover confirmed visibility of:

  * `PIPELINE_GROK_TEST_002`
  * `AUTH_TEST_001`
  * `MODULE_SYSLOG_TEST_001`

---

## Work Completed Step by Step

### 1. Built Elasticsearch cluster

* Installed and configured 3 Elasticsearch nodes
* Joined nodes into a single HA cluster
* Verified node status, master node, and shard placement
* Fixed repeated master discovery and cluster state issues

### 2. Enabled security and access control

* Reset the `elastic` password
* Verified authenticated REST access
* Created `logstash_writer` role
* Created `logstash_ingest` user
* Confirmed Logstash could authenticate and write into Elasticsearch

### 3. Deployed Logstash pipeline

* Installed Logstash on a dedicated VM
* Opened Beats port `5044`
* Configured Logstash output to all ES nodes
* Switched from raw index output to **data stream** output
* Fixed `create` action requirement for data streams

### 4. Configured Filebeat

* Installed Filebeat on the Logstash host
* Started with manual filestream input for:

  * `syslog`
  * `auth.log`
* Then enabled the Filebeat `system` module
* Fixed module loader path in `filebeat.yml`
* Enabled module parsing for:

  * `system.syslog`
  * `system.auth`

### 5. Fixed parsing

* Added Grok filter in Logstash
* Extracted:

  * `syslog_timestamp`
  * `syslog_host`
  * `process`
  * `log_message`
* Verified parsing with test logs

### 6. Validated ingest path

* Created test logs using `logger`
* Confirmed logs reached Elasticsearch
* Confirmed logs appeared in Kibana Discover
* Confirmed both syslog and auth log ingestion worked

### 7. Created Kibana data views

* Created data view for:

  * `logs-linux-prod`
* Used `@timestamp` as the time field
* Verified searching and visualizing log events in Discover

---

## Validation Evidence

The following checks were successfully verified:

* Elasticsearch cluster health = **green**
* Elasticsearch nodes = **3**
* Logstash API on `9600` = healthy
* Beats input on `5044` = listening
* Filebeat service = running
* Filebeat system module = enabled
* Grok parse success = confirmed
* Kibana Discover = showing results
* Search by test log = successful

---

## Example Test Events Verified

* `PIPELINE_GROK_TEST_002`
* `AUTH_TEST_001`
* `MODULE_SYSLOG_TEST_001`
* `SYSTEM_MODULE_TEST_001`

---

## Remaining Work

### Production hardening

* TLS between all components
* Secure Kibana and Elasticsearch endpoints
* Remove plaintext passwords from configs
* Move secrets to keystore / secure storage

### Observability improvements

* Add dashboards for:

  * log volume
  * logs by host
  * auth failures
  * syslog volume
* Add monitoring for:

  * Elasticsearch
  * Logstash
  * Filebeat
  * Kibana

### HA and resilience validation

* Test failover by stopping one Elasticsearch node
* Validate ingestion continues without interruption
* Verify replica recovery after node restart

### Expansion

* Add more log sources:

  * Nginx
  * Apache
  * Kubernetes
  * OpenStack
  * custom application logs
* Decide whether Kafka is needed later as a buffer layer

### Retention and lifecycle

* Tune ILM policy
* Define retention windows
* Confirm rollover behavior for production usage

---

## Notes

* This setup is now beyond lab-only configuration.
* Core ingestion is functional.
* The remaining work is mostly hardening, observability, and failover validation.

---

## Next Immediate Step

* Validate Elasticsearch node failover without breaking log ingestion
* Then move to TLS and hardening

```
```

> NOTE : Using Kimchi.dev



***
***

## 08-Jun-26

- Created PRD on ticketing system
- Increased GitLab Server Specs : 4/8 to 8/16 : CPU/RAM
- Planned rancher failover auto-healing
- MongoDB + MariaDB Observability : Prometheus + Grafana
- ELK Server setup
- Moved Storage of Current ELK Stack from zfs-local to primera-lvm
