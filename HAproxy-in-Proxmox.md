# Production-Grade HAProxy on Proxmox VE

This repository provides a comprehensive, step-by-step guide for deploying a highly optimized, production-ready HAProxy load balancer within a single Ubuntu 24.04 Virtual Machine (VM) on Proxmox Virtual Environment (VE). 

The configuration implements Center for Internet Security (CIS) hardening principles, advanced Linux kernel tuning for high-throughput networks, and modern TLS termination standards.

---

## 1. Proxmox VM Provisioning (CLI)

For repeatability, we use the Proxmox `qm` (QEMU Manager) command-line tool to provision the virtual machine. This approach allocates resources perfectly suited for high-concurrency TLS termination.

### Create the Virtual Machine
Run the following commands on your Proxmox host to create the VM (replace `100` with your desired VM ID and `local-zfs` with your storage pool):
```bash
# Create the base VM with 4 Cores, 4GB RAM, and virtio networking on VLAN 10
qm create 100 --name "HAProxy-01" --memory 4096 --cores 4 --cpu host --net0 virtio,bridge=vmbr0,tag=10

# Configure the SCSI controller for optimal ZFS performance
qm set 100 --scsihw virtio-scsi-single

# Attach a 40GB virtual disk backed by ZFS
qm set 100 --scsi0 local-zfs:40,discard=on,ssd=1

# Enable 'Start at boot' to ensure the VM powers on when the Proxmox host boots
qm set 100 --onboot 1
```
**Architectural Rationale:**
*   **`--cpu host`**: Enables pass-through of AES-NI cryptographic flags, dramatically improving SSL/TLS decryption performance.
*   **`virtio-scsi-single`**: Allows for advanced features like IO thread support and SCSI unmap (discard), maximizing the health and IOPS of the underlying ZFS pool.

---

## 2. Ubuntu OS Initialization & Hardening

Once you install Ubuntu 24.04 LTS on the VM, establish a secure baseline before exposing the server to traffic.

### Install Essential Tooling & HAProxy
Connect to your VM and install the necessary networking, monitoring, and load balancing packages:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y htop sysstat iotop tcpdump socat iperf3 net-tools git curl
sudo apt install -y haproxy keepalived
```

### SSH Hardening
To mitigate brute-force attacks, enforce an SSH-key-only access policy. Edit `/etc/ssh/sshd_config` to include:
```text
PasswordAuthentication no
ChallengeResponseAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```
Restart the service: `sudo systemctl restart ssh`. For further OS security, consider applying CIS Benchmark automation using tools like Ubuntu Security Guide (USG).

---

## 3. High-Performance Kernel Tuning (sysctl)

A load balancer's primary constraint is the Linux network stack. To handle massive connection volumes (100k+ connections), you must tune the TCP parameters beyond their default OS limits.

Create a new file `/etc/sysctl.d/99-haproxy.conf` and add the following:
```ini
# Scale the TCP Listen Backlog and Half-Open Connections
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.core.netdev_max_backlog = 65535

# Maximize available ephemeral ports for backend connections
net.ipv4.ip_local_port_range = 1024 65535

# Allow reuse of sockets in TIME_WAIT state
net.ipv4.tcp_tw_reuse = 1

# Allow binding to a Virtual IP (VIP) that does not exist locally yet (Crucial for future High-Availability/Keepalived)
net.ipv4.ip_nonlocal_bind = 1
```
Apply the changes immediately with `sudo sysctl -p /etc/sysctl.d/99-haproxy.conf`.

---

## 4. HAProxy Production Configuration

Edit `/etc/haproxy/haproxy.cfg`. The configuration below features global multi-threading, secure TLS 1.2+ defaults, rate limiting, and a statistics dashboard.

```haproxy
global
    log /dev/log local0
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    
    # Performance limits
    maxconn 100000
    nbthread 4
    
    # Modern SSL/TLS parameters
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets
    ssl-min-ver TLSv1.2

defaults
    log global
    mode http
    option httplog
    option dontlognull
    # Generous timeouts for client/server, tight connect timeouts
    timeout connect 5s
    timeout client  30s
    timeout server  30s
    timeout http-request 10s

# -----------------------------------------
# Statistics Dashboard
# -----------------------------------------
frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats admin if LOCALHOST

# -----------------------------------------
# Public Ingress (SSL Termination)
# -----------------------------------------
frontend https_front
    # Bind to port 443, enable SSL, specify combined PEM directory, enable HTTP/2
    bind *:443 ssl crt /etc/haproxy/certs/ alpn h2,http/1.1
    
    # Pass client IP to backend
    option forwardfor
    http-request add-header X-Forwarded-Proto https
    
    # Route traffic based on SNI/Host header
    use_backend app_backend if { req.hdr(host) -i myapp.example.com }

# -----------------------------------------
# Backend Servers
# -----------------------------------------
backend app_backend
    balance roundrobin
    # Enable L7 HTTP health checks
    option httpchk GET /health
    # Backend nodes
    server app01 192.168.10.50:80 check inter 5s fall 3 rise 2
    server app02 192.168.10.51:80 check inter 5s fall 3 rise 2
```
*Note: Ensure your TLS certificates are concatenated (Certificate + Intermediate + Private Key) into `.pem` files and placed in `/etc/haproxy/certs/`.*

Test your configuration syntax and safely restart the service:
```bash
haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
```

---

## 5. Next Steps for High Availability (Clustering)

While this repository configures a single highly-optimized HAProxy VM, true High Availability (HA) requires eliminating the single point of failure. 

Because we already enabled `net.ipv4.ip_nonlocal_bind=1` in the kernel, you are perfectly positioned to deploy a second cloned VM. Once cloned, you can configure **Keepalived** to manage a Virtual IP (VIP) using VRRP. Keepalived will float the shared VIP between the Master and Backup VM instantly if a node crashes, ensuring continuous uptime without DNS changes.
