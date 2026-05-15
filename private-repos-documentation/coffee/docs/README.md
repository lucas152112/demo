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

## 📝 **開發進度追蹤**

### **📋 了解系統目前做到哪裡**
- **工作日誌文件**: [`/doc/diary.md`](doc/diary.md) - 記錄所有開發活動、當前進度、已完成工作和下一步計劃
- **進度報告**: `progress_reports/` 目錄 - 自動化進度追蹤報告
- **線上演示**: [coffee.justit.cc](https://coffee.justit.cc) - 實際運作環境
- **當前進度**: 13% (9/67 項功能已完成)

### **🔍 如何追蹤開發狀態**
1. **閱讀工作日誌** - 查看 `/doc/diary.md` 了解最新開發動態
2. **檢查進度報告** - 查看 `progress_reports/` 中的自動化報告
3. **查看開發清單** - 參考 `COFFEE_DEVELOPMENT_CHECKLIST.md` 了解詳細功能列表
4. **體驗線上系統** - 登入演示帳號查看已實現功能

### **🎯 下一步開發重點**
- **短期目標**: 完成 Orders 服務最後 20%，開始 Inventory 服務開發
- **里程碑**: 突破 50% 完成率 (從當前 13% 進展到 50%)
- **技術重點**: 複製 JustProject 企業級模板到 Coffee 專案

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
- **全域 AI 按鍵** - 任何頁面可啟動 AI 對話
- **語音對話** - 支援文字和語音雙模式
- **系統查詢** - AI 可查詢系統內所有業務資料
- **營運建議** - 基於數據的智能營運建議

### **📱 跨平台支援**
- **Web 版** - 響應式網頁應用
- **Android App** - 原生 Android 應用
- **iOS App** - 原生 iOS 應用  
- **桌面版** - Windows/macOS/Linux 桌面應用
- **PWA 離線** - 支援離線操作

---

## 🛠️ **技術棧**

### **前端技術**
- **框架**: Nuxt.js + Vue 3
- **UI 組件**: Ant Design Vue
- **狀態管理**: Pinia
- **打包工具**: Vite
- **移動端**: Capacitor (Hybrid App)
- **桌面端**: Electron

### **後端技術**  
- **核心**: Rust + Axum Framework
- **資料庫**: MySQL 8.0 + Redis + MongoDB
- **認證**: JWT + OAuth 2.0
- **API**: RESTful + GraphQL
- **即時通訊**: WebSocket + Server-Sent Events
- **檔案儲存**: MinIO (S3 相容)

### **部署運維**
- **容器化**: Docker + Docker Compose
- **編排**: Kubernetes (K8s)
- **CI/CD**: GitLab CI/CD Pipeline
- **監控**: Prometheus + Grafana
- **日誌**: ELK Stack (Elasticsearch + Logstash + Kibana)
- **反向代理**: Nginx + Let's Encrypt SSL

### **AI 與整合**
- **AI 模型**: GPT-4 + Claude + Gemini 多模型支援
- **語音**: 語音轉文字 + 文字轉語音
- **支付**: 綠界科技 + LINE Pay + Apple Pay
- **發票**: 財政部電子發票 API
- **物流**: 7-11 + 全家 + 宅配通

---

## 🚀 **快速開始**

### **📋 系統需求**
- **Kubernetes 1.28+** (生產環境)
- **Docker 20.10+** (開發環境)
- **Node.js 18+** (前端開發)
- **Rust 1.70+** (後端開發)

### **🔧 本地開發環境**

```bash
# 1. 克隆專案
git clone https://github.com/justit-cc/coffee-management-system.git
cd coffee-management-system

# 2. 啟動開發環境
docker-compose up -d

# 3. 前端開發
cd frontend
npm install
npm run dev

# 4. 後端開發
cd backend
cargo run

# 5. 訪問系統
# 前端: http://localhost:3000
# 後端 API: http://localhost:8000
# 管理後台: http://localhost:3001
```

### **☸️ Kubernetes 部署**

```bash
# 1. 創建命名空間
kubectl create namespace coffee-system

# 2. 部署資料庫服務
kubectl apply -f k8s/database/

# 3. 部署後端微服務
kubectl apply -f k8s/backend/

# 4. 部署前端應用
kubectl apply -f k8s/frontend/

# 5. 配置 Nginx 反向代理
kubectl apply -f k8s/nginx/
```

---

## 📚 **文檔與資源**

### **📖 開發文檔**
- [系統架構設計](docs/architecture.md)
- [API 接口文檔](docs/api.md)
- [資料庫設計](docs/database.md)
- [部署指南](docs/deployment.md)
- [開發規範](docs/development.md)

### **🎯 商業文檔**
- [產品需求](docs/requirements.md)
- [功能規格](docs/specifications.md)
- [測試計劃](docs/testing.md)
- [用戶手冊](docs/user-guide.md)

### **🔗 相關鏈接**
- **線上演示**: [coffee.justit.cc](https://coffee.justit.cc)
- **API 文檔**: [api.coffee.justit.cc](https://api.coffee.justit.cc)
- **監控面板**: [monitoring.coffee.justit.cc](https://monitoring.coffee.justit.cc)
- **問題回報**: [GitHub Issues](https://github.com/justit-cc/coffee-management-system/issues)

---

## 📈 **開發進度**

### **✅ 已完成功能**
- [x] 🎯 **系統入口** - 註冊/登入介面
- [x] 🔐 **SSL 安全** - Let's Encrypt 證書自動化
- [x] ☸️ **K8s 部署** - 基礎容器化部署架構
- [x] 🌐 **Nginx 代理** - 反向代理和負載均衡
- [x] 🎨 **響應式設計** - 跨設備兼容的前端介面

### **🚧 開發中功能**
- [ ] 👥 **租戶管理服務** - 多租戶註冊和管理
- [ ] 🛒 **POS 點餐系統** - 完整的點餐結帳流程
- [ ] 📦 **庫存管理系統** - 智能庫存監控和補貨
- [ ] 💳 **支付整合** - 多種支付方式整合
- [ ] 🤖 **AI 智能助手** - 全功能 AI 客服系統

### **📋 規劃中功能**
- [ ] 📊 **數據分析平台** - 商業智能和預測分析
- [ ] 📱 **移動端 App** - Android/iOS 原生應用
- [ ] 🔔 **通知系統** - 多管道通知推送
- [ ] 🔄 **自動化運維** - CI/CD 和監控告警
- [ ] 🌍 **國際化** - 多語言和多地區支援

---

## 🤝 **參與貢獻**

### **🔧 開發貢獻**
1. **Fork 專案** 到您的 GitHub 帳號
2. **創建功能分支** (`git checkout -b feature/AmazingFeature`)
3. **提交更改** (`git commit -m 'Add some AmazingFeature'`)
4. **推送分支** (`git push origin feature/AmazingFeature`)
5. **開啟 Pull Request** 描述您的更改

### **📋 開發規範**
- **代碼風格**: 遵循 [開發規範](docs/development.md)
- **提交訊息**: 使用 [約定式提交](https://www.conventionalcommits.org/zh-hant/)
- **測試覆蓋**: 新功能需要包含單元測試
- **文檔更新**: 重大更改需要更新相關文檔

### **🐛 問題回報**
如果您發現 bug 或有功能建議，請：
1. 查看 [GitHub Issues](https://github.com/justit-cc/coffee-management-system/issues) 確認問題尚未被回報
2. 使用適當的問題模板創建新 Issue
3. 提供詳細的重現步驟和環境資訊

---

## 📄 **授權協議**

本專案採用 [MIT License](LICENSE) 授權協議。

```
MIT License

Copyright (c) 2026 Coffee Management System

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

## 📞 **聯絡我們**

### **🏢 商業合作**
- **客服專線**: 0800-COFFEE (0800-263333)
- **商務郵箱**: business@coffee.justit.cc
- **技術支援**: support@coffee.justit.cc

### **💬 社群討論**
- **GitHub Issues**: [專案問題討論](https://github.com/justit-cc/coffee-management-system/issues)
- **Discord**: [加入開發者社群](https://discord.gg/coffee-dev)
- **Telegram**: [Coffee SaaS 用戶群](https://t.me/coffee_saas)

### **📱 關注我們**
- **官方網站**: [coffee.justit.cc](https://coffee.justit.cc)
- **技術部落格**: [blog.coffee.justit.cc](https://blog.coffee.justit.cc)
- **Twitter**: [@CoffeeSaaS](https://twitter.com/CoffeeSaaS)

---

## 🎉 **致謝**

感謝所有為 Coffee Management System 專案做出貢獻的開發者、設計師、測試人員和用戶！

### **🏆 核心貢獻者**
- **專案發起人**: [@justit-cc](https://github.com/justit-cc)
- **首席架構師**: [@senior-architect](https://github.com/senior-architect)
- **前端團隊**: [@frontend-team](https://github.com/orgs/justit-cc/teams/frontend)
- **後端團隊**: [@backend-team](https://github.com/orgs/justit-cc/teams/backend)

### **🤝 特別感謝**
- **Kubernetes 社群** - 提供優秀的容器編排平台
- **Rust 社群** - 提供高效能的後端開發語言
- **Vue.js 社群** - 提供優雅的前端開發框架
- **所有 Beta 測試用戶** - 提供寶貴的回饋和建議

---

<div align="center">

**☕ 讓每一杯咖啡都有故事，讓每一家咖啡店都有未來 ☕**

**由 ❤️ 和 ☕ 驅動開發**

[⬆ 回到頂部](#-coffee-management-system)

</div>