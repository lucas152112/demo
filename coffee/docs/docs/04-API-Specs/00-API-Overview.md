# API 總覽

## 1. 概述

咖啡館連鎖管理系統採用微服務架構，所有 API 通過 API Gateway 統一提供服務。本文件說明整體 API 架構、認證機制、通用規範等。

**系統架構**:
- **API Gateway**: 統一入口，Port 8080
- **微服務**: 7 個業務微服務
- **認證方式**: JWT Bearer Token
- **協定**: HTTP/HTTPS + JSON
- **版本**: v1

---

## 2. 微服務列表

| 服務名稱 | 端點前綴 | 內部埠號 | 職責 |
|----------|----------|----------|------|
| **Auth Service** | `/api/v1/auth` | 8081 | 使用者認證與授權 |
| **Customers Service** | `/api/v1/customers` | 8082 | 客戶資料管理 |
| **Orders Service** | `/api/v1/orders` | 8083 | 訂單管理 |
| **Inventory Service** | `/api/v1/inventory` | 8084 | 庫存與商品管理 |
| **Loyalty Service** | `/api/v1/loyalty` | 8085 | 會員忠誠度管理 |
| **Notifications Service** | `/api/v1/notifications` | 8086 | 通知服務 |
| **Reports Service** | `/api/v1/reports` | 8087 | 報表與分析 |

**統一存取點**: `http://localhost:8080` (API Gateway)

---

## 3. 認證與授權

### 3.1 JWT Token 認證

所有需要認證的 API 都使用 JWT Bearer Token：

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Token 取得**: 透過 `POST /api/v1/auth/login` 取得  
**Token 有效期**: 24 小時  
**Token 格式**: JWT (JSON Web Token)

### 3.2 權限角色

| 角色 | 英文代碼 | 權限範圍 |
|------|----------|----------|
| 系統管理員 | admin | 全系統管理權限 |
| 總部管理 | headquarters | ERP 系統完整權限 |
| 店長 | store_manager | 門店完整管理權限 |
| 店員 | staff | POS 系統操作權限 |

---

## 4. 通用 API 規範

### 4.1 HTTP 方法

| 方法 | 用途 | 範例 |
|------|------|------|
| GET | 查詢資源 | `GET /api/v1/customers` |
| POST | 建立資源 | `POST /api/v1/customers` |
| PUT | 完整更新 | `PUT /api/v1/customers/123` |
| PATCH | 部分更新 | `PATCH /api/v1/customers/123` |
| DELETE | 刪除資源 | `DELETE /api/v1/customers/123` |

### 4.2 請求格式

**Headers**:
```http
Content-Type: application/json
Authorization: Bearer <token>  # 需要認證時
Accept: application/json
```

**JSON Body** (POST/PUT/PATCH):
```json
{
  "field1": "value1",
  "field2": "value2"
}
```

### 4.3 回應格式

#### 成功回應
```json
{
  "success": true,
  "message": "操作成功訊息",
  "data": {
    // 實際回應資料
  }
}
```

#### 錯誤回應
```json
{
  "success": false,
  "error_code": "ERROR_CODE",
  "message": "使用者友善的錯誤訊息",
  "details": {
    // 詳細錯誤資訊 (可選)
  }
}
```

### 4.4 分頁查詢

**Query Parameters**:
```http
GET /api/v1/customers?page=1&limit=20&sort=created_at&order=desc
```

**回應格式**:
```json
{
  "success": true,
  "message": "查詢成功",
  "data": {
    "items": [...],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 100,
      "total_pages": 5,
      "has_next": true,
      "has_prev": false
    }
  }
}
```

---

## 5. HTTP 狀態碼

| 狀態碼 | 意義 | 使用場景 |
|--------|------|----------|
| 200 | OK | 請求成功 |
| 201 | Created | 資源建立成功 |
| 204 | No Content | 刪除成功 |
| 400 | Bad Request | 請求參數錯誤 |
| 401 | Unauthorized | 未認證或 Token 無效 |
| 403 | Forbidden | 權限不足 |
| 404 | Not Found | 資源不存在 |
| 409 | Conflict | 資源衝突 (如重複建立) |
| 422 | Unprocessable Entity | 資料驗證失敗 |
| 429 | Too Many Requests | 請求頻率過高 |
| 500 | Internal Server Error | 內部伺服器錯誤 |

---

## 6. 通用錯誤代碼

| 錯誤代碼 | HTTP 狀態 | 描述 |
|----------|-----------|------|
| **認證錯誤** |
| INVALID_TOKEN | 401 | Token 無效或已過期 |
| TOKEN_REQUIRED | 401 | 缺少 Token |
| UNAUTHORIZED | 401 | 未授權 |
| FORBIDDEN | 403 | 權限不足 |
| **請求錯誤** |
| INVALID_REQUEST | 400 | 請求格式錯誤 |
| VALIDATION_ERROR | 422 | 資料驗證失敗 |
| RESOURCE_NOT_FOUND | 404 | 資源不存在 |
| RESOURCE_CONFLICT | 409 | 資源衝突 |
| **系統錯誤** |
| RATE_LIMIT_EXCEEDED | 429 | 請求頻率過高 |
| INTERNAL_ERROR | 500 | 內部伺服器錯誤 |
| SERVICE_UNAVAILABLE | 503 | 服務暫時不可用 |

---

## 7. API 安全規範

### 7.1 Rate Limiting

| API 類型 | 限制 |
|----------|------|
| 一般 API | 每分鐘 60 次請求 |
| 登入 API | 每分鐘 5 次請求 |
| 註冊 API | 每分鐘 3 次請求 |
| 密碼重設 | 每小時 3 次請求 |

### 7.2 資料驗證

- 所有輸入資料都進行嚴格驗證
- 輸出資料進行 XSS 過濾
- SQL Injection 防護
- CORS 設定：僅允許前端網域

### 7.3 HTTPS 要求

- 生產環境強制使用 HTTPS
- API Gateway 處理 SSL 終止
- 敏感資料傳輸加密

---

## 8. API 測試

### 8.1 測試環境

| 環境 | Base URL | 描述 |
|------|----------|------|
| 開發 | `http://localhost:8080` | 本地開發 |
| 測試 | `https://api-test.coffee.com` | 測試環境 |
| 生產 | `https://api.coffee.com` | 生產環境 |

### 8.2 測試工具

**推薦工具**:
- **Postman**: API 測試與文件
- **curl**: 命令列測試
- **HTTPie**: 友善的 CLI HTTP 客戶端

**範例測試**:
```bash
# 登入取得 Token
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@coffee.com", "password": "Admin123!"}'

# 使用 Token 查詢客戶
curl -X GET http://localhost:8080/api/v1/customers \
  -H "Authorization: Bearer <token>"
```

---

## 9. API 文件

| 服務 | 文件連結 |
|------|----------|
| **Auth API** | [01-Auth-API.md](./01-Auth-API.md) |
| **Customers API** | [02-Customers-API.md](./02-Customers-API.md) |
| **Orders API** | [03-Orders-API.md](./03-Orders-API.md) |
| **Inventory API** | [04-Inventory-API.md](./04-Inventory-API.md) |
| **Loyalty API** | [05-Loyalty-API.md](./05-Loyalty-API.md) |
| **Notifications API** | [06-Notifications-API.md](./06-Notifications-API.md) |
| **Reports API** | [07-Reports-API.md](./07-Reports-API.md) |

---

## 10. 開發指南

### 10.1 API 設計原則

1. **RESTful 設計**: 遵循 REST 原則
2. **一致性**: 統一的命名與格式
3. **可預測性**: 清晰的 URL 結構
4. **向後相容**: API 版本管理
5. **安全性**: 認證、授權、資料驗證

### 10.2 命名規範

**URL 路徑**: 使用小寫，單詞間用 `-` 分隔
```
GET /api/v1/customers
GET /api/v1/loyalty/point-history
POST /api/v1/orders/bulk-create
```

**JSON 欄位**: 使用 snake_case
```json
{
  "user_id": "123",
  "created_at": "2025-11-16T10:30:00Z",
  "full_name": "張三"
}
```

**查詢參數**: 使用 snake_case
```
GET /api/v1/orders?start_date=2025-01-01&end_date=2025-12-31
```

---

## 11. 版本管理

### 11.1 版本策略

- **URL 版本**: `/api/v1/`, `/api/v2/`
- **向後相容**: 舊版本持續支援 6 個月
- **廢棄通知**: 提前 3 個月通知

### 11.2 變更類型

| 變更類型 | 版本影響 | 範例 |
|----------|----------|------|
| 新增欄位 | 次版本 | 回應增加新欄位 |
| 新增端點 | 次版本 | 新增 API 端點 |
| 修改欄位 | 主版本 | 欄位類型變更 |
| 刪除欄位 | 主版本 | 移除回應欄位 |

---

## 12. 版本歷程

| 版本 | 日期 | 作者 | 說明 |
|------|------|------|------|
| 1.0 | 2025-11-16 | System | 初始 API 總覽建立 |

---

**下一步**: 請參閱各個服務的詳細 API 規格文件