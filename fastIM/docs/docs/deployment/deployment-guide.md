# FastIM 部署指南

## 📋 目錄
1. [系統要求](#系統要求)
2. [環境準備](#環境準備)
3. [資料庫部署](#資料庫部署)
4. [後端服務部署](#後端服務部署)
5. [前端客戶端部署](#前端客戶端部署)
6. [Kubernetes 部署](#kubernetes-部署)
7. [Docker 部署](#docker-部署)
8. [監控和日誌](#監控和日誌)
9. [SSL/TLS 配置](#ssltls-配置)
10. [部署驗證](#部署驗證)

---

## 🖥️ 系統要求

### 最低配置
- **CPU**: 2 核心 (ARM64/x86_64)
- **記憶體**: 4GB RAM
- **存儲**: 20GB 可用空間
- **網路**: 1Gbps 網路頻寬

### 推薦配置
- **CPU**: 4+ 核心 ARM64
- **記憶體**: 8GB+ RAM
- **存儲**: 50GB+ SSD
- **網路**: 10Gbps 網路頻寬

### 軟體依賴
- **作業系統**: Ubuntu 22.04+, CentOS 8+, macOS 12+
- **容器**: Docker 20.10+, Kubernetes 1.24+
- **資料庫**: MongoDB 5.0+, Redis 6.2+
- **語言環境**: Go 1.20+, Node.js 18+, Flutter 3.10+

---

## 🛠️ 環境準備

### 1. 基礎環境安裝

#### Ubuntu/Debian
```bash
# 更新系統
sudo apt update && sudo apt upgrade -y

# 安裝基礎工具
sudo apt install -y curl wget git vim unzip

# 安裝 Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# 安裝 Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.21.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 安裝 Kubernetes (可選)
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update && sudo apt install -y kubectl
```

#### CentOS/RHEL
```bash
# 更新系統
sudo yum update -y

# 安裝基礎工具
sudo yum install -y curl wget git vim unzip

# 安裝 Docker
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
```

### 2. Go 開發環境

```bash
# 下載並安裝 Go (ARM64)
wget https://golang.org/dl/go1.21.4.linux-arm64.tar.gz
sudo tar -C /usr/local -xzf go1.21.4.linux-arm64.tar.gz

# 配置環境變數
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
echo 'export GOPATH=$HOME/go' >> ~/.bashrc
echo 'export PATH=$PATH:$GOPATH/bin' >> ~/.bashrc
source ~/.bashrc

# 驗證安裝
go version
```

### 3. Node.js 開發環境

```bash
# 安裝 Node.js (使用 NodeSource)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# 驗證安裝
node --version
npm --version
```

### 4. Flutter 開發環境

```bash
# 下載 Flutter SDK
cd ~/
git clone https://github.com/flutter/flutter.git -b stable
echo 'export PATH="$PATH:`pwd`/flutter/bin"' >> ~/.bashrc
source ~/.bashrc

# 運行 Flutter Doctor 檢查依賴
flutter doctor

# 安裝 Android SDK (可選，用於 Android 開發)
wget https://dl.google.com/android/repository/commandlinetools-linux-8512546_latest.zip
mkdir -p ~/android-sdk/cmdline-tools
cd ~/android-sdk/cmdline-tools
unzip ~/commandlinetools-linux-8512546_latest.zip
mv cmdline-tools latest
echo 'export ANDROID_HOME=~/android-sdk' >> ~/.bashrc
echo 'export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin' >> ~/.bashrc
source ~/.bashrc
```

---

## 🗄️ 資料庫部署

### MongoDB 部署

#### 1. Docker 部署 (推薦)
```yaml
# docker-compose.mongodb.yml
version: '3.8'

services:
  mongodb:
    image: mongo:5.0
    container_name: fastim-mongodb
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: your_secure_password
      MONGO_INITDB_DATABASE: fastim
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
      - ./init-mongo.js:/docker-entrypoint-initdb.d/init-mongo.js:ro
    networks:
      - fastim-network

volumes:
  mongodb_data:

networks:
  fastim-network:
    driver: bridge
```

```bash
# 啟動 MongoDB
docker-compose -f docker-compose.mongodb.yml up -d

# 檢查狀態
docker-compose -f docker-compose.mongodb.yml logs -f mongodb
```

#### 2. 本地安裝
```bash
# Ubuntu
wget -qO - https://www.mongodb.org/static/pgp/server-5.0.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu $(lsb_release -cs)/mongodb-org/5.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-5.0.list
sudo apt update
sudo apt install -y mongodb-org

# 啟動服務
sudo systemctl start mongod
sudo systemctl enable mongod
```

#### 3. MongoDB 初始化腳本
```javascript
// init-mongo.js
db.auth('admin', 'your_secure_password')

db = db.getSiblingDB('fastim')

// 創建用戶
db.createUser({
  user: 'fastim_user',
  pwd: 'fastim_password',
  roles: [
    { role: 'readWrite', db: 'fastim' }
  ]
})

// 創建集合和索引
db.createCollection('users')
db.createCollection('messages')
db.createCollection('channels')
db.createCollection('sessions')

// 創建索引
db.messages.createIndex({ "channel": 1, "timestamp": -1 })
db.messages.createIndex({ "sender": 1, "timestamp": -1 })
db.users.createIndex({ "username": 1 }, { unique: true })
db.users.createIndex({ "email": 1 }, { unique: true })
db.channels.createIndex({ "name": 1 }, { unique: true })
db.sessions.createIndex({ "user_id": 1 })
db.sessions.createIndex({ "expires_at": 1 }, { expireAfterSeconds: 0 })

print('FastIM MongoDB 初始化完成')
```

### Redis 部署

#### 1. Docker 部署 (推薦)
```yaml
# docker-compose.redis.yml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    container_name: fastim-redis
    restart: unless-stopped
    command: redis-server --requirepass your_redis_password
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf
    networks:
      - fastim-network

volumes:
  redis_data:

networks:
  fastim-network:
    driver: bridge
```

#### 2. Redis 配置文件
```conf
# redis.conf
bind 0.0.0.0
port 6379
requirepass your_redis_password
maxmemory 2gb
maxmemory-policy allkeys-lru
save 900 1
save 300 10
save 60 10000
```

```bash
# 啟動 Redis
docker-compose -f docker-compose.redis.yml up -d

# 測試連接
redis-cli -a your_redis_password ping
```

---

## 🚀 後端服務部署

### 1. 源碼準備

```bash
# 克隆項目
git clone <your-fastim-repo>
cd fastim

# 檢查 Go 模組
cd backend
go mod tidy
go mod download
```

### 2. 配置文件

```yaml
# backend/config/config.yaml
server:
  port: "8080"
  host: "0.0.0.0"
  timeout:
    read: 15s
    write: 15s
    idle: 60s

database:
  mongodb:
    url: "mongodb://fastim_user:fastim_password@localhost:27017/fastim"
    database: "fastim"
    timeout: 10s
  redis:
    url: "redis://:your_redis_password@localhost:6379/0"
    pool_size: 10
    timeout: 5s

websocket:
  max_connections: 1000
  heartbeat_interval: 30s
  message_buffer_size: 256
  allow_origins:
    - "*"

security:
  jwt_secret: "your-jwt-secret-key-change-in-production"
  jwt_expiry: 24h
  cors_origins:
    - "http://localhost:3000"
    - "http://localhost:8080"

logging:
  level: "info"
  format: "json"
  output: "stdout"
```

### 3. 環境變數文件

```bash
# .env
SERVER_PORT=8080
SERVER_HOST=0.0.0.0

MONGODB_URL=mongodb://fastim_user:fastim_password@localhost:27017/fastim
REDIS_URL=redis://:your_redis_password@localhost:6379/0

JWT_SECRET=your-jwt-secret-key-change-in-production
LOG_LEVEL=info

# 生產環境變數
ENVIRONMENT=production
DEBUG=false
```

### 4. 構建和部署

#### 本地開發部署
```bash
cd backend

# 開發模式運行
go run cmd/fastim/main.go

# 構建二進制文件
go build -o bin/fastim cmd/fastim/main.go

# 運行二進制文件
./bin/fastim
```

#### 生產環境構建
```bash
# 多平台構建腳本 build.sh
#!/bin/bash

APP_NAME="fastim"
VERSION=$(git describe --tags --always --dirty)
BUILD_TIME=$(date -u '+%Y-%m-%d %H:%M:%S')
COMMIT=$(git rev-parse --short HEAD)

LDFLAGS="-s -w -X 'main.Version=${VERSION}' -X 'main.BuildTime=${BUILD_TIME}' -X 'main.Commit=${COMMIT}'"

echo "Building FastIM ${VERSION}..."

# Linux ARM64 (生產環境)
echo "Building for Linux ARM64..."
GOOS=linux GOARCH=arm64 go build -ldflags="${LDFLAGS}" -o bin/fastim-linux-arm64 cmd/fastim/main.go

# Linux AMD64
echo "Building for Linux AMD64..."
GOOS=linux GOARCH=amd64 go build -ldflags="${LDFLAGS}" -o bin/fastim-linux-amd64 cmd/fastim/main.go

# Darwin ARM64 (Apple Silicon)
echo "Building for Darwin ARM64..."
GOOS=darwin GOARCH=arm64 go build -ldflags="${LDFLAGS}" -o bin/fastim-darwin-arm64 cmd/fastim/main.go

# Windows AMD64
echo "Building for Windows AMD64..."
GOOS=windows GOARCH=amd64 go build -ldflags="${LDFLAGS}" -o bin/fastim-windows-amd64.exe cmd/fastim/main.go

echo "Build completed!"
ls -la bin/
```

```bash
# 執行構建
chmod +x build.sh
./build.sh
```

### 5. Systemd 服務配置

```ini
# /etc/systemd/system/fastim.service
[Unit]
Description=FastIM Real-time Chat Server
Documentation=https://github.com/your-org/fastim
After=network.target mongod.service redis.service
Wants=network-online.target

[Service]
Type=simple
User=fastim
Group=fastim
WorkingDirectory=/opt/fastim
ExecStart=/opt/fastim/bin/fastim-linux-arm64
ExecReload=/bin/kill -HUP $MAINPID
KillMode=mixed
KillSignal=SIGTERM
TimeoutSec=30

# 重啟策略
Restart=always
RestartSec=10
StartLimitInterval=60
StartLimitBurst=3

# 資源限制
MemoryLimit=1G
CPUQuota=200%

# 安全配置
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/opt/fastim/logs

# 環境變數
Environment=ENVIRONMENT=production
EnvironmentFile=/opt/fastim/.env

# 日誌配置
StandardOutput=journal
StandardError=journal
SyslogIdentifier=fastim

[Install]
WantedBy=multi-user.target
```

```bash
# 部署步驟
sudo useradd -r -s /bin/false fastim
sudo mkdir -p /opt/fastim/{bin,config,logs}
sudo cp bin/fastim-linux-arm64 /opt/fastim/bin/
sudo cp config/config.yaml /opt/fastim/config/
sudo cp .env /opt/fastim/
sudo chown -R fastim:fastim /opt/fastim

# 啟用服務
sudo systemctl daemon-reload
sudo systemctl enable fastim
sudo systemctl start fastim

# 檢查狀態
sudo systemctl status fastim
```

---

## 💻 前端客戶端部署

### Flutter 多平台客戶端

#### 1. 環境配置
```bash
cd frontend/flutter

# 安裝依賴
flutter pub get

# 檢查平台支持
flutter doctor -v
```

#### 2. 配置文件
```yaml
# frontend/flutter/lib/config/app_config.dart
class AppConfig {
  static const String apiBaseUrl = String.fromEnvironment(
    'API_BASE_URL',
    defaultValue: 'http://localhost:8080/api/v1',
  );
  
  static const String wsBaseUrl = String.fromEnvironment(
    'WS_BASE_URL', 
    defaultValue: 'ws://localhost:8080/api/v1/ws',
  );
  
  static const String appName = 'FastIM';
  static const String appVersion = '1.0.0';
  
  // 生產環境配置
  static const String prodApiBaseUrl = 'https://api.fastim.com/api/v1';
  static const String prodWsBaseUrl = 'wss://api.fastim.com/api/v1/ws';
}
```

#### 3. 多平台構建

**Linux 桌面應用**
```bash
# 構建 Linux 應用
flutter build linux --release

# 輸出目錄: build/linux/arm64/release/bundle/
# 打包成 .deb 包
sudo apt install -y alien fakeroot

# 創建打包腳本
cat << 'EOF' > package-linux.sh
#!/bin/bash
APP_NAME="fastim"
VERSION="1.0.0"
ARCH="arm64"

mkdir -p package/DEBIAN
mkdir -p package/opt/fastim
mkdir -p package/usr/share/applications
mkdir -p package/usr/share/pixmaps

# 複製文件
cp -r build/linux/arm64/release/bundle/* package/opt/fastim/

# 創建 desktop 文件
cat << DESKTOP > package/usr/share/applications/fastim.desktop
[Desktop Entry]
Name=FastIM
Comment=Real-time Chat Application
Exec=/opt/fastim/fastim
Icon=fastim
Terminal=false
Type=Application
Categories=Network;Chat;
DESKTOP

# 創建控制文件
cat << CONTROL > package/DEBIAN/control
Package: fastim
Version: ${VERSION}
Architecture: ${ARCH}
Maintainer: FastIM Team <team@fastim.com>
Description: FastIM Real-time Chat Application
 High-performance cross-platform real-time chat system
CONTROL

# 構建 deb 包
dpkg-deb --build package fastim_${VERSION}_${ARCH}.deb
EOF

chmod +x package-linux.sh
./package-linux.sh
```

**Windows 桌面應用**
```bash
# 構建 Windows 應用 (需要在 Linux 上交叉編譯)
flutter build windows --release

# 或使用 msix 打包
flutter pub add msix
flutter pub get
flutter build windows --release
dart run msix:create
```

**macOS 桌面應用**
```bash
# 構建 macOS 應用 (需要在 macOS 上執行)
flutter build macos --release

# 創建 DMG 安裝包
# 需要 create-dmg 工具
brew install create-dmg

create-dmg \
  --volname "FastIM Installer" \
  --volicon "assets/icons/app_icon.icns" \
  --window-pos 200 120 \
  --window-size 600 300 \
  --icon-size 100 \
  --icon "FastIM.app" 175 120 \
  --hide-extension "FastIM.app" \
  --app-drop-link 425 120 \
  "FastIM-1.0.0.dmg" \
  "build/macos/Build/Products/Release/"
```

**Android APK**
```bash
# 配置 Android 簽名
# android/key.properties
storePassword=your_keystore_password
keyPassword=your_key_password  
keyAlias=fastim
storeFile=../fastim-key.jks

# 生成簽名金鑰
keytool -genkey -v -keystore fastim-key.jks -keyalg RSA -keysize 2048 -validity 10000 -alias fastim

# 構建 APK
flutter build apk --release
flutter build appbundle --release
```

**iOS IPA**
```bash
# 構建 iOS 應用 (需要在 macOS 上執行)
flutter build ios --release

# 使用 Xcode 進行代碼簽名和打包
# 或使用命令行工具
flutter build ipa --export-options-plist=ios/ExportOptions.plist
```

### Web 客戶端

#### 1. React 前端構建
```bash
cd frontend/web

# 安裝依賴
npm install

# 開發模式
npm start

# 生產構建
npm run build

# 輸出到 build/ 目錄
```

#### 2. Nginx 配置
```nginx
# /etc/nginx/sites-available/fastim-web
server {
    listen 80;
    server_name fastim.yourdomain.com;
    
    # 重定向到 HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name fastim.yourdomain.com;
    
    # SSL 證書配置
    ssl_certificate /etc/letsencrypt/live/fastim.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/fastim.yourdomain.com/privkey.pem;
    
    # 安全頭部
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    
    # Web 應用根目錄
    root /var/www/fastim/build;
    index index.html index.htm;
    
    # 處理 React Router
    location / {
        try_files $uri $uri/ /index.html;
    }
    
    # API 代理
    location /api/ {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
    
    # WebSocket 代理
    location /api/v1/ws {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # 靜態資源緩存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

```bash
# 部署 Web 前端
sudo mkdir -p /var/www/fastim
sudo cp -r frontend/web/build/* /var/www/fastim/
sudo chown -R www-data:www-data /var/www/fastim

# 啟用 Nginx 配置
sudo ln -s /etc/nginx/sites-available/fastim-web /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## ☸️ Kubernetes 部署

### 1. Namespace 和 ConfigMap

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: fastim
  labels:
    name: fastim
---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fastim-config
  namespace: fastim
data:
  config.yaml: |
    server:
      port: "8080"
      host: "0.0.0.0"
    database:
      mongodb:
        url: "mongodb://fastim-user:fastim-password@mongodb-service:27017/fastim"
      redis:
        url: "redis://:redis-password@redis-service:6379/0"
    websocket:
      max_connections: 10000
      heartbeat_interval: 30s
    security:
      jwt_secret: "k8s-jwt-secret-key"
    logging:
      level: "info"
```

### 2. 資料庫部署

```yaml
# k8s/mongodb.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: fastim
spec:
  serviceName: mongodb-service
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:5.0
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: "admin"
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: root-password
        volumeMounts:
        - name: mongodb-storage
          mountPath: /data/db
  volumeClaimTemplates:
  - metadata:
      name: mongodb-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 20Gi
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
  namespace: fastim
spec:
  ports:
  - port: 27017
    targetPort: 27017
  selector:
    app: mongodb
```

```yaml
# k8s/redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: fastim
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        command:
        - redis-server
        - "--requirepass"
        - "$(REDIS_PASSWORD)"
        env:
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: password
        volumeMounts:
        - name: redis-storage
          mountPath: /data
      volumes:
      - name: redis-storage
        persistentVolumeClaim:
          claimName: redis-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: fastim
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
```

### 3. FastIM 後端部署

```yaml
# k8s/fastim-backend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastim-backend
  namespace: fastim
spec:
  replicas: 3
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
        image: fastim:latest
        ports:
        - containerPort: 8080
        env:
        - name: ENVIRONMENT
          value: "production"
        - name: CONFIG_PATH
          value: "/app/config/config.yaml"
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
        livenessProbe:
          httpGet:
            path: /api/v1/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /api/v1/health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: config-volume
        configMap:
          name: fastim-config
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
    port: 80
    targetPort: 8080
  type: ClusterIP
```

### 4. Ingress 配置

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fastim-ingress
  namespace: fastim
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    nginx.ingress.kubernetes.io/websocket-services: "fastim-backend-service"
spec:
  tls:
  - hosts:
    - fastim.yourdomain.com
    secretName: fastim-tls
  rules:
  - host: fastim.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: fastim-backend-service
            port:
              number: 80
```

### 5. Secret 配置

```yaml
# k8s/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
  namespace: fastim
type: Opaque
data:
  root-password: <base64-encoded-password>
  user-password: <base64-encoded-password>
---
apiVersion: v1
kind: Secret
metadata:
  name: redis-secret
  namespace: fastim
type: Opaque
data:
  password: <base64-encoded-password>
```

### 6. 部署腳本

```bash
#!/bin/bash
# k8s/deploy.sh

set -e

NAMESPACE="fastim"
IMAGE_TAG=${1:-latest}

echo "🚀 開始部署 FastIM 到 Kubernetes..."

# 創建 namespace
echo "📋 創建 namespace..."
kubectl apply -f namespace.yaml

# 創建 secrets
echo "🔐 創建 secrets..."
kubectl apply -f secrets.yaml

# 部署 MongoDB
echo "🗄️ 部署 MongoDB..."
kubectl apply -f mongodb.yaml

# 等待 MongoDB 就緒
echo "⏳ 等待 MongoDB 就緒..."
kubectl wait --for=condition=ready pod -l app=mongodb -n $NAMESPACE --timeout=300s

# 部署 Redis
echo "🔄 部署 Redis..."
kubectl apply -f redis.yaml

# 等待 Redis 就緒
echo "⏳ 等待 Redis 就緒..."
kubectl wait --for=condition=ready pod -l app=redis -n $NAMESPACE --timeout=300s

# 創建 ConfigMap
echo "📝 創建 ConfigMap..."
kubectl apply -f configmap.yaml

# 部署 FastIM 後端
echo "🚀 部署 FastIM 後端..."
kubectl set image deployment/fastim-backend fastim=fastim:$IMAGE_TAG -n $NAMESPACE || \
kubectl apply -f fastim-backend.yaml

# 等待部署完成
echo "⏳ 等待 FastIM 後端就緒..."
kubectl wait --for=condition=available deployment/fastim-backend -n $NAMESPACE --timeout=300s

# 部署 Ingress
echo "🌐 部署 Ingress..."
kubectl apply -f ingress.yaml

echo "✅ FastIM 部署完成！"

# 顯示服務狀態
echo "📊 服務狀態:"
kubectl get pods -n $NAMESPACE
kubectl get services -n $NAMESPACE
kubectl get ingress -n $NAMESPACE
```

---

## 🐳 Docker 部署

### 1. Docker Compose 完整部署

```yaml
# docker-compose.yml
version: '3.8'

services:
  mongodb:
    image: mongo:5.0
    container_name: fastim-mongodb
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
      MONGO_INITDB_DATABASE: fastim
    ports:
      - "${MONGO_PORT:-27017}:27017"
    volumes:
      - mongodb_data:/data/db
      - ./init-mongo.js:/docker-entrypoint-initdb.d/init-mongo.js:ro
    networks:
      - fastim-network
    healthcheck:
      test: ["CMD", "mongo", "--eval", "db.adminCommand('ping')"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis:
    image: redis:7-alpine
    container_name: fastim-redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    ports:
      - "${REDIS_PORT:-6379}:6379"
    volumes:
      - redis_data:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf
    networks:
      - fastim-network
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  fastim-backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: fastim-backend
    restart: unless-stopped
    ports:
      - "${SERVER_PORT:-8080}:8080"
    environment:
      - ENVIRONMENT=production
      - MONGODB_URL=mongodb://fastim_user:${MONGO_PASSWORD}@mongodb:27017/fastim
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/0
      - JWT_SECRET=${JWT_SECRET}
      - LOG_LEVEL=${LOG_LEVEL:-info}
    depends_on:
      mongodb:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - fastim-network
    volumes:
      - ./backend/logs:/app/logs
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/api/v1/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:alpine
    container_name: fastim-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./frontend/web/build:/usr/share/nginx/html
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - fastim-backend
    networks:
      - fastim-network

volumes:
  mongodb_data:
  redis_data:

networks:
  fastim-network:
    driver: bridge
```

### 2. 環境變數配置

```bash
# .env
# 資料庫配置
MONGO_ROOT_PASSWORD=secure_mongo_root_password
MONGO_PASSWORD=secure_mongo_user_password
REDIS_PASSWORD=secure_redis_password

# 服務配置
SERVER_PORT=8080
MONGO_PORT=27017
REDIS_PORT=6379

# 安全配置
JWT_SECRET=your-very-secure-jwt-secret-key
LOG_LEVEL=info

# SSL 配置 (可選)
SSL_ENABLED=false
```

### 3. Backend Dockerfile

```dockerfile
# backend/Dockerfile
FROM golang:1.21-alpine AS builder

# 安裝 build 依賴
RUN apk add --no-cache git ca-certificates tzdata

WORKDIR /app

# 複製 go mod 文件
COPY go.mod go.sum ./
RUN go mod download

# 複製源碼
COPY . .

# 構建應用
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags='-w -s -extldflags "-static"' \
    -o fastim cmd/fastim/main.go

FROM alpine:latest

# 安裝運行依賴
RUN apk --no-cache add ca-certificates tzdata curl

WORKDIR /app

# 創建非特權用戶
RUN addgroup -g 1001 -S fastim && \
    adduser -S fastim -u 1001

# 複製二進制文件
COPY --from=builder /app/fastim .
COPY --from=builder /app/config ./config

# 創建日誌目錄
RUN mkdir logs && chown -R fastim:fastim /app

USER fastim

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/api/v1/health || exit 1

CMD ["./fastim"]
```

### 4. 部署腳本

```bash
#!/bin/bash
# deploy-docker.sh

set -e

# 配置
COMPOSE_FILE="docker-compose.yml"
ENV_FILE=".env"

echo "🚀 開始 Docker 部署 FastIM..."

# 檢查文件存在
if [ ! -f "$COMPOSE_FILE" ]; then
    echo "❌ 找不到 $COMPOSE_FILE"
    exit 1
fi

if [ ! -f "$ENV_FILE" ]; then
    echo "❌ 找不到 $ENV_FILE"
    exit 1
fi

# 構建並啟動服務
echo "🔧 構建並啟動服務..."
docker-compose --file $COMPOSE_FILE --env-file $ENV_FILE up -d --build

# 等待服務就緒
echo "⏳ 等待服務啟動..."
sleep 10

# 檢查服務狀態
echo "📊 檢查服務狀態..."
docker-compose ps

# 檢查健康狀態
echo "🏥 檢查健康狀態..."
for service in fastim-mongodb fastim-redis fastim-backend; do
    echo "檢查 $service..."
    docker inspect --format='{{.State.Health.Status}}' $service || echo "無健康檢查"
done

# 顯示日誌
echo "📋 最近日誌:"
docker-compose logs --tail=20

echo "✅ FastIM Docker 部署完成！"
echo "🌐 服務地址: http://localhost:8080"
echo "📊 監控地址: http://localhost:8080/api/v1/stats"
```

---

## 📊 監控和日誌

### 1. Prometheus 監控

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "fastim_rules.yml"

scrape_configs:
  - job_name: 'fastim'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/api/v1/metrics'
    scrape_interval: 10s

  - job_name: 'mongodb'
    static_configs:
      - targets: ['mongodb-exporter:9216']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
```

### 2. Grafana 儀表板

```json
{
  "dashboard": {
    "title": "FastIM Monitoring",
    "panels": [
      {
        "title": "WebSocket 連接數",
        "type": "stat",
        "targets": [
          {
            "expr": "fastim_websocket_connections_total"
          }
        ]
      },
      {
        "title": "消息發送率",
        "type": "graph", 
        "targets": [
          {
            "expr": "rate(fastim_messages_sent_total[5m])"
          }
        ]
      }
    ]
  }
}
```

### 3. ELK Stack 日誌收集

```yaml
# logging/elasticsearch.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.5.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.5.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  logstash:
    image: docker.elastic.co/logstash/logstash:8.5.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    depends_on:
      - elasticsearch
```

---

## 🔒 SSL/TLS 配置

### 1. Let's Encrypt 自動證書

```bash
#!/bin/bash
# ssl-setup.sh

DOMAIN="fastim.yourdomain.com"
EMAIL="admin@yourdomain.com"

# 安裝 Certbot
sudo apt install -y certbot python3-certbot-nginx

# 獲取證書
sudo certbot --nginx -d $DOMAIN --email $EMAIL --agree-tos --non-interactive

# 設置自動更新
sudo crontab -l | { cat; echo "0 12 * * * /usr/bin/certbot renew --quiet"; } | sudo crontab -
```

### 2. Nginx SSL 配置

```nginx
# ssl-nginx.conf
server {
    listen 443 ssl http2;
    server_name fastim.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/fastim.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/fastim.yourdomain.com/privkey.pem;

    # SSL 配置
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    # 現代加密配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # HSTS
    add_header Strict-Transport-Security "max-age=63072000" always;

    # 其他配置...
}
```

---

## ✅ 部署驗證

### 1. 服務健康檢查腳本

```bash
#!/bin/bash
# health-check.sh

BASE_URL="https://fastim.yourdomain.com"
API_URL="$BASE_URL/api/v1"

echo "🏥 FastIM 部署健康檢查"
echo "=========================="

# 檢查 HTTP 服務
echo "📡 檢查 HTTP 服務..."
HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$API_URL/health")
if [ "$HTTP_STATUS" = "200" ]; then
    echo "✅ HTTP 服務正常 (200)"
else
    echo "❌ HTTP 服務異常 ($HTTP_STATUS)"
    exit 1
fi

# 檢查 WebSocket
echo "🔌 檢查 WebSocket 服務..."
# 使用 websocat 工具測試 WebSocket 連接
if command -v websocat &> /dev/null; then
    WS_URL="${BASE_URL/https/wss}/api/v1/ws"
    timeout 5 websocat "$WS_URL" --text <<< '{"type":"ping"}' | grep -q "pong"
    if [ $? -eq 0 ]; then
        echo "✅ WebSocket 服務正常"
    else
        echo "❌ WebSocket 服務異常"
        exit 1
    fi
else
    echo "⚠️  未安裝 websocat，跳過 WebSocket 測試"
fi

# 檢查資料庫連接
echo "🗄️ 檢查資料庫連接..."
DB_STATUS=$(curl -s "$API_URL/stats" | jq -r '.database.status')
if [ "$DB_STATUS" = "connected" ]; then
    echo "✅ 資料庫連接正常"
else
    echo "❌ 資料庫連接異常"
    exit 1
fi

# 檢查系統資源
echo "💻 檢查系統資源..."
STATS=$(curl -s "$API_URL/stats")
CPU_USAGE=$(echo "$STATS" | jq -r '.system.cpu_percent')
MEMORY_USAGE=$(echo "$STATS" | jq -r '.system.memory_percent')

echo "   CPU 使用率: $CPU_USAGE%"
echo "   記憶體使用率: $MEMORY_USAGE%"

# 檢查 WebSocket 連接數
CONNECTIONS=$(echo "$STATS" | jq -r '.websocket.active_connections')
echo "   活躍連接數: $CONNECTIONS"

echo ""
echo "✅ FastIM 部署驗證完成！所有服務正常運行。"
```

### 2. 自動化部署測試

```bash
#!/bin/bash
# deployment-test.sh

set -e

echo "🧪 開始部署測試..."

# 功能測試
echo "📋 執行功能測試..."
cd tests && go test ./... -v

# 性能測試
echo "⚡ 執行性能測試..."
cd load-tests && ./run-load-test.sh

# 安全測試
echo "🔐 執行安全測試..."
./security-scan.sh

echo "✅ 所有測試通過！"
```

---

## 🔧 故障排除

### 常見問題

1. **WebSocket 連接失敗**
   - 檢查防火牆設置
   - 確認 Nginx 配置正確
   - 檢查 SSL 證書

2. **資料庫連接超時**
   - 檢查 MongoDB/Redis 服務狀態
   - 確認網路連接
   - 檢查認證配置

3. **高記憶體使用**
   - 調整 Go GC 參數
   - 檢查 WebSocket 連接洩漏
   - 優化資料庫查詢

詳細故障排除指南請參考 [troubleshooting.md](troubleshooting.md)。

---

## 📚 相關文檔

- [API 文檔](api.md)
- [使用手冊](user-guide.md)
- [系統架構](architecture.md)
- [配置說明](configuration.md)
- [故障排除](troubleshooting.md)

---

**部署完成！** 🎉

如有問題，請參考相關文檔或聯繫技術支援。