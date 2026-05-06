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
