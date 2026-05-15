# 📚 咖啡館連鎖管理系統 - 快速參考指南

> 為開發團隊準備的快速查閱文件

---

## 🎯 專案資訊

**專案名稱**: 咖啡館連鎖管理系統  
**專案代號**: Coffee CRM/ERP  
**技術架構**: Go 微服務 + Vue 3/Nuxt 3  
**目標門店數**: 10-50家  
**開發週期**: 6個月 (MVP 3個月)

---

## 📁 文件導航

### 必讀文件 (開發前)
1. **[專案就緒報告](./PROJECT-READINESS-REPORT.md)** ⭐⭐⭐  
   → 閱讀時間: 10分鐘  
   → 了解專案當前狀態與下一步行動

2. **[文件總索引](./00-INDEX.md)** ⭐⭐⭐  
   → 閱讀時間: 5分鐘  
   → 快速找到所需文件

3. **[需求分析](./01-SA-SystemAnalysis/01-Requirements.md)** ⭐⭐⭐  
   → 閱讀時間: 30分鐘  
   → 了解系統功能與非功能需求

4. **[系統架構設計](./02-SD-SystemDesign/01-ArchitectureDesign.md)** ⭐⭐⭐  
   → 閱讀時間: 30分鐘  
   → 了解微服務架構與技術方案

5. **[資料庫ERD](./03-Database/01-ERD.md)** ⭐⭐⭐  
   → 閱讀時間: 40分鐘  
   → 了解資料表結構與關聯

### 開發時參考
- **[使用案例規格](./01-SA-SystemAnalysis/03-UseCaseSpecifications.md)** - 了解具體功能需求
- **[業務流程](./01-SA-SystemAnalysis/05-BusinessProcesses.md)** - 了解業務邏輯
- **[領域模型](./01-SA-SystemAnalysis/04-DomainModel.md)** - 了解領域實體與關係
- **API規格** - 待建立 ⚠️

### 專案管理參考
- **[系統範圍與限制](./01-SA-SystemAnalysis/06-ScopeAndConstraints.md)** - 了解做什麼/不做什麼
- **[文件完成摘要](./DOCUMENTATION-SUMMARY.md)** - 了解文件完成進度

---

## 🏗️ 系統架構一覽

### 微服務列表

| 服務名稱 | Port | 職責 | 資料庫Schema |
|---------|------|------|-------------|
| API Gateway | 8080 | 路由、認證 | - |
| Auth Service | 8081 | 使用者認證 | auth |
| Customers Service | 8082 | 客戶管理 | customers |
| Orders Service | 8083 | 訂單管理 | orders |
| Inventory Service | 8084 | 商品庫存採購 | inventory |
| Loyalty Service | 8085 | 會員點數 | customers + Redis |
| Notifications Service | 8086 | 通知郵件 | notifications |
| Reports Service | 8087 | 報表分析 | all (read-only) |
| Nuxt Frontend | 3000 | 前端介面 | - |

### 基礎設施

| 元件 | Port | 用途 |
|-----|------|------|
| MySQL | 3306 | 主資料庫 |
| Redis | 6379 | 快取與會話 |
| NATS | 4222 | 訊息佇列 |
| MailHog | 1025/8025 | 郵件測試 |

---

## 💾 資料庫速查

### 資料庫結構
MySQL 8.0+ 使用單一資料庫 `coffee_crm`,資料表使用前綴區分模組:
```
users_*          → Auth Service
customers_*      → Customers + Loyalty Services
orders_*         → Orders Service
inventory_*      → Inventory Service
notifications_*  → Notifications Service
```

### 核心資料表 (32個)

**Auth** (1):
- `users` - 使用者帳號

**Customers** (4):
- `customers` - 客戶資料
- `customer_tags` - 客戶標籤
- `loyalty_accounts` - 點數帳戶
- `point_transactions` - 點數異動

**Inventory** (11):
- `stores` - 門店
- `product_categories` - 商品分類
- `products` - 商品
- `product_variants` - 商品規格
- `stock` - 庫存
- `stock_movements` - 庫存異動
- `suppliers` - 供應商
- `purchase_orders` - 採購單
- `purchase_order_items` - 採購項目

**Orders** (4):
- `orders` - 訂單
- `order_items` - 訂單項目
- `payments` - 付款記錄
- `invoices` - 發票

**Notifications** (2):
- `notifications` - 通知記錄
- `notification_templates` - 通知範本

---

## 🔐 認證與授權

### JWT Token
**Header**: `Authorization: Bearer {token}`  
**有效期**: 24小時  
**Payload**:
```json
{
  "sub": "user_id",
  "email": "user@example.com",
  "role": "store_manager",
  "store_id": "store_uuid"
}
```

### 角色定義
- `admin` - 系統管理員 (所有權限)
- `headquarters_manager` - 總部管理 (ERP全權限)
- `store_manager` - 店長 (本店全權限)
- `staff` - 店員 (基本POS權限)

---

## 📊 核心業務流程

### 1. POS點餐結帳
```
選擇商品 → 加入購物車 → 綁定會員(可選) → 選擇付款方式 
→ 完成付款 → 扣減庫存 → 累積點數 → 產生發票(可選)
```

### 2. 採購入庫
```
建立採購單 → 提交審核 → 主管審核 → 供應商交貨 
→ 驗收入庫 → 更新庫存 → 產生庫存異動記錄
```

### 3. 會員點數
```
訂單完成 → 計算點數(消費$1=1點) → 累積至帳戶 
→ 檢查會員等級 → 自動升級(如符合條件)
```

---

## 🎨 前端模組劃分

### CRM (門店管理)
- **POS點餐系統** - 商品選擇、訂單結帳
- **客戶管理** - 客戶資料、消費記錄
- **會員管理** - 點數查詢、點數兌換
- **門店報表** - 日報、商品排行

### ERP (總部管理)
- **商品管理** - 商品主檔、分類、規格
- **庫存管理** - 庫存查詢、盤點、調撥
- **採購管理** - 採購單、供應商、入庫
- **報表分析** - 營收、銷售、客戶分析

### 系統管理
- **使用者管理** - 帳號、角色、權限
- **門店管理** - 門店資料、設定
- **系統設定** - 參數、通知範本

---

## 🚀 開發環境

### 必要工具
- Go 1.21+
- Node.js 18+ / npm
- Docker 24+ / Docker Compose
- MySQL 8.0+ (或使用Docker)
- Git

### 啟動指令
```bash
# 啟動所有服務
docker compose up --build

# 僅啟動基礎設施
docker compose up mysql redis nats mailhog

# 前端開發模式
cd webapp && npm run dev

# 後端服務開發 (範例: Auth Service)
cd services/auth && go run main.go
```

### 環境變數
參考各服務的 `docker-compose.yml` 中的 `environment` 設定

---

## 📋 開發檢查清單

### 後端服務開發
- [ ] 閱讀對應的使用案例規格
- [ ] 查看資料庫ERD了解資料表結構
- [ ] 遵循 API 規格 (待建立)
- [ ] 撰寫單元測試
- [ ] 遵循 Go 編碼規範 (待建立)
- [ ] 提供結構化日誌 (JSON format)
- [ ] 實作錯誤處理
- [ ] 更新 README (如有變更)

### 前端頁面開發
- [ ] 參考 UI 原型設計 (待建立)
- [ ] 遵循 Vue 3 / TypeScript 編碼規範 (待建立)
- [ ] 實作響應式設計 (RWD)
- [ ] 處理 API 錯誤與 Loading 狀態
- [ ] 撰寫元件測試
- [ ] 確保無障礙性 (Accessibility)

### 整合測試
- [ ] 測試完整業務流程
- [ ] 測試異常處理
- [ ] 測試權限控制
- [ ] 效能測試 (API 響應時間)

---

## ⚠️ 注意事項

### 金額處理
- ✅ 一律使用 **INTEGER (分)** 儲存,避免浮點數誤差
- ✅ 前端顯示時除以100轉為元
- ✅ 範例: $350 → 儲存為 35000 (分)

### 時間處理
- ✅ 資料庫使用 **DATETIME** 或 **TIMESTAMP**
- ✅ 統一使用 UTC+8 (台北時間)
- ✅ API 返回 ISO 8601 格式

### ID 生成
- ✅ 統一使用 **UUID** 作為主鍵
- ✅ 使用 MySQL 的 `UUID()` 或應用層生成

### 錯誤處理
- ✅ 統一錯誤格式 (JSON)
- ✅ 包含 `error_code`, `message`, `details`
- ✅ 記錄錯誤日誌

---

## 🔗 有用連結

### 技術文件
- [Go Gin Framework](https://gin-gonic.com/docs/)
- [Vue 3 官方文件](https://vuejs.org/)
- [Nuxt 3 官方文件](https://nuxt.com/)
- [MySQL 8.0 文件](https://dev.mysql.com/doc/refman/8.0/en/)
- [Docker Compose 文件](https://docs.docker.com/compose/)

### 專案文件
- [需求分析](./01-SA-SystemAnalysis/01-Requirements.md)
- [系統架構](./02-SD-SystemDesign/01-ArchitectureDesign.md)
- [資料庫ERD](./03-Database/01-ERD.md)
- [專案就緒報告](./PROJECT-READINESS-REPORT.md)

---

## ❓ 常見問題

### Q1: 如何找到某個功能的需求定義?
**A**: 查看 `01-SA-SystemAnalysis/01-Requirements.md` 的功能需求章節

### Q2: 如何知道某個API要怎麼設計?
**A**: 參考使用案例規格 + 架構設計,詳細 API 規格文件待建立

### Q3: 前端頁面要怎麼設計?
**A**: UI 原型文件待建立,可先參考業務流程了解操作步驟

### Q4: 資料表之間的關係是什麼?
**A**: 查看 `03-Database/01-ERD.md` 的 ERD 圖

### Q5: 某個欄位的資料型態是什麼?
**A**: 查看 `03-Database/01-ERD.md` 的資料表詳細設計

### Q6: 要用什麼技術實作某個功能?
**A**: 查看 `02-SD-SystemDesign/01-ArchitectureDesign.md`

---

## 📞 聯絡資訊

**技術問題**: 查閱對應技術文件  
**需求問題**: 查閱系統分析文件  
**專案問題**: 查閱專案管理文件

---

**最後更新**: 2025年11月16日  
**文件版本**: v1.0
