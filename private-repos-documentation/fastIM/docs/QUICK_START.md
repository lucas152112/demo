# FastIM 快速開始指南

**最後更新**: 2026年2月6日

## 📋 目錄

1. [系統概述](#系統概述)
2. [環境需求](#環境需求)
3. [本地開發](#本地開發)
4. [部署指南](#部署指南)
5. [測試驗證](#測試驗證)
6. [常見問題](#常見問題)

---

## 系統概述

FastIM 是一個**高性能跨平台實時聊天系統**，包含：

| 組件 | 技術 | 狀態 |
|------|------|------|
| **後端服務** | Go + WebSocket | ✅ 3個版本已構建 |
| **Web 前端** | Flutter Web | ✅ 已構建 |
| **數據庫** | MongoDB/MySQL + Redis | 📦 部署時安裝 |
| **容器化** | Docker + Kubernetes | 📚 配置已準備 |

---

## 環境需求

### 開發環境

```bash
# 最低要求
- Go 1.21+ (已安裝: 1.25.7)
- Flutter 3.30+ (已安裝: 3.38.9)
- Node.js 16+ (用於 Web 部署)
- macOS/Linux/Windows

# 可選但推薦
- Docker & Docker Compose
- Kubernetes kubectl
- Git
```

### 系統要求

| 組件 | 最低配置 | 推薦配置 |
|------|---------|---------|
| **CPU** | 1 核心 | 4 核心+ |
| **內存** | 512MB | 2GB+ |
| **存儲** | 100MB | 500MB+ |
| **網絡** | 10Mbps | 100Mbps+ |

---

## 本地開發

### 1. 克隆和準備

```bash
cd /Users/jackson/Documents/private/fastIM
ls -la  # 確認項目結構
```

### 2. 後端開發

#### 查看已構建的版本

```bash
ls -lh backend/fastim*

# 輸出:
# -rwxr-xr-x fastim-concurrent       17MB  (並發版本)
# -rwxr-xr-x fastim-dynamic          17MB  (動態心跳版本)
# -rwxr-xr-x fastim-progressive      17MB  (漸進式版本)
# -rwxr-xr-x fastim-*-linux          16MB  (Linux 部署版本)
```

#### 運行後端服務

```bash
# 運行 Concurrent 版本（推薦開發使用）
cd backend
./fastim-concurrent

# 或使用 go run (開發模式)
go run cmd/fastim/main.go
```

#### 開發後端

```bash
# 修改代碼後重新構建
cd backend
go build -o fastim ./cmd/fastim

# 運行新構建的版本
./fastim
```

### 3. 前端開發

#### Flutter Web 開發

```bash
cd frontend/flutter

# 啟動開發伺服器
flutter run -d chrome

# 或構建發佈版本
flutter build web --release
```

#### Web 應用測試

```bash
# 進入 Web 構建目錄
cd frontend/flutter/build/web

# 啟動本地伺服器
python3 -m http.server 8080

# 訪問 http://localhost:8080
```

#### Flutter 其他平台開發

```bash
# iOS (macOS + Xcode)
flutter run -d ios

# Android (Android Studio)
flutter run -d android

# Windows Desktop
flutter run -d windows

# macOS Desktop
flutter run -d macos

# Linux Desktop
flutter run -d linux
```

---

## 部署指南

### 快速部署（單機）

#### 1. 準備服務器

```bash
# SSH 連接到服務器
ssh root@your-server-ip

# 建立應用目錄
mkdir -p /opt/fastim/{bin,config,logs,data}
cd /opt/fastim
```

#### 2. 上傳構建産品

```bash
# 從本地上傳後端應用
scp backend/fastim-concurrent-linux root@your-server-ip:/opt/fastim/bin/

# 上傳配置文件
scp backend/.env root@your-server-ip:/opt/fastim/config/
```

#### 3. 配置環境變量

```bash
# 編輯配置文件
cat > /opt/fastim/config/.env << EOF
# 服務配置
PORT=8000
LOG_LEVEL=info

# 數據庫配置
MONGO_URL=mongodb://localhost:27017
REDIS_URL=redis://localhost:6379
MYSQL_URL=root:password@tcp(localhost:3306)/fastim

# WebSocket 配置
HEARTBEAT_INTERVAL=10s
WS_READ_TIMEOUT=60s
WS_WRITE_TIMEOUT=60s
EOF
```

#### 4. 啟動服務

```bash
# 給執行文件賦予權限
chmod +x /opt/fastim/bin/fastim-concurrent-linux

# 運行服務 (前台模式)
/opt/fastim/bin/fastim-concurrent-linux

# 或使用 systemd (後台模式)
sudo tee /etc/systemd/system/fastim.service > /dev/null << EOF
[Unit]
Description=FastIM Real-time Chat Service
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/fastim
ExecStart=/opt/fastim/bin/fastim-concurrent-linux
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# 啟動服務
sudo systemctl daemon-reload
sudo systemctl start fastim
sudo systemctl enable fastim
sudo systemctl status fastim
```

### 前端部署

#### 部署到 Nginx

```bash
# 上傳 Web 文件
scp -r frontend/flutter/build/web/* root@your-server-ip:/var/www/fastim/

# 配置 Nginx
sudo tee /etc/nginx/sites-available/fastim > /dev/null << EOF
server {
    listen 80;
    server_name your-domain.com;

    location / {
        root /var/www/fastim;
        try_files \$uri \$uri/ /index.html;
    }

    location /api {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    location /ws {
        proxy_pass http://localhost:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
EOF

# 啟用配置
sudo ln -s /etc/nginx/sites-available/fastim /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

### Kubernetes 部署

#### 部署到 K8s 集群

```bash
# 構建 Docker 鏡像
cd backend
docker build -t fastim:latest .

# 推送到 Docker Registry
docker tag fastim:latest your-registry/fastim:latest
docker push your-registry/fastim:latest

# 部署到 K8s
kubectl apply -f k8s/backend.yaml
kubectl apply -f k8s/database.yaml

# 驗證部署
kubectl get pods
kubectl get svc
```

---

## 測試驗證

### 後端測試

#### 檢查服務狀態

```bash
# 檢查端口
curl -s http://localhost:8000/health || echo "服務未運行"

# 查看日誌
tail -f /opt/fastim/logs/app.log
```

#### WebSocket 連接測試

```bash
# 使用 wscat 測試
npm install -g wscat
wscat -c ws://localhost:8000?username=testuser

# 發送消息
> {"type": "message", "channel": "general", "content": "Hello"}
```

### 前端測試

#### 訪問 Web 應用

```
http://your-server-ip:8000
或
http://your-domain.com
```

#### 測試功能清單

- [ ] 用戶登錄
- [ ] WebSocket 連接
- [ ] 發送消息
- [ ] 接收消息
- [ ] 頻道切換
- [ ] 用戶列表更新
- [ ] 連接斷開/重連
- [ ] 離線消息隊列

---

## 常見問題

### Q1: 如何選擇合適的版本？

| 場景 | 推薦版本 | 原因 |
|------|---------|------|
| 開發環境 | Concurrent | 功能完整，調試方便 |
| 高並發 (1000+) | Concurrent | 最高吞吐量 |
| 資源受限 | Dynamic Heartbeat | 最低資源消耗 |
| 生產環境 | Progressive | 最佳性能平衡 |

### Q2: 後端無法連接數據庫？

```bash
# 檢查 MongoDB
mongo --eval "db.adminCommand('ping')"

# 檢查 Redis
redis-cli ping

# 檢查 MySQL
mysql -u root -p -e "SELECT 1"

# 查看後端日誌
cat /opt/fastim/logs/app.log | grep -i error
```

### Q3: Web 應用無法連接後端？

```bash
# 檢查後端服務
curl -v http://localhost:8000

# 檢查防火牆
sudo iptables -L | grep 8000

# 檢查 CORS 配置
curl -i -X OPTIONS http://localhost:8000
```

### Q4: WebSocket 連接頻繁斷開？

```bash
# 檢查網絡狀況
ping your-server-ip
traceroute your-server-ip

# 增加心跳間隔
HEARTBEAT_INTERVAL=30s  # 增加到 30 秒

# 檢查防火牆/代理設置
```

### Q5: 如何更新應用？

```bash
# 1. 構建新版本
cd backend
go build -o fastim ./cmd/fastim

# 2. 停止舊服務
sudo systemctl stop fastim

# 3. 上傳新文件
scp fastim root@server:/opt/fastim/bin/

# 4. 啟動新服務
sudo systemctl start fastim

# 5. 驗證
sudo systemctl status fastim
```

---

## 文件結構速查

```
fastIM/
├── SYSTEM_ANALYSIS.md           ← 詳細系統分析
├── BUILD_REPORT.md              ← 構建詳細報告
├── QUICK_START.md               ← 本文件

├── backend/
│   ├── fastim*                  ← 已構建的可執行文件
│   ├── cmd/fastim/main.go      ← 後端入口
│   ├── internal/                ← 業務邏輯
│   └── pkg/                     ← 公共庫

├── frontend/
│   ├── flutter/
│   │   ├── build/web/          ← Web 構建産品 ✅
│   │   └── lib/                ← Dart 源代碼
│   └── web/                    ← 原始 Web 應用

├── k8s/                         ← Kubernetes 部署
├── scripts/                     ← 部署腳本
└── docs/                        ← 文檔
```

---

## 有用的命令

```bash
# 查看所有可用的版本
ls -lh /Users/jackson/Documents/private/fastIM/backend/fastim*

# 查看 Flutter Web 構建結果
du -sh /Users/jackson/Documents/private/fastIM/frontend/flutter/build/web

# 檢查後端依賴
cd backend && go list -m all

# 檢查前端依賴
cd frontend/flutter && flutter pub deps

# 清理構建文件
cd backend && go clean
cd frontend/flutter && flutter clean

# 重新構建所有版本
./scripts/rebuild-all.sh  # (可選腳本)
```

---

## 下一步

1. **閱讀詳細文檔**
   - [完整系統分析](SYSTEM_ANALYSIS.md)
   - [部署指南](docs/DEPLOYMENT_GUIDE.md)
   - [動態心跳優化](docs/DYNAMIC_HEARTBEAT.md)

2. **本地開發**
   - 運行後端服務
   - 啟動 Web 應用
   - 測試 WebSocket 連接

3. **部署上線**
   - 選擇合適的版本
   - 配置數據庫
   - 部署到服務器

4. **監控維護**
   - 設置日誌監控
   - 配置健康檢查
   - 自動備份配置

---

## 技術支援

- 📖 查閱文檔: [所有文檔](docs/)
- 🐛 報告問題: [問題追蹤](TASK_MANAGEMENT.md)
- ⏱️ 查看進度: [開發時間](DEVELOPMENT_TIME_TRACKING.md)

---

**快速開始完成！** 🚀

根據上述指南，你現在可以：
- ✅ 了解系統架構
- ✅ 運行本地開發
- ✅ 部署到生產環境
- ✅ 驗證和測試

祝構建順利！
