# lucas152112/fastIM

**Description**: 無延遲的私人IM

**Primary Language**: JavaScript

**Created**: 2026-02-04T08:56:01Z
**Last Updated**: 2026-02-21T12:26:55Z
**Stars**: 0 | **Forks**: 0
**Archived**: False

## Language Statistics

| Language | Bytes | Percentage |
|----------|-------|------------|
| JavaScript | 135,645 | 23.1% |
| Shell | 111,459 | 19.0% |
| Go | 88,652 | 15.1% |
| CSS | 83,019 | 14.1% |
| Dart | 77,664 | 13.2% |
| HTML | 64,817 | 11.0% |
| CMake | 9,295 | 1.6% |
| C++ | 5,808 | 1.0% |
| Ruby | 2,757 | 0.5% |
| Dockerfile | 2,591 | 0.4% |
| Kotlin | 2,139 | 0.4% |
| Swift | 1,948 | 0.3% |
| Java | 804 | 0.1% |
| C | 754 | 0.1% |
| Objective-C | 38 | 0.0% |

## README 完整性檢查

✅ **README 完整** - 包含所有必要部分

*檢查時間: 2026-03-19 14:26:57*

## Repository Information

- **GitHub URL**: https://github.com/lucas152112/fastIM
- **SSH URL**: git@github.com:lucas152112/fastIM.git
- **Clone URL**: https://github.com/lucas152112/fastIM.git

## System Architecture

### Overview

This project uses a modular architecture with the following technology stack:

| Language | Bytes | Percentage |
|----------|-------|------------|
| JavaScript | 135,645 | 23.1% |
| Shell | 111,459 | 19.0% |
| Go | 88,652 | 15.1% |
| CSS | 83,019 | 14.1% |
| Dart | 77,664 | 13.2% |
| HTML | 64,817 | 11.0% |
| CMake | 9,295 | 1.6% |
| C++ | 5,808 | 1.0% |
| Ruby | 2,757 | 0.5% |
| Dockerfile | 2,591 | 0.4% |
| Kotlin | 2,139 | 0.4% |
| Swift | 1,948 | 0.3% |
| Java | 804 | 0.1% |
| C | 754 | 0.1% |
| Objective-C | 38 | 0.0% |

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
# FastIM - 高性能跨平台實時聊天系統

## 🚀 專案概述

FastIM 是專為低延遲、高效率工作協作設計的實時聊天系統，支援多平台客戶端。

### ⚡ 核心特性
- **超低延遲**: WebSocket 實時通訊，<50ms 響應時間
- **跨平台**: iOS, Android, Linux, Windows, macOS, Web 全平台支援
- **高性能**: ARM64 原生優化，K8s 容器化部署
- **簡潔**: 專注實時性，無冗餘功能

## 📋 技術架構

### 後端技術棧
- **語言**: Go (高性能 WebSocket)
- **資料庫**: Redis (即時快取) + MongoDB (持久化)
- **部署**: Kubernetes + ARM64 優化
- **協議**: WebSocket + JSON

### 前端技術棧
- **跨平台**: Flutter (iOS/Android/Linux/Windows/macOS)
- **Web版本**: React + TypeScript
- **實時通訊**: WebSocket 客戶端
- **狀態管理**: Provider/Riverpod

## 📂 專案結構（已更新，包含今日新增管理後台）

```
fastIM/
├── backend/                      # Go 後端服務與管理 API
│   ├── main.go                   # 管理 API skeleton (新增)
│   ├── go.mod                    # module
│   ├── cmd/                      # 應用程式入口 (服務專屬)
│   ├── internal/                 # 內部業務邏輯
│   ├── pkg/                      # 公共套件
│   └── api/                      # API 定義與 Swagger 註解
├── front/                        # 公開前端說明 / 前端 API (/front/*)
│   └── README.md                 # i18n 與部署說明 (新增)
├── manage/                       # 管理後台 (Admin UI) - /manage 路徑
│   └── README.md                 # Nuxt.js + Ant Design 實作說明 (新增)
├── frontend/                     # 前端專案 (Flutter / Web)
│   ├── flutter/
│   └── web/
├── services/                     # 各微服務目錄（每個服務獨立）
│   └── example-service/          # 範本服務 (新增)
│       ├── main.go
│       ├── Dockerfile
│       └── main_test.go
├── utility/                      # 共用工具 (JWT, tokens, auth helpers) (新增)
│   └── jwt.go
├── ci/                           # CI/CD 範例與 workflow (新增)
│   └── github-actions-hotupdate.yml
├── k8s/                          # Kubernetes 部署配置
├── docs/                         # 專案文件
└── scripts/                      # 建構和部署腳本
```

> 說明：
- 管理後台（/manage）與前端使用不同 API namespace：/manage/* (admin) 與 /front/* (public)。
- 共用功能（JWT、token 等）放在 /utility，供各服務直接使用，避免重寫。
- 每個微服務皆放在 services/<service-name>，包含 Dockerfile、unit tests 與 Swagger 註解。

## 🚀 快速開始

### 後端部署
```bash
# K8s 部署
kubectl appl
```
