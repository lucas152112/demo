# Just Work 後端開發原則

本文檔定義 Just Work 後端開發的原則、規範和流程。

---

## 1. 開發原則

### 1.1 永不崩潰 (Zero Crash)

| 原則 | 說明 |
|------|------|
| Panic 處理 | 所有可能的錯誤必須有明確的處理邏輯 |
| 優雅降級 | 任何模組故障不能影響系統整體運作 |
| 自動恢復 | 服務必須具備自我修復能力 |

### 1.2 微服務原則

| 原則 | 說明 |
|------|------|
| 單一職責 | 每個服務只做一件事 |
| 鬆耦合 | 服務間透過 API 通訊 |
| 高內聚 | 相關功能在同一服務內 |
| 獨立部署 | 可獨立部署和擴展 |

### 1.3 資料庫原則

| 原則 | 說明 |
|------|------|
| 讀寫分離 | MySQL 主從架構 |
| 快取策略 | Redis 多級快取 |
| 最終一致性 | 容忍短期資料不一致 |

---

## 2. 程式碼規範

### 2.1 Rust 編碼風格

```rust
// ✅ 好的範例
#[derive(Debug, Serialize, Deserialize)]
pub struct UserResponse {
    pub id: Uuid,
    pub email: String,
    pub name: String,
    pub created_at: DateTime<Utc>,
}

impl UserResponse {
    pub fn from(user: &User) -> Self {
        Self {
            id: user.id,
            email: user.email.clone(),
            name: user.name.clone(),
            created_at: user.created_at,
        }
    }
}

// ❌ 不好的範例
struct UserResponse {  // 缺少 derive
    id: Uuid,          // 沒有 pub
    email: &str,       // 使用引用
}
```

### 2.2 錯誤處理

```rust
// ✅ 使用 thiserror 定義明確的錯誤類型
#[derive(Debug, thiserror::Error)]
pub enum ServiceError {
    #[error("Database error: {source}")]
    DatabaseError { source: sqlx::Error },
    
    #[error("Validation error: {field} - {reason}")]
    ValidationError { field: String, reason: String },
    
    #[error("Not found: {resource}")]
    NotFound { resource: String },
    
    #[error("Unauthorized: {message}")]
    Unauthorized { message: String },
}

// ✅ 錯誤轉換
impl From<sqlx::Error> for ServiceError {
    fn from(source: sqlx::Error) -> Self {
        Self::DatabaseError { source }
    }
}
```

### 2.3 API 處理函數

```rust
// ✅ 標準 API 處理函數
#[utoipa::path(
    get,
    path = "/api/v1/users",
    responses(
        (status = 200, description = "List of users", body = Vec<UserResponse>),
        (status = 401, description = "Unauthorized", body = ErrorResponse),
        (status = 500, description = "Internal server error", body = ErrorResponse),
    ),
    security(
        ("bearerAuth" = [])
    ),
    tag = "users",
)]
pub async fn list_users(
    State(pool): State<AppState>,
    Extension(user): Extension<User>,
) -> Result<Json<Vec<UserResponse>>, ServiceError> {
    // 1. 權限檢查
    user.require_permission(Permission::ListUsers)?;
    
    // 2. 查詢資料
    let users = UserRepository::list_all(&pool).await?;
    
    // 3. 轉換響應
    let response: Vec<UserResponse> = users.iter()
        .map(UserResponse::from)
        .collect();
    
    Ok(Json(response))
}
```

---

## 3. Git 分支策略

### 3.1 分支結構

```
develop (開發分支)
├── feature/user-service/auth      # 用戶服務-認證功能
├── feature/work-service/crud       # 工作服務-CRUD
└── fix/database-connection        # 修復資料庫連線

test (測試分支)
├── feature/user-service/auth
├── feature/work-service/crud
└── fix/database-connection

pp (預發布分支)
└── ...

prod (正式分支)
└── hotfix/critical-bug           # 緊急修復
```

### 3.2 合併規則

| 來源 | 目標 | 說明 |
|------|------|------|
| feature/xxx | develop | 新功能開發 |
| develop | test | 功能完成 |
| test | pp | 測試通過 |
| pp | prod | 預發布驗證 |
| hotfix/xxx | prod | 緊急修復 |

### 3.3 提交規範

```
<類型>(<服務>/<範圍>): <描述>

類型:
- feat: 新功能
- fix: 錯誤修復
- refactor: 重構
- docs: 文件更新
- test: 測試相關
- chore: 建置/工具

範例:
feat(user-service/auth): Add JWT token refresh

- Implement token refresh endpoint
- Add refresh token storage
- Add token expiration check

Closes #123
```

---

## 4. 微服務開發規範

### 4.1 服務結構

```
{service-name}/
├── src/
│   ├── mod.rs
│   ├── main.rs
│   ├── routes/
│   │   ├── mod.rs
│   │   └── user.rs
│   ├── models/
│   │   ├── mod.rs
│   │   └── user.rs
│   ├── services/
│   │   ├── mod.rs
│   │   └── user_service.rs
│   ├── repositories/
│   │   ├── mod.rs
│   │   └── user_repository.rs
│   └── middleware/
│       ├── mod.rs
│       └── auth.rs
├── Cargo.toml
├── Dockerfile
└── README.md
```

### 4.2 服務配置

```yaml
# {service}/config.yaml
service:
  name: user-service
  host: 0.0.0.0
  port: 8080
  
database:
  host: mysql
  port: 3306
  name: justwork_dev
  pool_size: 10
  
redis:
  host: redis
  port: 6379
  key_prefix: justwork:dev:user:

mongodb:
  host: mongo
  port: 27017
  database: justwork_dev
  collection_prefix: dev_

jwt:
  secret: ${JWT_SECRET}
  expiration: 3600
```

### 4.3 熱更新條件

```rust
// 熱更新條件檢查
async fn check_hot_reload(service_name: &str) -> bool {
    let status = get_service_status(service_name).await;
    status == ServiceStatus::Online
}

// 狀態更新
async fn update_service_status(service_name: &str, new_status: ServiceStatus) {
    // 寫入 Redis
    let key = format!("service:{}:status", service_name);
    redis::set(&key, format!("{:?}", new_status)).await;
    
    // 寫入資料庫
    sqlx::query!(
        "UPDATE services SET status = ?, updated_at = NOW() WHERE name = ?",
        new_status as i32,
        service_name
    ).execute(&pool).await.unwrap();
}
```

---

## 5. 測試規範

### 5.1 測試覆蓋率目標

| 測試類型 | 覆蓋率目標 |
|----------|-----------|
| 單元測試 | ≥ 80% |
| 整合測試 | ≥ 60% |
| API 測試 | 核心流程 100% |

### 5.2 單元測試範例

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::models::user::User;
    
    #[tokio::test]
    async fn test_user_from_domain() {
        let user = User {
            id: Uuid::new_v4(),
            email: "test@example.com".to_string(),
            name: "Test User".to_string(),
            created_at: Utc::now(),
        };
        
        let response = UserResponse::from(&user);
        
        assert_eq!(response.email, user.email);
        assert_eq!(response.name, user.name);
    }
    
    #[tokio::test]
    async fn test_validation_error() {
        let error = ServiceError::ValidationError {
            field: "email".to_string(),
            reason: "Invalid format".to_string(),
        };
        
        let error_response = ErrorResponse::from(&error);
        
        assert_eq!(error_response.error.code, "VALIDATION_ERROR");
    }
}
```

### 5.3 整合測試

```rust
#[cfg(test)]
mod integration_tests {
    use super::*;
    use test_context::test_context;
    
    struct TestContext {
        pool: SqlxPool<MySql>,
        redis: RedisConnection,
    }
    
    fn test_context(cx: &mut TestContext) {
        // 設定測試資料
        let user = seed_test_user(&cx.pool);
        cx.redis.set("test_key", "test_value");
    }
    
    #[tokio::test]
    async fn test_create_user(test_context: TestContext) {
        let new_user = CreateUserRequest {
            email: "new@example.com",
            name: "New User",
            password: "password123",
        };
        
        let response = create_user(&test_context.pool, new_user).await;
        
        assert!(response.is_ok());
        assert_eq!(response.unwrap().email, "new@example.com");
    }
}
```

---

## 6. API 文件規範

### 6.1 Swagger/OpenAPI

```rust
/// 用戶管理 API
///
/// 此 API 提供用戶的 CRUD 操作
#[utoipa::path(
    get,
    path = "/api/v1/users",
    responses(
        (status = 200, description = "成功獲取用戶列表", body = [UserResponse]),
        (status = 401, description = "未授權", body = ErrorResponse),
        (status = 500, description = "伺服器錯誤", body = ErrorResponse),
    ),
    security(
        ("bearerAuth" = [])
    ),
    tags = "Users",
)]
pub async fn list_users() -> Result<Json<Vec<UserResponse>>, ServiceError> {
    // 實作
}
```

### 6.2 API 版本管理

```
# 路由群組
api/v1/     # v1 版本
api/v2/     # v2 版本 (未來)
```

---

## 7. 安全性規範

### 7.1 認證與授權

```rust
// JWT 中間件
async fn auth_middleware(
    req: Request,
    next: Next,
) -> Result<Response, ServiceError> {
    let token = extract_bearer_token(&req)?;
    
    let claims = verify_jwt(&token)?;
    
    // 附加用戶資訊到請求
    let mut req = req;
    req.extensions_mut().insert(claims);
    
    Ok(next.run(req).await)
}

// 權限檢查
impl User {
    pub fn require_permission(&self, permission: Permission) -> Result<(), ServiceError> {
        if !self.has_permission(permission) {
            return Err(ServiceError::Unauthorized {
                message: format!("Missing permission: {:?}", permission),
            });
        }
        Ok(())
    }
}
```

### 7.2 敏感資料處理

| 類型 | 處理方式 |
|------|----------|
| 密碼 | bcrypt 雜湊 |
| API Key | 環境變數 |
| Token | JWT + 過期 |
| 日誌 | 脫敏處理 |

---

## 8. 效能要求

### 8.1 延遲目標

| 操作 | P95 | P99 |
|------|-----|-----|
| API 響應 | < 100ms | < 200ms |
| 資料庫查詢 | < 50ms | < 100ms |
| Redis 操作 | < 5ms | < 10ms |

### 8.2 並發要求

| 場景 | 目標 |
|------|------|
| API 併發 | 1000 req/s |
| 資料庫連線 | 100 concurrent |
| Redis 連線 | 100 concurrent |

---

## 9. 部署規範

### 9.1 環境對應

| 環境 | 分支 | 資料庫 | 配置檔 |
|------|------|--------|--------|
| dev | develop | justwork_dev | config/dev.yaml |
| test | test | justwork_test | config/test.yaml |
| pp | pp | justwork_pp | config/pp.yaml |
| prod | prod | justwork | config/prod.yaml |

### 9.2 Docker 建置

```dockerfile
# Dockerfile
FROM rust:1.75-alpine AS builder

WORKDIR /app
COPY . .
RUN cargo build --release --bin user-service

FROM alpine:3.19 AS runtime

RUN addgroup -S app && adduser -S -G app app
USER app

COPY --from=builder /app/target/release/user-service /usr/local/bin/

EXPOSE 8080

CMD ["user-service"]
```

---

## 10. 持續改進

### 10.1 定期檢視

| 項目 | 頻率 | 負責人 |
|------|------|--------|
| 程式碼審查 | 每週 | Tech Lead |
| 效能優化 | 每月 | Team |
| 架構檢視 | 每季 | Architect |
| 安全審計 | 每半年 | Security |

### 10.2 改進來源

- 系統監控數據
- 使用者回饋
- 技術債務清單
- 業界最佳實踐

---

## 參考文件

- [後端技術規格](./TECHNICAL_SPECIFICATION.md)
- [前端開發原則](../frontend/DEVELOPMENT.md)
- [管理後台開發原則](../admin/DEVELOPMENT.md)

---

**文件版本**: 0.1.0.0.0
**最後更新**: 2026-02-14
**維護團隊**: Backend Team
