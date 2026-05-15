# Just Work 管理後台開發原則

本文檔定義 Just Work 管理後台的開發原則、規範和流程。

---

## 1. 概述

管理後台是系統的核心管理模組，提供以下功能：

- 用戶管理
- 角色與權限管理
- 系統配置
- 服務監控
- 審計日誌

---

## 2. 技術架構

### 2.1 技術棧

| 層次 | 技術 |
|------|------|
| 框架 | Rust + Axum |
| API 文件 | Swagger/OpenAPI |
| 認證 | JWT + RBAC |
| 資料庫 | MySQL + Redis |

### 2.2 專案結構

```
admin/
├── src/
│   ├── main.rs
│   ├── lib.rs
│   ├── routes/
│   │   ├── mod.rs
│   │   ├── auth.rs
│   │   ├── users.rs
│   │   ├── roles.rs
│   │   ├── permissions.rs
│   │   ├── services.rs
│   │   └── logs.rs
│   ├── models/
│   │   ├── mod.rs
│   │   ├── user.rs
│   │   ├── role.rs
│   │   ├── permission.rs
│   │   └── service.rs
│   ├── services/
│   │   ├── mod.rs
│   │   ├── user_service.rs
│   │   ├── role_service.rs
│   │   └── system_service.rs
│   ├── middleware/
│   │   ├── mod.rs
│   │   ├── auth.rs
│   │   └── rbac.rs
│   └── config/
│       └── mod.rs
├── tests/
├── Cargo.toml
└── config.yaml
```

---

## 3. RBAC 權限模型

### 3.1 資料模型

```
用戶 (User)
  ├── 用戶角色 (UserRole)
  └── 用戶服務 (UserService)

角色 (Role)
  ├── 角色權限 (RolePermission)
  └── 角色選單 (RoleMenu)

權限 (Permission)
  └── 權限類型 (PermissionType)

選單 (Menu)
  └── 父子選單 (MenuHierarchy)
```

### 3.2 ER 圖

```sql
-- 用戶表
CREATE TABLE admin_users (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    uuid VARCHAR(36) NOT NULL UNIQUE,
    username VARCHAR(100) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    status TINYINT DEFAULT 1 COMMENT '1: active, 0: inactive',
    last_login_at DATETIME,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- 角色表
CREATE TABLE admin_roles (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    code VARCHAR(50) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    description VARCHAR(500),
    is_system TINYINT DEFAULT 0 COMMENT '系統內建角色',
    status TINYINT DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 權限表
CREATE TABLE admin_permissions (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    code VARCHAR(100) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    resource_type VARCHAR(50) COMMENT 'api, menu, button',
    resource_path VARCHAR(255),
    http_method VARCHAR(10),
    sort_order INT DEFAULT 0,
    status TINYINT DEFAULT 1
);

-- 選單表
CREATE TABLE admin_menus (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    icon VARCHAR(100),
    path VARCHAR(255),
    parent_id BIGINT UNSIGNED,
    sort_order INT DEFAULT 0,
    status TINYINT DEFAULT 1
);

-- 用戶-角色關聯
CREATE TABLE admin_user_roles (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT UNSIGNED NOT NULL,
    role_id BIGINT UNSIGNED NOT NULL,
    assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    assigned_by BIGINT UNSIGNED,
    UNIQUE KEY uk_user_role (user_id, role_id)
);

-- 角色-權限關聯
CREATE TABLE admin_role_permissions (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    role_id BIGINT UNSIGNED NOT NULL,
    permission_id BIGINT UNSIGNED NOT NULL,
    UNIQUE KEY uk_role_permission (role_id, permission_id)
);

-- 角色-選單關聯
CREATE TABLE admin_role_menus (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    role_id BIGINT UNSIGNED NOT NULL,
    menu_id BIGINT UNSIGNED NOT NULL,
    can_operate TINYINT DEFAULT 0,
    UNIQUE KEY uk_role_menu (role_id, menu_id)
);
```

---

## 4. API 設計

### 4.1 API 清單

```
認證模組
├── POST   /api/admin/auth/login          # 登入
├── POST   /api/admin/auth/logout         # 登出
├── POST   /api/admin/auth/refresh       # 刷新 Token
└── GET    /api/admin/auth/profile       # 取得個人資料

用戶管理
├── GET    /api/admin/users              # 列表
├── GET    /api/admin/users/:id          # 詳情
├── POST   /api/admin/users              # 建立
├── PUT    /api/admin/users/:id          # 更新
├── DELETE /api/admin/users/:id          # 刪除
├── POST   /api/admin/users/:id/roles    # 指派角色
└── POST   /api/admin/users/:id/reset-pwd # 重設密碼

角色管理
├── GET    /api/admin/roles              # 列表
├── GET    /api/admin/roles/:id          # 詳情
├── POST   /api/admin/roles              # 建立
├── PUT    /api/admin/roles/:id          # 更新
├── DELETE /api/admin/roles/:id          # 刪除
├── GET    /api/admin/roles/:id/permissions  # 取得角色權限
└── PUT    /api/admin/roles/:id/permissions  # 更新角色權限

權限管理
├── GET    /api/admin/permissions        # 列表
├── POST   /api/admin/permissions        # 建立
├── PUT    /api/admin/permissions/:id    # 更新
└── DELETE /api/admin/permissions/:id    # 刪除

選單管理
├── GET    /api/admin/menus              # 樹狀列表
├── GET    /api/admin/menus/:id          # 詳情
├── POST   /api/admin/menus              # 建立
├── PUT    /api/admin/menus/:id          # 更新
└── DELETE /api/admin/menus/:id          # 刪除

服務管理
├── GET    /api/admin/services           # 列表
├── GET    /api/admin/services/:id      # 詳情
├── PUT    /api/admin/services/:id/status # 更新狀態
└── GET    /api/admin/services/:id/logs # 服務日誌

審計日誌
├── GET    /api/admin/logs               # 列表
├── GET    /api/admin/logs/:id          # 詳情
└── GET    /api/admin/logs/export       # 匯出
```

### 4.2 響應格式

```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "request_id": "uuid",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

---

## 5. 開發規範

### 5.1 代碼結構

```rust
// routes/users.rs

/// 用戶列表
#[utoipa::path(
    get,
    path = "/api/admin/users",
    responses(
        (status = 200, description = "Success", body = PaginatedResponse<UserResponse>),
        (status = 401, description = "Unauthorized"),
        (status = 403, description = "Forbidden"),
    ),
    security(
        ("bearerAuth" = [])
    ),
    tag = "users",
)]
pub async fn list_users(
    State(state): State<AppState>,
    Query(params): Query<UserQuery>,
    Extension(user): Extension<AdminUser>,
) -> Result<Json<PaginatedResponse<UserResponse>>, AppError> {
    // 1. 權限檢查
    user.require_permission(Permission::ListUsers)?;
    
    // 2. 查詢資料
    let (users, total) = UserService::list(&state.db, &params).await?;
    
    // 3. 響應
    Ok(Json(PaginatedResponse {
        data: users.into_iter().map(UserResponse::from).collect(),
        total,
        page: params.page,
        page_size: params.page_size,
    }))
}
```

### 5.2 錯誤處理

```rust
#[derive(Debug, thiserror::Error)]
pub enum AdminError {
    #[error("Unauthorized: {message}")]
    Unauthorized { message: String },
    
    #[error("Forbidden: {message}")]
    Forbidden { message: String },
    
    #[error("Validation failed: {field} - {reason}")]
    Validation { field: String, reason: String },
    
    #[error("Not found: {resource}")]
    NotFound { resource: String },
    
    #[error("Conflict: {message}")]
    Conflict { message: String },
}
```

---

## 6. RBAC 中間件

```rust
// middleware/rbac.rs

pub async fn rbac_middleware(
    req: Request,
    next: Next,
) -> Result<Response, AdminError> {
    let path = req.uri().path();
    let method = req.method().as_str();
    
    // 跳過認證檢查的路徑
    if is_public_path(path) {
        return Ok(next.run(req).await);
    }
    
    // 取得當前用戶
    let user: AdminUser = req
        .extensions()
        .get()
        .ok_or(AdminError::Unauthorized {
            message: "Not authenticated".to_string(),
        })?
        .clone();
    
    // 檢查權限
    let required_permission = extract_permission(path, method);
    
    if !user.has_permission(&required_permission) {
        return Err(AdminError::Forbidden {
            message: format!("Missing permission: {}", required_permission),
        });
    }
    
    Ok(next.run(req).await)
}

fn extract_permission(path: &str, method: &str) -> String {
    // 從路徑提取資源
    let resource = path
        .trim_start_matches("/api/admin/")
        .replace(['/', ':'], "_");
    
    format!("{}:{}", resource.to_uppercase(), method.to_uppercase())
}
```

---

## 7. 服務狀態管理

```rust
// 服務狀態
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ServiceStatus {
    Online,      // 上架
    Offline,     // 下架
    Stopped,     // 停止
    Deprecated,  // 廢棄
}

// 狀態檢查
fn can_operate_service(user: &AdminUser, service: &Service) -> bool {
    // 只有管理員可以操作
    user.role.code == "admin"
}

// 熱更新條件
async fn check_hot_reload(service: &Service) -> bool {
    service.status == ServiceStatus::Online
}
```

---

## 8. 測試要求

### 8.1 測試覆蓋率

| 測試類型 | 目標 |
|----------|------|
| 單元測試 | ≥ 80% |
| 整合測試 | ≥ 70% |
| API 測試 | 100% 核心流程 |

### 8.2 測試範例

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::tests::{setup, TestContext};
    
    #[tokio::test]
    async fn test_create_user() {
        let ctx = setup().await;
        
        let request = CreateUserRequest {
            username: "test_user",
            email: "test@example.com",
            password: "password123",
            role_ids: vec![1],
        };
        
        let response = create_user(&ctx.state, &request, &ctx.admin_user)
            .await
            .unwrap();
        
        assert_eq!(response.username, "test_user");
    }
    
    #[tokio::test]
    async fn test_unauthorized_access() {
        let ctx = setup().await;
        
        let response = list_users(&ctx.state, QueryParams::default())
            .await;
            
        assert!(response.is_err());
        assert!(matches!(response.unwrap_err(), AdminError::Unauthorized { .. }));
    }
}
```

---

## 9. Git 分支策略

### 9.1 分支結構

```
develop
├── feature/admin-user-management
├── feature/admin-role-permission
├── feature/admin-service-monitor
└── fix/admin-login-issue

test
├── feature/admin-user-management
├── ...

pp
└── ...

prod
└── hotfix/admin-security-fix
```

### 9.2 合併規則

| 來源 | 目標 | 說明 |
|------|------|------|
| feature/* | develop | 新功能 |
| develop | test | 功能完成 |
| test | pp | 測試通過 |
| pp | prod | 預發布驗證 |
| hotfix/* | prod | 緊急修復 |

---

## 10. 部署規範

### 10.1 K8s 配置

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin-service
  labels:
    app: admin-service
    environment: dev
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: admin-service
        image: justwork/admin-service:v1.0.0
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: mysql-credentials
              key: url
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: jwt-secret
              key: secret
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"

---
apiVersion: v1
kind: Service
metadata:
  name: admin-service
spec:
  selector:
    app: admin-service
  ports:
  - port: 8080
    targetPort: 8080
```

---

## 11. 監控指標

### 11.1 API 指標

```
# API 響應時間
admin_api_requests_duration_seconds{method="GET",path="/api/admin/users"}

# API 錯誤率
admin_api_errors_total{path="/api/admin/users",error="unauthorized"}

# 登入嘗試
admin_login_attempts_total{status="success"}
admin_login_attempts_total{status="failed"}
```

### 11.2 RBAC 指標

```
admin_rbac_checks_total{result="allowed"}
admin_rbac_checks_total{result="denied"}
```

---

## 12. 安全性

### 12.1 認證

- JWT Token 認證
- Token 過期時間: 2 小時
- 刷新 Token 機制
- 密碼強度要求: 8+ 字元

### 12.2 授權

- RBAC 權限模型
- 最小權限原則
- 角色繼承

### 12.3 審計

- 所有操作記錄
- 登入日誌
- 權限變更日誌

---

## 參考文件

- [後端技術規格](../backend/TECHNICAL_SPECIFICATION.md)
- [後端開發原則](../backend/DEVELOPMENT.md)
- [前端開發原則](../frontend/DEVELOPMENT.md)

---

**文件版本**: 0.1.0.0.0
**最後更新**: 2026-02-14
**維護團隊**: Admin Team
