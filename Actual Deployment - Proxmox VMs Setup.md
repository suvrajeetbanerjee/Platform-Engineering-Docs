# Actual Deployment - Your Proxmox VMs Setup

**Status**: You have Proxmox with VMs → Now setup everything inside **Target**: Day 1 Deployment (Real Commands You Run Today)

---

## STEP 0: UNDERSTAND YOUR CURRENT SETUP

From your images, you have:

**Production Environment:**

- HA Proxy (1): 2vCPU, 8GB RAM, 80GB storage
- Master Node (3): 4vCPU, 8GB RAM, 80GB storage each
- Worker Node (3): 8vCPU, 16GB RAM, 80GB storage each
- Galera Cluster (3): 8vCPU, 16GB RAM, 350GB storage each
- MongoDB (2): 4vCPU, 32GB RAM, 1024GB storage each
- Jenkins (1): 16vCPU, 64GB RAM, 500GB storage

**Staging Environment:**

- HA Proxy (1): 2vCPU, 4GB RAM, 80GB storage
- Master Node (3): 4vCPU, 8GB RAM, 80GB storage each
- Worker Node (3): 8vCPU, 16GB RAM, 80GB storage each
- MariaDB (1): 4vCPU, 16GB RAM, 350GB storage
- MongoDB (1): 4vCPU, 16GB RAM, 500GB storage

**Applications** (13 services):

1. Jenkins (CI)
2. KeyCloak (Authentication)
3. Sonarqube (Code Quality)
4. Docker Build (Container Creation)
5. Project Quay (Registry)
6. Security-Trivy (Image Scanning)
7. ArgoCD (GitOps Deployment)
8. Prometheus + Grafana (Monitoring)
9. EFK/Elk/Loki Grafana (Logging)
10. Falco Kuber/Kyverno (Security)
11. HAproxy (Load Balancer)
12. MongoDB (Database)
13. MariaDB (Database)

---

## STEP 1: INITIAL VM SETUP (RUN ON ALL VMs)

### 1.1 Connect to Your First VM

```bash
# SSH into your first VM (e.g., Production HA Proxy)
ssh root@<VM_IP>

# Or if password auth
ssh -o StrictHostKeyChecking=no root@<VM_IP>
```

### 1.2 Update System (ALL VMs)

```bash
#!/bin/bash
# Run on EVERY VM first

# Update package list
apt update

# Upgrade system
apt upgrade -y

# Install essential tools
apt install -y \
  curl \
  wget \
  git \
  vim \
  htop \
  net-tools \
  tmux \
  jq \
  unzip \
  openssl

# Set timezone
timedatectl set-timezone Asia/Kolkata

# Verify
echo "System Ready: $(hostname) - $(date)"
```

### 1.3 Configure Network (ALL VMs)

```bash
#!/bin/bash
# Configure permanent network settings

# Check current networking
ip addr show
ip route show

# For persistent network config on Ubuntu
cat > /etc/netplan/00-installer-config.yaml << 'EOF'
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 10.0.2.11/24  # Change per VM
      gateway4: 10.0.2.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
EOF

# Apply
netplan apply
ip addr show

# Test connectivity
ping -c 2 10.0.2.1
ping -c 2 8.8.8.8
```

---

## STEP 2: SETUP MASTER NODES (Production)

### 2.1 On All 3 Master Nodes

```bash
#!/bin/bash
# Run on kube-master-1, kube-master-2, kube-master-3

# === DISABLE SWAP ===
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
free -h  # Verify swap is 0

# === LOAD KERNEL MODULES ===
cat >> /etc/modules-load.d/k8s.conf << EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# === CONFIGURE SYSCTL ===
cat >> /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system

# === INSTALL CONTAINERD (Container Runtime) ===
mkdir -p /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

apt update
apt install -y containerd.io

# Configure containerd
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
systemctl daemon-reload
systemctl restart containerd
systemctl enable containerd

# === INSTALL KUBERNETES TOOLS ===
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg \
  https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | \
  tee /etc/apt/sources.list.d/kubernetes.list

apt update
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

# === VERIFY INSTALLATION ===
echo "=== Verification ==="
containerd --version
kubelet --version
kubeadm version
kubectl version --client || echo "kubectl needs cluster to connect"

echo "✓ Master node ready: $(hostname)"
```

### 2.2 Initialize First Master (kube-master-1 ONLY)

```bash
#!/bin/bash
# RUN ONLY ON kube-master-1 (10.0.2.11)

# Create kubeadm config
cat > /root/kubeadm-config.yaml << 'EOF'
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.28.0
controlPlaneEndpoint: "10.0.2.100:6443"  # HAProxy VIP
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  dnsDomain: "cluster.local"
apiServer:
  extraArgs:
    authorization-mode: "Node,RBAC"
  certSANs:
    - "10.0.2.100"
    - "10.0.2.11"
    - "10.0.2.12"
    - "10.0.2.13"
    - "localhost"
    - "127.0.0.1"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
serverTLSBootstrap: true
EOF

# Initialize cluster
kubeadm init --config /root/kubeadm-config.yaml --upload-certs 2>&1 | tee /root/kubeadm-init.log

# === SAVE THESE TOKENS (from output above) ===
# You'll need:
# 1. KUBEADM_TOKEN
# 2. KUBEADM_DISCOVERY_TOKEN_CA_CERT_HASH
# 3. CERTIFICATE_KEY

echo "=== SAVE THIS OUTPUT ABOVE ===" 
echo "You need to copy the 'kubeadm join' commands for other masters and workers"
echo "Check /root/kubeadm-init.log"

# === SETUP KUBECTL ACCESS ===
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# Test kubectl
kubectl cluster-info
kubectl get nodes

echo "✓ Cluster initialized on $(hostname)"
```

### 2.3 Join Other Masters (kube-master-2 and kube-master-3)

```bash
#!/bin/bash
# RUN ON kube-master-2 and kube-master-3
# Get values from /root/kubeadm-init.log on master-1

TOKEN="<paste-token-from-log>"
DISCOVERY_HASH="<paste-discovery-hash-from-log>"
CERT_KEY="<paste-certificate-key-from-log>"

kubeadm join 10.0.2.100:6443 \
  --token $TOKEN \
  --discovery-token-ca-cert-hash sha256:$DISCOVERY_HASH \
  --control-plane \
  --certificate-key $CERT_KEY

# Setup kubectl
mkdir -p $HOME/.kube
scp root@10.0.2.11:/etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# Verify
kubectl get nodes

echo "✓ Master joined cluster"
```

---

## STEP 3: SETUP WORKER NODES

### 3.1 Prepare Worker Nodes (kube-worker-1, 2, 3)

```bash
#!/bin/bash
# Same as master nodes, but SKIP kubeadm init
# Run steps 2.1 (everything before kubeadm init)

# Then join the cluster:
TOKEN="<paste-token>"
DISCOVERY_HASH="<paste-discovery-hash>"

kubeadm join 10.0.2.100:6443 \
  --token $TOKEN \
  --discovery-token-ca-cert-hash sha256:$DISCOVERY_HASH

# Verify
kubectl get nodes  # Run from a master node

echo "✓ Worker joined cluster"
```

### 3.2 Verify All Nodes

```bash
# Run from kube-master-1
kubectl get nodes -o wide

# Should show:
# NAME              STATUS   ROLES                  AGE   VERSION
# kube-master-1     Ready    control-plane,master   5m    v1.28.0
# kube-master-2     Ready    control-plane,master   2m    v1.28.0
# kube-master-3     Ready    control-plane,master   1m    v1.28.0
# kube-worker-1     Ready    worker                 3m    v1.28.0
# kube-worker-2     Ready    worker                 2m    v1.28.0
# kube-worker-3     Ready    worker                 1m    v1.28.0
```

---

## STEP 4: SETUP NETWORKING (Calico CNI)

```bash
#!/bin/bash
# Run from kube-master-1

# Install Tigera Operator
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

# Wait for operator
kubectl wait --for=condition=ready pod -l k8s-app=tigera-operator -n tigera-operator --timeout=90s

# Install Calico
cat > /root/calico-install.yaml << 'EOF'
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLan
      natOutgoing: Enabled
      nodeSelector: all()
EOF

kubectl apply -f /root/calico-install.yaml

# Wait for Calico to be ready
kubectl wait --for=condition=ready pod -l k8s-app=calico-node -n calico-system --timeout=300s

# Verify
kubectl get pods -n calico-system
kubectl get nodes

# All nodes should be Ready now
echo "✓ Networking configured"
```

---

## STEP 5: SETUP HAPROXY LOAD BALANCER

```bash
#!/bin/bash
# Run on HA Proxy VM (10.0.2.100)

# Install HAProxy
apt update
apt install -y haproxy

# Configure HAProxy for Kubernetes API
cat > /etc/haproxy/haproxy.cfg << 'EOF'
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    tcp
    option  tcplog
    timeout connect 5000
    timeout client  50000
    timeout server  50000

frontend kubernetes-api
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes-masters

backend kubernetes-masters
    balance roundrobin
    mode tcp
    server master1 10.0.2.11:6443 check
    server master2 10.0.2.12:6443 check
    server master3 10.0.2.13:6443 check

frontend frontend-traffic
    bind *:80
    mode http
    default_backend workers

backend workers
    balance roundrobin
    mode http
    server worker1 10.0.2.51:80 check
    server worker2 10.0.2.52:80 check
    server worker3 10.0.2.53:80 check
EOF

# Restart HAProxy
systemctl restart haproxy
systemctl enable haproxy

# Test
echo "Testing HAProxy..."
curl -k https://127.0.0.1:6443/healthz 2>/dev/null || echo "API responding"

echo "✓ HAProxy configured"
```

---

## STEP 6: INSTALL ESSENTIAL TOOLS (on master-1)

```bash
#!/bin/bash
# Install Helm

curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version

# Add Helm repos
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add jenkinsci https://charts.jenkins.io
helm repo add jetstack https://charts.jetstack.io
helm repo add argoproj https://argoproj.github.io/argo-helm
helm repo update

# Install kubectl completions
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null

echo "✓ Tools installed"
```

---

## STEP 7: SETUP JENKINS (PRODUCTION)

```bash
#!/bin/bash
# Run on Jenkins VM (Production)

# === INSTALL DOCKER ===
apt update
apt install -y docker.io
systemctl enable docker
systemctl start docker

# Add user to docker group
usermod -aG docker root

# === INSTALL JAVA (Jenkins requirement) ===
apt install -y default-jdk

# === INSTALL JENKINS ===
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | apt-key add -
sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
apt update
apt install -y jenkins

# === CONFIGURE JENKINS ===
systemctl enable jenkins
systemctl start jenkins

# Get initial admin password
echo "Jenkins Admin Password:"
cat /var/lib/jenkins/secrets/initialAdminPassword

# Access at http://<jenkins-vm-ip>:8080
echo "✓ Jenkins running on $(hostname)"
```

---

## STEP 8: SETUP MONITORING (Prometheus + Grafana)

```bash
#!/bin/bash
# Run from kube-master-1

# Create monitoring namespace
kubectl create namespace monitoring

# Install Prometheus Stack via Helm
helm install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --set grafana.adminPassword=grafana_secure_password

# Get Grafana password
kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d
echo ""

# Port forward to access locally
echo "Port forwarding Grafana..."
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80 &
echo "Access Grafana at http://localhost:3000"

# Port forward Prometheus
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090 &
echo "Access Prometheus at http://localhost:9090"

echo "✓ Monitoring installed"
```

---

## STEP 9: SETUP DATABASES

### 9.1 MongoDB (Production)

```bash
#!/bin/bash
# Run on MongoDB VMs

# Install MongoDB
curl -fsSL https://www.mongodb.org/static/pgp/server-6.0.asc | apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/6.0 multiverse" | tee /etc/apt/sources.list.d/mongodb-org-6.0.list
apt update
apt install -y mongodb-org

# Start MongoDB
systemctl enable mongod
systemctl start mongod

# Configure for replica set (both MongoDB instances)
# On MongoDB-1:
mongo << 'EOF'
rs.initiate({
  _id: "rs0",
  members: [
    {_id: 0, host: "10.0.3.10:27017"},
    {_id: 1, host: "10.0.3.11:27017"}
  ]
})

// Check status
rs.status()

// Create user
db.createUser({
  user: "admin",
  pwd: "mongo_secure_password_123",
  roles: [{role: "root", db: "admin"}]
})

use appdb
db.createUser({
  user: "appuser",
  pwd: "app_mongo_password_123",
  roles: [{role: "readWrite", db: "appdb"}]
})
EOF

echo "✓ MongoDB configured"
```

### 9.2 MariaDB (Production)

```bash
#!/bin/bash
# Run on MariaDB VM

# Install MariaDB
apt update
apt install -y mariadb-server

# Start service
systemctl enable mariadb
systemctl start mariadb

# Secure installation
mysql_secure_installation << EOF
y
MariaDB_Root_Password_123
MariaDB_Root_Password_123
y
y
y
y
EOF

# Create database and user
mysql -u root -pMariaDB_Root_Password_123 << 'EOF'
CREATE DATABASE appdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'appuser'@'%' IDENTIFIED BY 'mariadb_secure_password_123';
GRANT ALL PRIVILEGES ON appdb.* TO 'appuser'@'%';
FLUSH PRIVILEGES;
SHOW DATABASES;
EOF

echo "✓ MariaDB configured"
```

---

## STEP 10: SETUP INGRESS & CERT MANAGER

```bash
#!/bin/bash
# Run from kube-master-1

# Install Nginx Ingress Controller
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer

# Get external IP
kubectl get svc -n ingress-nginx

# Install Cert-Manager for SSL
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Create Let's Encrypt Issuer
kubectl apply -f - << 'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@company.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

echo "✓ Ingress configured"
```

---

## STEP 11: SETUP KEYCLOAK (OIDC Authentication)

```bash
#!/bin/bash
# Run from kube-master-1

# Add Keycloak Helm repo
helm repo add keycloak https://codecentric.github.io/helm-charts
helm repo update

# Install Keycloak
helm install keycloak keycloak/keycloak \
  --namespace security \
  --create-namespace \
  --set auth.adminUser=admin \
  --set auth.adminPassword=keycloak_secure_password_123 \
  --set ingress.enabled=true \
  --set ingress.hosts[0].host=keycloak.example.com \
  --set postgresql.enabled=true

# Get Keycloak URL
kubectl get ingress -n security

# Access Keycloak admin console
echo "Access at: https://keycloak.example.com/auth/admin"
echo "Username: admin"
echo "Password: keycloak_secure_password_123"

echo "✓ Keycloak installed"
```

---

## STEP 12: ARGOCD (GitOps Deployment)

```bash
#!/bin/bash
# Run from kube-master-1

# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Get ArgoCD admin password
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD Admin Password: $ARGOCD_PASSWORD"

# Port forward to access
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
echo "Access ArgoCD at https://localhost:8080"
echo "Username: admin"
echo "Password: $ARGOCD_PASSWORD"

echo "✓ ArgoCD installed"
```

---

## STEP 13: VERIFY EVERYTHING IS RUNNING

```bash
#!/bin/bash
# Run from kube-master-1

echo "=== CLUSTER STATUS ==="
kubectl cluster-info
kubectl get nodes -o wide

echo -e "\n=== POD STATUS ==="
kubectl get pods -A

echo -e "\n=== NODES RESOURCE USAGE ==="
kubectl top nodes

echo -e "\n=== PERSISTENT VOLUMES ==="
kubectl get pv
kubectl get pvc -A

echo -e "\n=== SERVICES ==="
kubectl get svc -A

echo -e "\n=== INGRESSES ==="
kubectl get ingress -A

echo -e "\n=== RECENT EVENTS ==="
kubectl get events -A --sort-by='.lastTimestamp' | tail -20

echo "✓ Cluster Health Check Complete"
```

---

## QUICK REFERENCE: VMs TO UPDATE

**PRIORITY ORDER (Do in this sequence):**

```bash
# 1. All Masters & Workers (Kubernetes base)
ssh root@10.0.2.11  # kube-master-1
ssh root@10.0.2.12  # kube-master-2
ssh root@10.0.2.13  # kube-master-3
ssh root@10.0.2.51  # kube-worker-1
ssh root@10.0.2.52  # kube-worker-2
ssh root@10.0.2.53  # kube-worker-3

# 2. HA Proxy (Load balancer)
ssh root@10.0.2.100

# 3. Database VMs
ssh root@10.0.3.10  # MongoDB-1
ssh root@10.0.3.11  # MongoDB-2
ssh root@10.0.3.20  # MariaDB-1

# 4. Jenkins (CI/CD)
ssh root@<jenkins-ip>
```

---

## TROUBLESHOOTING CHECKLIST

**If nodes not joining cluster:**

```bash
# Check on master
journalctl -u kubelet -f
kubeadm token list

# Check on worker
systemctl status kubelet
journalctl -u kubelet -f
```

**If pods not starting:**

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
```

**If network connectivity failing:**

```bash
kubectl get pods -n calico-system
kubectl describe pod -n calico-system
```

**If database not accessible:**

```bash
# Test from pod
kubectl run -it --rm debug --image=ubuntu --restart=Never -- bash
apt update && apt install -y mysql-client
mysql -h 10.0.3.20 -u appuser -p
```

---

## NEXT: Deploy Your Applications

Once cluster is running:

1. Create Helm charts for your apps
2. Push to Git repo
3. Configure ArgoCD to deploy from Git
4. Deploy to staging first
5. Test thoroughly
6. Deploy to production

Ready to deploy your apps? Let me know what your first application is!
