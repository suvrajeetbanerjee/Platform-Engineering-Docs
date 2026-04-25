# KUBERNETES MANIFESTS & CONFIGURATION FILES REPOSITORY

## FILE: deployment-template.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: myapp-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
      - name: myapp
        image: quay.io/company/myapp:latest
        imagePullPolicy: IfNotPresent
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        env:
        - name: LOG_LEVEL
          value: "INFO"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: db.host
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: db.port
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db.user
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db.password
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: redis.url
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /startup
            port: http
          failureThreshold: 30
          periodSeconds: 10
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
          readOnly: true
        - name: cache-volume
          mountPath: /var/cache/app
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          readOnlyRootFilesystem: true
      volumes:
      - name: config-volume
        configMap:
          name: app-config
      - name: cache-volume
        emptyDir: {}
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - myapp
              topologyKey: kubernetes.io/hostname
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
              - key: node-role.kubernetes.io/worker
                operator: Exists
      tolerations:
      - key: workload
        operator: Equal
        value: general
        effect: NoSchedule
```

## FILE: service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: http
    protocol: TCP
    name: http
  selector:
    app: myapp
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

## FILE: ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    - api.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
```

## FILE: hpa.yaml

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Max
```

## FILE: configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  db.host: "mariadb.data-layer.svc.cluster.local"
  db.port: "3306"
  redis.url: "redis://redis.data-layer.svc.cluster.local:6379"
  log.level: "INFO"
  app.env: "production"
  app.version: "1.0.0"
```

## FILE: secrets.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
data:
  db.user: YXBwdXNlcg==  # base64 encoded: appuser
  db.password: c2VjdXJlX3Bhc3N3b3JkXzEyMw==  # base64 encoded: secure_password_123
  redis.password: cmVkaXNfc2VjdXJlX3Bhc3N3b3Jk  # base64 encoded: redis_secure_password
---
apiVersion: v1
kind: Secret
metadata:
  name: regcred
  namespace: production
type: docker.io/basic-auth
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

## FILE: networkpolicy.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: myapp-netpol
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
      port: 3306
    - protocol: TCP
      port: 27017
    - protocol: TCP
      port: 6379
  - to:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 443
```

## FILE: rbac.yaml

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
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]
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

## FILE: persistentvolume.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-pv-data
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  local:
    path: /mnt/storage/app-data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: storage
          operator: In
          values:
          - enabled
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-pvc-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 50Gi
```

## FILE: statefulset.yaml

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: myapp-stateful
  namespace: production
spec:
  serviceName: myapp-headless
  replicas: 3
  selector:
    matchLabels:
      app: myapp-stateful
  template:
    metadata:
      labels:
        app: myapp-stateful
    spec:
      containers:
      - name: myapp
        image: quay.io/company/myapp:latest
        ports:
        - containerPort: 8080
          name: web
        volumeMounts:
        - name: data
          mountPath: /var/data
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: local-path
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-headless
  namespace: production
spec:
  clusterIP: None
  selector:
    app: myapp-stateful
  ports:
  - port: 8080
    targetPort: 8080
```

## FILE: daemonset.yaml

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true
      hostPID: true
      containers:
      - name: node-exporter
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
        args:
          - --path.sysfs=/host/sys
          - --path.procfs=/host/proc
        volumeMounts:
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: proc
          mountPath: /host/proc
          readOnly: true
      tolerations:
      - effect: NoSchedule
        operator: Exists
      volumes:
      - name: sys
        hostPath:
          path: /sys
      - name: proc
        hostPath:
          path: /proc
```

## FILE: cronjob.yaml

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
  namespace: production
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          serviceAccountName: backup-sa
          containers:
          - name: backup
            image: quay.io/company/backup-tool:latest
            env:
            - name: BACKUP_DESTINATION
              value: s3://company-backups
            - name: DB_HOST
              value: mariadb.data-layer.svc.cluster.local
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: access-key
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: secret-key
            resources:
              requests:
                cpu: 500m
                memory: 512Mi
              limits:
                cpu: 1000m
                memory: 1Gi
          restartPolicy: OnFailure
```

## FILE: servicemonitor.yaml

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: production
  labels:
    app: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: http
    interval: 30s
    path: /metrics
```

## FILE: prometheusrule.yaml

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-rules
  namespace: production
spec:
  groups:
  - name: myapp.rules
    interval: 30s
    rules:
    - alert: HighErrorRate
      expr: 'rate(myapp_errors_total[5m]) > 0.1'
      for: 5m
      annotations:
        summary: "High error rate on {{ $labels.instance }}"
        description: "Error rate is {{ $value }}"
    
    - alert: HighLatency
      expr: 'myapp_request_duration_seconds_p99 > 1.0'
      for: 5m
      annotations:
        summary: "High latency on {{ $labels.instance }}"
    
    - alert: PodRestartingTooOften
      expr: 'rate(kube_pod_container_status_restarts_total[15m]) > 0.1'
      for: 5m
      annotations:
        summary: "Pod {{ $labels.pod }} is restarting too often"
    
    - alert: HighMemoryUsage
      expr: 'container_memory_usage_bytes{pod=~"myapp.*"} / container_spec_memory_limit_bytes > 0.9'
      for: 5m
      annotations:
        summary: "High memory usage on {{ $labels.pod }}"
    
    - alert: NodeNotReady
      expr: 'kube_node_status_condition{condition="Ready",status="true"} == 0'
      for: 10m
      annotations:
        summary: "Node {{ $labels.node }} is not ready"
```

## FILE: setup-cluster.sh

```bash
#!/bin/bash
set -e

echo "=== Setting up Kubernetes cluster ==="

# Configuration
NAMESPACE=${1:-default}
CHART_REPO="https://charts.helm.sh/stable"

# Create namespace
kubectl create namespace $NAMESPACE || true

# Install Ingress Controller (nginx)
echo "Installing Nginx Ingress Controller..."
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer

# Install Cert-Manager
echo "Installing Cert-Manager..."
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Create Let's Encrypt Issuer
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

# Install Prometheus Stack
echo "Installing Prometheus Stack..."
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values - <<EOF
prometheus:
  prometheusSpec:
    retention: 30d
grafana:
  adminPassword: grafana_secure_password
alertmanager:
  enabled: true
EOF

echo "=== Cluster setup complete ==="
echo "Ingress Controller: $(kubectl get svc -n ingress-nginx nginx-ingress-ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
echo "Prometheus: kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090"
echo "Grafana: kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80"
```

## FILE: health-check.sh

```bash
#!/bin/bash

echo "=== Kubernetes Cluster Health Check ==="

# Check cluster info
echo "Cluster Info:"
kubectl cluster-info
kubectl version --short

# Check node status
echo -e "\n=== Node Status ==="
kubectl get nodes -o wide
kubectl top nodes

# Check pod status
echo -e "\n=== Pod Status ==="
kubectl get pods -A -o wide
kubectl get pods -A -o json | jq '.items[] | select(.status.phase != "Running")'

# Check component status
echo -e "\n=== System Components ==="
kubectl get componentstatuses

# Check system namespace
echo -e "\n=== Kube-System Resources ==="
kubectl get all -n kube-system

# Check monitoring
echo -e "\n=== Monitoring ==="
kubectl get all -n monitoring

# Check persistent volumes
echo -e "\n=== Storage ==="
kubectl get pv
kubectl get pvc -A

# Check resource usage
echo -e "\n=== Resource Usage ==="
kubectl top pods -A --containers | head -20

# Check events
echo -e "\n=== Recent Events ==="
kubectl get events -A --sort-by='.lastTimestamp' | tail -10

echo -e "\n=== Health Check Complete ==="
```

## FILE: backup-databases.sh

```bash
#!/bin/bash

BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)
LOG_FILE="$BACKUP_DIR/backup_$DATE.log"

{
  echo "Starting database backup at $(date)"
  
  # MongoDB backup
  echo "Backing up MongoDB..."
  mongodump \
    --uri="mongodb://user:password@mongodb.data-layer.svc.cluster.local:27017/appdb?authSource=admin" \
    --out="$BACKUP_DIR/mongodb_$DATE" \
    --gzip 2>&1 || echo "MongoDB backup failed"
  
  # MariaDB backup
  echo "Backing up MariaDB..."
  mysqldump \
    -h mariadb.data-layer.svc.cluster.local \
    -u appuser \
    -p'password' \
    --single-transaction \
    --quick \
    appdb | gzip > "$BACKUP_DIR/mariadb_$DATE.sql.gz" 2>&1 || echo "MariaDB backup failed"
  
  # Upload to S3
  echo "Uploading to S3..."
  aws s3 sync "$BACKUP_DIR" "s3://company-backups/database/$DATE/" \
    --exclude "*" \
    --include "*_$DATE*" 2>&1 || echo "S3 upload failed"
  
  # Cleanup old backups (keep last 7 days)
  echo "Cleaning up old backups..."
  find "$BACKUP_DIR" -type d -name "*_*" -mtime +7 -exec rm -rf {} \; 2>&1 || true
  
  echo "Backup completed at $(date)"
  
} | tee "$LOG_FILE"
```

## FILE: update-applications.sh

```bash
#!/bin/bash

NAMESPACE=${1:-production}
DRY_RUN=${2:-false}

echo "=== Updating applications in namespace: $NAMESPACE ==="

# Pull latest images
kubectl set image deployment -n $NAMESPACE \
  "*" \
  "*=$(docker inspect --format='{{index .RepoDigests 0}}' $IMAGE)" \
  --dry-run=$DRY_RUN \
  --record

# Rollout status
kubectl rollout status deployment -n $NAMESPACE

echo "=== Update complete ==="
```
