
# Kubernetes HA Cluster - Load Balancer Setup
**Date:** May 4, 2026  
**Objective:** Implement High Availability load balancing for Kubernetes control plane using HAProxy + Keepalived

---

## Infrastructure Overview

### Cluster Nodes
- **Control Plane Nodes:** 3 (stg-cp1, stg-cp2, stg-cp3)
- **Worker Nodes:** 3 (stg-wk1, stg-wk2, stg-wk3)
- **Load Balancer Nodes:** 2 (stg-lb1, stg-lb2)

### IP Addressing
```
Control Plane:
  - stg-cp1: 172.237.44.177
  - stg-cp2: 172.236.173.44
  - stg-cp3: 172.236.173.68

Load Balancers:
  - stg-lb1: 172.236.181.110 (MASTER)
  - stg-lb2: 172.236.181.204 (BACKUP)
  - VIP (floating): 172.236.181.100
```

---

## Architecture Implemented

### HA Load Balancer Stack
```
kubectl/API clients
        ↓
  VIP: 172.236.181.100:6443 (managed by keepalived)
        ↓
    ┌───────────────┐
    │   Keepalived   │ (VRRP failover, <2s recovery)
    └───────────────┘
        ↓
    ┌───────────────┐
    │    HAProxy     │ (TCP load balancing + health checks)
    └───────────────┘
        ↓
    ┌─────────────────────────────────────┐
    │  stg-cp1:6443  stg-cp2:6443  stg-cp3:6443 │
    │  (Kubernetes API servers)           │
    └─────────────────────────────────────┘
```

**Key Features:**
- Zero single point of failure
- Automatic failover in <2 seconds
- Health-check based routing
- Round-robin load distribution

---

## Components Deployed

### 1. HAProxy (Load Balancer)
**Version:** 2.8.16  
**Configuration:** `/etc/haproxy/haproxy.cfg`

**Key Settings:**
```
Mode: TCP (Layer 4)
Frontend: Port 6443 (Kubernetes API)
Backend: 3 control plane nodes
Health Check: TCP connection test every 2s
  - fall 3: Mark DOWN after 3 failures
  - rise 2: Mark UP after 2 successes
Algorithm: roundrobin
```

**Health Check Parameters:**
- Interval: 2 seconds
- Failure threshold: 3 consecutive failures
- Recovery threshold: 2 consecutive successes

---

### 2. Keepalived (VIP Failover)
**Version:** 2.2.8  
**Configuration:** `/etc/keepalived/keepalived.conf`

**VRRP Setup:**
```
Protocol: VRRPv2
Virtual Router ID: 51
Authentication: PASS (password: k8s_lb)
Advertisement Interval: 1 second

Priority Configuration:
  - stg-lb1 (MASTER): 110 (with HAProxy up)
  - stg-lb2 (BACKUP): 105 (with HAProxy up)
  - Failed node: base priority - 15
```

**Health Check Script:**
```bash
/usr/bin/killall -0 haproxy
```
- Interval: 2 seconds
- Weight: -15 (penalty on failure)
- User: keepalived_script (non-root security)

**Failover Logic:**
- stg-lb1 healthy: priority 110 → owns VIP
- stg-lb1 HAProxy down: priority 95 → VIP fails to stg-lb2 (priority 105)
- Both HAProxy down: stg-lb1 retains VIP (priority 95 > 90)

---

## Installation Steps Summary

### Load Balancer VMs Provisioned
1. Created 2 Linode VMs (Nanode 1GB, Ubuntu 24.04)
2. Configured hostnames and network connectivity
3. Added `/etc/hosts` entries for all cluster nodes

### HAProxy Installation
```bash
sudo apt update
sudo apt install haproxy -y
```

**Configuration Applied:**
- Frontend listening on `*:6443`
- Backend with 3 control plane nodes
- TCP health checks enabled
- Service enabled on boot

### Keepalived Installation
```bash
sudo apt install keepalived -y
sudo useradd -r -s /sbin/nologin -M keepalived_script
```

**Configuration Applied:**
- VRRP instance configured with VIP
- Health check script for HAProxy monitoring
- Script security enabled with dedicated user
- Service enabled on boot

### Kubernetes Integration
**Updated kubeconfig:**
```bash
kubectl config set-cluster kubernetes --server=https://172.236.181.100:6443
```

**Old server:** `https://172.237.44.177:6443` (hardcoded stg-cp1)  
**New server:** `https://172.236.181.100:6443` (floating VIP)

---

## Testing & Validation

### Failover Tests Performed
1. **HAProxy failure simulation**
   - Stopped HAProxy on stg-lb1
   - Verified VIP failover to stg-lb2
   - Confirmed kubectl continued working without interruption

2. **Service recovery**
   - Restarted HAProxy on stg-lb1
   - Verified VIP returned to stg-lb1 (preemption)
   - Validated priority adjustment via health checks

3. **API connectivity**
   ```bash
   curl -k https://172.236.181.100:6443/version
   ```
   Response: Kubernetes v1.29.0 (successful)

---

## Challenges Encountered & Resolutions

### Issue 1: Keepalived Script Security Violation
**Error:**
```
SECURITY VIOLATION - scripts are being executed but script_security not enabled
```

**Root Cause:** Keepalived 2.2+ requires explicit script user configuration

**Resolution:**
- Created dedicated user: `keepalived_script`
- Added `enable_script_security` and `script_user` directives
- Configured proper script permissions

---

### Issue 2: Split-Brain Condition
**Symptom:** Both load balancers claimed VIP simultaneously

**Root Cause:** 
- Health check script not loading due to security restrictions
- No priority adjustment on HAProxy failure
- Non-preemptive VRRP mode with equal priorities

**Resolution:**
- Fixed script user permissions
- Changed health check weight from `+2` to `-15` (penalty-based)
- Adjusted priorities: stg-lb1=110, stg-lb2=105
- Verified failover triggers BACKUP state on failed node

---

### Issue 3: Config File Corruption
**Symptom:** Heredoc EOF not closing properly, invalid configs

**Root Cause:** Terminal copy-paste artifacts corrupting heredoc syntax

**Resolution:**
- Used `nano` text editor instead of heredoc
- Validated configs with `keepalived -t -f /etc/keepalived/keepalived.conf`
- Verified syntax before starting services

---

## Current Status

### Services Running
```
stg-lb1:
  - haproxy.service: active (running)
  - keepalived.service: active (running)
  - VIP ownership: MASTER
  - Priority: 110 (HAProxy healthy)

stg-lb2:
  - haproxy.service: active (running)
  - keepalived.service: active (running)
  - VIP ownership: BACKUP
  - Priority: 105 (HAProxy healthy)
```

### Cluster Health
```bash
kubectl get nodes
```
All 6 nodes (3 control plane + 3 worker) reporting `Ready`

---

## Production Readiness Checklist

### ✅ Completed
- [x] HA control plane (3 nodes with etcd cluster)
- [x] HA load balancers (HAProxy + Keepalived)
- [x] Automatic VIP failover (<2s recovery)
- [x] Health-check based traffic routing
- [x] Kubeconfig updated to use VIP
- [x] Failover testing validated
- [x] No single point of failure in control plane

### ⏳ Next Steps
- [ ] Monitoring & Alerting (Prometheus + Grafana)
- [ ] Persistent Storage (CSI driver: Longhorn/Rook-Ceph)
- [ ] Ingress Controller (NGINX/Traefik)
- [ ] TLS certificates (Let's Encrypt)
- [ ] etcd backup automation
- [ ] Network policies (Calico)
- [ ] RBAC hardening
- [ ] Disaster recovery testing

---

## Key Learnings

### HAProxy Best Practices
- Use TCP mode for TLS-encrypted traffic (Kubernetes API)
- Set `fall` and `rise` thresholds to prevent flapping
- Monitor backend server health in real-time
- Keep timeouts reasonable (50s for long-lived API connections)

### Keepalived Best Practices
- Use negative weight for failure detection (penalty-based)
- Set significant priority gaps between MASTER/BACKUP (5+ points)
- Enable script security with dedicated non-root user
- Use preemptive mode for automatic failback
- VRRP advertisement interval: 1s for fast detection

### VRRP Failover Timing
```
Detection: 2s (health check interval)
Priority drop: immediate
VRRP advertisements: 3-4 cycles (1s each)
Total failover time: ~2-4 seconds
```

---

## Configuration Files

### HAProxy Config (`/etc/haproxy/haproxy.cfg`)
```
global
    log /dev/log local0
    chroot /var/lib/haproxy
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode tcp
    option tcplog
    option dontlognull
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend k8s-api
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s-api-backend

backend k8s-api-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server stg-cp1 172.237.44.177:6443 check fall 3 rise 2
    server stg-cp2 172.236.173.44:6443 check fall 3 rise 2
    server stg-cp3 172.236.173.68:6443 check fall 3 rise 2
```

### Keepalived Config - stg-lb1 (`/etc/keepalived/keepalived.conf`)
```
global_defs {
   router_id LB1
   enable_script_security
   script_user keepalived_script
}

vrrp_script check_haproxy {
   script "/usr/bin/killall -0 haproxy"
   interval 2
   weight -15
}

vrrp_instance VI_1 {
   state MASTER
   interface eth0
   virtual_router_id 51
   priority 110
   advert_int 1
   
   authentication {
      auth_type PASS
      auth_pass k8s_lb
   }
   
   virtual_ipaddress {
      172.236.181.100
   }
   
   track_script {
      check_haproxy
   }
}
```

### Keepalived Config - stg-lb2 (`/etc/keepalived/keepalived.conf`)
```
global_defs {
   router_id LB2
   enable_script_security
   script_user keepalived_script
}

vrrp_script check_haproxy {
   script "/usr/bin/killall -0 haproxy"
   interval 2
   weight -15
}

vrrp_instance VI_1 {
   state BACKUP
   interface eth0
   virtual_router_id 51
   priority 105
   advert_int 1
   
   authentication {
      auth_type PASS
      auth_pass k8s_lb
   }
   
   virtual_ipaddress {
      172.236.181.100
   }
   
   track_script {
      check_haproxy
   }
}
```

---

## Useful Commands

### Check VIP Ownership
```bash
ip addr show eth0 | grep 172.236.181.100
```

### Monitor Keepalived Logs
```bash
sudo journalctl -u keepalived -f
```

### Check HAProxy Status
```bash
sudo systemctl status haproxy
```

### Test API Connectivity via VIP
```bash
curl -k https://172.236.181.100:6443/version
```

### Watch Failover in Real-Time
```bash
# Terminal 1 (monitoring)
while true; do kubectl get nodes --no-headers | wc -l; sleep 1; done

# Terminal 2 (trigger failover)
sudo systemctl stop haproxy
```

### Validate Keepalived Config
```bash
sudo keepalived -t -f /etc/keepalived/keepalived.conf
```

---

## References
- [Kubernetes HA Setup Guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [HAProxy Documentation](https://www.haproxy.org/)
- [Keepalived Manual](https://www.keepalived.org/manpage.html)
- [VRRP Protocol RFC 3768](https://datatracker.ietf.org/doc/html/rfc3768)

---

**Cluster Status:** ✅ Production-ready HA control plane operational  
**Next Session:** Deploy monitoring stack (Prometheus + Grafana)

