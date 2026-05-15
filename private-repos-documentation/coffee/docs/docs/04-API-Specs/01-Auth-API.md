# Auth API 規格

## 1. 概述

Auth Service 負責系統的使用者認證與授權，包括使用者註冊、登入、JWT Token 驗證等功能。

**服務資訊**:
- **服務名稱**: Auth Service
- **服務埠號**: 8081
- **資料庫**: coffee_auth (MySQL)
- **API 版本**: v1
- **Base URL**: `http://localhost:8080/api/v1/auth` (透過 API Gateway)

---

## 2. 認證機制

### 2.1 JWT Token 規格

**Token 格式**: Bearer Token  
**有效期**: 24 小時  
**簽發者**: auth-service  
**演算法**: HS256

**Payload 結構**:
```json
{
  "sub": "user_id",
  "email": "user@example.com",
  "name": "使用者姓名",
  "role": "admin|headquarters|store_manager|staff",
  "store_id": "store_uuid_or_null",
  "iat": 1634567890,
  "exp": 1634654290,
  "iss": "auth-service"
}
```

### 2.2 權限角色

| 角色 | 英文 | 描述 | 權限範圍 |
|------|------|------|----------|
| 系統管理員 | admin | 系統最高權限 | 全系統管理 |
| 總部管理 | headquarters | 總部管理人員 | ERP 系統存取 |
| 店長 | store_manager | 門店店長 | 門店完整管理 |
| 店員 | staff | 門店店員 | POS 系統操作 |

---

## 3. API 端點

### 3.1 使用者註冊

**端點**: `POST /api/v1/auth/register`  
**描述**: 註冊新使用者帳號  
**權限**: 需要 admin 角色

#### Request

**Headers**:
```http
Content-Type: application/json
Authorization: Bearer <jwt_token>  # admin 角色必須
```

**Body**:
```json
{
  "email": "user@example.com",
  "password": "SecurePassword123!",
  "name": "張三",
  "role": "staff",
  "store_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

**Body Schema**:
| 欄位 | 類型 | 必須 | 限制 | 描述 |
|------|------|------|------|------|
| email | string | ✓ | 有效 Email 格式 | 登入帳號 |
| password | string | ✓ | 8-50 字符，包含大小寫字母、數字、特殊符號 | 登入密碼 |
| name | string | ✓ | 2-50 字符 | 使用者姓名 |
| role | string | ✓ | admin\|headquarters\|store_manager\|staff | 使用者角色 |
| store_id | string | 條件 | UUID 格式，role=staff/store_manager 時必須 | 所屬門店 ID |

#### Response

**成功 (201 Created)**:
```json
{
  "success": true,
  "message": "使用者註冊成功",
  "data": {
    "user": {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "email": "user@example.com",
      "name": "張三",
      "role": "staff",
      "store_id": "550e8400-e29b-41d4-a716-446655440000",
      "status": "active",
      "created_at": "2025-11-16T10:30:00Z"
    }
  }
}
```

**錯誤回應**:
```json
{
  "success": false,
  "error_code": "EMAIL_ALREADY_EXISTS",
  "message": "此 Email 已被註冊",
  "details": {
    "field": "email",
    "value": "user@example.com"
  }
}
```

**錯誤代碼**:
| 錯誤代碼 | HTTP 狀態 | 描述 |
|----------|-----------|------|
| EMAIL_ALREADY_EXISTS | 409 | Email 已被註冊 |
| INVALID_EMAIL_FORMAT | 400 | Email 格式無效 |
| WEAK_PASSWORD | 400 | 密碼強度不足 |
| INVALID_ROLE | 400 | 角色無效 |
| STORE_NOT_FOUND | 400 | 門店不存在 |
| UNAUTHORIZED | 401 | 無權限執行操作 |

---

### 3.2 使用者登入

**端點**: `POST /api/v1/auth/login`  
**描述**: 使用者登入取得 JWT Token  
**權限**: 無需認證

#### Request

**Headers**:
```http
Content-Type: application/json
```

**Body**:
```json
{
  "email": "user@example.com",
  "password": "SecurePassword123!"
}
```

**Body Schema**:
| 欄位 | 類型 | 必須 | 描述 |
|------|------|------|------|
| email | string | ✓ | 登入 Email |
| password | string | ✓ | 登入密碼 |

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "登入成功",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expires_at": "2025-11-17T10:30:00Z",
    "user": {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "email": "user@example.com",
      "name": "張三",
      "role": "staff",
      "store_id": "550e8400-e29b-41d4-a716-446655440000",
      "status": "active",
      "last_login_at": "2025-11-16T10:30:00Z"
    }
  }
}
```

**錯誤回應**:
```json
{
  "success": false,
  "error_code": "INVALID_CREDENTIALS",
  "message": "帳號或密碼錯誤",
  "details": null
}
```

**錯誤代碼**:
| 錯誤代碼 | HTTP 狀態 | 描述 |
|----------|-----------|------|
| INVALID_CREDENTIALS | 401 | 帳號或密碼錯誤 |
| USER_LOCKED | 423 | 帳號被鎖定 |
| USER_INACTIVE | 403 | 帳號未啟用 |
| TOO_MANY_ATTEMPTS | 429 | 登入嘗試次數過多 |

---

### 3.3 Token 驗證

**端點**: `POST /api/v1/auth/verify`  
**描述**: 驗證 JWT Token 有效性  
**權限**: 需要有效 Token

#### Request

**Headers**:
```http
Authorization: Bearer <jwt_token>
```

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "Token 有效",
  "data": {
    "user": {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "email": "user@example.com",
      "name": "張三",
      "role": "staff",
      "store_id": "550e8400-e29b-41d4-a716-446655440000",
      "status": "active"
    },
    "expires_at": "2025-11-17T10:30:00Z"
  }
}
```

**錯誤回應**:
```json
{
  "success": false,
  "error_code": "INVALID_TOKEN",
  "message": "Token 無效或已過期",
  "details": null
}
```

**錯誤代碼**:
| 錯誤代碼 | HTTP 狀態 | 描述 |
|----------|-----------|------|
| INVALID_TOKEN | 401 | Token 無效或已過期 |
| TOKEN_REQUIRED | 401 | 缺少 Token |

---

### 3.4 密碼修改

**端點**: `PUT /api/v1/auth/password`  
**描述**: 修改使用者密碼  
**權限**: 需要有效 Token (使用者只能修改自己密碼，admin 可修改任何人)

#### Request

**Headers**:
```http
Content-Type: application/json
Authorization: Bearer <jwt_token>
```

**Body**:
```json
{
  "current_password": "OldPassword123!",
  "new_password": "NewSecurePassword456!"
}
```

**Body Schema**:
| 欄位 | 類型 | 必須 | 限制 | 描述 |
|------|------|------|------|------|
| current_password | string | ✓ | - | 目前密碼 |
| new_password | string | ✓ | 8-50 字符，包含大小寫字母、數字、特殊符號 | 新密碼 |

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "密碼修改成功",
  "data": null
}
```

**錯誤回應**:
```json
{
  "success": false,
  "error_code": "INVALID_CURRENT_PASSWORD",
  "message": "目前密碼錯誤",
  "details": null
}
```

**錯誤代碼**:
| 錯誤代碼 | HTTP 狀態 | 描述 |
|----------|-----------|------|
| INVALID_CURRENT_PASSWORD | 400 | 目前密碼錯誤 |
| WEAK_PASSWORD | 400 | 新密碼強度不足 |
| PASSWORD_SAME_AS_CURRENT | 400 | 新密碼與目前密碼相同 |

---

### 3.5 使用者資料查詢

**端點**: `GET /api/v1/auth/users`  
**描述**: 查詢使用者列表  
**權限**: admin 或 store_manager

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
| role | string | ✗ | all | 角色篩選 |
| store_id | string | ✗ | - | 門店篩選 (store_manager 只能查自己門店) |
| status | string | ✗ | all | 狀態篩選 |

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "查詢成功",
  "data": {
    "users": [
      {
        "id": "123e4567-e89b-12d3-a456-426614174000",
        "email": "user@example.com",
        "name": "張三",
        "role": "staff",
        "store_id": "550e8400-e29b-41d4-a716-446655440000",
        "status": "active",
        "created_at": "2025-11-16T10:30:00Z",
        "last_login_at": "2025-11-16T10:30:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 1,
      "total_pages": 1
    }
  }
}
```

---

### 3.6 使用者狀態管理

**端點**: `PATCH /api/v1/auth/users/{user_id}/status`  
**描述**: 啟用/停用/鎖定使用者  
**權限**: admin 角色

#### Request

**Headers**:
```http
Content-Type: application/json
Authorization: Bearer <jwt_token>
```

**Path Parameters**:
| 參數 | 類型 | 描述 |
|------|------|------|
| user_id | string | 使用者 UUID |

**Body**:
```json
{
  "status": "inactive",
  "reason": "離職"
}
```

**Body Schema**:
| 欄位 | 類型 | 必須 | 值域 | 描述 |
|------|------|------|------|------|
| status | string | ✓ | active\|inactive\|locked | 新狀態 |
| reason | string | ✗ | 最多 255 字符 | 變更原因 |

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "使用者狀態更新成功",
  "data": {
    "user_id": "123e4567-e89b-12d3-a456-426614174000",
    "old_status": "active",
    "new_status": "inactive",
    "updated_at": "2025-11-16T10:30:00Z"
  }
}
```

---

### 3.7 登出

**端點**: `POST /api/v1/auth/logout`  
**描述**: 使用者登出 (可選實現 Token 黑名單)  
**權限**: 需要有效 Token

#### Request

**Headers**:
```http
Authorization: Bearer <jwt_token>
```

#### Response

**成功 (200 OK)**:
```json
{
  "success": true,
  "message": "登出成功",
  "data": null
}
```

---

## 4. 通用回應格式

### 4.1 成功回應

```json
{
  "success": true,
  "message": "操作成功訊息",
  "data": {
    // 實際回應資料
  }
}
```

### 4.2 錯誤回應

```json
{
  "success": false,
  "error_code": "ERROR_CODE",
  "message": "錯誤訊息",
  "details": {
    // 錯誤詳細資訊 (可選)
  }
}
```

### 4.3 通用錯誤代碼

| 錯誤代碼 | HTTP 狀態 | 描述 |
|----------|-----------|------|
| INVALID_REQUEST | 400 | 請求參數無效 |
| UNAUTHORIZED | 401 | 未授權 |
| FORBIDDEN | 403 | 權限不足 |
| NOT_FOUND | 404 | 資源不存在 |
| VALIDATION_ERROR | 422 | 資料驗證失敗 |
| INTERNAL_ERROR | 500 | 內部伺服器錯誤 |

---

## 5. 安全考量

### 5.1 密碼安全
- 密碼使用 bcrypt 雜湊，成本因子 12
- 密碼強度要求：8-50 字符，必須包含大小寫字母、數字、特殊符號
- 登入失敗 5 次後鎖定帳號 15 分鐘

### 5.2 Token 安全
- JWT Secret 使用 256 位元隨機金鑰
- Token 有效期 24 小時，不自動續期
- 建議實現 Token 黑名單機制

### 5.3 API 安全
- 所有 API 實施 Rate Limiting: 每分鐘 60 次請求
- 登入 API 額外限制：每分鐘 5 次請求
- 使用 HTTPS 加密傳輸

---

## 6. 測試案例

### 6.1 註冊測試
```bash
# 正常註冊
curl -X POST http://localhost:8080/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <admin_token>" \
  -d '{
    "email": "test@example.com",
    "password": "Test123!@#",
    "name": "測試使用者",
    "role": "staff",
    "store_id": "550e8400-e29b-41d4-a716-446655440000"
  }'

# 重複 Email 測試
curl -X POST http://localhost:8080/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <admin_token>" \
  -d '{
    "email": "test@example.com",
    "password": "Test456!@#",
    "name": "重複測試",
    "role": "staff",
    "store_id": "550e8400-e29b-41d4-a716-446655440000"
  }'
```

### 6.2 登入測試
```bash
# 正常登入
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "Test123!@#"
  }'

# 錯誤密碼
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "WrongPassword"
  }'
```

### 6.3 Token 驗證測試
```bash
curl -X POST http://localhost:8080/api/v1/auth/verify \
  -H "Authorization: Bearer <jwt_token>"
```

---

## 7. 版本歷程

| 版本 | 日期 | 作者 | 說明 |
|------|------|------|------|
| 1.0 | 2025-11-16 | System | 初始 Auth API 規格建立 |

---

**備註**: 此 API 規格遵循 OpenAPI 3.0 標準，可用於生成 Swagger 文件和客戶端代碼。