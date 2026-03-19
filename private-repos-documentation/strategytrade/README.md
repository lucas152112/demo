# lucas152112/strategytrade

**Description**: 商品策略即時交易

**Primary Language**: Go

**Created**: 2025-11-08T17:47:54Z
**Last Updated**: 2025-11-08T18:58:31Z
**Stars**: 0 | **Forks**: 0
**Archived**: False

## Language Statistics

| Language | Bytes | Percentage |
|----------|-------|------------|
| Go | 33,194 | 100.0% |

## README 完整性檢查

⚠️ **README 不完整** - 缺少以下部分：
  - 簡介
  - 系統架構
  - 程式語言

**建議補充內容：**

### 📖 簡介
商品策略即時交易

### 🏗️ 系統架構
*在此描述系統架構...*

### 💻 程式語言
- 主要語言: Go


*檢查時間: 2026-03-19 14:26:57*

## Repository Information

- **GitHub URL**: https://github.com/lucas152112/strategytrade
- **SSH URL**: git@github.com:lucas152112/strategytrade.git
- **Clone URL**: https://github.com/lucas152112/strategytrade.git

## System Architecture

### Overview

This project uses a modular architecture with the following technology stack:

| Language | Bytes | Percentage |
|----------|-------|------------|
| Go | 33,194 | 100.0% |

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
# strategytrade
商品策略即時交易引擎 / Real-time commodity strategy trading engine

此專案提供以設定驅動的 Go 策略交易框架，整合即時行情、策略模組與訊號派送。  
This project provides a config-driven Go strategy trading framework that unifies real-time market data, pluggable strategies, and signal dispatching.

## 設定 / Configuration
- 共用設定檔：`config/common.config.json`；環境覆寫檔：`config/dev.config.json`, `config/prod.config.json`。 / Shared config lives in `config/common.config.json`, while environment overlays reside in `config/<env>.config.json`.
- 執行範例 / Run example:  
  `go run ./cmd/strategytrade -config config/common.config.json -env dev -log-dir log`

## 日誌 / Logging
- 日誌由 `internal/common/logger` 管理，並依日期輸出至 `log/` 目錄（`YYYY-MM-DD.log`）。 / Logging is handled by `internal/common/logger`, writing date-based files under `log/`.
- 可用 `-log-dir` 指定目錄、`-log-stdout` 同步輸出到 stdout 以利本地除錯。 / Use `-log-dir` to choose the directory and `-log-stdout` to mirror logs to stdout for local debugging.

```
