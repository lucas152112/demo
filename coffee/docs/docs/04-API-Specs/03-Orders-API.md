# Orders API 規格

## 1. 概述

Orders Service 負責訂單管理，包括 POS 點餐、訂單處理、付款管理、發票開立等功能。

**服務資訊**:
- **服務名稱**: Orders Service
- **服務埠號**: 8083
- **資料庫**: coffee_orders (MySQL)
- **API 版本**: v1
- **Base URL**: `http://localhost:8080/api/v1/orders` (透過 API Gateway)

---

## 2. 資料模型

### 2.1 訂單基本資料

```json
{
  "id": "123e4567-e89b-12d3-a456-426614174000",
  "order_number": "ORD-20251116-0001",
  "store_id": "550e8400-e29b-41d4-a716-446655440000",
  "user_id": "550e8400-e29b-41d4-a716-446655440001",
  "customer": {
    "id": "550e8400-e29b-41d4-a716-446655440002",
    "name": "王小明",
    "phone": "+886987654321",
    "member_number": "MB-20251116-0001"
  },
  "status": "completed",
  "items": [
    {
      "id": "item-001",
      "product_id": "prod-001",
      "product_name": "美式咖啡",
      "quantity": 2,
      "unit_price": 110.00,
      "subtotal": 220.00,
      "variants": {
        "size": "medium",
        "temperature": "hot"
      }
    }
  ],
  "pricing": {
    "subtotal": 220.00,
    "tax": 11.00,
    "discount": 22.00,
    "total": 209.00
  },
  "payment": {
    "method": "credit_card",
    "amount": 209.00,
    "received": 209.00,
    "change": 0.00,
    "transaction_id": "TXN123456",
    "status": "success"
  },
  "points": {
    "earned": 20,
    "redeemed": 0
  },
  "created_at": "2025-11-16T10:30:00Z",
  "completed_at": "2025-11-16T10:35:00Z"
}
```

### 2.2 訂單狀態

| 狀態 | 英文代碼 | 描述 | 允許操作 |
|------|----------|------|----------|
| 草稿 | draft | 正在建立中 | 新增項目、修改、結帳 |
| 已完成 | completed | 付款完成 | 查看、退款 |
| 已取消 | cancelled | 已取消 | 查看 |

### 2.3 付款方式

| 方式 | 英文代碼 | 描述 |
|------|----------|------|
| 現金 | cash | 現金付款 |
| 信用卡 | credit_card | 信用卡付款 |
| 行動支付 | mobile_pay | 行動支付 (Line Pay, Apple Pay 等) |

---

## 3. API 端點

### 3.1 建立訂單

**端點**: `POST /api/v1/orders`  
**描述**: 建立新訂單 (POS 下單)  
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
  "customer_id": "550e8400-e29b-41d4-a716-446655440002",
  "items": [
    {
      "product_id": "prod-001",
      "quantity": 2,
      "variants": {
        "size": "medium",
        "temperature": "hot"
      }
    },
    {
      "product_id": "prod-002",
      "quantity": 1,
      "variants": {
        "size": "large",
        "temperature": "iced"
      }
    }
  ]
}
```

**Body Schema**:
| 欄位 | 類型 | 必須 | 描述 |
|------|------|------|------|
| customer_id | string | ✗ | 客戶 UUID (非會員可為空) |
| items | array | ✓ | 訂單項目陣列 |
| items[].product_id | string | ✓ | 商品 UUID |
| items[].quantity | integer | ✓ | 數量 (1-99) |
| items[].variants | object | ✗ | 商品規格選項 |

#### Response

**成功 (201 Created)**:
```json
{
  "success": true,
  "message": "訂單建立成功",
  "data": {
    "order": {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "order_number": "ORD-20251116-0001",
      "status": "draft",
      "items": [
        {
          "id": "item-001",
          "product_id": "prod-001",
          "product_name": "美式咖啡",
          "quantity": 2,
          "unit_price": 110.00,
          "subtotal": 220.00,
          "variants": {
            "size": "medium",
            "temperature": "hot"
          }
        }
      ],
      "pricing": {
        "subtotal": 330.00,
        "tax": 16.50,
        "discount": 0.00,
        "total": 346.50
      },
      "customer": {
        "id": "550e8400-e29b-41d4-a716-446655440002",
        "name": "王小明",
        "member_level": "gold",
        "available_points": 1500
      },
      "created_at": "2025-11-16T10:30:00Z"
    }
  }
}
```

**錯誤代碼**:
| 錯誤代碼 | HTTP 狀態 | 描述 |
|----------|-----------|------|
| PRODUCT_NOT_FOUND | 404 | 商品不存在 |
| PRODUCT_OUT_OF_STOCK | 400 | 商品庫存不足 |
| INVALID_QUANTITY | 400 | 數量無效 |
| CUSTOMER_NOT_FOUND | 404 | 客戶不存在 |

---

### 3.2 訂單列表查詢

**端點**: `GET /api/v1/orders`  
**描述**: 查詢訂單列表，支援分頁和篩選  
**權限**: store_manager, staff, headquarters

#### Request

**Query Parameters**:
| 參數 | 類型 | 必須 | 預設值 | 描述 |
|------|------|------|-------|------|
| page | integer | ✗ | 1 | 頁數 |
| limit | integer | ✗ | 20 | 每頁筆數 (1-100) |
| store_id | string | ✗ | current | 門店篩選 (staff 只能查自己門店) |
| status | string | ✗ | all | 狀態篩選 |
| customer_id | string | ✗ | - | 客戶篩選 |
| user_id | string | ✗ | - | 操作員篩選 |
| date_start | string | ✗ | - | 日期起始 (YYYY-MM-DD) |
| date_end | string | ✗ | - | 日期結束 (YYYY-MM-DD) |
| payment_method | string | ✗ | - | 付款方式篩選 |
| order_number | string | ✗ | - | 訂單編號搜尋 |
| sort | string | ✗ | created_at | 排序欄位 |
| order | string | ✗ | desc | 排序方向 |

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "查詢成功",
  "data": {
    "orders": [
      {
        "id": "123e4567-e89b-12d3-a456-426614174000",
        "order_number": "ORD-20251116-0001",
        "status": "completed",
        "customer": {
          "name": "王小明",
          "member_number": "MB-20251116-0001"
        },
        "user_name": "店員A",
        "total": 346.50,
        "payment_method": "credit_card",
        "created_at": "2025-11-16T10:30:00Z",
        "completed_at": "2025-11-16T10:35:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 150,
      "total_pages": 8
    },
    "summary": {
      "total_orders": 150,
      "total_amount": 52350.00,
      "completed_orders": 145,
      "cancelled_orders": 5
    }
  }
}
```

---

### 3.3 訂單詳細資料

**端點**: `GET /api/v1/orders/{order_id}`  
**描述**: 取得訂單詳細資料  
**權限**: store_manager, staff, headquarters

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "查詢成功",
  "data": {
    "order": {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "order_number": "ORD-20251116-0001",
      "store": {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "name": "信義店",
        "address": "台北市信義區信義路五段7號"
      },
      "user": {
        "id": "550e8400-e29b-41d4-a716-446655440001",
        "name": "店員A"
      },
      "customer": {
        "id": "550e8400-e29b-41d4-a716-446655440002",
        "name": "王小明",
        "phone": "+886987654321",
        "member_number": "MB-20251116-0001",
        "member_level": "gold"
      },
      "status": "completed",
      "items": [
        {
          "id": "item-001",
          "product_id": "prod-001",
          "product_name": "美式咖啡",
          "quantity": 2,
          "unit_price": 110.00,
          "subtotal": 220.00,
          "variants": {
            "size": "medium",
            "temperature": "hot"
          }
        }
      ],
      "pricing": {
        "subtotal": 330.00,
        "tax": 16.50,
        "discount": 33.00,
        "discount_reason": "金卡會員10%折扣",
        "total": 313.50
      },
      "payment": {
        "method": "credit_card",
        "amount": 313.50,
        "received": 313.50,
        "change": 0.00,
        "transaction_id": "TXN123456",
        "status": "success",
        "paid_at": "2025-11-16T10:34:00Z"
      },
      "points": {
        "earned": 31,
        "redeemed": 0,
        "balance_before": 1500,
        "balance_after": 1531
      },
      "invoice": {
        "invoice_number": "AB-12345678",
        "tax_id": null,
        "issued_at": "2025-11-16T10:35:00Z",
        "printed": true
      },
      "created_at": "2025-11-16T10:30:00Z",
      "completed_at": "2025-11-16T10:35:00Z"
    }
  }
}
```

---

### 3.4 修改訂單

**端點**: `PUT /api/v1/orders/{order_id}`  
**描述**: 修改草稿狀態的訂單  
**權限**: store_manager, staff

#### Request

**Body**:
```json
{
  "customer_id": "550e8400-e29b-41d4-a716-446655440002",
  "items": [
    {
      "product_id": "prod-001",
      "quantity": 1,
      "variants": {
        "size": "large",
        "temperature": "hot"
      }
    }
  ]
}
```

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "訂單更新成功",
  "data": {
    "order": {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "order_number": "ORD-20251116-0001",
      "status": "draft",
      "items": [...],
      "pricing": {
        "subtotal": 150.00,
        "tax": 7.50,
        "discount": 0.00,
        "total": 157.50
      },
      "updated_at": "2025-11-16T10:32:00Z"
    }
  }
}
```

**錯誤代碼**:
| 錯誤代碼 | HTTP 狀態 | 描述 |
|----------|-----------|------|
| ORDER_NOT_EDITABLE | 400 | 訂單狀態不允許修改 |
| ORDER_NOT_FOUND | 404 | 訂單不存在 |

---

### 3.5 訂單結帳

**端點**: `POST /api/v1/orders/{order_id}/checkout`  
**描述**: 訂單結帳付款  
**權限**: store_manager, staff

#### Request

**Body**:
```json
{
  "payment_method": "credit_card",
  "received_amount": 313.50,
  "points_to_redeem": 100,
  "tax_id": "12345678",
  "transaction_id": "TXN123456"
}
```

**Body Schema**:
| 欄位 | 類型 | 必須 | 描述 |
|------|------|------|------|
| payment_method | string | ✓ | 付款方式 (cash/credit_card/mobile_pay) |
| received_amount | number | 條件 | 收款金額 (現金付款時必須) |
| points_to_redeem | integer | ✗ | 要兌換的點數 |
| tax_id | string | ✗ | 統一編號 (開發票用) |
| transaction_id | string | 條件 | 交易編號 (信用卡/行動支付時必須) |

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "結帳成功",
  "data": {
    "order": {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "order_number": "ORD-20251116-0001",
      "status": "completed",
      "total": 213.50,
      "payment": {
        "method": "credit_card",
        "amount": 213.50,
        "received": 213.50,
        "change": 0.00,
        "transaction_id": "TXN123456",
        "status": "success"
      },
      "points": {
        "earned": 21,
        "redeemed": 100,
        "balance_before": 1500,
        "balance_after": 1421
      },
      "invoice": {
        "invoice_number": "AB-12345678",
        "tax_id": "12345678"
      },
      "completed_at": "2025-11-16T10:35:00Z"
    }
  }
}
```

**錯誤代碼**:
| 錯誤代碼 | HTTP 狀態 | 描述 |
|----------|-----------|------|
| ORDER_ALREADY_COMPLETED | 400 | 訂單已完成 |
| INSUFFICIENT_POINTS | 400 | 點數不足 |
| PAYMENT_FAILED | 400 | 付款失敗 |
| INSUFFICIENT_AMOUNT | 400 | 收款金額不足 |

---

### 3.6 取消訂單

**端點**: `POST /api/v1/orders/{order_id}/cancel`  
**描述**: 取消訂單  
**權限**: store_manager, staff

#### Request

**Body**:
```json
{
  "reason": "客戶要求取消"
}
```

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "訂單取消成功",
  "data": {
    "order_id": "123e4567-e89b-12d3-a456-426614174000",
    "status": "cancelled",
    "cancelled_at": "2025-11-16T10:40:00Z",
    "cancel_reason": "客戶要求取消"
  }
}
```

---

### 3.7 訂單退款

**端點**: `POST /api/v1/orders/{order_id}/refund`  
**描述**: 訂單部分或全額退款  
**權限**: store_manager

#### Request

**Body**:
```json
{
  "refund_amount": 313.50,
  "reason": "商品有問題",
  "refund_points": true,
  "items": [
    {
      "item_id": "item-001",
      "quantity": 1
    }
  ]
}
```

**Body Schema**:
| 欄位 | 類型 | 必須 | 描述 |
|------|------|------|------|
| refund_amount | number | ✓ | 退款金額 |
| reason | string | ✓ | 退款原因 |
| refund_points | boolean | ✗ | 是否退還點數 |
| items | array | ✗ | 退款項目 (部分退款) |

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "退款處理成功",
  "data": {
    "refund_id": "refund-001",
    "order_id": "123e4567-e89b-12d3-a456-426614174000",
    "refund_amount": 313.50,
    "points_refunded": 31,
    "status": "processed",
    "processed_at": "2025-11-16T11:00:00Z"
  }
}
```

---

### 3.8 重新列印發票

**端點**: `POST /api/v1/orders/{order_id}/reprint-invoice`  
**描述**: 重新列印發票  
**權限**: store_manager, staff

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "發票重新列印成功",
  "data": {
    "order_id": "123e4567-e89b-12d3-a456-426614174000",
    "invoice_number": "AB-12345678",
    "print_count": 2,
    "printed_at": "2025-11-16T11:05:00Z"
  }
}
```

---

### 3.9 今日銷售統計

**端點**: `GET /api/v1/orders/daily-summary`  
**描述**: 取得今日銷售統計  
**權限**: store_manager, staff

#### Request

**Query Parameters**:
| 參數 | 類型 | 必須 | 描述 |
|------|------|------|------|
| date | string | ✗ | 查詢日期 (YYYY-MM-DD，預設今日) |
| store_id | string | ✗ | 門店 ID (staff 自動使用所屬門店) |

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "查詢成功",
  "data": {
    "date": "2025-11-16",
    "store": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "信義店"
    },
    "summary": {
      "total_orders": 45,
      "completed_orders": 42,
      "cancelled_orders": 3,
      "total_revenue": 15680.50,
      "total_tax": 784.03,
      "total_discount": 1568.05,
      "average_order_value": 373.35
    },
    "payment_methods": [
      {
        "method": "credit_card",
        "count": 25,
        "amount": 9408.30
      },
      {
        "method": "cash",
        "count": 12,
        "amount": 4476.15
      },
      {
        "method": "mobile_pay",
        "count": 5,
        "amount": 1796.05
      }
    ],
    "hourly_sales": [
      {
        "hour": 9,
        "orders": 3,
        "revenue": 1120.50
      },
      {
        "hour": 10,
        "orders": 8,
        "revenue": 2980.00
      }
    ],
    "top_products": [
      {
        "product_name": "美式咖啡",
        "quantity": 18,
        "revenue": 1980.00
      },
      {
        "product_name": "拿鐵咖啡",
        "quantity": 12,
        "revenue": 1680.00
      }
    ]
  }
}
```

---

### 3.10 快速查找訂單

**端點**: `GET /api/v1/orders/search/{order_number}`  
**描述**: 以訂單編號快速查找訂單  
**權限**: store_manager, staff

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "查詢成功",
  "data": {
    "order": {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "order_number": "ORD-20251116-0001",
      "status": "completed",
      "customer_name": "王小明",
      "total": 313.50,
      "created_at": "2025-11-16T10:30:00Z"
    }
  }
}
```

---

## 4. 事件通知

### 4.1 訂單完成事件

**主題**: `orders.completed`  
**內容**:
```json
{
  "event_type": "order_completed",
  "order_id": "123e4567-e89b-12d3-a456-426614174000",
  "order_number": "ORD-20251116-0001",
  "store_id": "550e8400-e29b-41d4-a716-446655440000",
  "customer_id": "550e8400-e29b-41d4-a716-446655440002",
  "total_amount": 313.50,
  "points_earned": 31,
  "items": [
    {
      "product_id": "prod-001",
      "quantity": 2
    }
  ],
  "completed_at": "2025-11-16T10:35:00Z"
}
```

### 4.2 訂單取消事件

**主題**: `orders.cancelled`  
**內容**:
```json
{
  "event_type": "order_cancelled",
  "order_id": "123e4567-e89b-12d3-a456-426614174000",
  "reason": "客戶要求取消",
  "cancelled_at": "2025-11-16T10:40:00Z"
}
```

---

## 5. 業務規則

### 5.1 訂單編號生成

格式：`ORD-YYYYMMDD-XXXX`
- ORD: Order 前綴
- YYYYMMDD: 建立日期
- XXXX: 當日流水號 (0001~9999)

### 5.2 金額計算

1. 小計 = Σ(單價 × 數量)
2. 折扣 = 會員折扣 + 優惠券折扣
3. 稅額 = (小計 - 折扣) × 稅率 (5%)
4. 總額 = 小計 - 折扣 + 稅額

### 5.3 點數計算

- 賺取：總金額 × 會員等級倍數 (regular:1x, silver:1.2x, gold:1.5x, platinum:2x)
- 兌換：100 點 = $1
- 點數異動記錄會發送到 Loyalty Service

### 5.4 庫存扣減

- 訂單完成時自動扣減商品庫存
- 訂單取消時自動回補庫存
- 庫存異動會發送事件到 Inventory Service

---

## 6. 測試案例

### 6.1 建立訂單測試

```bash
# 正常建立訂單
curl -X POST http://localhost:8080/api/v1/orders \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "customer_id": "customer-001",
    "items": [
      {
        "product_id": "prod-001",
        "quantity": 2,
        "variants": {
          "size": "medium",
          "temperature": "hot"
        }
      }
    ]
  }'
```

### 6.2 結帳測試

```bash
# 信用卡結帳
curl -X POST http://localhost:8080/api/v1/orders/order-001/checkout \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "payment_method": "credit_card",
    "points_to_redeem": 50,
    "transaction_id": "TXN123456"
  }'

# 現金結帳
curl -X POST http://localhost:8080/api/v1/orders/order-001/checkout \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "payment_method": "cash",
    "received_amount": 500.00
  }'
```

### 6.3 查詢測試

```bash
# 訂單列表
curl -X GET "http://localhost:8080/api/v1/orders?date_start=2025-11-16&status=completed" \
  -H "Authorization: Bearer <token>"

# 今日統計
curl -X GET "http://localhost:8080/api/v1/orders/daily-summary?date=2025-11-16" \
  -H "Authorization: Bearer <token>"
```

---

## 7. 版本歷程

| 版本 | 日期 | 作者 | 說明 |
|------|------|------|------|
| 1.0 | 2025-11-16 | System | 初始 Orders API 規格建立 |

---

**備註**: 此 API 與 Inventory Service、Loyalty Service、Customers Service 緊密協作。