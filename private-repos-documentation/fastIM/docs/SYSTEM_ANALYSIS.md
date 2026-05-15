# FastIM 系統架構分析報告

**生成時間**: 2026-02-05  
**系統**: FastIM - 高性能跨平台實時聊天系統

## 📋 系統概述

FastIM 是一個專為**超低延遲、高效率工作協作**設計的實時聊天系統，支援多平台客戶端。

### 🎯 核心特性
- **超低延遲**: WebSocket 實時通訊，<50ms 響應時間
- **跨平台支援**: iOS, Android, Linux, Windows, macOS, Web
- **高性能架構**: ARM64 原生優化，K8s 容器化部署
- **靈活部署**: 三個優化版本（並發、動態心跳、漸進式）

---

## 🏗️ 技術架構詳解

### 後端技術棧
| 技術 | 版本 | 用途 |
|------|------|------|
| Go | 1.21+ | 高性能 WebSocket 伺服器 |
| MongoDB | Latest | 訊息持久化存儲 |
| Redis | Latest | 即時快取和連接狀態管理 |
| MySQL | Latest | 用戶數據和輕量化版本 |
| Gin | v1.9.1 | HTTP 框架 |
| Gorilla WebSocket | v1.5.0 | WebSocket 通訊 |

### 前端技術棧
| 平台 | 技術 | 語言 | 狀態 |
|------|------|------|------|
| iOS/Android/Linux/Windows/macOS | Flutter | Dart | 開發中 |
| Web | React (計劃) | TypeScript | 計劃中 |
| 通訊協議 | WebSocket + JSON | JSON | 已實現 |

### 部署架構
- **容器化**: Docker + Kubernetes
- **負載均衡**: 分散式 WebSocket Hub
- **CPU架構**: ARM64 優化編譯
- **集群**: Master + Worker + Gateway 三節點

---

## 📂 完整項目結構

```
fastIM/
├── backend/                      # Go 後端服務
│   ├── cmd/
│   │   └── fastim/main.go       # 應用入口點
│   ├── internal/
│   │   ├── handler/              # HTTP 處理器
│   │   │   ├── handlers.go
│   │   │   └── handlers.go.old
│   │   ├── service/              # 業務邏輯層
│   │   │   ├── service.go
│   │   │   └── mysql_message_service.go
│   │   └── websocket/            # WebSocket 實時通訊
│   │       ├── concurrent_hub.go    # 並發 Hub
│   │       ├── dynamic_heartbeat.go # 動態心跳
│   │       ├── progressive_heartbeat.go # 漸進式心跳
│   │       └── hub.go.old
│   ├── pkg/
│   │   ├── config/              # 配置管理
│   │   ├── database/            # 數據庫連接
│   │   └── logger/              # 日誌系統
│   ├── database/
│   │   └── mysql_schema.sql     # 數據庫架構
│   ├── Dockerfile               # 容器化配置
│   ├── go.mod                   # Go 模塊定義
│   └── go.sum                   # 依賴版本鎖定
│
├── frontend/
│   ├── flutter/                 # Flutter 跨平台應用
│   │   ├── lib/
│   │   │   ├── main.dart
│   │   │   ├── models/          # 數據模型
│   │   │   ├── screens/         # UI 頁面
│   │   │   ├── services/        # 業務邏輯
│   │   │   ├── utils/           # 工具函數
│   │   │   └── widgets/         # 可重用組件
│   │   ├── pubspec.yaml         # Flutter 依賴配置
│   │   └── README.md
│   └── web/                     # Web 版本
│       └── index.html           # HTML 入口
│
├── k8s/                         # Kubernetes 部署
│   ├── backend.yaml             # 後端部署配置
│   └── database.yaml            # 數據庫部署配置
│
├── docs/                        # 文檔
│   ├── DEPLOYMENT_GUIDE.md      # 部署指南
│   ├── DEPLOY_SCRIPT.md         # 部署腳本文檔
│   └── DYNAMIC_HEARTBEAT.md     # 動態心跳優化說明
│
├── scripts/                     # 構建和部署腳本
│   ├── deploy-concurrent.sh     # 並發版本部署
│   ├── deploy-dynamic-heartbeat.sh # 動態心跳版本部署
│   ├── deploy-progressive.sh    # 漸進式版本部署
│   ├── deploy-web-fixed.sh      # Web 固定部署
│   ├── deploy-web.sh            # Web 通用部署
│   ├── deploy.sh                # 通用部署腳本
│   └── README.md                # 腳本文檔
│
├── README.md                    # 項目主文檔
├── MEMORY.md                    # 系統記憶
├── TASK_MANAGEMENT.md           # 任務管理
└── DEVELOPMENT_TIME_TRACKING.md # 開發時間追蹤
```

---

## 🚀 三大部署版本詳解

### 1️⃣ Concurrent (並發) 版本
**特點**: 多工並發處理  
**優化**: 充分利用多核心 CPU，提高吞吐量  
**適用場景**: 高並發聊天場景，大量用戶同時在線  

構建方式:
```bash
cd backend
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build -o fastim ./cmd/fastim
```

關鍵文件: `internal/websocket/concurrent_hub.go`

---

### 2️⃣ Dynamic Heartbeat (動態心跳) 版本
**特點**: 智能心跳機制 + MySQL 輕量化  
**優化**: 
- 根據網絡狀況動態調整心跳間隔
- 使用 MySQL 替代 MongoDB，減少資源消耗
- 更適合資源受限的環境

構建方式:
```bash
go get github.com/go-sql-driver/mysql
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build -o fastim ./cmd/fastim
```

關鍵文件: `internal/websocket/dynamic_heartbeat.go`  
數據庫: `backend/database/mysql_schema.sql`

---

### 3️⃣ Progressive Heartbeat (漸進式心跳) 版本
**特點**: 逐步優化的心跳策略  
**優化**:
- 初始頻繁心跳，逐漸降低頻率
- 檢測到異常立即恢復高頻
- 最平衡的性能和資源消耗

構建方式:
```bash
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build -o fastim ./cmd/fastim
```

關鍵文件: `internal/websocket/progressive_heartbeat.go`  
文檔: `docs/DYNAMIC_HEARTBEAT.md`

---

## 🛠️ 主要模塊功能

### 1. WebSocket Hub (即時通訊核心)
- **concurrent_hub.go**: 並發處理多個連接
- **dynamic_heartbeat.go**: 動態調整心跳間隔
- **progressive_heartbeat.go**: 漸進式心跳優化

功能:
- 連接管理: 添加、移除、廣播消息
- 心跳檢測: 保活連接，檢測斷線
- 消息路由: 將消息轉發到指定用戶

### 2. Service 層 (業務邏輯)
- **service.go**: 通用業務邏輯
- **mysql_message_service.go**: 消息存儲服務

功能:
- 消息存儲和檢索
- 用戶會話管理
- 離線消息隊列

### 3. Handler 層 (HTTP 接口)
- **handlers.go**: API 端點實現

功能:
- 用戶認證和授權
- 消息上傳下載
- 連接狀態查詢

### 4. 配置和日誌
- **pkg/config/config.go**: 環境變量配置
- **pkg/logger/logger.go**: 結構化日誌
- **pkg/database/database.go**: 數據庫初始化

---

## 📊 性能指標對比

| 指標 | Concurrent | Dynamic Heartbeat | Progressive |
|------|-----------|------------------|------------|
| 延遲 | < 50ms | < 50ms | < 50ms |
| CPU使用 | 中 | 低 | 中 |
| 內存使用 | 中 | 低 | 低 |
| 吞吐量 | 高 | 中 | 高 |
| 資源消耗 | 中 | 低 | 低 |
| 網絡心跳 | 固定10s | 動態5-30s | 逐步調整 |
| 數據庫 | MongoDB | MySQL | MongoDB |

---

## 🚀 部署流程

### 前置條件
1. ✅ Go 1.21+ 已安裝
2. ✅ Linux ARM64 環境（或使用 Docker）
3. ✅ 部署集群: Master、Worker、Gateway 三節點
4. ✅ Kubernetes 環境（可選，用於生產部署）

### 構建順序
1. **驗證環境**: 檢查 Go 版本和依賴
2. **構建後端**: 編譯三個版本的 FastIM
3. **準備前端**: Flutter 應用依賴和構建
4. **打包部署**: 生成部署包並上傳到服務器

### 部署命令
```bash
# 並發版本
./scripts/deploy-concurrent.sh

# 動態心跳版本
./scripts/deploy-dynamic-heartbeat.sh

# 漸進式心跳版本
./scripts/deploy-progressive.sh
```

---

## 📱 前端支援平台

### Flutter 跨平台應用
- ✅ iOS (需要 Xcode)
- ✅ Android (需要 Android Studio)
- ✅ Linux Desktop
- ✅ Windows Desktop
- ✅ macOS Desktop

### 構建前端
```bash
cd frontend/flutter
flutter pub get
flutter run        # 開發環境
flutter build web  # Web 構建
```

---

## 🔧 配置參數

### 環境變量 (.env)
```bash
# 數據庫配置
MONGO_URL=mongodb://localhost:27017
REDIS_URL=redis://localhost:6379
MYSQL_URL=root:password@tcp(localhost:3306)/fastim

# 服務配置
LOG_LEVEL=info
PORT=8000
HEARTBEAT_INTERVAL=10s

# WebSocket 配置
WS_READ_TIMEOUT=60s
WS_WRITE_TIMEOUT=60s
WS_MESSAGE_SIZE_LIMIT=65536
```

---

## 📝 開發進度

- [x] 項目初始化
- [x] Go 後端基礎架構
- [x] 三種 WebSocket Hub 實現
- [x] 消息存儲服務
- [x] HTTP API 接口
- [ ] Flutter 客戶端開發
- [ ] Web 客戶端開發
- [ ] 全面測試和優化
- [ ] 部署和上線

---

## 📚 相關文檔

- [部署指南](docs/DEPLOYMENT_GUIDE.md)
- [部署腳本說明](docs/DEPLOY_SCRIPT.md)
- [動態心跳優化](docs/DYNAMIC_HEARTBEAT.md)
- [開發時間追蹤](DEVELOPMENT_TIME_TRACKING.md)
- [任務管理](TASK_MANAGEMENT.md)

---

**系統分析完成** ✅
