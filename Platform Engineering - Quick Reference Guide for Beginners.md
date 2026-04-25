# Platform Engineering - Quick Reference Guide for Beginners

**For**: Team members new to Kubernetes, Proxmox, and cloud-native architecture \
**Goal**: Answer your most common questions in simple terms

---

## YOUR QUESTIONS ANSWERED

### "What is all this, and why do we need it?"

You're building a **private cloud** - basically, a smaller version of AWS/Google Cloud running on your own hardware. Instead of paying cloud providers, you:

- Keep data in-house (security)
- Save costs at scale
- Have full control

Think of it like this:

- **Proxmox** = The foundation (like the actual data center servers)
- **Kubernetes** = The orchestrator (decides where applications run)
- **CI/CD** = The assembly line (automatically builds, tests, and deploys code)
- **Databases** = Where data lives (MongoDB, MariaDB)
- **Monitoring** = Your dashboard (sees if anything is broken)

---

## PART 1: UNDERSTANDING THE ARCHITECTURE

### Layer 1: Hardware (Proxmox)

```
Your Physical Servers (with Proxmox)
  ├── Virtual Machine: Master-1 (Kubernetes controller)
  ├── Virtual Machine: Master-2 (Kubernetes controller)
  ├── Virtual Machine: Master-3 (Kubernetes controller)
  ├── Virtual Machine: Worker-1 (runs apps)
  ├── Virtual Machine: Worker-2 (runs apps)
  ├── Virtual Machine: Worker-3 (runs apps)
  ├── Virtual Machine: MongoDB (stores data)
  ├── Virtual Machine: MariaDB (stores data)
  └── Virtual Machine: Redis (fast cache)
```

**Why multiple masters?** If one goes down, the cluster keeps working (High Availability - HA).

### Layer 2: Kubernetes (The Orchestrator)

Kubernetes is like a **smart scheduler** for containers.

You say: "I want 3 copies of my app running" Kubernetes does: "Okay, I'll put one on Worker-1, one on Worker-2, one on Worker-3. If Worker-1 crashes, I'll move it to Worker-3."

**Key concepts:**

- **Pod**: Smallest unit (usually = 1 container)
- **Deployment**: Says "I want 3 replicas of this app"
- **Service**: A stable address so apps can find each other
- **Namespace**: Like folders (separates staging from production)
- **ConfigMap**: Configuration files
- **Secret**: Passwords, API keys (stored securely)

### Layer 3: Applications

Your React/Next.js frontend talks to:

- API Gateway (HAProxy) → Routes traffic to backend
- Backend services (Node.js, Python, etc.) running in Kubernetes
- Databases (MongoDB, MariaDB)
- Cache (Redis)

---

## PART 2: OIDC & AUTHENTICATION

### What is OIDC?

**OIDC** = OpenID Connect = A standard way for users to log in.

Instead of:

- You managing 100 user passwords
- Users forgetting passwords

OIDC lets users:

- Click "Login with SSO"
- Use their company account
- Automatically authenticated everywhere

**How it works:**

```
User clicks "Login"
  ↓
App redirects to KeyCloak (our identity server)
  ↓
User enters username/password (only here, once)
  ↓
KeyCloak says "Yes, that's them" and gives a token
  ↓
App gets the token, user is now logged in
  ↓
Token works everywhere (no more password prompts)
```

### Implementation Steps:

1. **Install KeyCloak** (identity server)

```bash
helm install keycloak keycloak/keycloak \
  --set auth.adminUser=admin \
  --set auth.adminPassword=secure_password
```

2. **Create a "Realm" in KeyCloak** (like a project)
    
    - Go to KeyCloak admin console
    - Create realm called "Company"
3. **Create a Client** (your app)
    
    - Client ID: "myapp"
    - Valid redirect URI: "https://myapp.example.com/callback"
4. **Connect your app**
    

```javascript
// Example: React app with OIDC library
import { AuthProvider, useAuth } from 'react-oidc-context';

const oidcConfig = {
  authority: 'https://keycloak.example.com/realms/Company',
  client_id: 'myapp',
  redirect_uri: window.location.origin,
};

function App() {
  return (
    <AuthProvider config={oidcConfig}>
      <YourApp />
    </AuthProvider>
  );
}
```

---

## PART 3: HELM - PACKAGE MANAGER FOR KUBERNETES

### What is Helm?

Think of it like **Docker for Kubernetes manifests**.

Instead of writing:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
kubectl apply -f pvc.yaml
kubectl apply -f configmap.yaml
# ... 10 more files
```

With Helm, you write one command:

```bash
helm install myapp ./myapp-chart
```

And all 15 files deploy automatically.

### Helm Chart Structure:

```
myapp-chart/
├── Chart.yaml              # Metadata (version, description)
├── values.yaml             # Default configuration
├── values-prod.yaml        # Production-specific settings
├── values-staging.yaml     # Staging-specific settings
└── templates/
    ├── deployment.yaml     # How to run the app
    ├── service.yaml        # How to access it
    ├── ingress.yaml        # External routes
    ├── configmap.yaml      # Configuration
    ├── secret.yaml         # Passwords/keys
    └── hpa.yaml           # Auto-scaling rules
```

### Using Helm:

```bash
# Install
helm install myapp ./myapp-chart -f values-prod.yaml

# Update (like pushing code)
helm upgrade myapp ./myapp-chart -f values-prod.yaml

# Rollback (like git revert)
helm rollback myapp 1

# List all deployments
helm list

# Uninstall
helm uninstall myapp
```

---

## PART 4: VOLUMES & PERSISTENT STORAGE

### Problem: Apps need to remember data

```
Container deletes → Data is GONE ❌
We need: Data that survives container restarts ✅
```

### Types of Storage:

1. **emptyDir**: Temporary storage (deleted with pod)

```yaml
volumes:
- name: cache
  emptyDir: {}
# Use for: Temporary logs, caches, scratch space
```

2. **ConfigMap**: Configuration files

```yaml
volumes:
- name: config
  configMap:
    name: app-config
# Use for: Application settings, config files
```

3. **Secret**: Passwords, API keys

```yaml
volumes:
- name: secrets
  secret:
    secretName: app-secrets
# Use for: Database passwords, API keys, certificates
```

4. **PersistentVolume (PV)**: Real storage

```yaml
# Storage exists on a physical disk
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-storage
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  local:
    path: /mnt/storage
```

5. **PersistentVolumeClaim (PVC)**: "I need 50GB"

```yaml
# Pod says "Give me 50GB of storage"
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

### How it works:

```
Pod says: "I need storage"
  ↓
Kubernetes looks at available PersistentVolumes
  ↓
Finds one big enough
  ↓
Connects it to the pod
  ↓
Pod can now write files that survive restarts
```

---

## PART 5: TRAFFIC FLOW - How Data Moves

### A user visits your website:

```
User's browser
  ↓ (DNS: myapp.example.com → 1.2.3.4)
HAProxy Load Balancer (port 80/443)
  ↓ (distributes load across multiple backend services)
Kubernetes Service "myapp" 
  ↓ (routes to healthy pods)
Pod 1: myapp running on Worker-1
Pod 2: myapp running on Worker-2
Pod 3: myapp running on Worker-3
  ↓ (if pod needs data)
MongoDB or MariaDB
  ↓ (cached for speed)
Redis Cache
```

### Database Traffic:

```
API Pod says: "Get user data for ID 123"
  ↓
Looks in Redis cache first (super fast)
  ↓
If not found, queries MariaDB
  ↓
MariaDB returns data
  ↓
API puts it in Redis (remember for next time)
  ↓
Sends response to user
```

### Monitoring Traffic:

```
Prometheus scrapes metrics from all pods
  ↓
Every 30 seconds, collects:
  - CPU usage
  - Memory usage
  - Request count
  - Error count
  ↓
Stores in time-series database
  ↓
Grafana visualizes it (pretty dashboards)
  ↓
If error rate > 5%, alert triggers
  ↓
Slack notification sent to team
```

---

## PART 6: SECURITY - MULTIPLE LAYERS

### Layer 1: OIDC Authentication

✅ Only logged-in users can access

```
User → OIDC → Verified identity token
```

### Layer 2: RBAC (Role-Based Access Control)

✅ Users only see what they should

Example: New dev can deploy to staging, but NOT production

```yaml
Role: developer
  - Can: deploy to staging namespace
  - Cannot: access production
```

### Layer 3: Network Policies

✅ Pods can only talk to the pods they need

Example: Frontend CANNOT directly contact database

```yaml
Frontend Pod → Can talk to → API Pod
API Pod → Can talk to → Database Pod
Database Pod → CANNOT talk to → Frontend (blocked!)
```

### Layer 4: Pod Security

✅ Pods run with minimum permissions

```yaml
securityContext:
  runAsNonRoot: true        # Don't run as root (safer)
  allowPrivilegeEscalation: false  # Can't get more permissions
  readOnlyRootFilesystem: true     # Can't modify /
  capabilities:
    drop:
      - ALL               # No special powers
```

### Layer 5: Secrets Management

✅ Passwords never in code

```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secrets
      key: password
```

Password is stored encrypted in Kubernetes, only decrypted when pod needs it.

---

## PART 7: PROVISIONING & DEPLOYMENT

### "How do I deploy my app?"

**Traditional way (OLD):**

1. SSH into server
2. Download code
3. Manually start service
4. Pray it works

**Modern way (NEW) - GitOps:**

1. Push code to Git
2. Jenkins automatically tests it
3. If tests pass, builds container image
4. Pushes image to container registry
5. ArgoCD automatically deploys to Kubernetes

### The Process:

```
Developer commits code
  ↓ (GitHub webhook triggers)
Jenkins Pipeline Starts
  ├─ Build: npm run build (create app files)
  ├─ Test: npm run test (verify it works)
  ├─ Scan: Sonarqube (check code quality)
  ├─ Security: Trivy (scan for vulnerabilities)
  ├─ Docker Build: docker build (create container)
  ├─ Push: push to Quay registry
  └─ Update: update Git with new image version
  ↓ (ArgoCD watches Git)
ArgoCD Detects Change
  ├─ Git says: "Deploy image v1.5.0"
  ├─ Kubernetes currently running v1.4.0
  └─ ArgoCD: "I'll fix that"
  ↓
Kubernetes Update
  ├─ Stop old pods (v1.4.0)
  ├─ Start new pods (v1.5.0)
  └─ Health check: "Is it working?"
  ↓
Prometheus Monitoring
  ├─ Check error rate
  ├─ Check response time
  └─ If problems: automatic rollback to v1.4.0
```

### Provisioning From Scratch:

1. **Infrastructure** (Week 1)

```bash
# Proxmox: Create 3 master + 3 worker VMs
# Setup networking
# Install OS
```

2. **Kubernetes** (Week 1-2)

```bash
# kubeadm init (on master 1)
# kubeadm join (on master 2,3 and workers)
# Install Calico networking
# Install Helm
```

3. **Infrastructure Services** (Week 2)

```bash
# Jenkins (for CI)
# ArgoCD (for CD)
# Prometheus + Grafana (for monitoring)
# Elasticsearch + Kibana (for logging)
# KeyCloak (for authentication)
```

4. **Databases** (Week 2)

```bash
# MongoDB (create VMs or run in K8s)
# MariaDB (create VMs or run in K8s)
# Redis (for caching)
# Setup backups
```

5. **Your Application** (Week 3)

```bash
# Create Helm chart
# Create staging namespace
# Deploy to staging
# Test
# Deploy to production
```

---

## PART 8: TROUBLESHOOTING

### "My pod won't start!"

```bash
# Step 1: What's wrong?
kubectl describe pod <pod-name> -n production

# Step 2: Check logs
kubectl logs <pod-name> -n production

# Step 3: Check resources
kubectl top pods -n production

# Step 4: Check events
kubectl get events -n production

# Fix: Usually one of:
# - Image pull error: Fix Docker credentials
# - Insufficient resources: Add more worker nodes
# - ConfigMap/Secret missing: Create them
# - Health check failing: Fix app
```

### "My app is slow!"

```bash
# Check CPU usage
kubectl top pods -n production --sort-by=cpu

# Check memory usage
kubectl top pods -n production --sort-by=memory

# Check request latency
# Go to Grafana → Search for "request duration"

# Solutions:
# - Increase resource limits
# - Optimize database queries
# - Enable caching (Redis)
# - Scale to more replicas (HPA)
```

### "Database connection failing!"

```bash
# Test connection from pod
kubectl run -it --rm debug --image=ubuntu:latest --restart=Never -- bash
apt update && apt install -y mariadb-client
mysql -h mariadb.data-layer.svc.cluster.local -u user -p

# If fails:
# 1. Check database pod is running: kubectl get pods -n data-layer
# 2. Check credentials in ConfigMap/Secret
# 3. Check Network Policy isn't blocking traffic
# 4. Check database service exists: kubectl get svc -n data-layer
```

---

## PART 9: COMMON COMMANDS CHEAT SHEET

```bash
# Basic cluster info
kubectl cluster-info
kubectl get nodes
kubectl top nodes

# Deployments
kubectl get deployment -n production
kubectl describe deployment myapp -n production
kubectl logs deployment/myapp -n production

# Pods
kubectl get pods -n production
kubectl describe pod <pod-name> -n production
kubectl logs <pod-name> -n production
kubectl exec -it <pod-name> -n production -- /bin/bash

# Services & Networking
kubectl get svc -n production
kubectl get ingress -n production
kubectl port-forward svc/myapp 8080:80 -n production  # Access locally

# Configs & Secrets
kubectl get configmaps -n production
kubectl create secret generic db-password --from-literal=password=secret123 -n production

# Scaling
kubectl scale deployment myapp --replicas=5 -n production
kubectl autoscale deployment myapp --min=2 --max=10 --cpu-percent=70 -n production

# Updates
kubectl rollout status deployment/myapp -n production
kubectl rollout history deployment/myapp -n production
kubectl rollout undo deployment/myapp -n production

# Helm
helm install myapp ./chart -f values.yaml
helm upgrade myapp ./chart -f values.yaml
helm rollback myapp 1
helm list -n production
helm uninstall myapp -n production

# Debugging
kubectl describe node <node-name>           # Check node issues
kubectl get events -A --sort-by='.lastTimestamp'  # See what happened
kubectl top pods -A --sort-by=memory        # Memory hogs
```

---

## PART 10: LEARNING RESOURCES

**Official Documentation:**

- Kubernetes Basics: https://kubernetes.io/docs/tutorials/kubernetes-basics/
- Helm Docs: https://helm.sh/docs/
- Proxmox Docs: https://pve.proxmox.com/pve-docs/

**Free Courses:**

- Kubernetes for Beginners: https://www.youtube.com/@TechWorldwithNana (YouTube)
- Docker & Kubernetes: Udemy courses
- Kubernetes Architecture: Pluralsight

**Tools to Practice:**

- Minikube (local Kubernetes): https://minikube.sigs.k8s.io/
- Kind (Docker-based K8s): https://kind.sigs.k8s.io/
- Killercoda (interactive labs): https://killercoda.com/

**Key Takeaways:**

1. **Kubernetes is a scheduler** - tells where to run apps
2. **Helm is a package manager** - automates Kubernetes configuration
3. **OIDC is authentication** - lets users securely log in
4. **Volumes are storage** - makes data survive restarts
5. **Network policies restrict traffic** - only pods that need to talk can talk
6. **GitOps automates deployment** - code push automatically deploys
7. **Monitoring tells you when things break** - metrics + alerts + dashboards
8. **Everything is version controlled** - Git is your source of truth

**Next Steps:**

1. ✅ Read this guide (you're doing it!)
2. ✅ Understand the three diagrams above
3. ✅ Follow the implementation guide step-by-step
4. ✅ Practice with Minikube locally first
5. ✅ Deploy to staging, test thoroughly
6. ✅ Deploy to production with confidence
7. ✅ Monitor and optimize based on metrics
8. ✅ Document your learnings for the team

---

**Remember:** You're not expected to know everything. The cloud-native world is vast. This guide gives you the 20% that solves 80% of problems. For the rest, Google is your friend. Ask questions. Read docs. Practice. You've got this!

**Questions?** Check the full implementation guide for detailed steps on each component.
