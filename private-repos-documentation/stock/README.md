# lucas152112/stock

**Description**: 進銷存系統

**Primary Language**: Not specified

**Created**: 2026-02-12T01:17:19Z
**Last Updated**: 2026-02-12T04:10:39Z
**Stars**: 0 | **Forks**: 0
**Archived**: False

## Repository Information

- **GitHub URL**: https://github.com/lucas152112/stock
- **SSH URL**: git@github.com:lucas152112/stock.git
- **Clone URL**: https://github.com/lucas152112/stock.git

## System Architecture

### Overview

This project follows a standard layered architecture pattern.

### Components

- **Presentation Layer** - User interface components
- **Business Logic Layer** - Core application logic
- **Data Access Layer** - Database interaction
- **Infrastructure Layer** - External services and utilities

### Deployment

Deployed as containerized services with automated CI/CD pipelines.

### Dependencies

Dependencies managed through standard package managers for the primary language.
## README Content

```
# 📦 進銷存管理系統 - SaaS多租戶微服務版本

**企業級SaaS進銷存管理解決方案**  
基於 Rust微服務 + Nuxt.js 構建的高效能跨平台多租戶進銷存系統

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/rust-1.70+-orange.svg)](https://www.rust-lang.org)
[![Nuxt](https://img.shields.io/badge/nuxt-3.x-00C58E.svg)](https://nuxt.com)
[![MySQL](https://img.shields.io/badge/mysql-8.0-blue.svg)](https://www.mysql.com)
[![SaaS](https://img.shields.io/badge/SaaS-Multi--Tenant-brightgreen.svg)](https://en.wikipedia.org/wiki/Software_as_a_service)
[![Microservices](https://img.shields.io/badge/Architecture-Microservices-blue.svg)](https://microservices.io/)

---

## 🎯 系統概述

### SaaS多租戶微服務架構特色
- 🏢 **多公司支援** - 每個客戶公司獨立資料庫和權限
- 🌐 **專用URL** - 每個公司擁有專屬的網址 (如：company-a.yourdomain.com)
- 🔒 **資料隔離** - 完全獨立的租戶資料，確保安全性
- ⚙️ **微服務架構** - 每個API微服務獨立部署，彈性擴展
- 🚀 **自動感知** - 系統自動感知微服務變化，自動重新編譯部署
- 🎛️ **微服務管理** - 專用管理平台控制微服務上下線
- 💰 **收費管理** - 月付、年付、終身方案靈活計費
- 💾 **自動備份** - 每個租戶獨立備份，一鍵恢復

### 微服務部署特性
- **獨立目錄** - 每個微服務獨立一個目錄
- **隨時上下線** - 微服務可隨時啟動/停止，不影響其他服務
- **自動感知** - 系統自動感知微服務變化
- **自動編譯部署** - 代碼更新後自動重新編譯部署
- **手動控制** - 微服務狀態不能自動變更，需手動管理
- **預設不啟動** - 新部署的微服務預設為停止狀態
- **狀態保持** - 更新時保持原有的啟動/停止狀態

---

## 🏗️ 微服務架構圖

```
┌─────────────────────────────────────────────────────────┐
│                🎛️ 微服務管理平台                        │
│         (控制所有微服務的上下線狀態)                      │
└─────────────────────────────────────────────────────────┘
                             │
┌─────────────────────────────────────────────────────────┐
│              🌐 API Gateway + 負載均衡                   │
│               (自動發現和路由到活躍微服務)                │
└─────────────────────────────────────────────────────────┘
                             │
┌─────────────────────────────────────────────────────────┐
│                  🎯 租戶路由層                           │
│            (*.yourdomain.com → 對應租戶)                │
└─────────────────────────────────────────────────────────┘
                             │
┌─────────────────┬
```
