# lucas152112/TickServer

**Description**: Tick收發

**Primary Language**: Go

**Created**: 2025-09-17T09:38:45Z
**Last Updated**: 2025-11-30T16:21:07Z
**Stars**: 0 | **Forks**: 0
**Archived**: False

## Language Statistics

| Language | Bytes | Percentage |
|----------|-------|------------|
| Go | 235,570 | 56.9% |
| Shell | 117,432 | 28.4% |
| HTML | 39,354 | 9.5% |
| Python | 17,993 | 4.3% |
| Lua | 2,868 | 0.7% |
| Batchfile | 463 | 0.1% |

## Repository Information

- **GitHub URL**: https://github.com/lucas152112/TickServer
- **SSH URL**: git@github.com:lucas152112/TickServer.git
- **Clone URL**: https://github.com/lucas152112/TickServer.git

## System Architecture

### Overview

This project uses a modular architecture with the following technology stack:

| Language | Bytes | Percentage |
|----------|-------|------------|
| Go | 235,570 | 56.9% |
| Shell | 117,432 | 28.4% |
| HTML | 39,354 | 9.5% |
| Python | 17,993 | 4.3% |
| Lua | 2,868 | 0.7% |
| Batchfile | 463 | 0.1% |

### Key Components

1. **Core Application** - Main business logic and functionality
2. **Data Layer** - Database and data access components
3. **API Layer** - RESTful or GraphQL interfaces
4. **Client Interface** - User-facing applications or services

### Deployment

- **Containerization**: Docker-based deployment
- **Orchestration**: Kubernetes for scalable deployment
- **CI/CD**: Automated testing and deployment pipeline

### Dependencies

- See package manager files for detailed dependencies
## README Content

```
# TickServer

期貨 Tick 資料收發伺服器 - 提供 TCP 即時資料接收與 Redis 儲存服務

[![Go Version](https://img.shields.io/badge/Go-1.22.2-blue.svg)](https://golang.org/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

## 📋 功能特性

- ✅ **TCP 伺服器** - 即時接收 Tick 資料 (Port 30001)
- ✅ **HTTP API** - 狀態查詢服務 (Port 8080)
- ✅ **Redis 儲存** - 高效能資料持久化
- ✅ **使用者認證** - 安全的帳號密碼驗證
- ✅ **交易時段控制** - 可設定啟動/結束時間
- ✅ **多模式支援** - test/day/night 模式
- ✅ **詳細日誌** - UTF-8 編碼，日期輪換
- ✅ **測試工具** - 完整的測試套件

## 🚀 快速開始

### 安裝與編譯

```powershell
# 克隆專案
git clone https://github.com/lucas152112/TickServer.git
cd TickServer

# 安裝依賴
go mod download

# 編譯主程式
go build -o bin/TickServer.exe main.go
```

### 執行程式

```powershell
# 測試模式 (不限時間)
.\bin\TickServer.exe test

# 日間模式 (依設定時間)
.\bin\TickServer.exe day

# 夜間模式 (依設定時間)
.\bin\TickServer.exe night
```

### 測試連線

```powershell
# 編譯並執行測試客戶端
go build -o test/test_client.exe test/test_client.go
.\test\test_client.exe

# 查看 HTTP 狀態
curl http://localhost:8080/status
```

### 查看日誌

```cmd
# 使用批次檔查看 (自動處理 UTF-8 編碼)
viewlog.bat log\test_2025-11-04.log

# 列出可用的日誌檔案
viewlog.bat
```

## 📚 文件

- **[專案初始化指南](PROJECT_INIT.md)** - 完整的專案說明與設定
- **[快速參考](QUICK_REFERENCE.md)** - 常用命令速查
- **[開發日誌](Develop_Dairy.md)** - 所有變更記錄
- **[開發日誌指南](DEVELOP_LOG_GUIDE.md)** - 如何更新開發日誌
- **[編碼問題解決](ENCODING_FIX_SUMMARY.md)** - 日誌中文顯示修正
- **[VS Code 錯誤說明](VS_CODE_ERRORS_EXPLAINED.md)** - IDE 誤報處理

## 🔧 技術棧

- **Go** 1.22.2
- **Gin** v1.10.1 - HTTP 框架
- **Redis** v8.11.5 - 資料儲存

## 📊 專案結構

```
TickServer/
├── main.go              # 主程式
├── common/              # 共用模組
│   └── logger.go       # 日誌處理
├── test/               # 測試工具
├── log/                # 日誌檔案
├── setting.json        # 設定檔
└── viewlog.bat         # 日誌查看器
```

## 🧪 測試

```powershell
# 完整測試 (包含編譯、啟動、測試、驗證)
.\complete_test.ps1

# 日誌功能驗證
.\verify_logging.ps1

# Redis Mock 服務 (測試用)
.\redis_mock.ps1
```

## 📝 設定檔範例

```json
{
  "tcp_port": "30001",
  "http_port": "8080",
  "redis_addr": "localhost:6379",
  "datamode": "test",
  "m
```
