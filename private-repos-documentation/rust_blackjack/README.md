# lucas152112/rust_blackjack

**Description**: rust版本多平台Blackjack

**Primary Language**: Rust

**Created**: 2025-12-31T08:01:52Z
**Last Updated**: 2025-12-31T09:36:35Z
**Stars**: 0 | **Forks**: 0
**Archived**: False

## Language Statistics

| Language | Bytes | Percentage |
|----------|-------|------------|
| Rust | 115,295 | 94.3% |
| TypeScript | 5,353 | 4.4% |
| Fluent | 1,224 | 1.0% |
| HTML | 370 | 0.3% |

## README 完整性檢查

✅ **README 完整** - 包含所有必要部分

*檢查時間: 2026-03-19 14:26:57*

## Repository Information

- **GitHub URL**: https://github.com/lucas152112/rust_blackjack
- **SSH URL**: git@github.com:lucas152112/rust_blackjack.git
- **Clone URL**: https://github.com/lucas152112/rust_blackjack.git

## System Architecture

### Overview

This project uses a modular architecture with the following technology stack:

| Language | Bytes | Percentage |
|----------|-------|------------|
| Rust | 115,295 | 94.3% |
| TypeScript | 5,353 | 4.4% |
| Fluent | 1,224 | 1.0% |
| HTML | 370 | 0.3% |

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
# Big Blackjack (大點數 21 點)

這是一個使用 Rust 語言與 Bevy 遊戲引擎開發的 MMORPG Blackjack (21點) 遊戲。專案採用前後端分離架構，具備高性能、可擴展的資料庫系統。

## 核心技術棧

- **前端 (Frontend)**: Bevy Engine 0.15 (跨平台遊戲引擎)
- **後端 (Backend)**: Axum (高性能 Web 框架), WebSocket
- **獨立工具類 (Utils)**: 每個專案具備獨立的 `utils` 目錄，確保核心邏輯與通訊協議的完全解耦
- **資料庫 (Database)**: 
  - **MySQL**: 玩家核心資料、錢包點數 (持久化)
  - **Redis**: Session 管理、登入狀態 (快取/叢集支援)
  - **MongoDB**: 玩家個性化配置 (JSON 結構)
- **通訊協議**: WebSocket + Serde (JSON 序列化)
- **容器化**: Docker & Docker Compose

## 專案結構

```text
.
├── backend/            # 後端伺服器原始碼
│   ├── config/         # 環境設定檔 (local, dev, test, pp, prod)
│   ├── src/            # 資料庫管理、網絡處理
│   └── db/             # 資料庫初始化腳本
├── frontend/           # Bevy 客戶端原始碼
│   ├── assets/         # 圖片、字體資源
│   ├── config/         # 前端環境設定
│   └── src/            # UI 系統、遊戲狀態管理、i18n
├── manage/             # 管理後台 (React + Rocket)
│   ├── front/          # 管理後台前端 (Vite + AntD Pro)
│   └── backend/        # 管理後台後端 (Rocket API)
├── docker-compose.yml  # 資料庫環境配置文件
└── .env                # 環境模式切換 (.local, .dev, .test, .pp, .prod)
```

## 開發分支策略

本專案採用 Git Flow 簡化版策略：
- **main**: 穩定版本分支，僅用於存放通過測試且準備發布的代碼。
- **develop**: 主要開發分支，所有新功能的開發與整合皆在此分支進行。

開發新功能時，請確保您處於 `develop` 分支：
```bash
git checkout develop
```

## 快速開始

### 1. 環境準備

確保您的系統已安裝：
- Rust (最新穩定版)
- Docker & Docker Compose
- Node.js (v18+) & pnpm (v8+)

### 2. 啟動資料庫

在根目錄執行：
```bash
docker-compose up -d
```
這將啟動 MySQL (8.4), Redis (穩定版) 與 MongoDB。

### 3. 設定環境

修改根目錄的 `.env` 檔案：
```env
RUN_MODE=local
```
可選模式：`local`, `dev`, `test`, `pp`, `prod`。

### 4. 啟動後端

```bash
cargo run -p big-blackjack-server
```

### 5. 啟動前端

```bash
cargo run -p big-blackjack-client
```

## 功能特色

- **多環境支援**: 自動根據 `RUN_MODE` 切換資料庫連線（支援讀寫分離與叢集模式）。
- **i18n 多語系**: 支援英文 (en-US) 與正體中文 (zh-TW)，遊戲中可動態切換（快速鍵 E/Z）。
- **模組化機器人**: 可在設定檔中配置每桌固定或隨機數量的 AI 玩家。
- **三層存儲設計**: 登錄資訊先由 Redis 讀取，缺失則同步 MySQL，設定檔存放於 MongoDB。
- **自動化日誌**: 根據日期滾動建立日誌，並區分系統日誌與個別使用者操作日誌。

## 遊戲開發日誌
詳細開發過程請參考 `develop_dairy.md`。
任務進度追蹤請參考 `Agent.md`
```
