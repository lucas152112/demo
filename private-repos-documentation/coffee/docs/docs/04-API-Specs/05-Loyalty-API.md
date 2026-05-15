# 會員忠誠度管理 API 規格 (Loyalty Management API)

## 1. API 概觀

### 服務名稱
- **服務名稱**: Loyalty Service
- **Port**: 3005
- **Base URL**: `/api/v1/loyalty`
- **業務職責**: 會員點數管理、等級制度、獎勵機制、優惠券管理

### 核心功能
- 會員點數累積與兌換
- 會員等級管理
- 優惠券發放與使用
- 生日禮品管理
- 推薦獎勵機制
- 忠誠度分析

---

## 2. 認證與授權

### 認證方式
- Bearer Token (JWT)
- 所有 API 需要驗證 `Authorization: Bearer <token>`

### 權限等級
- **LOYALTY_READ**: 會員忠誠度查詢權限
- **LOYALTY_WRITE**: 點數異動權限
- **LOYALTY_MANAGE**: 獎勵機制管理權限
- **LOYALTY_ADMIN**: 忠誠度系統管理權限

---

## 3. 資料模型

### 3.1 會員點數帳戶 (LoyaltyAccount)
```json
{
  "account_id": "string",            // 帳戶ID
  "customer_id": "string",           // 客戶ID
  "current_points": "integer",       // 目前點數
  "total_earned": "integer",         // 累積獲得點數
  "total_redeemed": "integer",       // 累積兌換點數
  "pending_points": "integer",       // 待入帳點數
  "expiring_soon": "integer",        // 即將到期點數 (30天內)
  "member_tier": "string",           // 會員等級
  "tier_progress": "number",         // 等級進度 (0-100%)
  "next_tier_points": "integer",     // 升級所需點數
  "last_activity": "datetime",       // 最後活動時間
  "status": "string",                // 帳戶狀態 (active/suspended/closed)
  "created_at": "datetime",          // 創建時間
  "updated_at": "datetime"           // 更新時間
}
```

### 3.2 點數異動記錄 (PointTransaction)
```json
{
  "transaction_id": "string",        // 交易ID
  "account_id": "string",           // 帳戶ID
  "customer_id": "string",          // 客戶ID
  "transaction_type": "string",     // 交易類型 (earn/redeem/expire/adjust)
  "points": "integer",              // 點數數量 (+/-)
  "balance_after": "integer",       // 交易後餘額
  "source": "string",               // 來源 (purchase/birthday/referral/manual)
  "source_reference": "string",    // 來源參照 (訂單號/活動編號等)
  "description": "string",          // 描述
  "store_id": "string",            // 門店ID
  "operator_id": "string",         // 操作員ID
  "expire_date": "date",           // 到期日 (僅獲得點數)
  "status": "string",              // 狀態 (completed/pending/cancelled)
  "created_at": "datetime",        // 創建時間
  "processed_at": "datetime"       // 處理時間
}
```

### 3.3 會員等級 (MemberTier)
```json
{
  "tier_id": "string",              // 等級ID
  "tier_name": "string",            // 等級名稱
  "tier_level": "integer",          // 等級序號
  "required_points": "integer",     // 所需累積點數
  "required_spend": "number",       // 所需累積消費
  "point_multiplier": "number",     // 點數倍數
  "birthday_bonus": "integer",      // 生日獎勵點數
  "tier_benefits": "array",         // 等級權益
  "tier_color": "string",           // 等級色彩
  "tier_icon": "string",           // 等級圖示
  "valid_period": "integer",        // 有效期間 (月)
  "is_active": "boolean",          // 是否啟用
  "created_at": "datetime",        // 創建時間
  "updated_at": "datetime"         // 更新時間
}
```

### 3.4 優惠券 (Coupon)
```json
{
  "coupon_id": "string",            // 優惠券ID
  "coupon_code": "string",          // 優惠券代碼
  "coupon_type": "string",          // 類型 (discount/free_item/points)
  "title": "string",               // 優惠券標題
  "description": "string",         // 描述
  "discount_type": "string",       // 折扣類型 (percentage/fixed_amount)
  "discount_value": "number",      // 折扣值
  "minimum_spend": "number",       // 最低消費
  "maximum_discount": "number",    // 最高折扣
  "usage_limit": "integer",        // 使用次數限制
  "usage_count": "integer",        // 已使用次數
  "per_customer_limit": "integer", // 每客戶使用限制
  "valid_from": "datetime",        // 生效時間
  "valid_until": "datetime",       // 到期時間
  "applicable_stores": "array",    // 適用門店
  "applicable_products": "array",  // 適用商品
  "status": "string",              // 狀態 (active/inactive/expired)
  "created_at": "datetime",        // 創建時間
  "updated_at": "datetime"         // 更新時間
}
```

### 3.5 優惠券發放記錄 (CouponIssue)
```json
{
  "issue_id": "string",             // 發放ID
  "coupon_id": "string",           // 優惠券ID
  "customer_id": "string",         // 客戶ID
  "issued_reason": "string",       // 發放原因
  "issued_by": "string",           // 發放人
  "issue_date": "datetime",        // 發放時間
  "used_date": "datetime",         // 使用時間
  "used_order_id": "string",       // 使用訂單ID
  "status": "string",              // 狀態 (issued/used/expired)
  "created_at": "datetime",        // 創建時間
  "updated_at": "datetime"         // 更新時間
}
```

---

## 4. API 端點

### 4.1 會員點數查詢

#### 4.1.1 查詢會員點數餘額
```
GET /api/v1/loyalty/customers/{customer_id}/balance
```

**權限**: `LOYALTY_READ`

**Response:**
```json
{
  "success": true,
  "data": {
    "account_id": "LOY-001",
    "customer_id": "CUST-001",
    "current_points": 1250,
    "total_earned": 5000,
    "total_redeemed": 3750,
    "pending_points": 100,
    "expiring_soon": 50,
    "member_tier": "Gold",
    "tier_progress": 75.5,
    "next_tier_points": 500,
    "last_activity": "2024-01-15T10:30:00Z",
    "status": "active"
  }
}
```

#### 4.1.2 查詢點數異動記錄
```
GET /api/v1/loyalty/customers/{customer_id}/transactions
```

**權限**: `LOYALTY_READ`

**Query Parameters:**
- `transaction_type` (string, optional): 交易類型篩選
- `date_from` (date, optional): 開始日期
- `date_to` (date, optional): 結束日期
- `page` (integer, optional): 頁碼
- `limit` (integer, optional): 每頁筆數

**Response:**
```json
{
  "success": true,
  "data": {
    "transactions": [
      {
        "transaction_id": "PT-001",
        "transaction_type": "earn",
        "points": 50,
        "balance_after": 1250,
        "source": "purchase",
        "source_reference": "ORD-001",
        "description": "消費獲得點數",
        "store_id": "ST-001",
        "expire_date": "2025-01-15",
        "created_at": "2024-01-15T10:30:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 100,
      "total_pages": 5
    }
  }
}
```

#### 4.1.3 查詢即將到期點數
```
GET /api/v1/loyalty/customers/{customer_id}/expiring-points
```

**權限**: `LOYALTY_READ`

**Query Parameters:**
- `days` (integer, optional): 天數範圍 (預設: 30)

### 4.2 點數異動

#### 4.2.1 消費獲得點數
```
POST /api/v1/loyalty/points/earn
```

**權限**: `LOYALTY_WRITE`

**Request Body:**
```json
{
  "customer_id": "CUST-001",
  "order_id": "ORD-001",
  "store_id": "ST-001",
  "order_amount": 250.00,
  "base_points": 250,
  "bonus_points": 50,
  "description": "消費獲得點數"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "transaction_id": "PT-001",
    "points_earned": 300,
    "balance_after": 1550,
    "tier_updated": false,
    "new_tier": null,
    "expire_date": "2025-01-15"
  }
}
```

#### 4.2.2 點數兌換
```
POST /api/v1/loyalty/points/redeem
```

**權限**: `LOYALTY_WRITE`

**Request Body:**
```json
{
  "customer_id": "CUST-001",
  "points": 500,
  "redeem_type": "discount",
  "redeem_value": 5.00,
  "order_id": "ORD-002",
  "store_id": "ST-001",
  "description": "點數兌換現金抵用"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "transaction_id": "PT-002",
    "points_redeemed": 500,
    "balance_after": 1050,
    "redeem_value": 5.00
  }
}
```

#### 4.2.3 手動調整點數
```
POST /api/v1/loyalty/points/adjust
```

**權限**: `LOYALTY_MANAGE`

**Request Body:**
```json
{
  "customer_id": "CUST-001",
  "adjustment": 100,
  "reason": "生日禮品",
  "operator_id": "USER-001",
  "notes": "生日當月額外贈送"
}
```

#### 4.2.4 點數過期處理
```
POST /api/v1/loyalty/points/expire
```

**權限**: `LOYALTY_ADMIN`

**Request Body:**
```json
{
  "expire_date": "2024-01-15",
  "dry_run": false
}
```

### 4.3 會員等級管理

#### 4.3.1 查詢會員等級制度
```
GET /api/v1/loyalty/tiers
```

**權限**: `LOYALTY_READ`

**Response:**
```json
{
  "success": true,
  "data": {
    "tiers": [
      {
        "tier_id": "TIER-001",
        "tier_name": "Bronze",
        "tier_level": 1,
        "required_points": 0,
        "required_spend": 0,
        "point_multiplier": 1.0,
        "birthday_bonus": 50,
        "tier_benefits": ["基礎會員權益"],
        "tier_color": "#CD7F32"
      },
      {
        "tier_id": "TIER-002", 
        "tier_name": "Silver",
        "tier_level": 2,
        "required_points": 1000,
        "required_spend": 1000,
        "point_multiplier": 1.2,
        "birthday_bonus": 100,
        "tier_benefits": ["生日折扣", "優先服務"],
        "tier_color": "#C0C0C0"
      }
    ]
  }
}
```

#### 4.3.2 更新會員等級
```
PUT /api/v1/loyalty/customers/{customer_id}/tier
```

**權限**: `LOYALTY_MANAGE`

**Request Body:**
```json
{
  "new_tier_id": "TIER-002",
  "reason": "達成升級條件",
  "operator_id": "USER-001"
}
```

#### 4.3.3 批量等級檢查
```
POST /api/v1/loyalty/tiers/batch-check
```

**權限**: `LOYALTY_ADMIN`

### 4.4 優惠券管理

#### 4.4.1 查詢客戶優惠券
```
GET /api/v1/loyalty/customers/{customer_id}/coupons
```

**權限**: `LOYALTY_READ`

**Query Parameters:**
- `status` (string, optional): 狀態篩選 (issued/used/expired)
- `coupon_type` (string, optional): 類型篩選

**Response:**
```json
{
  "success": true,
  "data": {
    "coupons": [
      {
        "issue_id": "CI-001",
        "coupon_id": "COUP-001",
        "coupon_code": "BIRTHDAY2024",
        "title": "生日專屬優惠",
        "description": "全品項9折",
        "discount_type": "percentage",
        "discount_value": 10,
        "minimum_spend": 100,
        "valid_until": "2024-02-15T23:59:59Z",
        "status": "issued",
        "issue_date": "2024-01-15T10:30:00Z"
      }
    ]
  }
}
```

#### 4.4.2 發放優惠券
```
POST /api/v1/loyalty/coupons/issue
```

**權限**: `LOYALTY_MANAGE`

**Request Body:**
```json
{
  "customer_id": "CUST-001",
  "coupon_id": "COUP-001",
  "issued_reason": "生日禮品",
  "issued_by": "SYSTEM",
  "custom_expire_date": "2024-02-15T23:59:59Z"
}
```

#### 4.4.3 使用優惠券
```
POST /api/v1/loyalty/coupons/use
```

**權限**: `LOYALTY_WRITE`

**Request Body:**
```json
{
  "customer_id": "CUST-001",
  "coupon_code": "BIRTHDAY2024",
  "order_id": "ORD-003",
  "order_amount": 150.00,
  "store_id": "ST-001"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "issue_id": "CI-001",
    "discount_applied": 15.00,
    "final_amount": 135.00,
    "used_date": "2024-01-15T10:30:00Z"
  }
}
```

#### 4.4.4 驗證優惠券
```
POST /api/v1/loyalty/coupons/validate
```

**權限**: `LOYALTY_READ`

**Request Body:**
```json
{
  "customer_id": "CUST-001",
  "coupon_code": "BIRTHDAY2024",
  "order_amount": 150.00,
  "store_id": "ST-001",
  "products": ["PROD-001", "PROD-002"]
}
```

### 4.5 生日禮品管理

#### 4.5.1 查詢生日會員
```
GET /api/v1/loyalty/birthdays
```

**權限**: `LOYALTY_READ`

**Query Parameters:**
- `month` (integer, optional): 月份篩選
- `days_ahead` (integer, optional): 提前天數 (預設: 7)

#### 4.5.2 發放生日禮品
```
POST /api/v1/loyalty/birthdays/gifts
```

**權限**: `LOYALTY_MANAGE`

**Request Body:**
```json
{
  "customer_ids": ["CUST-001", "CUST-002"],
  "gift_type": "points",
  "gift_value": 200,
  "coupon_id": "COUP-BIRTHDAY",
  "message": "生日快樂！"
}
```

### 4.6 推薦獎勵

#### 4.6.1 記錄推薦關係
```
POST /api/v1/loyalty/referrals
```

**權限**: `LOYALTY_WRITE`

**Request Body:**
```json
{
  "referrer_id": "CUST-001",
  "referee_id": "CUST-003",
  "referral_code": "REF-001-2024"
}
```

#### 4.6.2 推薦獎勵發放
```
POST /api/v1/loyalty/referrals/{referral_id}/reward
```

**權限**: `LOYALTY_MANAGE`

### 4.7 忠誠度分析

#### 4.7.1 會員活躍度分析
```
GET /api/v1/loyalty/analytics/activity
```

**權限**: `LOYALTY_READ`

**Query Parameters:**
- `date_from` (date, required): 開始日期
- `date_to` (date, required): 結束日期
- `store_id` (string, optional): 門店篩選

#### 4.7.2 點數使用分析
```
GET /api/v1/loyalty/analytics/points-usage
```

**權限**: `LOYALTY_READ`

#### 4.7.3 會員等級分布
```
GET /api/v1/loyalty/analytics/tier-distribution
```

**權限**: `LOYALTY_READ`

---

## 5. 錯誤代碼

### 點數相關錯誤
- `LOYALTY_001`: 點數餘額不足
- `LOYALTY_002`: 點數已過期
- `LOYALTY_003`: 點數帳戶不存在
- `LOYALTY_004`: 點數異動失敗
- `LOYALTY_005`: 等級升級條件不符

### 優惠券相關錯誤
- `COUPON_001`: 優惠券不存在
- `COUPON_002`: 優惠券已過期
- `COUPON_003`: 優惠券已使用
- `COUPON_004`: 優惠券使用條件不符
- `COUPON_005`: 優惠券使用次數已達上限

### 權限相關錯誤
- `AUTH_001`: 認證失敗
- `AUTH_002`: 權限不足
- `AUTH_003`: Token 過期

---

## 6. 業務規則

### 6.1 點數規則
- 點數有效期 1 年
- 消費 $1 = 1 基礎點數
- 會員等級影響點數倍數
- 點數最小兌換單位 100 點

### 6.2 等級規則
- 根據年度累積點數或消費金額升級
- 等級每年重新評估
- 降級需連續 2 年未達標

### 6.3 優惠券規則
- 每張優惠券有使用期限
- 部分優惠券有最低消費限制
- 優惠券不可轉讓

---

## 7. 整合介面

### 7.1 客戶服務整合
- 查詢客戶基本資料
- 驗證客戶狀態

### 7.2 訂單服務整合
- 接收訂單完成通知
- 計算獲得點數

### 7.3 通知服務整合
- 發送點數異動通知
- 發送等級升級通知
- 發送優惠券到期提醒

---

## 8. 性能要求

### 8.1 回應時間
- 點數查詢: < 200ms
- 點數異動: < 500ms
- 優惠券驗證: < 300ms

### 8.2 併發處理
- 支援同時 1000+ 點數查詢
- 支援同時 500+ 點數異動

---

## 9. 測試案例

### 9.1 功能測試
- 點數累積與兌換
- 會員等級升降級
- 優惠券發放與使用
- 生日禮品發放

### 9.2 異常測試
- 點數不足處理
- 優惠券過期處理
- 並發點數異動
- 系統異常恢復

---

**版本**: v1.0  
**建立日期**: 2025-11-16  
**最後更新**: 2025-11-16  
**負責人**: 開發團隊