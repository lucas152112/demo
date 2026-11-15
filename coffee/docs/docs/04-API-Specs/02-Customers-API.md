# Customers API 規格

## 1. 概述

Customers Service 負責客戶資料管理，包括客戶資料 CRUD、標籤管理、會員等級管理等功能。

**服務資訊**:
- **服務名稱**: Customers Service
- **服務埠號**: 8082
- **資料庫**: coffee_customers (MySQL)
- **API 版本**: v1
- **Base URL**: `http://localhost:8080/api/v1/customers` (透過 API Gateway)

---

## 2. 資料模型

### 2.1 客戶基本資料

```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "owner_user_id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "王小明",
  "phone": "+886987654321",
  "email": "wang@example.com",
  "birthday": "1990-05-15",
  "member_number": "MB-20251116-0001",
  "member_level": "gold",
  "total_spent": 15680.50,
  "total_visits": 42,
  "tags": ["VIP", "咖啡愛好者"],
  "created_at": "2025-11-16T10:30:00Z",
  "last_visit_at": "2025-11-15T14:20:00Z"
}
```

### 2.2 會員等級

| 等級 | 英文代碼 | 升級條件 | 權益 |
|------|----------|----------|------|
| 一般會員 | regular | 初始等級 | 基本點數累積 |
| 銀卡會員 | silver | 累計消費 ≥ $3,000 | 1.2倍點數、生日優惠 |
| 金卡會員 | gold | 累計消費 ≥ $10,000 | 1.5倍點數、專屬優惠 |
| 白金會員 | platinum | 累計消費 ≥ $30,000 | 2倍點數、優先服務 |

---

## 3. API 端點

### 3.1 建立客戶

**端點**: `POST /api/v1/customers`  
**描述**: 新增客戶資料  
**權限**: store_manager, staff

#### Request

**Headers**:
```http
Content-Type: application/json
Authorization: Bearer <jwt_token>
```

**Body**:
```json
{
  "name": "王小明",
  "phone": "+886987654321",
  "email": "wang@example.com",
  "birthday": "1990-05-15",
  "tags": ["VIP", "咖啡愛好者"]
}
```

**Body Schema**:
| 欄位 | 類型 | 必須 | 限制 | 描述 |
|------|------|------|------|------|
| name | string | ✓ | 2-100 字符 | 客戶姓名 |
| phone | string | ✓ | 手機號碼格式 | 手機號碼 |
| email | string | ✗ | 有效 Email 格式 | Email 地址 |
| birthday | string | ✗ | YYYY-MM-DD 格式 | 生日 |
| tags | array | ✗ | 標籤名稱陣列 | 客戶標籤 |

#### Response

**成功 (201 Created)**:
```json
{
  "success": true,
  "message": "客戶建立成功",
  "data": {
    "customer": {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "owner_user_id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "王小明",
      "phone": "+886987654321",
      "email": "wang@example.com",
      "birthday": "1990-05-15",
      "member_number": "MB-20251116-0001",
      "member_level": "regular",
      "total_spent": 0.00,
      "total_visits": 0,
      "tags": ["VIP", "咖啡愛好者"],
      "created_at": "2025-11-16T10:30:00Z",
      "last_visit_at": null
    }
  }
}
```

**錯誤代碼**:
| 錯誤代碼 | HTTP 狀態 | 描述 |
|----------|-----------|------|
| PHONE_ALREADY_EXISTS | 409 | 手機號碼已存在 |
| INVALID_PHONE_FORMAT | 400 | 手機號碼格式無效 |
| INVALID_EMAIL_FORMAT | 400 | Email 格式無效 |
| INVALID_BIRTHDAY_FORMAT | 400 | 生日格式無效 |

---

### 3.2 客戶列表查詢

**端點**: `GET /api/v1/customers`  
**描述**: 查詢客戶列表，支援分頁和篩選  
**權限**: store_manager, staff, headquarters

#### Request

**Headers**:
```http
Authorization: Bearer <jwt_token>
```

**Query Parameters**:
| 參數 | 類型 | 必須 | 預設值 | 描述 |
|------|------|------|-------|------|
| page | integer | ✗ | 1 | 頁數 |
| limit | integer | ✗ | 20 | 每頁筆數 (1-100) |
| search | string | ✗ | - | 搜尋關鍵字 (姓名/手機/會員編號) |
| member_level | string | ✗ | all | 會員等級篩選 |
| tag | string | ✗ | - | 標籤篩選 |
| created_start | string | ✗ | - | 建立日期起始 (YYYY-MM-DD) |
| created_end | string | ✗ | - | 建立日期結束 (YYYY-MM-DD) |
| sort | string | ✗ | created_at | 排序欄位 |
| order | string | ✗ | desc | 排序方向 (asc/desc) |

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "查詢成功",
  "data": {
    "customers": [
      {
        "id": "123e4567-e89b-12d3-a456-426614174000",
        "name": "王小明",
        "phone": "+886987654321",
        "email": "wang@example.com",
        "member_number": "MB-20251116-0001",
        "member_level": "gold",
        "total_spent": 15680.50,
        "total_visits": 42,
        "tags": ["VIP", "咖啡愛好者"],
        "created_at": "2025-11-16T10:30:00Z",
        "last_visit_at": "2025-11-15T14:20:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 150,
      "total_pages": 8,
      "has_next": true,
      "has_prev": false
    },
    "summary": {
      "total_customers": 150,
      "regular_members": 80,
      "silver_members": 40,
      "gold_members": 25,
      "platinum_members": 5
    }
  }
}
```

---

### 3.3 客戶詳細資料

**端點**: `GET /api/v1/customers/{customer_id}`  
**描述**: 取得客戶詳細資料  
**權限**: store_manager, staff, headquarters

#### Request

**Headers**:
```http
Authorization: Bearer <jwt_token>
```

**Path Parameters**:
| 參數 | 類型 | 描述 |
|------|------|------|
| customer_id | string | 客戶 UUID |

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "查詢成功",
  "data": {
    "customer": {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "owner_user_id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "王小明",
      "phone": "+886987654321",
      "email": "wang@example.com",
      "birthday": "1990-05-15",
      "member_number": "MB-20251116-0001",
      "member_level": "gold",
      "total_spent": 15680.50,
      "total_visits": 42,
      "tags": ["VIP", "咖啡愛好者"],
      "created_at": "2025-11-16T10:30:00Z",
      "last_visit_at": "2025-11-15T14:20:00Z"
    },
    "loyalty": {
      "current_points": 1568,
      "total_earned": 3120,
      "total_redeemed": 1552,
      "tier_progress": {
        "current_tier": "gold",
        "next_tier": "platinum",
        "progress_percent": 52.27,
        "amount_needed": 14319.50
      }
    }
  }
}
```

**錯誤代碼**:
| 錯誤代碼 | HTTP 狀態 | 描述 |
|----------|-----------|------|
| CUSTOMER_NOT_FOUND | 404 | 客戶不存在 |

---

### 3.4 更新客戶資料

**端點**: `PUT /api/v1/customers/{customer_id}`  
**描述**: 更新客戶資料  
**權限**: store_manager, staff

#### Request

**Headers**:
```http
Content-Type: application/json
Authorization: Bearer <jwt_token>
```

**Body**:
```json
{
  "name": "王小明",
  "phone": "+886987654321",
  "email": "wang.new@example.com",
  "birthday": "1990-05-15"
}
```

**Body Schema**:
| 欄位 | 類型 | 必須 | 限制 | 描述 |
|------|------|------|------|------|
| name | string | ✓ | 2-100 字符 | 客戶姓名 |
| phone | string | ✓ | 手機號碼格式 | 手機號碼 |
| email | string | ✗ | 有效 Email 格式 | Email 地址 |
| birthday | string | ✗ | YYYY-MM-DD 格式 | 生日 |

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "客戶資料更新成功",
  "data": {
    "customer": {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "name": "王小明",
      "phone": "+886987654321",
      "email": "wang.new@example.com",
      "birthday": "1990-05-15",
      "member_number": "MB-20251116-0001",
      "member_level": "gold",
      "total_spent": 15680.50,
      "total_visits": 42,
      "tags": ["VIP", "咖啡愛好者"],
      "updated_at": "2025-11-16T11:30:00Z"
    }
  }
}
```

---

### 3.5 手機號碼搜尋

**端點**: `GET /api/v1/customers/search/phone/{phone}`  
**描述**: 以手機號碼搜尋客戶 (POS 快速查詢)  
**權限**: store_manager, staff

#### Request

**Headers**:
```http
Authorization: Bearer <jwt_token>
```

**Path Parameters**:
| 參數 | 類型 | 描述 |
|------|------|------|
| phone | string | 手機號碼 |

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "查詢成功",
  "data": {
    "customer": {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "name": "王小明",
      "phone": "+886987654321",
      "member_number": "MB-20251116-0001",
      "member_level": "gold",
      "current_points": 1568
    }
  }
}
```

**錯誤代碼**:
| 錯誤代碼 | HTTP 狀態 | 描述 |
|----------|-----------|------|
| CUSTOMER_NOT_FOUND | 404 | 找不到該手機號碼的客戶 |

---

### 3.6 標籤管理

#### 3.6.1 新增標籤

**端點**: `POST /api/v1/customers/{customer_id}/tags`  
**描述**: 為客戶新增標籤  
**權限**: store_manager, staff

**Body**:
```json
{
  "tags": ["常客", "喜歡拿鐵"]
}
```

#### 3.6.2 移除標籤

**端點**: `DELETE /api/v1/customers/{customer_id}/tags`  
**描述**: 移除客戶標籤

**Body**:
```json
{
  "tags": ["常客"]
}
```

#### 3.6.3 取得所有標籤

**端點**: `GET /api/v1/customers/tags`  
**描述**: 取得系統中所有標籤列表

**Response**:
```json
{
  "success": true,
  "message": "查詢成功",
  "data": {
    "tags": [
      {
        "name": "VIP",
        "customer_count": 25,
        "created_at": "2025-11-16T10:30:00Z"
      },
      {
        "name": "常客",
        "customer_count": 80,
        "created_at": "2025-11-16T10:30:00Z"
      }
    ]
  }
}
```

---

### 3.7 會員等級管理

#### 3.7.1 手動調整會員等級

**端點**: `PATCH /api/v1/customers/{customer_id}/member-level`  
**描述**: 手動調整客戶會員等級  
**權限**: store_manager

**Body**:
```json
{
  "member_level": "platinum",
  "reason": "客服主管調整"
}
```

#### 3.7.2 重新計算會員等級

**端點**: `POST /api/v1/customers/{customer_id}/recalculate-level`  
**描述**: 根據消費金額重新計算會員等級  
**權限**: store_manager

**Response**:
```json
{
  "success": true,
  "message": "會員等級重新計算完成",
  "data": {
    "customer_id": "123e4567-e89b-12d3-a456-426614174000",
    "old_level": "silver",
    "new_level": "gold",
    "total_spent": 15680.50,
    "calculated_at": "2025-11-16T12:00:00Z"
  }
}
```

---

### 3.8 消費統計

**端點**: `GET /api/v1/customers/{customer_id}/statistics`  
**描述**: 取得客戶消費統計資料  
**權限**: store_manager, headquarters

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "查詢成功",
  "data": {
    "customer_id": "123e4567-e89b-12d3-a456-426614174000",
    "summary": {
      "total_spent": 15680.50,
      "total_visits": 42,
      "average_order_value": 373.35,
      "first_visit": "2024-05-15T14:30:00Z",
      "last_visit": "2025-11-15T14:20:00Z",
      "visit_frequency_days": 7.2
    },
    "monthly_stats": [
      {
        "month": "2025-11",
        "visits": 3,
        "spent": 1120.50,
        "avg_order": 373.50
      },
      {
        "month": "2025-10",
        "visits": 8,
        "spent": 2980.00,
        "avg_order": 372.50
      }
    ],
    "favorite_products": [
      {
        "product_name": "美式咖啡",
        "order_count": 18,
        "total_spent": 1980.00
      },
      {
        "product_name": "拿鐵咖啡",
        "order_count": 12,
        "total_spent": 1680.00
      }
    ]
  }
}
```

---

### 3.9 批次匯入

**端點**: `POST /api/v1/customers/import`  
**描述**: 批次匯入客戶資料  
**權限**: admin, headquarters

#### Request

**Headers**:
```http
Content-Type: multipart/form-data
Authorization: Bearer <jwt_token>
```

**Body**:
```
file: customers.csv
format: csv
```

**CSV 格式**:
```csv
name,phone,email,birthday,tags
王小明,+886987654321,wang@example.com,1990-05-15,"VIP,常客"
李小華,+886976543210,li@example.com,1985-08-20,常客
```

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "匯入完成",
  "data": {
    "import_id": "import-20251116-001",
    "total_records": 100,
    "success_count": 95,
    "error_count": 5,
    "errors": [
      {
        "line": 10,
        "error": "手機號碼已存在",
        "data": "張小美,+886987654321,..."
      }
    ]
  }
}
```

---

### 3.10 匯出客戶資料

**端點**: `POST /api/v1/customers/export`  
**描述**: 匯出客戶資料  
**權限**: admin, headquarters, store_manager

#### Request

**Body**:
```json
{
  "format": "csv",
  "filters": {
    "member_level": "gold",
    "created_start": "2025-01-01",
    "created_end": "2025-12-31"
  },
  "fields": ["name", "phone", "email", "member_level", "total_spent"]
}
```

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "匯出任務已建立",
  "data": {
    "export_id": "export-20251116-001",
    "status": "processing",
    "download_url": null,
    "estimated_time": "2分鐘",
    "created_at": "2025-11-16T12:00:00Z"
  }
}
```

---

## 4. 事件通知

當客戶資料異動時，系統會發送事件到 NATS：

### 4.1 客戶建立事件

**主題**: `customers.created`  
**內容**:
```json
{
  "event_type": "customer_created",
  "customer_id": "123e4567-e89b-12d3-a456-426614174000",
  "customer_data": {
    "name": "王小明",
    "phone": "+886987654321",
    "member_number": "MB-20251116-0001"
  },
  "created_by": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": "2025-11-16T10:30:00Z"
}
```

### 4.2 會員等級異動事件

**主題**: `customers.level_changed`  
**內容**:
```json
{
  "event_type": "member_level_changed",
  "customer_id": "123e4567-e89b-12d3-a456-426614174000",
  "old_level": "silver",
  "new_level": "gold",
  "total_spent": 15680.50,
  "timestamp": "2025-11-16T10:30:00Z"
}
```

---

## 5. 業務規則

### 5.1 會員編號生成

格式：`MB-YYYYMMDD-XXXX`
- MB: Member 前綴
- YYYYMMDD: 建立日期
- XXXX: 當日流水號 (0001~9999)

### 5.2 會員等級自動升級

- 當客戶 `total_spent` 異動時自動檢查升級條件
- 升級後發送通知事件
- 降級需要手動調整

### 5.3 資料驗證規則

- 手機號碼必須唯一
- 會員編號必須唯一
- Email 格式驗證 (如果提供)
- 生日不能是未來日期

---

## 6. 測試案例

### 6.1 建立客戶測試

```bash
# 正常建立客戶
curl -X POST http://localhost:8080/api/v1/customers \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "name": "測試客戶",
    "phone": "+886912345678",
    "email": "test@example.com",
    "birthday": "1990-01-01",
    "tags": ["測試", "新客戶"]
  }'

# 重複手機號碼測試
curl -X POST http://localhost:8080/api/v1/customers \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "name": "重複測試",
    "phone": "+886912345678",
    "email": "duplicate@example.com"
  }'
```

### 6.2 搜尋測試

```bash
# 手機號碼搜尋
curl -X GET "http://localhost:8080/api/v1/customers/search/phone/+886912345678" \
  -H "Authorization: Bearer <token>"

# 列表查詢
curl -X GET "http://localhost:8080/api/v1/customers?search=測試&member_level=regular&page=1&limit=10" \
  -H "Authorization: Bearer <token>"
```

---

## 7. 版本歷程

| 版本 | 日期 | 作者 | 說明 |
|------|------|------|------|
| 1.0 | 2025-11-16 | System | 初始 Customers API 規格建立 |

---

**備註**: 此 API 與 Loyalty Service 緊密協作，客戶等級變動會觸發點數規則調整。