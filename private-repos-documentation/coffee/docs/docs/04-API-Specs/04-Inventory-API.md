# 庫存管理 API 規格 (Inventory Management API)

## 1. API 概觀

### 服務名稱
- **服務名稱**: Inventory Service
- **Port**: 3004
- **Base URL**: `/api/v1/inventory`
- **業務職責**: 商品庫存管理、庫存盤點、庫存異動記錄

### 核心功能
- 庫存查詢與更新
- 庫存異動記錄
- 庫存盤點管理
- 庫存預警
- 庫存報表
- 商品庫存追蹤

---

## 2. 認證與授權

### 認證方式
- Bearer Token (JWT)
- 所有 API 需要驗證 `Authorization: Bearer <token>`

### 權限等級
- **INVENTORY_READ**: 庫存查詢權限
- **INVENTORY_WRITE**: 庫存異動權限
- **INVENTORY_AUDIT**: 庫存盤點權限
- **INVENTORY_ADMIN**: 庫存管理員權限

---

## 3. 資料模型

### 3.1 庫存資訊 (InventoryItem)
```json
{
  "inventory_id": "string",           // 庫存ID
  "store_id": "string",              // 門店ID
  "product_id": "string",            // 商品ID
  "product_sku": "string",           // 商品SKU
  "product_name": "string",          // 商品名稱
  "current_stock": "integer",        // 目前庫存
  "available_stock": "integer",      // 可用庫存
  "reserved_stock": "integer",       // 預留庫存
  "minimum_stock": "integer",        // 最小庫存
  "maximum_stock": "integer",        // 最大庫存
  "unit_cost": "number",             // 單位成本
  "last_updated": "datetime",        // 最後更新時間
  "status": "string"                 // 狀態 (active/inactive)
}
```

### 3.2 庫存異動記錄 (InventoryMovement)
```json
{
  "movement_id": "string",           // 異動ID
  "inventory_id": "string",          // 庫存ID
  "movement_type": "string",         // 異動類型 (in/out/adjust/transfer)
  "movement_reason": "string",       // 異動原因
  "quantity": "integer",             // 異動數量 (+/-)
  "previous_stock": "integer",       // 異動前庫存
  "after_stock": "integer",         // 異動後庫存
  "unit_cost": "number",            // 單位成本
  "reference_id": "string",         // 關聯單據ID
  "reference_type": "string",       // 關聯單據類型
  "operator_id": "string",          // 操作員ID
  "movement_date": "datetime",      // 異動時間
  "notes": "string"                 // 備註
}
```

### 3.3 盤點單 (StockCount)
```json
{
  "count_id": "string",             // 盤點ID
  "count_number": "string",         // 盤點單號
  "store_id": "string",            // 門店ID
  "count_type": "string",          // 盤點類型 (full/partial/cycle)
  "status": "string",              // 狀態 (draft/counting/completed/cancelled)
  "start_date": "datetime",        // 開始日期
  "end_date": "datetime",          // 結束日期
  "operator_id": "string",         // 盤點人ID
  "supervisor_id": "string",       // 督導ID
  "description": "string",         // 盤點說明
  "created_at": "datetime",        // 創建時間
  "updated_at": "datetime"         // 更新時間
}
```

### 3.4 盤點明細 (StockCountItem)
```json
{
  "count_item_id": "string",        // 盤點明細ID
  "count_id": "string",            // 盤點ID
  "inventory_id": "string",        // 庫存ID
  "product_id": "string",          // 商品ID
  "book_quantity": "integer",      // 帳面數量
  "count_quantity": "integer",     // 盤點數量
  "variance": "integer",           // 差異數量
  "variance_amount": "number",     // 差異金額
  "variance_reason": "string",     // 差異原因
  "count_status": "string",        // 盤點狀態 (pending/counted/verified)
  "counted_at": "datetime",        // 盤點時間
  "counted_by": "string"           // 盤點人
}
```

---

## 4. API 端點

### 4.1 庫存查詢

#### 4.1.1 查詢門店庫存清單
```
GET /api/v1/inventory/stores/{store_id}/items
```

**權限**: `INVENTORY_READ`

**Query Parameters:**
- `product_id` (string, optional): 商品ID篩選
- `sku` (string, optional): SKU篩選
- `category` (string, optional): 商品分類篩選
- `low_stock` (boolean, optional): 僅顯示低庫存商品
- `page` (integer, optional): 頁碼 (預設: 1)
- `limit` (integer, optional): 每頁筆數 (預設: 20)

**Response:**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "inventory_id": "INV-001",
        "store_id": "ST-001", 
        "product_id": "PROD-001",
        "product_sku": "COFFEE-LAT-001",
        "product_name": "拿鐵咖啡豆",
        "current_stock": 50,
        "available_stock": 45,
        "reserved_stock": 5,
        "minimum_stock": 20,
        "maximum_stock": 100,
        "unit_cost": 15.50,
        "last_updated": "2024-01-15T10:30:00Z",
        "status": "active"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 150,
      "total_pages": 8
    }
  }
}
```

#### 4.1.2 查詢特定商品庫存
```
GET /api/v1/inventory/stores/{store_id}/products/{product_id}
```

**權限**: `INVENTORY_READ`

**Response:**
```json
{
  "success": true,
  "data": {
    "inventory_id": "INV-001",
    "store_id": "ST-001",
    "product_id": "PROD-001", 
    "product_sku": "COFFEE-LAT-001",
    "product_name": "拿鐵咖啡豆",
    "current_stock": 50,
    "available_stock": 45,
    "reserved_stock": 5,
    "minimum_stock": 20,
    "maximum_stock": 100,
    "unit_cost": 15.50,
    "last_updated": "2024-01-15T10:30:00Z",
    "status": "active"
  }
}
```

#### 4.1.3 查詢庫存預警清單
```
GET /api/v1/inventory/alerts
```

**權限**: `INVENTORY_READ`

**Query Parameters:**
- `store_id` (string, optional): 門店ID篩選
- `alert_type` (string, optional): 預警類型 (low_stock/out_of_stock/overstock)

**Response:**
```json
{
  "success": true,
  "data": {
    "alerts": [
      {
        "alert_id": "ALERT-001",
        "inventory_id": "INV-001",
        "store_id": "ST-001",
        "product_name": "拿鐵咖啡豆",
        "current_stock": 15,
        "minimum_stock": 20,
        "alert_type": "low_stock",
        "severity": "warning",
        "created_at": "2024-01-15T10:30:00Z"
      }
    ]
  }
}
```

### 4.2 庫存異動

#### 4.2.1 庫存入庫
```
POST /api/v1/inventory/movements/inbound
```

**權限**: `INVENTORY_WRITE`

**Request Body:**
```json
{
  "store_id": "ST-001",
  "items": [
    {
      "product_id": "PROD-001",
      "quantity": 50,
      "unit_cost": 15.50,
      "reference_id": "PO-001",
      "reference_type": "purchase_order",
      "notes": "新進貨"
    }
  ],
  "movement_reason": "進貨入庫",
  "operator_id": "USER-001"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "movements": [
      {
        "movement_id": "MOV-001",
        "inventory_id": "INV-001",
        "movement_type": "in",
        "quantity": 50,
        "previous_stock": 30,
        "after_stock": 80,
        "movement_date": "2024-01-15T10:30:00Z"
      }
    ]
  }
}
```

#### 4.2.2 庫存出庫
```
POST /api/v1/inventory/movements/outbound
```

**權限**: `INVENTORY_WRITE`

**Request Body:**
```json
{
  "store_id": "ST-001",
  "items": [
    {
      "product_id": "PROD-001",
      "quantity": 10,
      "reference_id": "ORD-001",
      "reference_type": "sale_order",
      "notes": "銷售出貨"
    }
  ],
  "movement_reason": "銷售出庫",
  "operator_id": "USER-001"
}
```

#### 4.2.3 庫存調整
```
POST /api/v1/inventory/movements/adjustment
```

**權限**: `INVENTORY_WRITE`

**Request Body:**
```json
{
  "store_id": "ST-001",
  "items": [
    {
      "product_id": "PROD-001",
      "adjustment_quantity": -5,
      "adjustment_reason": "損耗",
      "notes": "過期品處理"
    }
  ],
  "operator_id": "USER-001"
}
```

#### 4.2.4 庫存調撥
```
POST /api/v1/inventory/movements/transfer
```

**權限**: `INVENTORY_WRITE`

**Request Body:**
```json
{
  "from_store_id": "ST-001",
  "to_store_id": "ST-002",
  "items": [
    {
      "product_id": "PROD-001",
      "quantity": 20,
      "notes": "門店調撥"
    }
  ],
  "transfer_reason": "庫存平衡",
  "operator_id": "USER-001"
}
```

### 4.3 庫存盤點

#### 4.3.1 建立盤點單
```
POST /api/v1/inventory/stock-counts
```

**權限**: `INVENTORY_AUDIT`

**Request Body:**
```json
{
  "store_id": "ST-001",
  "count_type": "full",
  "description": "月度全盤點",
  "start_date": "2024-01-20T09:00:00Z",
  "product_ids": [],
  "operator_id": "USER-001"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "count_id": "COUNT-001",
    "count_number": "SC-20240115-001",
    "store_id": "ST-001",
    "count_type": "full",
    "status": "draft",
    "start_date": "2024-01-20T09:00:00Z",
    "description": "月度全盤點",
    "created_at": "2024-01-15T10:30:00Z"
  }
}
```

#### 4.3.2 查詢盤點單列表
```
GET /api/v1/inventory/stock-counts
```

**權限**: `INVENTORY_READ`

**Query Parameters:**
- `store_id` (string, optional): 門店ID篩選
- `status` (string, optional): 狀態篩選
- `count_type` (string, optional): 盤點類型篩選

#### 4.3.3 查詢盤點單詳情
```
GET /api/v1/inventory/stock-counts/{count_id}
```

**權限**: `INVENTORY_READ`

#### 4.3.4 更新盤點數量
```
PUT /api/v1/inventory/stock-counts/{count_id}/items/{count_item_id}
```

**權限**: `INVENTORY_AUDIT`

**Request Body:**
```json
{
  "count_quantity": 45,
  "variance_reason": "正常損耗",
  "counted_by": "USER-001"
}
```

#### 4.3.5 完成盤點
```
POST /api/v1/inventory/stock-counts/{count_id}/complete
```

**權限**: `INVENTORY_AUDIT`

**Request Body:**
```json
{
  "supervisor_id": "USER-002",
  "apply_adjustments": true,
  "notes": "盤點完成，已套用庫存調整"
}
```

### 4.4 庫存報表

#### 4.4.1 庫存匯總報表
```
GET /api/v1/inventory/reports/summary
```

**權限**: `INVENTORY_READ`

**Query Parameters:**
- `store_id` (string, optional): 門店ID篩選
- `date_from` (string, optional): 開始日期
- `date_to` (string, optional): 結束日期

**Response:**
```json
{
  "success": true,
  "data": {
    "summary": {
      "total_products": 150,
      "total_value": 125000.00,
      "low_stock_items": 8,
      "out_of_stock_items": 2,
      "overstocked_items": 5
    },
    "by_category": [
      {
        "category": "咖啡豆",
        "total_quantity": 1200,
        "total_value": 18000.00,
        "item_count": 15
      }
    ]
  }
}
```

#### 4.4.2 庫存異動報表
```
GET /api/v1/inventory/reports/movements
```

**權限**: `INVENTORY_READ`

**Query Parameters:**
- `store_id` (string, optional): 門店ID篩選
- `movement_type` (string, optional): 異動類型篩選
- `date_from` (date, required): 開始日期
- `date_to` (date, required): 結束日期

#### 4.4.3 庫存周轉率報表
```
GET /api/v1/inventory/reports/turnover
```

**權限**: `INVENTORY_READ`

### 4.5 庫存預留

#### 4.5.1 預留庫存
```
POST /api/v1/inventory/reservations
```

**權限**: `INVENTORY_WRITE`

**Request Body:**
```json
{
  "store_id": "ST-001",
  "items": [
    {
      "product_id": "PROD-001",
      "quantity": 5,
      "reference_id": "ORD-001",
      "reference_type": "order",
      "expire_at": "2024-01-15T18:00:00Z"
    }
  ]
}
```

#### 4.5.2 釋放預留庫存
```
DELETE /api/v1/inventory/reservations/{reservation_id}
```

**權限**: `INVENTORY_WRITE`

#### 4.5.3 確認預留庫存 (轉為實際消耗)
```
POST /api/v1/inventory/reservations/{reservation_id}/confirm
```

**權限**: `INVENTORY_WRITE`

---

## 5. 錯誤代碼

### 庫存相關錯誤
- `INVENTORY_001`: 庫存不足
- `INVENTORY_002`: 商品不存在於該門店
- `INVENTORY_003`: 庫存已被預留
- `INVENTORY_004`: 庫存異動數量不正確
- `INVENTORY_005`: 盤點單狀態不正確
- `INVENTORY_006`: 庫存預留已過期
- `INVENTORY_007`: 庫存調撥門店不匹配

### 權限相關錯誤
- `AUTH_001`: 認證失敗
- `AUTH_002`: 權限不足
- `AUTH_003`: Token 過期

---

## 6. 業務規則

### 6.1 庫存管理規則
- 可用庫存 = 目前庫存 - 預留庫存
- 庫存不可為負數
- 最小庫存必須 ≤ 最大庫存
- 庫存調撥需要兩邊門店都有該商品

### 6.2 盤點規則
- 盤點期間庫存暫停異動
- 盤點完成後自動調整庫存差異
- 盤點單一旦開始不可修改商品清單

### 6.3 預留規則
- 預留庫存有時效性 (預設 4 小時)
- 超時未確認自動釋放
- 同一商品可多次預留

---

## 7. 整合介面

### 7.1 商品服務整合
- 查詢商品基本資料
- 驗證商品狀態

### 7.2 訂單服務整合
- 接收庫存預留請求
- 接收庫存扣減請求

### 7.3 採購服務整合
- 接收進貨通知
- 查詢採購狀態

### 7.4 通知服務整合
- 發送庫存預警通知
- 發送盤點完成通知

---

## 8. 性能要求

### 8.1 回應時間
- 庫存查詢: < 200ms
- 庫存異動: < 500ms
- 報表生成: < 2s

### 8.2 併發處理
- 支援同時 1000+ 庫存查詢
- 支援同時 100+ 庫存異動

### 8.3 資料一致性
- 庫存異動採用事務處理
- 預留庫存採用樂觀鎖定
- 盤點期間庫存鎖定

---

## 9. 測試案例

### 9.1 功能測試
- 正常庫存查詢
- 庫存入出庫作業
- 庫存盤點流程
- 庫存預留機制

### 9.2 異常測試
- 庫存不足處理
- 並發庫存異動
- 網路中斷恢復
- 資料庫連線異常

### 9.3 效能測試
- 大量庫存資料查詢
- 高併發庫存異動
- 複雜報表生成

---

**版本**: v1.0  
**建立日期**: 2025-11-16  
**最後更新**: 2025-11-16  
**負責人**: 開發團隊