# 💻 Coffee系統階段5：開發編譯測試部署

## 🎯 **階段5目標：實際部署與生產環境配置**

**執行時間**: 2026-02-13 03:00  
**預計完成**: 2026-02-13 04:30 (90分鐘)

---

## 🏗️ **完整系統部署架構**

### **🌐 生產環境架構圖**

```
Coffee系統生產環境架構:

                    ┌─────────────────────────────────────┐
                    │          Load Balancer              │
                    │      (Nginx Ingress)               │
                    └─────────────────┬───────────────────┘
                                      │
                    ┌─────────────────┴───────────────────┐
                    │         API Gateway                 │
                    │      (coffee-gateway:8000)         │
                    └─────────────────┬───────────────────┘
                                      │
        ┌─────────────────────────────┼─────────────────────────────┐
        │                             │                             │
        ▼                             ▼                             ▼
┌───────────────┐            ┌───────────────┐            ┌───────────────┐
│   Platform    │            │   Business    │            │   Support     │
│  Services     │            │   Services    │            │   Services    │
├───────────────┤            ├───────────────┤            ├───────────────┤
│tenant:8000    │            │order:8010     │            │monitoring:8001│
│config:8006    │            │payment:8011   │            │analytics:8013 │
│billing:8007   │            │inventory:8012 │            │notification   │
└───────────────┘            └───────────────┘            └───────────────┘
        │                             │                             │
        └─────────────────────────────┼─────────────────────────────┘
                                      │
                    ┌─────────────────┴───────────────────┐
                    │           Data Layer                │
                    ├─────────────────┬───────────────────┤
                    │   PostgreSQL    │      Redis        │
                    │   (Primary DB)  │     (Cache)       │
                    ├─────────────────┼───────────────────┤
                    │    MongoDB      │    File Storage   │
                    │   (Audit Log)   │      (MinIO)      │
                    └─────────────────┴───────────────────┘
```

### **🔧 K8s集群配置**

#### **命名空間劃分**
```yaml
# k8s/namespaces.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: coffee-prod
  labels:
    environment: production
    team: coffee
---
apiVersion: v1
kind: Namespace
metadata:
  name: coffee-staging
  labels:
    environment: staging
    team: coffee
---
apiVersion: v1
kind: Namespace
metadata:
  name: coffee-monitoring
  labels:
    component: monitoring
    team: coffee
---
apiVersion: v1
kind: Namespace
metadata:
  name: coffee-ingress
  labels:
    component: ingress
    team: coffee
```

#### **資源配額與限制**
```yaml
# k8s/resource-quotas.yaml
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: coffee-prod-quota
  namespace: coffee-prod
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    persistentvolumeclaims: "10"
    pods: "50"
    services: "20"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: coffee-prod-limits
  namespace: coffee-prod
spec:
  limits:
  - default:
      cpu: "1"
      memory: 1Gi
    defaultRequest:
      cpu: "100m"
      memory: 256Mi
    type: Container
```

---

## 📊 **資料庫部署與配置**

### **🐘 PostgreSQL高可用部署**

#### **主從複製配置**
```yaml
# k8s/databases/postgresql.yaml
---
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres-cluster
  namespace: coffee-prod
spec:
  instances: 3
  
  postgresql:
    parameters:
      max_connections: "200"
      shared_buffers: "256MB"
      effective_cache_size: "1GB"
      maintenance_work_mem: "64MB"
      checkpoint_completion_target: "0.7"
      wal_buffers: "16MB"
      default_statistics_target: "100"
      random_page_cost: "1.1"
      effective_io_concurrency: "200"
    
  bootstrap:
    initdb:
      database: coffee_prod
      owner: coffee_user
      secret:
        name: postgres-credentials
        
  storage:
    size: 100Gi
    storageClass: fast-ssd
    
  monitoring:
    enabled: true
    
  backup:
    retentionPolicy: "30d"
    barmanObjectStore:
      destinationPath: "s3://coffee-backups/postgres"
      s3Credentials:
        accessKeyId:
          name: backup-credentials
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: backup-credentials
          key: SECRET_ACCESS_KEY
      wal:
        retention: "5d"
      data:
        retention: "30d"
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
  namespace: coffee-prod
type: kubernetes.io/basic-auth
stringData:
  username: coffee_user
  password: "your-secure-password-here"
```

### **🔴 Redis集群部署**

```yaml
# k8s/databases/redis.yaml
apiVersion: redis.redis.opstreelabs.in/v1beta1
kind: RedisCluster
metadata:
  name: redis-cluster
  namespace: coffee-prod
spec:
  clusterSize: 6
  clusterVersion: v7.0
  persistenceEnabled: true
  
  redisExporter:
    enabled: true
    image: oliver006/redis_exporter:latest
    
  storage:
    volumeClaimTemplate:
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi
        storageClassName: fast-ssd
        
  securityContext:
    runAsUser: 1000
    fsGroup: 1000
    
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values: ["redis-cluster"]
          topologyKey: kubernetes.io/hostname
```

### **🍃 MongoDB副本集部署**

```yaml
# k8s/databases/mongodb.yaml
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: mongodb-replica-set
  namespace: coffee-prod
spec:
  members: 3
  type: ReplicaSet
  version: "7.0.4"
  
  security:
    authentication:
      modes: ["SCRAM"]
    tls:
      enabled: false
      
  users:
  - name: coffee_admin
    db: admin
    passwordSecretRef:
      name: mongodb-admin-password
    roles:
    - name: clusterAdmin
      db: admin
    - name: userAdminAnyDatabase
      db: admin
    - name: readWriteAnyDatabase
      db: admin
      
  - name: coffee_app
    db: coffee_audit
    passwordSecretRef:
      name: mongodb-app-password
    roles:
    - name: readWrite
      db: coffee_audit
      
  additionalMongodConfig:
    storage.wiredTiger.engineConfig.journalCompressor: zlib
    storage.wiredTiger.collectionConfig.blockCompressor: snappy
    
  statefulSet:
    spec:
      volumeClaimTemplates:
      - metadata:
          name: data-volume
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
          storageClassName: fast-ssd
---
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-admin-password
  namespace: coffee-prod
type: Opaque
stringData:
  password: "mongodb-admin-secure-password"
---
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-app-password
  namespace: coffee-prod
type: Opaque
stringData:
  password: "mongodb-app-secure-password"
```

---

## 🚀 **微服務完整部署**

### **📦 微服務部署配置**

#### **Tenant Management Service**
```yaml
# k8s/services/tenant-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tenant-service
  namespace: coffee-prod
  labels:
    app: tenant-service
    tier: platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tenant-service
  template:
    metadata:
      labels:
        app: tenant-service
        tier: platform
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: coffee-service-account
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      containers:
      - name: tenant-service
        image: ghcr.io/lucas152112/coffee/tenant-service:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
          name: http
        env:
        - name: RUST_LOG
          value: "info"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: url
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: jwt-secret
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /home/coffee/.cache
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
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
                  values: ["tenant-service"]
              topologyKey: kubernetes.io/hostname
---
apiVersion: v1
kind: Service
metadata:
  name: tenant-service
  namespace: coffee-prod
  labels:
    app: tenant-service
spec:
  selector:
    app: tenant-service
  ports:
  - name: http
    port: 8000
    targetPort: 8000
    protocol: TCP
  type: ClusterIP
```

#### **Order Management Service (高併發配置)**
```yaml
# k8s/services/order-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: coffee-prod
  labels:
    app: order-service
    tier: business
spec:
  replicas: 5  # 高併發需要更多副本
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        tier: business
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8010"
    spec:
      containers:
      - name: order-service
        image: ghcr.io/lucas152112/coffee/order-service:latest
        ports:
        - containerPort: 8010
        env:
        - name: RUST_LOG
          value: "info"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-credentials
              key: url
        - name: WEBSOCKET_ENABLED
          value: "true"
        resources:
          requests:
            memory: "512Mi"
            cpu: "200m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8010
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8010
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: coffee-prod
spec:
  selector:
    app: order-service
  ports:
  - name: http
    port: 8010
    targetPort: 8010
  - name: websocket
    port: 8011
    targetPort: 8011
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: coffee-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 20
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
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

### **🌐 API Gateway與Ingress配置**

```yaml
# k8s/ingress/api-gateway.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: coffee-prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: api-gateway
        image: ghcr.io/lucas152112/coffee/api-gateway:latest
        ports:
        - containerPort: 8000
        env:
        - name: RUST_LOG
          value: "info"
        - name: UPSTREAM_SERVICES
          value: |
            tenant-service:8000,
            order-service:8010,
            payment-service:8011,
            inventory-service:8012,
            analytics-service:8013,
            monitoring-service:8001,
            config-service:8006,
            billing-service:8007
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
  namespace: coffee-prod
spec:
  selector:
    app: api-gateway
  ports:
  - port: 80
    targetPort: 8000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: coffee-ingress
  namespace: coffee-prod
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "1000"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    nginx.ingress.kubernetes.io/websocket-services: "order-service"
spec:
  tls:
  - hosts:
    - coffee.justit.cc
    - api.coffee.justit.cc
    secretName: coffee-tls
  rules:
  - host: coffee.justit.cc
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-gateway
            port:
              number: 80
  - host: api.coffee.justit.cc
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-gateway
            port:
              number: 80
```

---

## 📊 **監控與可觀測性實施**

### **🔍 Prometheus監控部署**

```yaml
# k8s/monitoring/prometheus.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: coffee-monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        ports:
        - containerPort: 9090
        args:
        - '--config.file=/etc/prometheus/prometheus.yml'
        - '--storage.tsdb.path=/prometheus/'
        - '--web.console.libraries=/etc/prometheus/console_libraries'
        - '--web.console.templates=/etc/prometheus/consoles'
        - '--web.enable-lifecycle'
        - '--storage.tsdb.retention.time=30d'
        volumeMounts:
        - name: prometheus-config-volume
          mountPath: /etc/prometheus/
        - name: prometheus-storage-volume
          mountPath: /prometheus/
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
      volumes:
      - name: prometheus-config-volume
        configMap:
          name: prometheus-config
      - name: prometheus-storage-volume
        persistentVolumeClaim:
          claimName: prometheus-storage
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-storage
  namespace: coffee-monitoring
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: fast-ssd
```

### **📈 Grafana儀表板部署**

```yaml
# k8s/monitoring/grafana.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: coffee-monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 3000
        env:
        - name: GF_SECURITY_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: grafana-credentials
              key: admin-password
        - name: GF_DATABASE_TYPE
          value: "postgres"
        - name: GF_DATABASE_HOST
          value: "postgres-cluster-rw:5432"
        - name: GF_DATABASE_NAME
          value: "grafana"
        - name: GF_DATABASE_USER
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: grafana-user
        - name: GF_DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-credentials
              key: grafana-password
        volumeMounts:
        - name: grafana-storage
          mountPath: /var/lib/grafana
        - name: grafana-config
          mountPath: /etc/grafana/provisioning
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: grafana-storage
        persistentVolumeClaim:
          claimName: grafana-storage
      - name: grafana-config
        configMap:
          name: grafana-config
```

---

## 🔒 **安全配置實施**

### **🛡️ RBAC權限配置**

```yaml
# k8s/security/rbac.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coffee-service-account
  namespace: coffee-prod
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: coffee-prod
  name: coffee-service-role
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: coffee-service-rolebinding
  namespace: coffee-prod
subjects:
- kind: ServiceAccount
  name: coffee-service-account
  namespace: coffee-prod
roleRef:
  kind: Role
  name: coffee-service-role
  apiGroup: rbac.authorization.k8s.io
```

### **🔐 Network Policy網路隔離**

```yaml
# k8s/security/network-policies.yaml
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: coffee-network-policy
  namespace: coffee-prod
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: coffee-ingress
    - podSelector:
        matchLabels:
          app: api-gateway
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 8000
    - protocol: TCP
      port: 8010
    - protocol: TCP
      port: 8011
    - protocol: TCP
      port: 8012
    - protocol: TCP
      port: 8013
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
  - to:
    - podSelector:
        matchLabels:
          app: postgres-cluster
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          app: redis-cluster
    ports:
    - protocol: TCP
      port: 6379
  - to:
    - podSelector:
        matchLabels:
          app: mongodb-replica-set
    ports:
    - protocol: TCP
      port: 27017
  - to: []
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 443
```

---

## 🚀 **前端應用部署**

### **📱 Nuxt.js前端部署**

```yaml
# k8s/frontend/coffee-frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coffee-frontend
  namespace: coffee-prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: coffee-frontend
  template:
    metadata:
      labels:
        app: coffee-frontend
    spec:
      containers:
      - name: coffee-frontend
        image: ghcr.io/lucas152112/coffee/frontend:latest
        ports:
        - containerPort: 3000
        env:
        - name: NUXT_PUBLIC_API_BASE_URL
          value: "https://api.coffee.justit.cc"
        - name: NUXT_PUBLIC_WS_URL
          value: "wss://api.coffee.justit.cc"
        - name: NODE_ENV
          value: "production"
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: coffee-frontend
  namespace: coffee-prod
spec:
  selector:
    app: coffee-frontend
  ports:
  - port: 80
    targetPort: 3000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: coffee-frontend-ingress
  namespace: coffee-prod
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-buffer-size: "16k"
    nginx.ingress.kubernetes.io/client-max-body-size: "50m"
spec:
  tls:
  - hosts:
    - app.coffee.justit.cc
    secretName: coffee-frontend-tls
  rules:
  - host: app.coffee.justit.cc
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: coffee-frontend
            port:
              number: 80
```

---

## 🔧 **完整部署腳本**

### **📜 一鍵部署腳本**

```bash
#!/bin/bash
# deploy-coffee-system.sh
set -e

echo "🚀 開始Coffee系統完整部署..."

# 顏色定義
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# 函數定義
log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

check_prerequisites() {
    log_info "檢查部署前提條件..."
    
    # 檢查kubectl
    if ! command -v kubectl &> /dev/null; then
        log_error "kubectl未安裝"
        exit 1
    fi
    
    # 檢查helm
    if ! command -v helm &> /dev/null; then
        log_error "helm未安裝"
        exit 1
    fi
    
    # 檢查集群連接
    if ! kubectl cluster-info &> /dev/null; then
        log_error "無法連接到Kubernetes集群"
        exit 1
    fi
    
    log_info "前提條件檢查通過 ✅"
}

deploy_namespaces() {
    log_info "創建命名空間..."
    kubectl apply -f k8s/namespaces.yaml
    kubectl apply -f k8s/resource-quotas.yaml
    log_info "命名空間創建完成 ✅"
}

deploy_secrets() {
    log_info "部署機密配置..."
    
    # 創建資料庫密碼
    kubectl create secret generic database-credentials \
        --from-literal=url="postgresql://coffee_user:$(openssl rand -base64 32)@postgres-cluster-rw:5432/coffee_prod" \
        --namespace=coffee-prod \
        --dry-run=client -o yaml | kubectl apply -f -
    
    # 創建Redis密碼
    kubectl create secret generic redis-credentials \
        --from-literal=url="redis://:$(openssl rand -base64 32)@redis-cluster:6379/0" \
        --namespace=coffee-prod \
        --dry-run=client -o yaml | kubectl apply -f -
    
    # 創建應用密鑰
    kubectl create secret generic app-secrets \
        --from-literal=jwt-secret="$(openssl rand -base64 64)" \
        --namespace=coffee-prod \
        --dry-run=client -o yaml | kubectl apply -f -
    
    log_info "機密配置部署完成 ✅"
}

deploy_databases() {
    log_info "部署資料庫..."
    
    # 部署PostgreSQL
    kubectl apply -f k8s/databases/postgresql.yaml
    log_info "等待PostgreSQL就緒..."
    kubectl wait --for=condition=Ready cluster/postgres-cluster -n coffee-prod --timeout=600s
    
    # 部署Redis
    kubectl apply -f k8s/databases/redis.yaml
    log_info "等待Redis集群就緒..."
    kubectl wait --for=condition=Ready rediscluster/redis-cluster -n coffee-prod --timeout=300s
    
    # 部署MongoDB
    kubectl apply -f k8s/databases/mongodb.yaml
    log_info "等待MongoDB副本集就緒..."
    kubectl wait --for=condition=Ready mongodbcommunity/mongodb-replica-set -n coffee-prod --timeout=600s
    
    log_info "資料庫部署完成 ✅"
}

deploy_microservices() {
    log_info "部署微服務..."
    
    # 部署所有微服務
    for service in tenant monitoring config billing order payment inventory analytics; do
        log_info "部署 ${service}-service..."
        kubectl apply -f k8s/services/${service}-service.yaml
    done
    
    # 等待所有微服務就緒
    log_info "等待所有微服務就緒..."
    for service in tenant monitoring config billing order payment inventory analytics; do
        kubectl wait --for=condition=Available deployment/${service}-service -n coffee-prod --timeout=300s
    done
    
    log_info "微服務部署完成 ✅"
}

deploy_gateway_ingress() {
    log_info "部署API Gateway和Ingress..."
    kubectl apply -f k8s/ingress/api-gateway.yaml
    kubectl wait --for=condition=Available deployment/api-gateway -n coffee-prod --timeout=300s
    log_info "API Gateway和Ingress部署完成 ✅"
}

deploy_frontend() {
    log_info "部署前端應用..."
    kubectl apply -f k8s/frontend/coffee-frontend.yaml
    kubectl wait --for=condition=Available deployment/coffee-frontend -n coffee-prod --timeout=300s
    log_info "前端應用部署完成 ✅"
}

deploy_monitoring() {
    log_info "部署監控系統..."
    kubectl apply -f k8s/monitoring/prometheus.yaml
    kubectl apply -f k8s/monitoring/grafana.yaml
    
    kubectl wait --for=condition=Available deployment/prometheus -n coffee-monitoring --timeout=300s
    kubectl wait --for=condition=Available deployment/grafana -n coffee-monitoring --timeout=300s
    
    log_info "監控系統部署完成 ✅"
}

setup_security() {
    log_info "配置安全策略..."
    kubectl apply -f k8s/security/rbac.yaml
    kubectl apply -f k8s/security/network-policies.yaml
    log_info "安全策略配置完成 ✅"
}

run_health_checks() {
    log_info "執行健康檢查..."
    
    # 檢查所有Pod狀態
    log_info "檢查Pod狀態..."
    kubectl get pods -n coffee-prod
    
    # 檢查服務端點
    log_info "檢查API Gateway健康狀態..."
    kubectl port-forward -n coffee-prod svc/api-gateway 8080:80 &
    PF_PID=$!
    sleep 5
    
    if curl -f http://localhost:8080/health; then
        log_info "API Gateway健康檢查通過 ✅"
    else
        log_error "API Gateway健康檢查失敗 ❌"
    fi
    
    kill $PF_PID 2>/dev/null || true
    
    log_info "健康檢查完成 ✅"
}

cleanup_on_error() {
    log_error "部署過程中發生錯誤，開始清理..."
    # 這裡可以添加清理邏輯
}

# 主要部署流程
main() {
    trap cleanup_on_error ERR
    
    log_info "========================================"
    log_info "  Coffee系統生產環境部署開始"
    log_info "========================================"
    
    check_prerequisites
    deploy_namespaces
    deploy_secrets
    deploy_databases
    deploy_microservices
    deploy_gateway_ingress
    deploy_frontend
    deploy_monitoring
    setup_security
    run_health_checks
    
    log_info "========================================"
    log_info "  Coffee系統部署完成！🎉"
    log_info "========================================"
    log_info "前端URL: https://app.coffee.justit.cc"
    log_info "API URL: https://api.coffee.justit.cc"
    log_info "監控URL: https://monitor.coffee.justit.cc"
    log_info "========================================"
}

# 執行部署
main "$@"
```

**🎯 Coffee系統階段5完整部署配置完成！**

**下一步：執行最終階段6部署上線！**