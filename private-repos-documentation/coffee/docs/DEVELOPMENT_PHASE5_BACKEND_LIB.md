# 🦀 Coffee系統共享後端函式庫開發

## 📦 **Rust共享庫架構**

### **專案結構**
```
coffee-shared-lib/
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── auth/
│   │   ├── mod.rs
│   │   ├── jwt_handler.rs
│   │   ├── rbac.rs
│   │   └── middleware.rs
│   ├── database/
│   │   ├── mod.rs
│   │   ├── connection_pool.rs
│   │   ├── migrations.rs
│   │   └── query_builder.rs
│   ├── models/
│   │   ├── mod.rs
│   │   ├── user.rs
│   │   ├── tenant.rs
│   │   ├── product.rs
│   │   └── order.rs
│   ├── services/
│   │   ├── mod.rs
│   │   ├── notification_service.rs
│   │   ├── file_service.rs
│   │   └── audit_service.rs
│   ├── utils/
│   │   ├── mod.rs
│   │   ├── validation.rs
│   │   ├── encryption.rs
│   │   └── date_utils.rs
│   └── ai/
│       ├── mod.rs
│       ├── chat_client.rs
│       ├── voice_processor.rs
│       └── permission_checker.rs
```

---

## 🔐 **認證模組開發**

### **JWT處理器**
```rust
// src/auth/jwt_handler.rs
use axum::{
    extract::{Request, State},
    http::{HeaderMap, StatusCode},
    middleware::Next,
    response::Response,
};
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use std::time::{SystemTime, UNIX_EPOCH};
use uuid::Uuid;

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Claims {
    pub sub: String,        // 使用者ID
    pub tenant_id: String,  // 租戶ID  
    pub role: String,       // 角色
    pub permissions: Vec<String>, // 權限列表
    pub exp: usize,         // 過期時間
    pub iat: usize,         // 發行時間
    pub jti: String,        // JWT ID
}

#[derive(Debug, Clone)]
pub struct JwtHandler {
    encoding_key: EncodingKey,
    decoding_key: DecodingKey,
    validation: Validation,
}

impl JwtHandler {
    pub fn new(secret: &str) -> Self {
        Self {
            encoding_key: EncodingKey::from_secret(secret.as_bytes()),
            decoding_key: DecodingKey::from_secret(secret.as_bytes()),
            validation: Validation::default(),
        }
    }

    /// 生成JWT Token
    pub fn generate_token(
        &self,
        user_id: &str,
        tenant_id: &str,
        role: &str,
        permissions: Vec<String>,
        expires_in_hours: usize,
    ) -> Result<String, Box<dyn std::error::Error>> {
        let now = SystemTime::now()
            .duration_since(UNIX_EPOCH)?
            .as_secs() as usize;

        let claims = Claims {
            sub: user_id.to_string(),
            tenant_id: tenant_id.to_string(),
            role: role.to_string(),
            permissions,
            exp: now + (expires_in_hours * 3600),
            iat: now,
            jti: Uuid::new_v4().to_string(),
        };

        let token = encode(&Header::default(), &claims, &self.encoding_key)?;
        Ok(token)
    }

    /// 驗證JWT Token
    pub fn verify_token(&self, token: &str) -> Result<Claims, Box<dyn std::error::Error>> {
        let token_data = decode::<Claims>(token, &self.decoding_key, &self.validation)?;
        Ok(token_data.claims)
    }

    /// 刷新Token
    pub fn refresh_token(
        &self,
        token: &str,
        extends_hours: usize,
    ) -> Result<String, Box<dyn std::error::Error>> {
        let claims = self.verify_token(token)?;
        
        // 生成新的Token，保持相同權限
        self.generate_token(
            &claims.sub,
            &claims.tenant_id,
            &claims.role,
            claims.permissions,
            extends_hours,
        )
    }

    /// 從請求頭提取Token
    pub fn extract_token_from_headers(headers: &HeaderMap) -> Option<String> {
        headers
            .get("authorization")
            .and_then(|auth_header| auth_header.to_str().ok())
            .and_then(|auth_str| {
                if auth_str.starts_with("Bearer ") {
                    Some(auth_str[7..].to_string())
                } else {
                    None
                }
            })
    }
}

/// JWT認證中間件
pub async fn jwt_middleware(
    State(jwt_handler): State<JwtHandler>,
    mut request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let headers = request.headers();
    
    let token = JwtHandler::extract_token_from_headers(headers)
        .ok_or(StatusCode::UNAUTHORIZED)?;

    let claims = jwt_handler
        .verify_token(&token)
        .map_err(|_| StatusCode::UNAUTHORIZED)?;

    // 將claims添加到請求擴展中
    request.extensions_mut().insert(claims);

    Ok(next.run(request).await)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_jwt_generation_and_verification() {
        let handler = JwtHandler::new("test_secret");
        
        let permissions = vec!["read:orders".to_string(), "write:products".to_string()];
        let token = handler
            .generate_token("user123", "tenant456", "manager", permissions.clone(), 24)
            .unwrap();

        let claims = handler.verify_token(&token).unwrap();
        assert_eq!(claims.sub, "user123");
        assert_eq!(claims.tenant_id, "tenant456");
        assert_eq!(claims.role, "manager");
        assert_eq!(claims.permissions, permissions);
    }
}
```

### **RBAC權限控制**
```rust
// src/auth/rbac.rs
use serde::{Deserialize, Serialize};
use std::collections::{HashMap, HashSet};
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Role {
    pub id: String,
    pub name: String,
    pub description: Option<String>,
    pub permissions: HashSet<String>,
    pub tenant_id: String,
    pub is_system: bool, // 系統預設角色
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Permission {
    pub id: String,
    pub resource: String,  // 資源類型: orders, products, customers
    pub action: String,    // 操作: create, read, update, delete
    pub conditions: Option<HashMap<String, String>>, // 條件限制
}

#[derive(Debug, Clone)]
pub struct RBACManager {
    roles: HashMap<String, Role>,
    permissions: HashMap<String, Permission>,
}

impl RBACManager {
    pub fn new() -> Self {
        let mut manager = Self {
            roles: HashMap::new(),
            permissions: HashMap::new(),
        };
        
        // 初始化系統預設權限和角色
        manager.init_system_roles_and_permissions();
        manager
    }

    /// 初始化系統預設角色和權限
    fn init_system_roles_and_permissions(&mut self) {
        // 系統權限
        let permissions = vec![
            // 訂單權限
            ("orders:create", "orders", "create"),
            ("orders:read", "orders", "read"),
            ("orders:update", "orders", "update"),
            ("orders:delete", "orders", "delete"),
            
            // 商品權限
            ("products:create", "products", "create"),
            ("products:read", "products", "read"),
            ("products:update", "products", "update"),
            ("products:delete", "products", "delete"),
            
            // 客戶權限
            ("customers:create", "customers", "create"),
            ("customers:read", "customers", "read"),
            ("customers:update", "customers", "update"),
            ("customers:delete", "customers", "delete"),
            
            // 報表權限
            ("reports:read", "reports", "read"),
            ("reports:export", "reports", "export"),
            
            // 系統管理權限
            ("system:tenant_manage", "system", "tenant_manage"),
            ("system:user_manage", "system", "user_manage"),
            ("system:role_manage", "system", "role_manage"),
        ];

        for (id, resource, action) in permissions {
            self.permissions.insert(
                id.to_string(),
                Permission {
                    id: id.to_string(),
                    resource: resource.to_string(),
                    action: action.to_string(),
                    conditions: None,
                },
            );
        }

        // 系統預設角色
        self.create_system_roles();
    }

    fn create_system_roles(&mut self) {
        // 超級管理員 (平台級)
        let mut super_admin_permissions = HashSet::new();
        for permission_id in self.permissions.keys() {
            super_admin_permissions.insert(permission_id.clone());
        }
        
        self.roles.insert(
            "super_admin".to_string(),
            Role {
                id: "super_admin".to_string(),
                name: "超級管理員".to_string(),
                description: Some("平台超級管理員，擁有所有權限".to_string()),
                permissions: super_admin_permissions,
                tenant_id: "system".to_string(),
                is_system: true,
            },
        );

        // 租戶管理員
        let tenant_admin_permissions: HashSet<String> = vec![
            "orders:create", "orders:read", "orders:update", "orders:delete",
            "products:create", "products:read", "products:update", "products:delete",
            "customers:create", "customers:read", "customers:update", "customers:delete",
            "reports:read", "reports:export",
            "system:user_manage", "system:role_manage",
        ].iter().map(|s| s.to_string()).collect();

        self.roles.insert(
            "tenant_admin".to_string(),
            Role {
                id: "tenant_admin".to_string(),
                name: "租戶管理員".to_string(),
                description: Some("租戶管理員，管理租戶內所有資源".to_string()),
                permissions: tenant_admin_permissions,
                tenant_id: "".to_string(), // 會在使用時設定
                is_system: true,
            },
        );

        // 店長
        let manager_permissions: HashSet<String> = vec![
            "orders:create", "orders:read", "orders:update",
            "products:read", "products:update",
            "customers:create", "customers:read", "customers:update",
            "reports:read",
        ].iter().map(|s| s.to_string()).collect();

        self.roles.insert(
            "manager".to_string(),
            Role {
                id: "manager".to_string(),
                name: "店長".to_string(),
                description: Some("門店管理者".to_string()),
                permissions: manager_permissions,
                tenant_id: "".to_string(),
                is_system: true,
            },
        );

        // 店員
        let staff_permissions: HashSet<String> = vec![
            "orders:create", "orders:read",
            "products:read",
            "customers:create", "customers:read",
        ].iter().map(|s| s.to_string()).collect();

        self.roles.insert(
            "staff".to_string(),
            Role {
                id: "staff".to_string(),
                name: "店員".to_string(),
                description: Some("門店工作人員".to_string()),
                permissions: staff_permissions,
                tenant_id: "".to_string(),
                is_system: true,
            },
        );
    }

    /// 檢查使用者是否有指定權限
    pub fn check_permission(
        &self,
        user_roles: &[String],
        tenant_id: &str,
        required_permission: &str,
    ) -> bool {
        for role_id in user_roles {
            if let Some(role) = self.roles.get(role_id) {
                // 檢查租戶權限隔離 (系統角色除外)
                if !role.is_system && role.tenant_id != tenant_id {
                    continue;
                }
                
                if role.permissions.contains(required_permission) {
                    return true;
                }
            }
        }
        false
    }

    /// 檢查資源存取權限
    pub fn check_resource_access(
        &self,
        user_roles: &[String],
        tenant_id: &str,
        resource: &str,
        action: &str,
        resource_tenant_id: Option<&str>,
    ) -> bool {
        let permission_key = format!("{}:{}", resource, action);
        
        // 基本權限檢查
        if !self.check_permission(user_roles, tenant_id, &permission_key) {
            return false;
        }

        // 租戶隔離檢查
        if let Some(resource_tenant) = resource_tenant_id {
            if resource_tenant != tenant_id {
                // 檢查是否為跨租戶權限 (只有超級管理員才有)
                return self.check_permission(user_roles, tenant_id, "system:cross_tenant");
            }
        }

        true
    }

    /// 創建自定義角色
    pub fn create_role(
        &mut self,
        name: String,
        tenant_id: String,
        permissions: HashSet<String>,
        description: Option<String>,
    ) -> Result<String, String> {
        // 驗證權限是否存在
        for permission in &permissions {
            if !self.permissions.contains_key(permission) {
                return Err(format!("權限不存在: {}", permission));
            }
        }

        let role_id = Uuid::new_v4().to_string();
        let role = Role {
            id: role_id.clone(),
            name,
            description,
            permissions,
            tenant_id,
            is_system: false,
        };

        self.roles.insert(role_id.clone(), role);
        Ok(role_id)
    }

    /// 獲取角色的所有權限
    pub fn get_role_permissions(&self, role_id: &str) -> Option<&HashSet<String>> {
        self.roles.get(role_id).map(|role| &role.permissions)
    }

    /// 獲取使用者的所有權限 (合併多個角色)
    pub fn get_user_permissions(&self, user_roles: &[String]) -> HashSet<String> {
        let mut all_permissions = HashSet::new();
        
        for role_id in user_roles {
            if let Some(permissions) = self.get_role_permissions(role_id) {
                all_permissions.extend(permissions.iter().cloned());
            }
        }
        
        all_permissions
    }
}

/// 權限檢查宏
#[macro_export]
macro_rules! require_permission {
    ($rbac:expr, $user_roles:expr, $tenant_id:expr, $permission:expr) => {
        if !$rbac.check_permission($user_roles, $tenant_id, $permission) {
            return Err(axum::http::StatusCode::FORBIDDEN);
        }
    };
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_rbac_permission_check() {
        let rbac = RBACManager::new();
        
        let user_roles = vec!["manager".to_string()];
        
        // 店長應該有讀取訂單權限
        assert!(rbac.check_permission(&user_roles, "tenant1", "orders:read"));
        
        // 店長不應該有刪除訂單權限
        assert!(!rbac.check_permission(&user_roles, "tenant1", "orders:delete"));
        
        // 店長不應該有系統管理權限
        assert!(!rbac.check_permission(&user_roles, "tenant1", "system:tenant_manage"));
    }

    #[test]
    fn test_tenant_isolation() {
        let rbac = RBACManager::new();
        
        let user_roles = vec!["manager".to_string()];
        
        // 檢查資源存取 - 同租戶
        assert!(rbac.check_resource_access(
            &user_roles,
            "tenant1",
            "orders",
            "read",
            Some("tenant1")
        ));
        
        // 檢查資源存取 - 不同租戶 (應該拒絕)
        assert!(!rbac.check_resource_access(
            &user_roles,
            "tenant1",
            "orders",
            "read",
            Some("tenant2")
        ));
    }
}
```

### **認證中間件**
```rust
// src/auth/middleware.rs
use axum::{
    extract::{Request, State},
    http::StatusCode,
    middleware::Next,
    response::Response,
};
use crate::auth::{jwt_handler::Claims, rbac::RBACManager};

/// 權限檢查中間件
pub fn require_permission(permission: &'static str) -> impl Fn(State<RBACManager>, Request, Next) -> std::pin::Pin<Box<dyn std::future::Future<Output = Result<Response, StatusCode>>>> + Clone {
    move |State(rbac), mut request: Request, next: Next| {
        let required_permission = permission;
        Box::pin(async move {
            // 從請求擴展中獲取JWT claims
            let claims = request
                .extensions()
                .get::<Claims>()
                .ok_or(StatusCode::UNAUTHORIZED)?;

            // 檢查權限
            let user_roles = vec![claims.role.clone()]; // 簡化處理，實際可能有多角色
            
            if !rbac.check_permission(&user_roles, &claims.tenant_id, required_permission) {
                return Err(StatusCode::FORBIDDEN);
            }

            Ok(next.run(request).await)
        })
    }
}

/// 租戶隔離中間件
pub async fn tenant_isolation_middleware(
    mut request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    // 從JWT claims中獲取租戶ID
    let claims = request
        .extensions()
        .get::<Claims>()
        .ok_or(StatusCode::UNAUTHORIZED)?;

    // 將租戶ID添加到請求頭中，供後續處理使用
    request.headers_mut().insert(
        "x-tenant-id",
        claims.tenant_id.parse().map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?,
    );

    Ok(next.run(request).await)
}

/// 管理員權限檢查中間件  
pub async fn admin_only_middleware(
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let claims = request
        .extensions()
        .get::<Claims>()
        .ok_or(StatusCode::UNAUTHORIZED)?;

    // 檢查是否為管理員角色
    if !["super_admin", "tenant_admin"].contains(&claims.role.as_str()) {
        return Err(StatusCode::FORBIDDEN);
    }

    Ok(next.run(request).await)
}
```

---

## 🗄️ **資料庫模組開發**

### **連線池管理**
```rust
// src/database/connection_pool.rs
use sqlx::{PgPool, Pool, Postgres, Row};
use std::time::Duration;
use tokio::sync::OnceCell;
use uuid::Uuid;

pub struct DatabaseConfig {
    pub host: String,
    pub port: u16,
    pub username: String,
    pub password: String,
    pub database: String,
    pub max_connections: u32,
    pub min_connections: u32,
    pub connect_timeout: Duration,
    pub idle_timeout: Duration,
}

impl Default for DatabaseConfig {
    fn default() -> Self {
        Self {
            host: "localhost".to_string(),
            port: 5432,
            username: "coffee".to_string(),
            password: "coffee123".to_string(),
            database: "coffee_db".to_string(),
            max_connections: 20,
            min_connections: 5,
            connect_timeout: Duration::from_secs(30),
            idle_timeout: Duration::from_secs(300),
        }
    }
}

pub struct DatabaseManager {
    pools: std::collections::HashMap<String, PgPool>,
    config: DatabaseConfig,
}

impl DatabaseManager {
    pub async fn new(config: DatabaseConfig) -> Result<Self, sqlx::Error> {
        Ok(Self {
            pools: std::collections::HashMap::new(),
            config,
        })
    }

    /// 獲取租戶專用資料庫連線池
    pub async fn get_tenant_pool(&mut self, tenant_id: &str) -> Result<&PgPool, sqlx::Error> {
        if !self.pools.contains_key(tenant_id) {
            let pool = self.create_tenant_pool(tenant_id).await?;
            self.pools.insert(tenant_id.to_string(), pool);
        }
        
        Ok(self.pools.get(tenant_id).unwrap())
    }

    /// 創建租戶專用連線池
    async fn create_tenant_pool(&self, tenant_id: &str) -> Result<PgPool, sqlx::Error> {
        let database_url = format!(
            "postgresql://{}:{}@{}:{}/{}",
            self.config.username,
            self.config.password,
            self.config.host,
            self.config.port,
            format!("{}_{}", self.config.database, tenant_id)
        );

        let pool = sqlx::postgres::PgPoolOptions::new()
            .max_connections(self.config.max_connections)
            .min_connections(self.config.min_connections)
            .acquire_timeout(self.config.connect_timeout)
            .idle_timeout(self.config.idle_timeout)
            .connect(&database_url)
            .await?;

        // 確保資料庫Schema存在
        self.ensure_tenant_schema(&pool, tenant_id).await?;

        Ok(pool)
    }

    /// 確保租戶Schema存在
    async fn ensure_tenant_schema(&self, pool: &PgPool, tenant_id: &str) -> Result<(), sqlx::Error> {
        // 檢查Schema是否存在
        let schema_exists: bool = sqlx::query_scalar(
            "SELECT EXISTS(SELECT 1 FROM information_schema.schemata WHERE schema_name = $1)"
        )
        .bind(tenant_id)
        .fetch_one(pool)
        .await?;

        if !schema_exists {
            // 創建Schema
            let create_schema_sql = format!("CREATE SCHEMA IF NOT EXISTS {}", tenant_id);
            sqlx::query(&create_schema_sql).execute(pool).await?;

            // 運行遷移
            self.run_tenant_migrations(pool, tenant_id).await?;
        }

        Ok(())
    }

    /// 運行租戶資料庫遷移
    async fn run_tenant_migrations(&self, pool: &PgPool, tenant_id: &str) -> Result<(), sqlx::Error> {
        // 設置搜索路徑到租戶Schema
        let set_search_path = format!("SET search_path TO {}, public", tenant_id);
        sqlx::query(&set_search_path).execute(pool).await?;

        // 創建基本表結構
        self.create_tenant_tables(pool).await?;

        Ok(())
    }

    /// 創建租戶表結構
    async fn create_tenant_tables(&self, pool: &PgPool) -> Result<(), sqlx::Error> {
        // 用戶表
        sqlx::query(r#"
            CREATE TABLE IF NOT EXISTS users (
                id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
                username VARCHAR(100) UNIQUE NOT NULL,
                email VARCHAR(255) UNIQUE NOT NULL,
                password_hash VARCHAR(255) NOT NULL,
                display_name VARCHAR(255),
                role VARCHAR(50) NOT NULL DEFAULT 'staff',
                is_active BOOLEAN DEFAULT true,
                last_login_at TIMESTAMPTZ,
                created_at TIMESTAMPTZ DEFAULT NOW(),
                updated_at TIMESTAMPTZ DEFAULT NOW()
            )
        "#).execute(pool).await?;

        // 商品表
        sqlx::query(r#"
            CREATE TABLE IF NOT EXISTS products (
                id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
                name VARCHAR(255) NOT NULL,
                description TEXT,
                price DECIMAL(10,2) NOT NULL,
                cost DECIMAL(10,2),
                category_id UUID,
                sku VARCHAR(100) UNIQUE,
                image_url VARCHAR(500),
                is_active BOOLEAN DEFAULT true,
                created_at TIMESTAMPTZ DEFAULT NOW(),
                updated_at TIMESTAMPTZ DEFAULT NOW()
            )
        "#).execute(pool).await?;

        // 客戶表
        sqlx::query(r#"
            CREATE TABLE IF NOT EXISTS customers (
                id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
                phone VARCHAR(20) UNIQUE,
                email VARCHAR(255),
                name VARCHAR(255),
                birthday DATE,
                gender VARCHAR(10),
                loyalty_points INT DEFAULT 0,
                total_spent DECIMAL(12,2) DEFAULT 0,
                visit_count INT DEFAULT 0,
                last_visit_at TIMESTAMPTZ,
                created_at TIMESTAMPTZ DEFAULT NOW(),
                updated_at TIMESTAMPTZ DEFAULT NOW()
            )
        "#).execute(pool).await?;

        // 訂單表
        sqlx::query(r#"
            CREATE TABLE IF NOT EXISTS orders (
                id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
                order_number VARCHAR(50) UNIQUE NOT NULL,
                customer_id UUID REFERENCES customers(id),
                user_id UUID REFERENCES users(id),
                total_amount DECIMAL(10,2) NOT NULL,
                discount_amount DECIMAL(10,2) DEFAULT 0,
                tax_amount DECIMAL(10,2) DEFAULT 0,
                final_amount DECIMAL(10,2) NOT NULL,
                payment_method VARCHAR(50),
                payment_status VARCHAR(20) DEFAULT 'pending',
                order_status VARCHAR(20) DEFAULT 'pending',
                notes TEXT,
                created_at TIMESTAMPTZ DEFAULT NOW(),
                updated_at TIMESTAMPTZ DEFAULT NOW()
            )
        "#).execute(pool).await?;

        // 訂單項目表
        sqlx::query(r#"
            CREATE TABLE IF NOT EXISTS order_items (
                id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
                order_id UUID REFERENCES orders(id) ON DELETE CASCADE,
                product_id UUID REFERENCES products(id),
                product_name VARCHAR(255) NOT NULL,
                quantity INT NOT NULL,
                unit_price DECIMAL(10,2) NOT NULL,
                total_price DECIMAL(10,2) NOT NULL,
                customizations JSONB,
                created_at TIMESTAMPTZ DEFAULT NOW()
            )
        "#).execute(pool).await?;

        // 庫存表
        sqlx::query(r#"
            CREATE TABLE IF NOT EXISTS inventory (
                id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
                product_id UUID REFERENCES products(id) ON DELETE CASCADE,
                current_stock INT NOT NULL DEFAULT 0,
                reserved_stock INT DEFAULT 0,
                min_stock_level INT DEFAULT 0,
                max_stock_level INT DEFAULT 1000,
                last_restock_at TIMESTAMPTZ,
                created_at TIMESTAMPTZ DEFAULT NOW(),
                updated_at TIMESTAMPTZ DEFAULT NOW(),
                UNIQUE(product_id)
            )
        "#).execute(pool).await?;

        // 創建索引
        self.create_indexes(pool).await?;

        Ok(())
    }

    /// 創建資料庫索引
    async fn create_indexes(&self, pool: &PgPool) -> Result<(), sqlx::Error> {
        let indexes = vec![
            "CREATE INDEX IF NOT EXISTS idx_orders_customer_id ON orders(customer_id)",
            "CREATE INDEX IF NOT EXISTS idx_orders_created_at ON orders(created_at)",
            "CREATE INDEX IF NOT EXISTS idx_orders_status ON orders(order_status)",
            "CREATE INDEX IF NOT EXISTS idx_order_items_order_id ON order_items(order_id)",
            "CREATE INDEX IF NOT EXISTS idx_order_items_product_id ON order_items(product_id)",
            "CREATE INDEX IF NOT EXISTS idx_customers_phone ON customers(phone)",
            "CREATE INDEX IF NOT EXISTS idx_customers_email ON customers(email)",
            "CREATE INDEX IF NOT EXISTS idx_products_category_id ON products(category_id)",
            "CREATE INDEX IF NOT EXISTS idx_products_sku ON products(sku)",
            "CREATE INDEX IF NOT EXISTS idx_inventory_product_id ON inventory(product_id)",
        ];

        for index_sql in indexes {
            sqlx::query(index_sql).execute(pool).await?;
        }

        Ok(())
    }

    /// 健康檢查
    pub async fn health_check(&self, tenant_id: &str) -> Result<bool, sqlx::Error> {
        if let Some(pool) = self.pools.get(tenant_id) {
            sqlx::query("SELECT 1").fetch_one(pool).await?;
            Ok(true)
        } else {
            Ok(false)
        }
    }

    /// 獲取連線池狀態
    pub fn get_pool_status(&self, tenant_id: &str) -> Option<(u32, u32)> {
        self.pools.get(tenant_id).map(|pool| {
            (pool.size(), pool.num_idle())
        })
    }
}

/// 租戶資料庫管理器單例
static DATABASE_MANAGER: OnceCell<std::sync::Mutex<DatabaseManager>> = OnceCell::const_new();

pub async fn init_database_manager(config: DatabaseConfig) -> Result<(), sqlx::Error> {
    let manager = DatabaseManager::new(config).await?;
    DATABASE_MANAGER.set(std::sync::Mutex::new(manager))
        .map_err(|_| sqlx::Error::Configuration("Database manager already initialized".into()))?;
    Ok(())
}

pub async fn get_tenant_pool(tenant_id: &str) -> Result<PgPool, Box<dyn std::error::Error>> {
    let manager = DATABASE_MANAGER.get()
        .ok_or("Database manager not initialized")?;
    
    let mut manager = manager.lock().unwrap();
    let pool = manager.get_tenant_pool(tenant_id).await?;
    Ok(pool.clone())
}
```

---

**🏗️ 共享後端函式庫第一部分開發完成！**

包含：
- ✅ **JWT認證處理器**: Token生成/驗證/刷新
- ✅ **RBAC權限控制**: 多角色權限管理+租戶隔離  
- ✅ **認證中間件**: 權限檢查+租戶隔離+管理員驗證
- ✅ **資料庫連線池**: 多租戶資料庫+自動Schema+連線管理

**下一步**: 繼續開發資料模型和服務層？