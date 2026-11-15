# 通知服務 API 規格 (Notifications Service API)

## 1. API 概觀

### 服務名稱
- **服務名稱**: Notifications Service
- **Port**: 3006
- **Base URL**: `/api/v1/notifications`
- **業務職責**: 系統通知發送、訊息推播、郵件發送、簡訊通知、系統提醒

### 核心功能
- 即時通知推播
- 電子郵件發送
- 簡訊發送
- 系統內訊息
- 通知範本管理
- 通知歷史記錄
- 通知偏好設定

---

## 2. 認證與授權

### 認證方式
- Bearer Token (JWT)
- 所有 API 需要驗證 `Authorization: Bearer <token>`

### 權限等級
- **NOTIFICATION_READ**: 通知查詢權限
- **NOTIFICATION_SEND**: 通知發送權限
- **NOTIFICATION_MANAGE**: 通知管理權限
- **NOTIFICATION_ADMIN**: 通知系統管理權限

---

## 3. 資料模型

### 3.1 通知訊息 (Notification)
```json
{
  "notification_id": "string",       // 通知ID
  "title": "string",                // 標題
  "message": "string",              // 訊息內容
  "type": "string",                 // 通知類型 (system/order/promotion/alert)
  "category": "string",             // 類別 (info/warning/error/success)
  "priority": "string",             // 優先級 (low/medium/high/urgent)
  "channels": "array",              // 發送管道 (push/email/sms/in_app)
  "recipients": "array",            // 收件人清單
  "template_id": "string",          // 範本ID
  "template_data": "object",        // 範本資料
  "scheduled_at": "datetime",       // 預定發送時間
  "sent_at": "datetime",           // 實際發送時間
  "status": "string",              // 狀態 (draft/scheduled/sending/sent/failed)
  "delivery_status": "object",     // 各管道發送狀態
  "retry_count": "integer",        // 重試次數
  "created_by": "string",          // 建立者
  "created_at": "datetime",        // 建立時間
  "updated_at": "datetime"         // 更新時間
}
```

### 3.2 通知範本 (NotificationTemplate)
```json
{
  "template_id": "string",          // 範本ID
  "template_name": "string",        // 範本名稱
  "template_code": "string",        // 範本代碼
  "description": "string",          // 描述
  "type": "string",                // 類型 (email/sms/push/in_app)
  "subject": "string",             // 主旨 (email適用)
  "content": "string",             // 內容範本
  "variables": "array",            // 變數清單
  "style_config": "object",        // 樣式設定
  "is_active": "boolean",          // 是否啟用
  "version": "string",             // 版本
  "created_at": "datetime",        // 建立時間
  "updated_at": "datetime"         // 更新時間
}
```

### 3.3 通知偏好設定 (NotificationPreference)
```json
{
  "preference_id": "string",        // 偏好設定ID
  "user_id": "string",             // 使用者ID
  "notification_type": "string",   // 通知類型
  "channels": {                    // 管道偏好
    "push": "boolean",
    "email": "boolean", 
    "sms": "boolean",
    "in_app": "boolean"
  },
  "quiet_hours": {                 // 勿擾時間
    "enabled": "boolean",
    "start_time": "string",
    "end_time": "string"
  },
  "frequency": "string",           // 頻率 (immediate/daily/weekly)
  "language": "string",            // 語言偏好
  "timezone": "string",            // 時區
  "created_at": "datetime",        // 建立時間
  "updated_at": "datetime"         // 更新時間
}
```

### 3.4 發送記錄 (DeliveryLog)
```json
{
  "log_id": "string",              // 記錄ID
  "notification_id": "string",     // 通知ID
  "recipient": "string",           // 收件人
  "channel": "string",             // 發送管道
  "status": "string",              // 狀態 (sent/failed/pending)
  "response_code": "string",       // 回應代碼
  "response_message": "string",    // 回應訊息
  "provider": "string",            // 服務提供商
  "cost": "number",               // 發送成本
  "sent_at": "datetime",          // 發送時間
  "delivered_at": "datetime",     // 送達時間
  "opened_at": "datetime",        // 開啟時間
  "clicked_at": "datetime"        // 點擊時間
}
```

### 3.5 通知訂閱 (NotificationSubscription)
```json
{
  "subscription_id": "string",      // 訂閱ID
  "user_id": "string",             // 使用者ID
  "device_token": "string",        // 裝置Token (Push適用)
  "endpoint": "string",            // 推播端點
  "platform": "string",           // 平台 (ios/android/web)
  "is_active": "boolean",          // 是否啟用
  "created_at": "datetime",        // 建立時間
  "updated_at": "datetime"         // 更新時間
}
```

---

## 4. API 端點

### 4.1 通知發送

#### 4.1.1 發送即時通知
```
POST /api/v1/notifications/send
```

**權限**: `NOTIFICATION_SEND`

**Request Body:**
```json
{
  "title": "訂單完成通知",
  "message": "您的訂單 #ORD-001 已完成製作，請至櫃台取餐",
  "type": "order",
  "category": "info",
  "priority": "medium",
  "channels": ["push", "in_app"],
  "recipients": [
    {
      "type": "user",
      "id": "USER-001",
      "contact": {
        "email": "customer@example.com",
        "phone": "+886912345678"
      }
    }
  ],
  "template_data": {
    "order_id": "ORD-001",
    "customer_name": "王小明",
    "store_name": "台北信義店"
  },
  "scheduled_at": null
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "notification_id": "NOTIF-001",
    "status": "sent",
    "delivery_status": {
      "push": "sent",
      "in_app": "delivered"
    },
    "sent_at": "2024-01-15T10:30:00Z"
  }
}
```

#### 4.1.2 使用範本發送通知
```
POST /api/v1/notifications/send-template
```

**權限**: `NOTIFICATION_SEND`

**Request Body:**
```json
{
  "template_code": "ORDER_COMPLETED",
  "recipients": ["USER-001"],
  "channels": ["push", "email"],
  "template_data": {
    "order_id": "ORD-001",
    "customer_name": "王小明",
    "order_total": "250.00"
  },
  "scheduled_at": "2024-01-15T14:00:00Z"
}
```

#### 4.1.3 批量發送通知
```
POST /api/v1/notifications/send-bulk
```

**權限**: `NOTIFICATION_SEND`

**Request Body:**
```json
{
  "template_code": "PROMOTION_ALERT",
  "recipient_groups": ["all_customers", "gold_members"],
  "channels": ["push", "email"],
  "template_data": {
    "promotion_title": "週年慶特惠",
    "discount": "20%",
    "valid_until": "2024-01-31"
  },
  "send_immediately": false,
  "scheduled_at": "2024-01-20T09:00:00Z"
}
```

#### 4.1.4 發送系統警告
```
POST /api/v1/notifications/send-alert
```

**權限**: `NOTIFICATION_SEND`

**Request Body:**
```json
{
  "alert_type": "inventory_low",
  "severity": "warning",
  "title": "庫存不足警告",
  "message": "商品 [拿鐵咖啡豆] 庫存不足，目前剩餘 5 包",
  "recipients": ["MANAGER-001"],
  "data": {
    "product_id": "PROD-001",
    "current_stock": 5,
    "minimum_stock": 20,
    "store_id": "ST-001"
  }
}
```

### 4.2 通知查詢

#### 4.2.1 查詢使用者通知清單
```
GET /api/v1/notifications/users/{user_id}/notifications
```

**權限**: `NOTIFICATION_READ`

**Query Parameters:**
- `type` (string, optional): 通知類型篩選
- `status` (string, optional): 狀態篩選 (unread/read/all)
- `date_from` (date, optional): 開始日期
- `date_to` (date, optional): 結束日期
- `page` (integer, optional): 頁碼
- `limit` (integer, optional): 每頁筆數

**Response:**
```json
{
  "success": true,
  "data": {
    "notifications": [
      {
        "notification_id": "NOTIF-001",
        "title": "訂單完成通知",
        "message": "您的訂單 #ORD-001 已完成製作",
        "type": "order",
        "category": "info",
        "priority": "medium",
        "is_read": false,
        "created_at": "2024-01-15T10:30:00Z"
      }
    ],
    "unread_count": 5,
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 50,
      "total_pages": 3
    }
  }
}
```

#### 4.2.2 查詢通知詳情
```
GET /api/v1/notifications/{notification_id}
```

**權限**: `NOTIFICATION_READ`

#### 4.2.3 標記通知為已讀
```
PUT /api/v1/notifications/{notification_id}/read
```

**權限**: `NOTIFICATION_READ`

#### 4.2.4 批量標記已讀
```
PUT /api/v1/notifications/users/{user_id}/mark-read
```

**權限**: `NOTIFICATION_READ`

**Request Body:**
```json
{
  "notification_ids": ["NOTIF-001", "NOTIF-002"],
  "mark_all": false
}
```

### 4.3 通知範本管理

#### 4.3.1 查詢範本清單
```
GET /api/v1/notifications/templates
```

**權限**: `NOTIFICATION_READ`

**Query Parameters:**
- `type` (string, optional): 範本類型篩選
- `is_active` (boolean, optional): 是否啟用

**Response:**
```json
{
  "success": true,
  "data": {
    "templates": [
      {
        "template_id": "TPL-001",
        "template_name": "訂單完成通知",
        "template_code": "ORDER_COMPLETED",
        "type": "push",
        "is_active": true,
        "created_at": "2024-01-15T10:30:00Z"
      }
    ]
  }
}
```

#### 4.3.2 查詢範本詳情
```
GET /api/v1/notifications/templates/{template_id}
```

**權限**: `NOTIFICATION_READ`

**Response:**
```json
{
  "success": true,
  "data": {
    "template_id": "TPL-001",
    "template_name": "訂單完成通知",
    "template_code": "ORDER_COMPLETED",
    "description": "顧客訂單完成後的推播通知",
    "type": "push",
    "subject": "",
    "content": "您的訂單 #{order_id} 已完成製作，請至 {store_name} 取餐",
    "variables": ["order_id", "store_name", "customer_name"],
    "is_active": true,
    "version": "1.0"
  }
}
```

#### 4.3.3 建立通知範本
```
POST /api/v1/notifications/templates
```

**權限**: `NOTIFICATION_MANAGE`

**Request Body:**
```json
{
  "template_name": "庫存預警通知",
  "template_code": "INVENTORY_ALERT",
  "description": "商品庫存不足時的預警通知",
  "type": "email",
  "subject": "庫存預警 - {product_name}",
  "content": "商品 {product_name} 庫存不足，目前剩餘 {current_stock} 個，最低庫存為 {minimum_stock}",
  "variables": ["product_name", "current_stock", "minimum_stock"],
  "style_config": {
    "background_color": "#fff3cd",
    "text_color": "#856404"
  }
}
```

#### 4.3.4 更新通知範本
```
PUT /api/v1/notifications/templates/{template_id}
```

**權限**: `NOTIFICATION_MANAGE`

#### 4.3.5 刪除通知範本
```
DELETE /api/v1/notifications/templates/{template_id}
```

**權限**: `NOTIFICATION_MANAGE`

### 4.4 通知偏好設定

#### 4.4.1 查詢使用者通知偏好
```
GET /api/v1/notifications/users/{user_id}/preferences
```

**權限**: `NOTIFICATION_READ`

**Response:**
```json
{
  "success": true,
  "data": {
    "preferences": [
      {
        "preference_id": "PREF-001",
        "notification_type": "order",
        "channels": {
          "push": true,
          "email": false,
          "sms": false,
          "in_app": true
        },
        "quiet_hours": {
          "enabled": true,
          "start_time": "22:00",
          "end_time": "08:00"
        },
        "frequency": "immediate",
        "language": "zh-TW"
      }
    ]
  }
}
```

#### 4.4.2 更新通知偏好
```
PUT /api/v1/notifications/users/{user_id}/preferences
```

**權限**: `NOTIFICATION_READ`

**Request Body:**
```json
{
  "notification_type": "promotion",
  "channels": {
    "push": false,
    "email": true,
    "sms": false,
    "in_app": false
  },
  "quiet_hours": {
    "enabled": true,
    "start_time": "22:00",
    "end_time": "08:00"
  },
  "frequency": "weekly"
}
```

### 4.5 推播訂閱管理

#### 4.5.1 註冊推播訂閱
```
POST /api/v1/notifications/subscriptions
```

**權限**: `NOTIFICATION_READ`

**Request Body:**
```json
{
  "user_id": "USER-001",
  "device_token": "ABC123DEF456",
  "platform": "ios",
  "endpoint": "https://fcm.googleapis.com/fcm/send/ABC123"
}
```

#### 4.5.2 更新訂閱狀態
```
PUT /api/v1/notifications/subscriptions/{subscription_id}
```

**權限**: `NOTIFICATION_READ`

#### 4.5.3 取消訂閱
```
DELETE /api/v1/notifications/subscriptions/{subscription_id}
```

**權限**: `NOTIFICATION_READ`

### 4.6 發送記錄查詢

#### 4.6.1 查詢發送記錄
```
GET /api/v1/notifications/delivery-logs
```

**權限**: `NOTIFICATION_READ`

**Query Parameters:**
- `notification_id` (string, optional): 通知ID篩選
- `channel` (string, optional): 管道篩選
- `status` (string, optional): 狀態篩選
- `date_from` (date, optional): 開始日期
- `date_to` (date, optional): 結束日期

#### 4.6.2 查詢發送統計
```
GET /api/v1/notifications/delivery-stats
```

**權限**: `NOTIFICATION_READ`

**Query Parameters:**
- `date_from` (date, required): 開始日期
- `date_to` (date, required): 結束日期
- `group_by` (string, optional): 分組方式 (channel/type/day)

**Response:**
```json
{
  "success": true,
  "data": {
    "total_sent": 10000,
    "total_delivered": 9500,
    "total_failed": 500,
    "delivery_rate": 95.0,
    "by_channel": {
      "push": {
        "sent": 6000,
        "delivered": 5800,
        "delivery_rate": 96.7
      },
      "email": {
        "sent": 3000,
        "delivered": 2900,
        "delivery_rate": 96.7
      },
      "sms": {
        "sent": 1000,
        "delivered": 800,
        "delivery_rate": 80.0
      }
    }
  }
}
```

### 4.7 系統通知管理

#### 4.7.1 建立系統公告
```
POST /api/v1/notifications/system-announcements
```

**權限**: `NOTIFICATION_ADMIN`

**Request Body:**
```json
{
  "title": "系統維護通知",
  "content": "系統將於 2024/01/20 凌晨 2:00-4:00 進行維護",
  "type": "maintenance",
  "priority": "high",
  "target_audience": "all_users",
  "display_from": "2024-01-15T09:00:00Z",
  "display_until": "2024-01-20T04:00:00Z"
}
```

#### 4.7.2 查詢系統公告
```
GET /api/v1/notifications/system-announcements
```

**權限**: `NOTIFICATION_READ`

#### 4.7.3 測試通知發送
```
POST /api/v1/notifications/test
```

**權限**: `NOTIFICATION_ADMIN`

**Request Body:**
```json
{
  "channel": "email",
  "recipient": "test@example.com",
  "template_id": "TPL-001",
  "template_data": {
    "test_field": "測試資料"
  }
}
```

---

## 5. 錯誤代碼

### 通知相關錯誤
- `NOTIFICATION_001`: 通知範本不存在
- `NOTIFICATION_002`: 收件人資訊不完整
- `NOTIFICATION_003`: 通知發送失敗
- `NOTIFICATION_004`: 不支援的通知類型
- `NOTIFICATION_005`: 範本變數缺失
- `NOTIFICATION_006`: 發送頻率限制

### 服務提供商錯誤
- `PROVIDER_001`: 郵件服務異常
- `PROVIDER_002`: 簡訊服務異常
- `PROVIDER_003`: 推播服務異常
- `PROVIDER_004`: 服務配額不足

---

## 6. 業務規則

### 6.1 發送規則
- 尊重用戶勿擾時間設定
- 遵循發送頻率限制
- 優先級影響發送順序
- 失敗自動重試 (最多 3 次)

### 6.2 範本規則
- 範本變數必須完整提供
- 範本內容長度限制 (簡訊 160 字元)
- 範本版本管理

### 6.3 權限規則
- 系統管理員可發送所有類型通知
- 一般使用者僅可發送特定類型通知
- 批量發送需要特殊權限

---

## 7. 整合介面

### 7.1 第三方服務整合

#### 推播服務
- Firebase Cloud Messaging (Android)
- Apple Push Notification Service (iOS)
- Web Push API

#### 郵件服務
- SendGrid
- Amazon SES
- SMTP

#### 簡訊服務
- Twilio
- 台灣簡訊服務商

### 7.2 內部服務整合
- 客戶服務：查詢客戶聯絡資訊
- 訂單服務：接收訂單狀態更新
- 庫存服務：接收庫存預警
- 忠誠度服務：接收會員活動通知

---

## 8. 性能要求

### 8.1 回應時間
- 即時通知發送: < 500ms
- 批量通知處理: < 5s
- 通知查詢: < 200ms

### 8.2 吞吐量
- 支援每分鐘 10,000+ 通知發送
- 支援並發 1000+ API 請求

### 8.3 可靠性
- 99.9% 服務可用性
- 重要通知保證送達機制
- 發送失敗自動重試

---

## 9. 安全要求

### 9.1 資料保護
- 通知內容敏感資訊遮罩
- 個人聯絡資訊加密存儲
- API 存取日誌記錄

### 9.2 防濫用機制
- API 呼叫頻率限制
- 發送數量配額管理
- 異常發送行為監控

---

## 10. 測試案例

### 10.1 功能測試
- 各管道通知發送
- 範本系統功能
- 偏好設定功能
- 批量發送處理

### 10.2 整合測試
- 第三方服務整合
- 內部服務通訊
- 錯誤處理機制

### 10.3 負載測試
- 高併發發送
- 大量批量處理
- 系統穩定性測試

---

**版本**: v1.0  
**建立日期**: 2025-11-16  
**最後更新**: 2025-11-16  
**負責人**: 開發團隊