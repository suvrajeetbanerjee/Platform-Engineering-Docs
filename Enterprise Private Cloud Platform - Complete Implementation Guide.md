# Enterprise Private Cloud Platform - Complete Implementation Guide

## Table of Contents

1. Pre-Deployment Planning
2. Infrastructure Setup (Proxmox)
3. Kubernetes Installation on Baremetal
4. CI/CD Pipeline Configuration
5. Database Setup
6. Application Deployment
7. Security Implementation
8. Monitoring & Logging
9. Disaster Recovery
10. Troubleshooting Guide

---

## 1. PRE-DEPLOYMENT PLANNING

### 1.1 Resource Requirements

**Proxmox Hypervisor:**

- Bare metal server with 256GB+ RAM
- 16+ CPU cores
- 10TB+ SSD storage (for VMs and data)
- 10Gbps network interface

**Network Planning:**

```
Management Network:   10.0.1.0/24
Kubernetes Network:   10.0.2.0/24
Pod CIDR:            10.244.0.0/16
Service CIDR:        10.96.0.0/12
Database Network:    10.0.3.0/24
```

### 1.2 VM Configuration Strategy

**Master Nodes (3 replicas for HA):**

- vCPU: 4 per node
- RAM: 8GB per node
- Storage: 80GB SSD per node
- OS: Ubuntu 20.04 LTS or RHEL 8

**Worker Nodes (Start with 3, scale as needed):**

- vCPU: 8 per node
- RAM: 16GB per node
- Storage: 100GB SSD per node
- OS: Ubuntu 20.04 LTS or RHEL 8

**Database VMs:**

- MongoDB: 4vCPU, 16GB RAM, 500GB storage
- MariaDB: 4vCPU, 16GB RAM, 350GB storage
- Redis: 2vCPU, 8GB RAM, 100GB storage

---

## 2. INFRASTRUCTURE SETUP (PROXMOX)

### 2.1 Proxmox Installation & Configuration

```bash
# Step 1: Install Proxmox VE on bare metal
# Download ISO from https://www.proxmox.com/en/proxmox-ve
# Use USB boot and follow standard installation

# Step 2: Post-installation configuration
ssh root@proxmox-ip

# Update system
apt update && apt upgrade -y

# Remove enterprise repo (if not licensed)
rm /etc/apt/sources.list.d/pve-enterprise.list
echo "deb http://download.proxmox.com/debian/pve bullseye pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
apt update

# Enable useful features
echo 1 > /proc/sys/net/ipv4/ip_forward

# Configure network bridge for Kubernetes networking
# Edit /etc/network/interfaces to add VM network bridge
```

### 2.2 Create VM Templates

```bash
# Create Ubuntu 20.04 template
# 1. Download cloud image
wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img

# 2. Create VM from image
qm create 9000 --name ubuntu-20.04-template \
  --memory 2048 --cores 2 --sockets 1 \
  --net0 virtio,bridge=vmbr0

# 3. Import disk
qm importdisk 9000 focal-server-cloudimg-amd64.img local-lvm

# 4. Resize disk for templates
qm resize 9000 scsi0 +20G

# 5. Convert to template
qm template 9000
```

### 2.3 Create Master & Worker Node VMs

```bash
#!/bin/bash
# Create 3 Master Nodes

for i in {1..3}; do
  VMID=$((100 + i))
  qm clone 9000 $VMID --name kube-master-$i --full
  qm set $VMID --cores 4 --memory 8192 --net0 virtio,bridge=vmbr0,ip=10.0.2.$((10+i))/24,gw=10.0.2.1
  qm start $VMID
done

# Create 3 Worker Nodes
for i in {1..3}; do
  VMID=$((200 + i))
  qm clone 9000 $VMID --name kube-worker-$i --full
  qm set $VMID --cores 8 --memory 16384 --net0 virtio,bridge=vmbr0,ip=10.0.2.$((50+i))/24,gw=10.0.2.1
  qm start $VMID
done

# Create Database VMs
qm clone 9000 301 --name mongodb-1 --full
qm set 301 --cores 4 --memory 16384 --net0 virtio,bridge=vmbr0,ip=10.0.3.10/24,gw=10.0.3.1
qm start 301

qm clone 9000 302 --name mongodb-2 --full
qm set 302 --cores 4 --memory 16384 --net0 virtio,bridge=vmbr0,ip=10.0.3.11/24,gw=10.0.3.1
qm start 302

qm clone 9000 303 --name mariadb-1 --full
qm set 303 --cores 4 --memory 16384 --net0 virtio,bridge=vmbr0,ip=10.0.3.20/24,gw=10.0.3.1
qm start 303
```

---

## 3. KUBERNETES INSTALLATION ON BAREMETAL

### 3.1 Prerequisites on All Nodes

```bash
#!/bin/bash
# Run on all master and worker nodes

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl
cat << EOF | sudo tee /etc/sysctl.d/99-kubernetes.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system

# Install container runtime (containerd)
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
sudo apt-get update && sudo apt-get install -y containerd.io

# Configure containerd
mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd

# Install kubeadm, kubelet, kubectl
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet=1.28.0-00 kubeadm=1.28.0-00 kubectl=1.28.0-00
sudo apt-mark hold kubelet kubeadm kubectl
```

### 3.2 Initialize Control Plane (Master-1)

```bash
#!/bin/bash
# Run on kube-master-1 only

# Create kubeadm config file
cat << EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.28.0
controlPlaneEndpoint: "10.0.2.100:6443"
networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  dnsDomain: "cluster.local"
apiServer:
  certSANs:
    - "10.0.2.100"
    - "localhost"
    - "127.0.0.1"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
serverTLSBootstrap: true
EOF

# Initialize control plane
sudo kubeadm init --config kubeadm-config.yaml --upload-certs | tee init-output.log

# Save join token and certificate key
export KUBEADM_INIT_DRY_RUN_OUTPUT_PRINTED=true
echo "Save the join token and certificate key from output above!"

# Setup kubectl access
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify
kubectl get nodes
```

### 3.3 Add Master Nodes (Master-2 & Master-3)

```bash
#!/bin/bash
# Run on kube-master-2 and kube-master-3
# Use the --certificate-key from kubeadm init output

sudo kubeadm join 10.0.2.100:6443 \
  --token <TOKEN_FROM_INIT> \
  --discovery-token-ca-cert-hash sha256:<HASH_FROM_INIT> \
  --control-plane \
  --certificate-key <CERTIFICATE_KEY_FROM_INIT>

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 3.4 Add Worker Nodes

```bash
#!/bin/bash
# Run on all worker nodes

sudo kubeadm join 10.0.2.100:6443 \
  --token <TOKEN_FROM_INIT> \
  --discovery-token-ca-cert-hash sha256:<HASH_FROM_INIT>
```

### 3.5 Install CNI Plugin (Calico)

```bash
#!/bin/bash
# Run on master node with kubectl access

# Install Calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

# Wait for operator
kubectl wait --for=condition=ready pod -l k8s-app=tigera-operator -n tigera-operator --timeout=90s

# Create Calico installation
cat << EOF | kubectl apply -f -
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

# Verify
kubectl get nodes -o wide
```

### 3.6 Setup HAProxy Load Balancer for API Server

```bash
# Install on HAProxy VM (10.0.2.100)
sudo apt-get install -y haproxy

cat << EOF | sudo tee /etc/haproxy/haproxy.cfg
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

frontend kubernetes-master
    bind *:6443
    mode tcp
    option tcplog
    default_backend kubernetes-master

backend kubernetes-master
    balance roundrobin
    mode tcp
    option tcplog
    server master1 10.0.2.11:6443 check
    server master2 10.0.2.12:6443 check
    server master3 10.0.2.13:6443 check
EOF

sudo systemctl restart haproxy
```

---

## 4. CI/CD PIPELINE CONFIGURATION

### 4.1 Jenkins Installation via Helm

```bash
#!/bin/bash
# Add Jenkins Helm repository
helm repo add jenkinsci https://charts.jenkins.io
helm repo update

# Create namespace
kubectl create namespace ci-cd

# Create Jenkins values file
cat << EOF > jenkins-values.yaml
controller:
  image: jenkins/jenkins:latest
  installPlugins:
    - kubernetes:latest
    - workflow-aggregator:latest
    - git:latest
    - pipeline-stage-view:latest
    - blueocean:latest
    - role-based-authorization-strategy:latest
    - kubernetes-credentials-provider:latest
  JCasC:
    enabled: true
    configScripts:
      basic-config: |
        unclassified:
          location:
            url: http://jenkins.ci-cd.svc.cluster.local:8080/
  persistence:
    enabled: true
    storageClassName: local-path
    size: 50Gi
  serviceType: LoadBalancer

agent:
  enabled: true
  image: jenkins/inbound-agent:latest
  podName: jenkins-agent

rbac:
  create: true
EOF

# Install Jenkins
helm install jenkins jenkinsci/jenkins -f jenkins-values.yaml -n ci-cd

# Get admin password
kubectl exec --namespace ci-cd -it svc/jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-passwd
```

### 4.2 ArgoCD Installation

```bash
#!/bin/bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
# Access at https://localhost:8080
```

### 4.3 Sonarqube Installation

```bash
#!/bin/bash
# Add Sonarqube Helm chart
helm repo add sonarqube https://SonarSource.github.io/helm-chart-sonarqube
helm repo update

# Create values file
cat << EOF > sonarqube-values.yaml
image:
  repository: sonarqube
  tag: 9.9-community
  pullPolicy: IfNotPresent

persistence:
  enabled: true
  size: 20Gi

postgresql:
  enabled: true
  auth:
    username: sonarqube
    password: sonarqube123
    postgresPassword: postgres123

service:
  type: LoadBalancer
EOF

# Install Sonarqube
helm install sonarqube sonarqube/sonarqube -f sonarqube-values.yaml -n ci-cd

# Access at http://sonarqube-ip:9000
# Default credentials: admin/admin
```

### 4.4 Jenkinsfile Example

```groovy
pipeline {
    agent any
    
    environment {
        REGISTRY = "quay.io/yourcompany"
        IMAGE_NAME = "myapp"
        SONAR_HOST = "http://sonarqube.ci-cd:9000"
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/yourcompany/myapp.git'
            }
        }
        
        stage('Build') {
            steps {
                sh '''
                    npm install
                    npm run build
                '''
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh 'npm run test'
            }
        }
        
        stage('Code Quality Analysis') {
            steps {
                sh '''
                    sonar-scanner \
                      -Dsonar.projectKey=myapp \
                      -Dsonar.sources=. \
                      -Dsonar.host.url=${SONAR_HOST}
                '''
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER} .
                '''
            }
        }
        
        stage('Image Scanning') {
            steps {
                sh '''
                    trivy image --severity HIGH,CRITICAL \
                      ${REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }
        }
        
        stage('Push to Registry') {
            steps {
                sh '''
                    docker push ${REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}
                '''
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                sh '''
                    kubectl set image deployment/myapp-staging \
                      myapp=${REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER} \
                      -n staging
                '''
            }
        }
        
        stage('Integration Tests') {
            steps {
                sh 'npm run test:integration'
            }
        }
        
        stage('Approval') {
            steps {
                input 'Deploy to production?'
            }
        }
        
        stage('Deploy to Production') {
            steps {
                sh '''
                    kubectl set image deployment/myapp \
                      myapp=${REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER} \
                      -n production
                '''
            }
        }
        
        stage('Smoke Tests') {
            steps {
                sh 'npm run test:smoke'
            }
        }
    }
    
    post {
        always {
            publishHTML([
                reportDir: 'coverage',
                reportFiles: 'index.html',
                reportName: 'Code Coverage Report'
            ])
        }
    }
}
```

---

## 5. DATABASE SETUP

### 5.1 MongoDB Replica Set

```bash
#!/bin/bash
# Install MongoDB on dedicated VMs

# On each MongoDB node
sudo apt-get install -y mongodb-org

# Configure replica set
cat << EOF | mongo
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "10.0.3.10:27017" },
    { _id: 1, host: "10.0.3.11:27017" }
  ]
})

# Verify replica set status
rs.status()
EOF

# Create application database and user
mongo << EOF
db.createUser({
  user: "appuser",
  pwd: "secure_password_123",
  roles: [
    { role: "readWrite", db: "appdb" }
  ]
})
EOF
```

### 5.2 MariaDB Installation

```bash
#!/bin/bash
# Install MariaDB
sudo apt-get install -y mariadb-server

# Secure installation
sudo mysql_secure_installation << EOF
y
secure_password_123
secure_password_123
y
y
y
y
EOF

# Create application database and user
mysql -u root -p << EOF
CREATE DATABASE appdb CHARACTER SET utf8mb4;
CREATE USER 'appuser'@'%' IDENTIFIED BY 'secure_password_123';
GRANT ALL PRIVILEGES ON appdb.* TO 'appuser'@'%';
FLUSH PRIVILEGES;
EOF
```

### 5.3 Redis in Kubernetes

```bash
#!/bin/bash
# Install Redis via Helm
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Create Redis instance
helm install redis bitnami/redis \
  -n data-layer \
  --set auth.enabled=true \
  --set auth.password=redis_secure_password
```

### 5.4 Database Backup Strategy

```bash
#!/bin/bash
# MongoDB Backup Cron Job
0 2 * * * /usr/bin/mongodump --out /backups/mongodb-$(date +\%Y\%m\%d) --gzip

# MariaDB Backup Cron Job
0 2 * * * mysqldump -u root -p'password' --all-databases | gzip > /backups/mariadb-$(date +\%Y\%m\%d).sql.gz

# Upload to S3 (or NFS shared storage)
0 3 * * * aws s3 sync /backups s3://company-backups/database/
```

---

## 6. APPLICATION DEPLOYMENT

### 6.1 Helm Chart Structure

```
myapp-helm/
├── Chart.yaml
├── values.yaml
├── values-prod.yaml
├── values-staging.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   └── networkpolicy.yaml
```

### 6.2 Example Deployment Helm Values

```yaml
# values.yaml
replicaCount: 3

image:
  repository: quay.io/company/myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - myapp.example.com

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: db.host
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secrets
        key: db.password

livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

### 6.3 Deployment Commands

```bash
#!/bin/bash
# Create namespaces
kubectl create namespace production
kubectl create namespace staging
kubectl create namespace development

# Deploy to staging
helm install myapp ./myapp-helm \
  -f ./myapp-helm/values-staging.yaml \
  -n staging

# Deploy to production
helm install myapp ./myapp-helm \
  -f ./myapp-helm/values-prod.yaml \
  -n production

# Upgrade application
helm upgrade myapp ./myapp-helm \
  -f ./myapp-helm/values-prod.yaml \
  -n production
```

---

## 7. SECURITY IMPLEMENTATION

### 7.1 Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: data-layer
      ports:
        - protocol: TCP
          port: 27017  # MongoDB
    - to:
        - namespaceSelector:
            matchLabels:
              name: data-layer
      ports:
        - protocol: TCP
          port: 3306   # MariaDB
    - to:
        - podSelector: {}
      ports:
        - protocol: TCP
          port: 53     # DNS
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: TCP
          port: 443    # HTTPS
```

### 7.2 RBAC Configuration

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-role-binding
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: myapp-role
subjects:
  - kind: ServiceAccount
    name: myapp-sa
    namespace: production
```

### 7.3 Pod Security Policies

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'MustRunAs'
    seLinuxOptions:
      level: "s0:c123,c456"
  fsGroup:
    rule: 'MustRunAs'
    fsGroupOptions:
      ranges:
        - min: 1
          max: 65535
  readOnlyRootFilesystem: true
```

### 7.4 OIDC/KeyCloak Integration

```bash
#!/bin/bash
# Install KeyCloak via Helm
helm repo add keycloak https://codecentric.github.io/helm-charts
helm repo update

# Create values file
cat << EOF > keycloak-values.yaml
image:
  repository: keycloak/keycloak
  tag: 20.0.0

auth:
  adminUser: admin
  adminPassword: secure_admin_password

postgresql:
  enabled: true
  auth:
    password: postgres_password

ingress:
  enabled: true
  hosts:
    - keycloak.example.com
EOF

# Install
helm install keycloak keycloak/keycloak \
  -f keycloak-values.yaml \
  -n keycloak \
  --create-namespace
```

---

## 8. MONITORING & LOGGING

### 8.1 Prometheus Installation

```bash
#!/bin/bash
# Add Prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create values file
cat << EOF > prometheus-values.yaml
prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 50Gi

grafana:
  enabled: true
  adminPassword: grafana_secure_password
  persistence:
    enabled: true
    size: 10Gi

alertmanager:
  enabled: true
  config:
    route:
      group_by: ['alertname', 'cluster']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      receiver: 'team-channel'
    receivers:
      - name: 'team-channel'
        slack_configs:
          - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
            channel: '#alerts'
EOF

# Install
helm install prometheus prometheus-community/kube-prometheus-stack \
  -f prometheus-values.yaml \
  -n monitoring \
  --create-namespace
```

### 8.2 ELK Stack for Logging

```bash
#!/bin/bash
# Install Elasticsearch
helm repo add elastic https://Helm.elastic.co
helm repo update

# Create namespace
kubectl create namespace logging

# Install Elasticsearch
helm install elasticsearch elastic/elasticsearch \
  -n logging \
  --set replicas=3 \
  --set persistence.size=30Gi

# Install Kibana
helm install kibana elastic/kibana \
  -n logging \
  --set elasticsearchHosts=http://elasticsearch-master:9200

# Install Logstash/Filebeat for log collection
helm install filebeat elastic/filebeat \
  -n logging
```

### 8.3 Application Health Checks

```yaml
# Configure in deployment
livenessProbe:
  httpGet:
    path: /api/health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /api/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3

startupProbe:
  httpGet:
    path: /api/startup
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

---

## 9. DISASTER RECOVERY

### 9.1 Backup Strategy

```bash
#!/bin/bash
# Install Velero for Kubernetes backup
wget https://github.com/vmware-tanzu/velero/releases/download/v1.11.0/velero-v1.11.0-linux-amd64.tar.gz
tar xzf velero-v1.11.0-linux-amd64.tar.gz

# Create S3 bucket for backups
aws s3 mb s3://company-k8s-backups

# Install Velero
./velero install \
  --secret-file ./credentials-velero \
  --bucket company-k8s-backups \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.7.0

# Create scheduled backup
velero create schedule production-daily \
  --schedule="0 2 * * *" \
  --include-namespaces production
```

### 9.2 Database Backup

```bash
#!/bin/bash
# MongoDB backup script
#!/bin/bash
BACKUP_DIR="/backups/mongodb"
DATE=$(date +%Y%m%d_%H%M%S)

mongodump \
  --uri="mongodb://user:password@10.0.3.10:27017/appdb?authSource=admin" \
  --out=$BACKUP_DIR/backup_$DATE \
  --gzip

# Upload to S3
aws s3 sync $BACKUP_DIR s3://company-backups/mongodb/

# MariaDB backup
BACKUP_DIR="/backups/mariadb"
DATE=$(date +%Y%m%d_%H%M%S)

mysqldump \
  -h 10.0.3.20 \
  -u appuser \
  -p'secure_password_123' \
  --single-transaction \
  --quick \
  appdb | gzip > $BACKUP_DIR/backup_$DATE.sql.gz

# Upload to S3
aws s3 sync $BACKUP_DIR s3://company-backups/mariadb/
```

### 9.3 Restore Procedures

```bash
#!/bin/bash
# Restore Kubernetes cluster
velero restore create --from-schedule production-daily

# Restore MongoDB
mongorestore \
  --uri="mongodb://user:password@10.0.3.10:27017/?authSource=admin" \
  --gzip \
  /backups/mongodb/backup_20240101_120000/

# Restore MariaDB
mysql \
  -h 10.0.3.20 \
  -u root \
  -p'root_password' \
  appdb < /backups/mariadb/backup_20240101_120000.sql
```

---

## 10. TROUBLESHOOTING GUIDE

### 10.1 Common Issues & Solutions

**Issue: Pods not starting**

```bash
# Check pod status
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>

# Check node resources
kubectl top nodes
kubectl top pods -n <namespace>

# Check persistent volume
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>
```

**Issue: Network connectivity problems**

```bash
# Test pod-to-pod connectivity
kubectl exec -it <pod-1> -n <namespace> -- ping <service-name>

# Check Calico
kubectl get daemonset -n calico-system

# Check DNS
kubectl exec -it <pod> -n <namespace> -- nslookup kubernetes.default
```

**Issue: Database connection failures**

```bash
# Test MongoDB connection
kubectl run -it --rm debug --image=mongo:5.0 --restart=Never -- mongo mongodb://user:password@mongodb-service:27017/appdb

# Test MariaDB connection
kubectl run -it --rm debug --image=mariadb:10.6 --restart=Never -- mysql -h mariadb-service -u appuser -p -e "select 1;"
```

**Issue: High memory usage**

```bash
# Find memory hogs
kubectl top pods -n <namespace> --sort-by=memory

# Check for memory leaks
kubectl logs <pod-name> -n <namespace> --tail=100

# Increase resource limits
kubectl set resources deployment <name> -n <namespace> --limits=memory=1Gi
```

### 10.2 Monitoring Commands

```bash
#!/bin/bash
# Cluster health
kubectl cluster-info
kubectl get nodes -o wide
kubectl get componentstatuses

# Resource usage
kubectl top nodes
kubectl top pods -A

# Check events
kubectl get events -A --sort-by='.lastTimestamp'

# Network policies
kubectl get networkpolicies -A

# RBAC audit
kubectl get clusterroles | grep admin
kubectl get rolebindings -A
```

---

## IMPLEMENTATION TIMELINE

**Week 1:**

- Proxmox setup and VM creation
- Kubernetes cluster initialization
- CNI plugin installation

**Week 2:**

- CI/CD pipeline setup (Jenkins, ArgoCD)
- Database installation and configuration
- Backup and disaster recovery setup

**Week 3:**

- Security implementation (RBAC, Network Policies, OIDC)
- Application deployment
- Testing and validation

**Week 4:**

- Monitoring and logging setup
- Performance tuning
- Documentation and handover

---

## FINAL CHECKLIST

- [ ] All VMs created and networking configured
- [ ] Kubernetes cluster running with all nodes healthy
- [ ] HAProxy load balancer functioning
- [ ] Jenkins pipeline working end-to-end
- [ ] ArgoCD synchronized with Git repositories
- [ ] All databases running and accessible
- [ ] Monitoring (Prometheus/Grafana) active
- [ ] Logging (ELK) collecting all logs
- [ ] Backup jobs running and tested
- [ ] Security policies implemented
- [ ] OIDC authentication working
- [ ] Disaster recovery tested
- [ ] Documentation complete
- [ ] Team trained on all systems
- [ ] Production cutover successful

---

## SUPPORT & RESOURCES

**Kubernetes Documentation:** https://kubernetes.io/docs/ **Helm:** https://helm.sh/ **Proxmox:** https://www.proxmox.com/en/proxmox-ve/documentation **Jenkins:** https://www.jenkins.io/doc/ **ArgoCD:** https://argo-cd.readthedocs.io/ **Prometheus:** https://prometheus.io/docs/ **ELK Stack:** https://www.elastic.co/guide/

---

**Document Version:** 1.0 **Last Updated:** January 2024 **Next Review:** April 2024
