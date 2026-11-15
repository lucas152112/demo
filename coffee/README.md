# lucas152112/coffee

**Description**: 連鎖咖啡館管理系統

**Primary Language**: Rust

**Created**: 2025-11-15T17:15:20Z
**Last Updated**: 2026-02-15T17:29:37Z
**Stars**: 0 | **Forks**: 0
**Archived**: False

## Language Statistics

| Language | Bytes | Percentage |
|----------|-------|------------|
| Rust | 436,054 | 76.5% |
| Shell | 57,211 | 10.0% |
| Vue | 49,152 | 8.6% |
| Go | 21,604 | 3.8% |
| Dockerfile | 5,826 | 1.0% |
| TypeScript | 186 | 0.0% |

## README 完整性檢查

✅ **README 完整** - 包含所有必要部分

*檢查時間: 2026-03-19 14:26:57*

## Repository Information

- **GitHub URL**: https://github.com/lucas152112/coffee
- **SSH URL**: git@github.com:lucas152112/coffee.git
- **Clone URL**: https://github.com/lucas152112/coffee.git

## System Architecture

### Overview

This project uses a modular architecture with the following technology stack:

| Language | Bytes | Percentage |
|----------|-------|------------|
| Rust | 436,054 | 76.5% |
| Shell | 57,211 | 10.0% |
| Vue | 49,152 | 8.6% |
| Go | 21,604 | 3.8% |
| Dockerfile | 5,826 | 1.0% |
| TypeScript | 186 | 0.0% |

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
# ☕ Coffee Management System

> 企業級多租戶咖啡店 SaaS 管理平台

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Kubernetes](https://img.shields.io/badge/platform-Kubernetes-blue.svg)](https://kubernetes.io/)
[![Rust](https://img.shields.io/badge/backend-Rust-orange.svg)](https://www.rust-lang.org/)
[![Vue.js](https://img.shields.io/badge/frontend-Vue.js-green.svg)](https://vuejs.org/)

## 🚀 **系統簡介**

Coffee Management System 是一個現代化的多租戶咖啡店管理平台，採用微服務架構，為咖啡店提供完整的數位化營運解決方案。

### **🎯 核心價值**
- **💼 多租戶 SaaS** - 一套系統服務多家咖啡店
- **🛒 完整 POS** - 從點餐到結帳的完整流程  
- **📊 智能分析** - 實時銷售數據分析
- **🤖 AI 助手** - 智能客服和營運建議
- **📱 跨平台** - Web/Android/iOS 全平台支援

---

## 🌐 **線上體驗**

### **🔗 系統入口**
- **網址**: [coffee.justit.cc](https://coffee.justit.cc)
- **演示帳號**: demo@coffee.justit.cc
- **演示密碼**: demo123456

### **🎯 立即體驗**
1. 訪問 [coffee.justit.cc](https://coffee.justit.cc)
2. 使用演示帳號登入
3. 體驗完整系統功能
4. 或註冊專屬咖啡店帳號

---

## 🏗️ **系統架構**

### **☸️ Kubernetes 微服務架構**
```
├── 🌐 Gateway Service (Nginx 反向代理)
├── 👥 Tenant Service (租戶管理)
├── 📊 Monitoring Service (系統監控)
├── 🛒 Order Service (訂單管理)
├── 💳 Payment Service (支付處理)
├── 📦 Inventory Service (庫存管理)
├── 📈 Analytics Service (數據分析)
├── 🤖 AI Chat Service (AI 智能助手)
└── 🔔 Notification Service (通知服務)
```

### **🗄️ 多資料庫架構**
- **MySQL 8.0** - 核心業務關聯式資料
- **Redis** - 記憶體快取和會話管理
- **MongoDB** - 文檔型資料和操作日誌

### **🔒 安全與認證**
- **Let's Encrypt SSL** - 全站 HTTPS 加密
- **JWT 認證** - 無狀態身份驗證
- **多層權限** - 租戶/角色/功能權限控制
- **API 安全** - Rate Limiting 和 CORS 保護

---

## 📋 **功能特色**

### **🏪 多租戶管理**
- **獨立數據隔離** - 每家咖啡店數據完全隔離
- **自助註冊** - 咖啡店可自助註冊加入平台
- **彈性計費** - 依使用量靈活計費模式
- **品牌客製化** - 每個租戶可客製化介面

### **🛒 完整 POS 系統**
- **商品管理** - 飲品/餐點/套餐管理
- **點餐系統** - 桌邊點餐、外帶、外送
- **結帳系統** - 多種支付方式整合
- **發票系統** - 電子發票自動開立

### **📦 智能庫存管理**
- **即時庫存** - 實時庫存數量追蹤
- **自動補貨** - 低庫存自動提醒補貨
- **成本控制** - 原料成本精確計算
- **供應商管理** - 供應商資訊與採購記錄

### **📊 數據分析報表**
- **銷售報表** - 日/週/月銷售統計
- **商品分析** - 熱銷商品排行榜
- **客戶分析** - 客戶消費行為分析
- **營收預測** - AI 驅動的營收預測

### **🤖 AI 智能助手**
- **全域 AI 按鍵** - 任何頁面可啟
```
