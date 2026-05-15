# Just Work 管理後台需求規格

## 概述

本文檔定義 Just Work 管理後台的完整功能需求。

---

## 1. AI 模型接入配置

### 1.1 功能描述

管理員可以配置 AI 模型接入參數，並測試 API 響應。

### 1.2 功能清單

| # | 功能 | 說明 |
|---|------|------|
| 1 | 模型列表 | 顯示所有已配置的 AI 模型 |
| 2 | 新增模型 | 新增 AI 模型配置 |
| 3 | 編輯模型 | 編輯現有模型配置 |
| 4 | 刪除模型 | 刪除模型配置 |
| 5 | 優先順序 | 設定模型使用優先順序 |
| 6 | API 測試 | 測試 API 提示文字與響應 |

### 1.3 模型配置欄位

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| 模型名稱 | String | 是 | 例如：GPT-4, Claude-3 |
| API URL | String | 是 | API 端點 |
| API Key | String | 是 | 認證金鑰 |
| 模型版本 | String | 是 | 例如：gpt-4-turbo |
| 優先順序 | Integer | 否 | 數字越小越優先 |
| 最大 Token | Integer | 否 | 預設 4096 |
| 超時時間 | Integer | 否 | 預設 60 秒 |
| 狀態 | Enum | 是 | 啟用/停用 |

### 1.4 API 測試介面

```
┌─────────────────────────────────────────────┐
│ 模型選擇: [GPT-4 ▼]                        │
├─────────────────────────────────────────────┤
│ 提示文字:                                    │
│ ┌─────────────────────────────────────────┐ │
│ │ 請用繁體中文回答以下問題...                │ │
│ └─────────────────────────────────────────┘ │
├─────────────────────────────────────────────┤
│ [測試] [清除]                              │
├─────────────────────────────────────────────┤
│ 回應:                                        │
│ ┌─────────────────────────────────────────┐ │
│ │ [API 響應會顯示在這裡]                    │ │
│ └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

---

## 2. 使用者管理

### 2.1 功能清單

| # | 功能 | 說明 |
|---|------|------|
| 1 | 使用者列表 | 顯示所有管理後台使用者 |
| 2 | 新增使用者 | 新增管理員帳號 |
| 3 | 編輯使用者 | 編輯使用者資料 |
| 4 | 刪除使用者 | 軟刪除使用者 |
| 5 | 重設密碼 | 重設使用者密碼 |
| 6 | 指派角色 | 指派使用者角色 |
| 7 | 帳號狀態 | 啟用/停用帳號 |

### 2.2 使用者欄位

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| 用戶名 | String | 是 | 登入用戶名 |
| Email | Email | 是 | 聯絡 Email |
| 密碼 | String | 是 | 首次建立時填寫 |
| 顯示名稱 | String | 否 | 顯示名稱 |
| 角色 | List | 是 | 指派角色 |
| 狀態 | Enum | 是 | 啟用/停用 |
| 最後登入 | DateTime | 否 | 最後登入時間 |

---

## 3. 角色管理

### 3.1 功能清單

| # | 功能 | 說明 |
|---|------|------|
| 1 | 角色列表 | 顯示所有角色 |
| 2 | 新增角色 | 建立新角色 |
| 3 | 編輯角色 | 編輯角色資料 |
| 4 | 刪除角色 | 刪除角色 |
| 5 | 指派權限 | 指派角色權限 |
| 6 | 指派選單 | 指派角色可見選單 |

### 3.2 角色欄位

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| 角色代碼 | String | 是 | 唯一識別碼 |
| 角色名稱 | String | 是 | 顯示名稱 |
| 說明 | String | 否 | 角色說明 |
| 系統內建 | Boolean | 否 | 是否系統內建 |
| 狀態 | Enum | 是 | 啟用/停用 |

---

## 4. 權限管理

### 4.1 功能清單

| # | 功能 | 說明 |
|---|------|------|
| 1 | 權限列表 | 顯示所有權限 |
| 2 | 新增權限 | 建立新權限 |
| 3 | 編輯權限 | 編輯權限資料 |
| 4 | 刪除權限 | 刪除權限 |

### 4.2 權限欄位

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| 權限代碼 | String | 是 | 唯一識別碼 |
| 權限名稱 | String | 是 | 顯示名稱 |
| 資源類型 | Enum | 是 | API / Menu / Button |
| 資源路徑 | String | 否 | API 路徑 |
| HTTP 方法 | String | 否 | GET / POST / PUT / DELETE |
| 排序 | Integer | 否 | 顯示排序 |
| 狀態 | Enum | 是 | 啟用/停用 |

---

## 5. 選單管理

### 5.1 功能清單

| # | 功能 | 說明 |
|---|------|------|
| 1 | 選單列表 | 樹狀顯示所有選單 |
| 2 | 新增選單 | 建立新選單 |
| 3 | 編輯選單 | 編輯選單資料 |
| 4 | 刪除選單 | 刪除選單 |
| 5 | 拖曳排序 | 調整選單順序 |

### 5.2 選單欄位

| 欄位 | 類型 | 必填 | 說明 |
|------|------|------|------|
| 選單名稱 | String | 是 | 顯示名稱 |
| 圖示 | String | 否 | 選單圖示 |
| 路徑 | String | 否 | 路由路徑 |
| 上層選單 | Tree | 否 | 父選單 |
| 排序 | Integer | 否 | 顯示排序 |
| 狀態 | Enum | 是 | 啟用/停用 |

---

## 6. 操作日誌

### 6.1 功能清單

| # | 功能 | 說明 |
|---|------|------|
| 1 | 日誌列表 | 顯示所有操作日誌 |
| 2 | 日誌詳情 | 查看日誌詳細內容 |
| 3 | 日誌匯出 | 匯出日誌 |
| 4 | 日誌清理 | 清理歷史日誌 |

### 6.2 日誌欄位

| 欄位 | 類型 | 說明 |
|------|------|------|
| 操作者 | String | 執行操作的使用者 |
| 操作類型 | Enum | 登入 / 新增 / 編輯 / 刪除 / 其他 |
| 操作模組 | String | 所屬模組 |
| 操作內容 | String | 詳細描述 |
| IP 位址 | IP | 來源 IP |
| 狀態 | Enum | 成功 / 失敗 |
| 建立時間 | DateTime | 記錄時間 |

---

## 7. 頁面權限對照

| 頁面 | 必要權限 |
|------|----------|
| AI 模型配置 | ai_model:read, ai_model:write |
| 使用者管理 | user:read, user:write |
| 角色管理 | role:read, role:write |
| 權限管理 | permission:read, permission:write |
| 選單管理 | menu:read, menu:write |
| 操作日誌 | log:read |
| 儀表板 | dashboard:read |

---

## 8. 資料庫 Schema

### 8.1 AI 模型表

```sql
CREATE TABLE ai_models (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL COMMENT '模型名稱',
    api_url VARCHAR(500) NOT NULL COMMENT 'API URL',
    api_key VARCHAR(500) NOT NULL COMMENT 'API Key',
    model_version VARCHAR(100) COMMENT '模型版本',
    priority INT DEFAULT 0 COMMENT '優先順序',
    max_tokens INT DEFAULT 4096 COMMENT '最大 Token',
    timeout INT DEFAULT 60 COMMENT '超時時間(秒)',
    status TINYINT DEFAULT 1 COMMENT '1:啟用, 0:停用',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_priority (priority),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 8.2 操作日誌表

```sql
CREATE TABLE admin_operation_logs (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED COMMENT '操作者ID',
    user_name VARCHAR(100) COMMENT '操作者名稱',
    action_type VARCHAR(50) NOT NULL COMMENT '操作類型',
    module VARCHAR(100) NOT NULL COMMENT '操作模組',
    description TEXT COMMENT '操作內容',
    ip_address VARCHAR(45) COMMENT 'IP 位址',
    status TINYINT DEFAULT 1 COMMENT '1:成功, 0:失敗',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user (user_id),
    INDEX idx_action (action_type),
    INDEX idx_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 9. API 端點

### 9.1 AI 模型

| 方法 | 路徑 | 說明 |
|------|------|------|
| GET | /api/admin/ai-models | 列表 |
| POST | /api/admin/ai-models | 建立 |
| GET | /api/admin/ai-models/:id | 詳情 |
| PUT | /api/admin/ai-models/:id | 更新 |
| DELETE | /api/admin/ai-models/:id | 刪除 |
| POST | /api/admin/ai-models/:id/test | API 測試 |

### 9.2 使用者

| 方法 | 路徑 | 說明 |
|------|------|------|
| GET | /api/admin/users | 列表 |
| POST | /api/admin/users | 建立 |
| GET | /api/admin/users/:id | 詳情 |
| PUT | /api/admin/users/:id | 更新 |
| DELETE | /api/admin/users/:id | 刪除 |
| POST | /api/admin/users/:id/roles | 指派角色 |
| POST | /api/admin/users/:id/password | 重設密碼 |

### 9.3 角色

| 方法 | 路徑 | 說明 |
|------|------|------|
| GET | /api/admin/roles | 列表 |
| POST | /api/admin/roles | 建立 |
| GET | /api/admin/roles/:id | 詳情 |
| PUT | /api/admin/roles/:id | 更新 |
| DELETE | /api/admin/roles/:id | 刪除 |
| GET | /api/admin/roles/:id/permissions | 取得權限 |
| PUT | /api/admin/roles/:id/permissions | 更新權限 |
| GET | /api/admin/roles/:id/menus | 取得選單 |
| PUT | /api/admin/roles/:id/menus | 更新選單 |

### 9.4 權限

| 方法 | 路徑 | 說明 |
|------|------|------|
| GET | /api/admin/permissions | 列表 |
| POST | /api/admin/permissions | 建立 |
| GET | /api/admin/permissions/:id | 詳情 |
| PUT | /api/admin/permissions/:id | 更新 |
| DELETE | /api/admin/permissions/:id | 刪除 |

### 9.5 選單

| 方法 | 路徑 | 說明 |
|------|------|------|
| GET | /api/admin/menus | 樹狀列表 |
| POST | /api/admin/menus | 建立 |
| GET | /api/admin/menus/:id | 詳情 |
| PUT | /api/admin/menus/:id | 更新 |
| DELETE | /api/admin/menus/:id | 刪除 |

### 9.6 操作日誌

| 方法 | 路徑 | 說明 |
|------|------|------|
| GET | /api/admin/logs | 列表 |
| GET | /api/admin/logs/:id | 詳情 |
| GET | /api/admin/logs/export | 匯出 |

---

## 10. 文件版本

**文件版本**: 0.1.0.0.0
**最後更新**: 2026-02-14
**維護團隊**: Admin Team

---

## 參考文件

- [後端技術規格](../backend/TECHNICAL_SPECIFICATION.md)
- [後端開發原則](../backend/DEVELOPMENT.md)
- [管理後台開發原則](./DEVELOPMENT.md)
