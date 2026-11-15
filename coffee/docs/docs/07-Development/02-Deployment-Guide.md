# 部署指南 (Deployment Guide)

## 1. 部署概述

### 1.1 部署架構
本系統採用微服務架構，支援 Docker 容器化部署和 Kubernetes 編排。主要包含以下組件：

```
部署架構圖：
┌─────────────────────────────────────────┐
│               Load Balancer             │
│              (Nginx/HAProxy)            │
└─────────────┬───────────────────────────┘
              │
┌─────────────▼───────────────────────────┐
│            API Gateway                  │
│          (Gin + Go 1.21)               │
└─────┬───┬───┬───┬───┬───┬───┬───────────┘
      │   │   │   │   │   │   │
      ▼   ▼   ▼   ▼   ▼   ▼   ▼
    ┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐
    │Auth││Cust││Order││Inv││Loy││Not││Rep│
    └───┘└───┘└───┘└───┘└───┘└───┘└───┘
      │   │   │   │   │   │   │
      └───┼───┼───┼───┼───┼───┼───────┐
          │   │   │   │   │   │       │
        ┌─▼───▼───▼───▼───▼───▼─┐   ┌─▼─┐
        │       MySQL 8.0      │   │Redis│
        └─────────────────────────┘   └───┘
```

### 1.2 支援的部署方式
- **開發環境**: Docker Compose (單機部署)
- **測試環境**: Docker Swarm (小規模集群)
- **生產環境**: Kubernetes (高可用集群)
- **雲端部署**: AWS EKS, Azure AKS, Google GKE

---

## 2. 環境需求

### 2.1 硬體需求
```yaml
最小需求:
  CPU: 4 cores
  RAM: 8 GB
  Storage: 50 GB SSD
  Network: 1 Gbps

建議配置:
  CPU: 8 cores
  RAM: 16 GB
  Storage: 100 GB SSD
  Network: 10 Gbps

生產環境 (3 節點):
  Master Node:
    CPU: 4 cores
    RAM: 8 GB
    Storage: 50 GB SSD
  
  Worker Node (x2):
    CPU: 8 cores
    RAM: 16 GB
    Storage: 100 GB SSD
```

### 2.2 軟體需求
```yaml
作業系統:
  - Ubuntu 20.04 LTS 或更高
  - CentOS 8 或更高
  - RHEL 8 或更高

必要軟體:
  - Docker 24.0+
  - Docker Compose 2.0+
  - Git 2.30+

Kubernetes 部署 (額外需求):
  - kubectl 1.28+
  - Helm 3.12+
  - kubeadm 1.28+ (自建集群)

監控工具 (可選):
  - Prometheus 2.40+
  - Grafana 9.0+
  - ELK Stack 8.5+
```

---

## 3. Docker Compose 部署 (開發/測試環境)

### 3.1 準備部署環境
```bash
# 1. 克隆專案
git clone https://github.com/your-org/coffee-chain-system.git
cd coffee-chain-system

# 2. 複製環境配置文件
cp .env.example .env
cp config/development.yaml.example config/development.yaml

# 3. 設定環境變數
vim .env
```

### 3.2 環境變數配置
```bash
# .env
# 資料庫配置
DB_ROOT_PASSWORD=your_strong_password
DB_USER=coffee_user
DB_PASSWORD=user_password
DB_NAME=coffee_crm

# Redis 配置
REDIS_PASSWORD=redis_password

# JWT 配置
JWT_SECRET=your_jwt_secret_key_at_least_32_chars

# 環境設定
ENVIRONMENT=development
LOG_LEVEL=debug

# 網路配置
EXTERNAL_PORT=8080
```

### 3.3 Docker Compose 配置文件
```yaml
# docker-compose.yml
version: '3.8'

services:
  # API Gateway
  gateway:
    build:
      context: ./gateway
      dockerfile: Dockerfile
    ports:
      - "${EXTERNAL_PORT:-8080}:8080"
    environment:
      - ENV=${ENVIRONMENT}
      - LOG_LEVEL=${LOG_LEVEL}
      - DB_HOST=mysql
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_NAME=${DB_NAME}
      - REDIS_HOST=redis
      - REDIS_PASSWORD=${REDIS_PASSWORD}
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - coffee-network
    restart: unless-stopped

  # 認證服務
  auth-service:
    build:
      context: ./services/auth
      dockerfile: Dockerfile
    environment:
      - ENV=${ENVIRONMENT}
      - LOG_LEVEL=${LOG_LEVEL}
      - DB_HOST=mysql
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_NAME=${DB_NAME}
      - REDIS_HOST=redis
      - REDIS_PASSWORD=${REDIS_PASSWORD}
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - coffee-network
    restart: unless-stopped

  # 客戶服務
  customers-service:
    build:
      context: ./services/customers
      dockerfile: Dockerfile
    environment:
      - ENV=${ENVIRONMENT}
      - LOG_LEVEL=${LOG_LEVEL}
      - DB_HOST=mysql
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_NAME=${DB_NAME}
      - REDIS_HOST=redis
      - REDIS_PASSWORD=${REDIS_PASSWORD}
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - coffee-network
    restart: unless-stopped

  # 其他服務...（簡化顯示）

  # 資料庫
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USER}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
      - ./config/mysql.cnf:/etc/mysql/conf.d/custom.cnf
    ports:
      - "3306:3306"
    networks:
      - coffee-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 20s
      retries: 10

  # Redis 快取
  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    networks:
      - coffee-network
    restart: unless-stopped

  # 前端應用
  webapp:
    build:
      context: ./webapp
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NUXT_PUBLIC_API_BASE_URL=http://localhost:${EXTERNAL_PORT:-8080}
    depends_on:
      - gateway
    networks:
      - coffee-network
    restart: unless-stopped

volumes:
  mysql_data:
  redis_data:

networks:
  coffee-network:
    driver: bridge
```

### 3.4 部署步驟
```bash
# 1. 建構所有服務
docker-compose build

# 2. 啟動所有服務
docker-compose up -d

# 3. 查看服務狀態
docker-compose ps

# 4. 查看日誌
docker-compose logs -f

# 5. 執行資料庫遷移 (首次部署)
docker-compose exec gateway ./migrate up

# 6. 建立初始管理員帳號
docker-compose exec gateway ./scripts/create-admin.sh

# 7. 驗證部署
curl http://localhost:8080/health
```

### 3.5 常用管理指令
```bash
# 停止所有服務
docker-compose down

# 重啟特定服務
docker-compose restart gateway

# 查看服務日誌
docker-compose logs -f auth-service

# 進入容器 Shell
docker-compose exec mysql bash

# 備份資料庫
docker-compose exec mysql mysqldump -u root -p${DB_ROOT_PASSWORD} coffee_crm > backup.sql

# 清理未使用的映像
docker system prune -f
```

---

## 4. Kubernetes 部署 (生產環境)

### 4.1 集群準備
```bash
# 1. 安裝 kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# 2. 安裝 Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 3. 驗證集群連接
kubectl cluster-info
kubectl get nodes
```

### 4.2 命名空間和資源配額
```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: coffee-chain
  labels:
    name: coffee-chain
    env: production

---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: coffee-chain-quota
  namespace: coffee-chain
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    persistentvolumeclaims: "10"
    services: "10"
    secrets: "10"
    configmaps: "10"
```

### 4.3 ConfigMap 和 Secret
```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coffee-chain-config
  namespace: coffee-chain
data:
  database.yaml: |
    host: mysql-service
    port: "3306"
    name: coffee_crm
    max_open_conns: 25
    max_idle_conns: 10
    max_lifetime: "5m"
  
  redis.yaml: |
    host: redis-service
    port: "6379"
    db: "0"
    pool_size: 10

---
apiVersion: v1
kind: Secret
metadata:
  name: coffee-chain-secrets
  namespace: coffee-chain
type: Opaque
data:
  db-root-password: eW91cl9zdHJvbmdfcGFzc3dvcmQ=  # base64 encoded
  db-user-password: dXNlcl9wYXNzd29yZA==          # base64 encoded
  redis-password: cmVkaXNfcGFzc3dvcmQ=              # base64 encoded
  jwt-secret: eW91cl9qd3Rfc2VjcmV0X2tleQ==        # base64 encoded
```

### 4.4 資料庫部署
```yaml
# k8s/mysql.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: coffee-chain
spec:
  serviceName: mysql-service
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: coffee-chain-secrets
              key: db-root-password
        - name: MYSQL_DATABASE
          value: coffee_crm
        - name: MYSQL_USER
          value: coffee_user
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: coffee-chain-secrets
              key: db-user-password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
        - name: mysql-config
          mountPath: /etc/mysql/conf.d
        livenessProbe:
          exec:
            command:
            - mysqladmin
            - ping
            - -h
            - localhost
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - mysql
            - -h
            - localhost
            - -u
            - root
            - -p$(MYSQL_ROOT_PASSWORD)
            - -e
            - "SELECT 1"
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
      volumes:
      - name: mysql-config
        configMap:
          name: mysql-config
  volumeClaimTemplates:
  - metadata:
      name: mysql-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 20Gi

---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: coffee-chain
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
  type: ClusterIP
```

### 4.5 微服務部署
```yaml
# k8s/gateway.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gateway
  namespace: coffee-chain
spec:
  replicas: 3
  selector:
    matchLabels:
      app: gateway
  template:
    metadata:
      labels:
        app: gateway
    spec:
      containers:
      - name: gateway
        image: coffee-chain/gateway:latest
        ports:
        - containerPort: 8080
        env:
        - name: ENV
          value: "production"
        - name: LOG_LEVEL
          value: "info"
        - name: DB_HOST
          value: "mysql-service"
        - name: DB_USER
          value: "coffee_user"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: coffee-chain-secrets
              key: db-user-password
        - name: DB_NAME
          value: "coffee_crm"
        - name: REDIS_HOST
          value: "redis-service"
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: coffee-chain-secrets
              key: redis-password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: coffee-chain-secrets
              key: jwt-secret
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
          initialDelaySeconds: 5
          periodSeconds: 5
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
  name: gateway-service
  namespace: coffee-chain
spec:
  selector:
    app: gateway
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

### 4.6 Ingress 配置
```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: coffee-chain-ingress
  namespace: coffee-chain
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - api.coffee-chain.com
    - app.coffee-chain.com
    secretName: coffee-chain-tls
  rules:
  - host: api.coffee-chain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gateway-service
            port:
              number: 80
  - host: app.coffee-chain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-service
            port:
              number: 80
```

### 4.7 部署腳本
```bash
#!/bin/bash
# deploy.sh

set -e

# 變數設定
NAMESPACE="coffee-chain"
IMAGE_TAG=${1:-latest}

echo "開始部署 Coffee Chain System..."

# 1. 創建命名空間
kubectl apply -f k8s/namespace.yaml

# 2. 部署 ConfigMap 和 Secrets
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secrets.yaml

# 3. 部署資料庫
echo "部署 MySQL..."
kubectl apply -f k8s/mysql.yaml
kubectl wait --for=condition=ready pod -l app=mysql -n $NAMESPACE --timeout=300s

# 4. 部署 Redis
echo "部署 Redis..."
kubectl apply -f k8s/redis.yaml
kubectl wait --for=condition=ready pod -l app=redis -n $NAMESPACE --timeout=120s

# 5. 更新映像標籤
sed -i "s|coffee-chain/gateway:latest|coffee-chain/gateway:$IMAGE_TAG|g" k8s/gateway.yaml
sed -i "s|coffee-chain/auth:latest|coffee-chain/auth:$IMAGE_TAG|g" k8s/auth.yaml
# ... 其他服務

# 6. 部署微服務
echo "部署微服務..."
kubectl apply -f k8s/gateway.yaml
kubectl apply -f k8s/auth.yaml
kubectl apply -f k8s/customers.yaml
kubectl apply -f k8s/orders.yaml
kubectl apply -f k8s/inventory.yaml
kubectl apply -f k8s/loyalty.yaml
kubectl apply -f k8s/notifications.yaml
kubectl apply -f k8s/reports.yaml

# 7. 部署前端
kubectl apply -f k8s/webapp.yaml

# 8. 設定 Ingress
kubectl apply -f k8s/ingress.yaml

# 9. 等待部署完成
echo "等待所有服務啟動..."
kubectl wait --for=condition=ready pod -l app=gateway -n $NAMESPACE --timeout=300s
kubectl wait --for=condition=ready pod -l app=webapp -n $NAMESPACE --timeout=300s

# 10. 顯示部署狀態
echo "部署完成！"
kubectl get pods -n $NAMESPACE
kubectl get services -n $NAMESPACE
kubectl get ingress -n $NAMESPACE

echo "API Gateway: http://$(kubectl get service gateway-service -n $NAMESPACE -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
echo "Web App: http://$(kubectl get ingress coffee-chain-ingress -n $NAMESPACE -o jsonpath='{.spec.rules[1].host}')"
```

---

## 5. 監控和日誌

### 5.1 Prometheus 監控
```yaml
# k8s/monitoring/prometheus.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: coffee-chain
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    
    scrape_configs:
    - job_name: 'coffee-chain-services'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
          - coffee-chain
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  namespace: coffee-chain
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config
          mountPath: /etc/prometheus
        - name: storage
          mountPath: /prometheus
        command:
        - /bin/prometheus
        - --config.file=/etc/prometheus/prometheus.yml
        - --storage.tsdb.path=/prometheus
        - --web.console.libraries=/usr/share/prometheus/console_libraries
        - --web.console.templates=/usr/share/prometheus/consoles
      volumes:
      - name: config
        configMap:
          name: prometheus-config
      - name: storage
        emptyDir: {}
```

### 5.2 Grafana 儀表板
```yaml
# k8s/monitoring/grafana.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: coffee-chain
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      access: proxy
      url: http://prometheus-service:9090
      isDefault: true

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: coffee-chain
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
          value: admin
        volumeMounts:
        - name: grafana-datasources
          mountPath: /etc/grafana/provisioning/datasources
      volumes:
      - name: grafana-datasources
        configMap:
          name: grafana-datasources
```

---

## 6. 備份和災難復原

### 6.1 資料庫備份腳本
```bash
#!/bin/bash
# backup.sh

set -e

NAMESPACE="coffee-chain"
BACKUP_DIR="/backups/coffee-chain"
DATE=$(date +%Y%m%d_%H%M%S)

# 建立備份目錄
mkdir -p $BACKUP_DIR

# 1. 資料庫備份
echo "開始備份 MySQL..."
kubectl exec -n $NAMESPACE mysql-0 -- mysqldump \
  -u root -p$(kubectl get secret coffee-chain-secrets -n $NAMESPACE -o jsonpath='{.data.db-root-password}' | base64 -d) \
  --single-transaction --routines --triggers coffee_crm > $BACKUP_DIR/mysql_$DATE.sql

# 2. Redis 備份
echo "開始備份 Redis..."
kubectl exec -n $NAMESPACE redis-0 -- redis-cli --rdb /tmp/redis_$DATE.rdb
kubectl cp $NAMESPACE/redis-0:/tmp/redis_$DATE.rdb $BACKUP_DIR/redis_$DATE.rdb

# 3. 配置備份
echo "備份 Kubernetes 配置..."
kubectl get all,configmaps,secrets,ingress -n $NAMESPACE -o yaml > $BACKUP_DIR/k8s_config_$DATE.yaml

# 4. 壓縮備份
echo "壓縮備份檔案..."
tar -czf $BACKUP_DIR/backup_$DATE.tar.gz -C $BACKUP_DIR mysql_$DATE.sql redis_$DATE.rdb k8s_config_$DATE.yaml

# 5. 上傳到雲端存儲 (示例使用 AWS S3)
if command -v aws &> /dev/null; then
    echo "上傳備份到 S3..."
    aws s3 cp $BACKUP_DIR/backup_$DATE.tar.gz s3://coffee-chain-backups/
fi

# 6. 清理本地備份 (保留最近 7 天)
find $BACKUP_DIR -name "backup_*.tar.gz" -mtime +7 -delete

echo "備份完成: backup_$DATE.tar.gz"
```

### 6.2 災難復原腳本
```bash
#!/bin/bash
# restore.sh

set -e

NAMESPACE="coffee-chain"
BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
    echo "使用方式: $0 <backup_file>"
    exit 1
fi

echo "開始復原系統..."

# 1. 解壓備份檔案
TEMP_DIR=$(mktemp -d)
tar -xzf $BACKUP_FILE -C $TEMP_DIR

# 2. 停止服務
kubectl scale deployment --replicas=0 -n $NAMESPACE \
  gateway auth-service customers-service orders-service \
  inventory-service loyalty-service notifications-service reports-service

# 3. 復原資料庫
echo "復原 MySQL..."
kubectl exec -n $NAMESPACE mysql-0 -- mysql \
  -u root -p$(kubectl get secret coffee-chain-secrets -n $NAMESPACE -o jsonpath='{.data.db-root-password}' | base64 -d) \
  coffee_crm < $TEMP_DIR/mysql_*.sql

# 4. 復原 Redis
echo "復原 Redis..."
kubectl cp $TEMP_DIR/redis_*.rdb $NAMESPACE/redis-0:/tmp/restore.rdb
kubectl exec -n $NAMESPACE redis-0 -- redis-cli --rdb /tmp/restore.rdb

# 5. 重啟服務
kubectl scale deployment --replicas=3 -n $NAMESPACE gateway
kubectl scale deployment --replicas=2 -n $NAMESPACE \
  auth-service customers-service orders-service \
  inventory-service loyalty-service notifications-service reports-service

# 6. 等待服務啟動
kubectl wait --for=condition=ready pod -l app=gateway -n $NAMESPACE --timeout=300s

echo "復原完成！"
rm -rf $TEMP_DIR
```

---

## 7. 維護和故障排除

### 7.1 常見問題診斷
```bash
#!/bin/bash
# troubleshoot.sh

NAMESPACE="coffee-chain"

echo "Coffee Chain System 診斷工具"
echo "================================"

# 1. 檢查所有 Pod 狀態
echo "1. 檢查 Pod 狀態:"
kubectl get pods -n $NAMESPACE -o wide

# 2. 檢查服務狀態
echo -e "\n2. 檢查服務狀態:"
kubectl get services -n $NAMESPACE

# 3. 檢查資源使用情況
echo -e "\n3. 檢查資源使用:"
kubectl top pods -n $NAMESPACE

# 4. 檢查最近的事件
echo -e "\n4. 最近事件:"
kubectl get events -n $NAMESPACE --sort-by=.metadata.creationTimestamp

# 5. 檢查 Ingress 狀態
echo -e "\n5. Ingress 狀態:"
kubectl get ingress -n $NAMESPACE

# 6. 健康檢查
echo -e "\n6. 健康檢查:"
GATEWAY_IP=$(kubectl get service gateway-service -n $NAMESPACE -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
if [ ! -z "$GATEWAY_IP" ]; then
    curl -f http://$GATEWAY_IP/health && echo "✓ Gateway health check passed" || echo "✗ Gateway health check failed"
fi

# 7. 檢查日誌 (最新的錯誤)
echo -e "\n7. 最近的錯誤日誌:"
for app in gateway auth-service customers-service; do
    echo "--- $app ---"
    kubectl logs -n $NAMESPACE -l app=$app --tail=10 | grep -i error || echo "No errors found"
done
```

### 7.2 性能監控腳本
```bash
#!/bin/bash
# monitor.sh

NAMESPACE="coffee-chain"

# 監控循環
while true; do
    clear
    echo "Coffee Chain System 實時監控"
    echo "=============================="
    date

    # CPU 和記憶體使用情況
    echo -e "\n資源使用情況:"
    kubectl top pods -n $NAMESPACE

    # 服務響應時間
    echo -e "\n服務響應時間:"
    GATEWAY_IP=$(kubectl get service gateway-service -n $NAMESPACE -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    if [ ! -z "$GATEWAY_IP" ]; then
        for endpoint in health ready; do
            response_time=$(curl -o /dev/null -s -w "%{time_total}" http://$GATEWAY_IP/$endpoint)
            echo "$endpoint: ${response_time}s"
        done
    fi

    # 活躍連線數
    echo -e "\n活躍連線數:"
    kubectl get pods -n $NAMESPACE -l app=gateway -o name | xargs -I {} kubectl exec {} -n $NAMESPACE -- netstat -an | grep ESTABLISHED | wc -l

    sleep 30
done
```

### 7.3 日誌查看腳本
```bash
#!/bin/bash
# logs.sh

NAMESPACE="coffee-chain"
SERVICE=${1:-gateway}
LINES=${2:-100}

echo "查看 $SERVICE 服務日誌 (最新 $LINES 行)"
echo "================================"

# 獲取所有該服務的 Pod
PODS=$(kubectl get pods -n $NAMESPACE -l app=$SERVICE -o jsonpath='{.items[*].metadata.name}')

if [ -z "$PODS" ]; then
    echo "未找到 $SERVICE 服務的 Pod"
    exit 1
fi

# 為每個 Pod 顯示日誌
for pod in $PODS; do
    echo "--- Pod: $pod ---"
    kubectl logs -n $NAMESPACE $pod --tail=$LINES
    echo ""
done

# 實時跟蹤日誌 (可選)
read -p "是否要實時跟蹤日誌？(y/N) " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    kubectl logs -n $NAMESPACE -l app=$SERVICE -f
fi
```

---

## 8. 升級和回滾

### 8.1 滾動升級腳本
```bash
#!/bin/bash
# upgrade.sh

NAMESPACE="coffee-chain"
NEW_TAG=$1

if [ -z "$NEW_TAG" ]; then
    echo "使用方式: $0 <new_image_tag>"
    exit 1
fi

echo "開始滾動升級到版本: $NEW_TAG"

# 升級所有服務
SERVICES=("gateway" "auth-service" "customers-service" "orders-service" 
          "inventory-service" "loyalty-service" "notifications-service" "reports-service")

for service in "${SERVICES[@]}"; do
    echo "升級 $service..."
    
    # 更新映像
    kubectl set image deployment/$service -n $NAMESPACE \
        $service=coffee-chain/$service:$NEW_TAG
    
    # 等待滾動升級完成
    kubectl rollout status deployment/$service -n $NAMESPACE
    
    # 檢查健康狀態
    kubectl wait --for=condition=ready pod -l app=$service -n $NAMESPACE --timeout=300s
    
    echo "$service 升級完成"
done

echo "所有服務升級完成！"
```

### 8.2 回滾腳本
```bash
#!/bin/bash
# rollback.sh

NAMESPACE="coffee-chain"
SERVICE=${1:-all}

if [ "$SERVICE" = "all" ]; then
    SERVICES=("gateway" "auth-service" "customers-service" "orders-service" 
              "inventory-service" "loyalty-service" "notifications-service" "reports-service")
else
    SERVICES=("$SERVICE")
fi

echo "開始回滾服務..."

for service in "${SERVICES[@]}"; do
    echo "回滾 $service..."
    
    # 回滾到上一個版本
    kubectl rollout undo deployment/$service -n $NAMESPACE
    
    # 等待回滾完成
    kubectl rollout status deployment/$service -n $NAMESPACE
    
    echo "$service 回滾完成"
done

echo "回滾操作完成！"
```

---

**版本**: v1.0  
**建立日期**: 2025-01-16  
**最後更新**: 2025-01-16  
**負責人**: DevOps 團隊

**注意事項**:
- 生產環境部署前請先在測試環境驗證
- 確保所有敏感資訊使用 Secret 管理
- 定期備份數據和配置
- 監控系統資源使用情況
- 保持 Kubernetes 和相關工具的版本更新