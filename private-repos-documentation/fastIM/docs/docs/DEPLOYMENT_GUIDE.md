# FastIM 後端部署操作文件

## 📋 概述
本文件記錄 FastIM 實時聊天系統後端在 ARM64 K8s 集群上的完整部署流程。

## 🛠️ 前置需求

### 硬體環境
- **K8s Master**: 103.251.113.35 (4核心8GB) - 密碼: Hiq7fJSY@
- **K8s Worker**: 103.251.113.34 (6核心16GB) - 密碼: iYEckXo)4  
- **Gateway**: 103.251.113.36 (4核心8GB) - 密碼: AwtfFs]O5

### 軟體需求
- Kubernetes 集群運行中
- Docker 和 containerd 已安裝
- kubectl 配置正確
- Go 1.21+ (本地編譯用)

## 🚀 部署步驟

### 步驟 1: 準備原始碼
```bash
# 1.1 克隆專案
cd /root/.openclaw/workspace
git clone git@github.com:lucas152112/fastIM.git
cd fastIM/backend

# 1.2 編譯 ARM64 二進位檔案
go mod tidy
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build -o fastim ./cmd/fastim

# 1.3 驗證編譯結果
ls -lh fastim  # 應顯示約 16MB 的執行檔
```

### 步驟 2: 建立 Docker 映像
```bash
# 2.1 打包專案
cd /root/.openclaw/workspace/fastIM
tar czf fastim-backend.tar.gz backend/

# 2.2 傳送到 K8s Master 節點
sshpass -p 'Hiq7fJSY@' scp -o ProxyCommand="sshpass -p 'AwtfFs]O5' ssh -W %h:%p root@103.251.113.36" \
  fastim-backend.tar.gz root@103.251.113.35:/tmp/

# 2.3 在 Master 節點建立映像
ssh -o ProxyCommand="sshpass -p 'AwtfFs]O5' ssh -W %h:%p root@103.251.113.36" root@103.251.113.35 << 'EOF'
cd /tmp && tar xzf fastim-backend.tar.gz && cd backend

# 建立簡化的 Dockerfile
cat > Dockerfile.simple << 'DOCKEREOF'
FROM alpine:latest
RUN apk add --no-cache ca-certificates
WORKDIR /root/
COPY fastim .
EXPOSE 8080
CMD ["./fastim"]
DOCKEREOF

# 建立映像
docker build -f Dockerfile.simple -t fastim:latest .
EOF
```

### 步驟 3: 部署資料庫服務
```bash
# 3.1 建立命名空間和資料庫
ssh -o ProxyCommand="sshpass -p 'AwtfFs]O5' ssh -W %h:%p root@103.251.113.36" root@103.251.113.35 << 'EOF'
export KUBECONFIG=/etc/kubernetes/admin.conf

# 建立命名空間
kubectl create namespace fastim --dry-run=client -o yaml | kubectl apply -f -

# 部署資料庫服務
cat > /tmp/fastim-database.yaml << 'YAMLEOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastim-mongo
  namespace: fastim
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastim-mongo
  template:
    metadata:
      labels:
        app: fastim-mongo
    spec:
      containers:
      - name: mongo
        image: mongo:7.0
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_DATABASE
          value: "fastim"
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "400m"
---
apiVersion: v1
kind: Service
metadata:
  name: fastim-mongo-service
  namespace: fastim
spec:
  selector:
    app: fastim-mongo
  ports:
  - port: 27017
    targetPort: 27017
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastim-redis
  namespace: fastim
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastim-redis
  template:
    metadata:
      labels:
        app: fastim-redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
---
apiVersion: v1
kind: Service
metadata:
  name: fastim-redis-service
  namespace: fastim
spec:
  selector:
    app: fastim-redis
  ports:
  - port: 6379
    targetPort: 6379
YAMLEOF

kubectl apply -f /tmp/fastim-database.yaml
EOF
```

### 步驟 4: 載入映像到集群
```bash
# 4.1 載入映像到 Master 節點 containerd
ssh -o ProxyCommand="sshpass -p 'AwtfFs]O5' ssh -W %h:%p root@103.251.113.36" root@103.251.113.35 \
  "docker save fastim:latest | ctr -n k8s.io images import -"

# 4.2 載入映像到 Worker 節點
ssh -o ProxyCommand="sshpass -p 'AwtfFs]O5' ssh -W %h:%p root@103.251.113.36" root@103.251.113.35 \
  "docker save fastim:latest" | \
ssh -o ProxyCommand="sshpass -p 'AwtfFs]O5' ssh -W %h:%p root@103.251.113.36" root@103.251.113.34 \
  "ctr -n k8s.io images import -"
```

### 步驟 5: 部署 FastIM 後端
```bash
ssh -o ProxyCommand="sshpass -p 'AwtfFs]O5' ssh -W %h:%p root@103.251.113.36" root@103.251.113.35 << 'EOF'
export KUBECONFIG=/etc/kubernetes/admin.conf

cat > /tmp/fastim-backend.yaml << 'YAMLEOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastim-backend
  namespace: fastim
spec:
  replicas: 1
  selector:
    matchLabels:
      app: fastim-backend
  template:
    metadata:
      labels:
        app: fastim-backend
    spec:
      containers:
      - name: fastim
        image: docker.io/library/fastim:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        env:
        - name: PORT
          value: "8080"
        - name: MONGO_URL
          value: "mongodb://fastim-mongo-service:27017/fastim"
        - name: REDIS_URL
          value: "redis://fastim-redis-service:6379"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: fastim-backend-service
  namespace: fastim
spec:
  selector:
    app: fastim-backend
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  name: fastim-external-service
  namespace: fastim
spec:
  selector:
    app: fastim-backend
  ports:
  - name: http
    port: 8080
    targetPort: 8080
    nodePort: 30800
  type: NodePort
YAMLEOF

kubectl apply -f /tmp/fastim-backend.yaml
EOF
```

## 🔍 驗證部署

### 檢查 Pod 狀態
```bash
kubectl get pods -n fastim
# 預期輸出: 所有 Pod 都顯示 1/1 Running
```

### 檢查服務狀態  
```bash
kubectl get svc -n fastim
# 預期輸出: 服務正常運行，NodePort 30800 開放
```

### 健康檢查
```bash
# 內部健康檢查
kubectl exec -n fastim deployment/fastim-backend -- wget -qO- http://localhost:8080/api/v1/health

# 外部健康檢查
curl http://103.251.113.34:30800/api/v1/health
curl http://103.251.113.35:30800/api/v1/health
```

## 📡 API 端點

| 端點 | 方法 | 說明 |
|------|------|------|
| `/api/v1/health` | GET | 健康檢查 |
| `/api/v1/ws?username=name` | WebSocket | WebSocket 連接 |
| `/api/v1/auth/login` | POST | 使用者登入 |
| `/api/v1/messages/:channel` | GET | 獲取訊息歷史 |

## 🌐 訪問方式

- **外部訪問**: http://103.251.113.34:30800 或 http://103.251.113.35:30800
- **WebSocket**: ws://103.251.113.34:30800/api/v1/ws?username=你的名稱
- **內部服務**: http://fastim-backend-service.fastim.svc.cluster.local:8080

## 🛠️ 常見問題

### 問題 1: Pod 顯示 ErrImagePull
**原因**: 映像未正確載入到所有節點
**解決**: 重新執行步驟 4，確保映像載入到所有節點

### 問題 2: 服務無法訪問
**原因**: NodePort 防火牆阻擋或服務未啟動
**解決**: 檢查 Pod 日誌並確認防火牆設定

### 問題 3: 資料庫連接失敗
**原因**: MongoDB 或 Redis 服務未準備完成
**解決**: 等待資料庫 Pod 變為 Running 狀態

## 📊 監控和日誌

### 檢查日誌
```bash
# FastIM 後端日誌
kubectl logs -n fastim deployment/fastim-backend -f

# MongoDB 日誌  
kubectl logs -n fastim deployment/fastim-mongo -f

# Redis 日誌
kubectl logs -n fastim deployment/fastim-redis -f
```

### 性能監控
```bash
# 資源使用情況
kubectl top pods -n fastim

# 詳細 Pod 資訊
kubectl describe pod -n fastim -l app=fastim-backend
```

## 🔄 更新和維護

### 更新應用程式
1. 重新編譯並建立映像
2. 載入新映像到所有節點
3. 重啟 Deployment: `kubectl rollout restart deployment/fastim-backend -n fastim`

### 擴展服務
```bash
# 增加副本數
kubectl scale deployment fastim-backend --replicas=3 -n fastim
```

### 清理環境
```bash
# 刪除整個 namespace（包含所有資源）
kubectl delete namespace fastim
```

## 📝 注意事項

1. **映像管理**: 每次更新都需要載入映像到所有 K8s 節點
2. **資源限制**: 根據實際使用情況調整 CPU 和記憶體限制
3. **安全性**: 生產環境建議設定 TLS 和身份認證
4. **備份**: 定期備份 MongoDB 資料
5. **監控**: 建議整合 Grafana 和 Prometheus 進行監控

---

**建立時間**: 2026-02-04 17:06  
**版本**: v1.0  
**維護人**: Jackson