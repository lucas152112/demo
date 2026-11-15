# 🌐 Coffee系統 Phase 1: 平台管理系統開發

## 📋 **平台管理系統概述**

平台管理系統是Coffee多租戶架構的核心，負責：
- 租戶生命週期管理 (創建、配置、監控、計費)
- 平台級監控與運營
- 系統資源分配與優化
- 全域配置管理

---

## 🏢 **租戶管理服務開發**

### **租戶管理API控制器**
```rust
// backend/tenant-management/src/handlers/tenant_handler.rs
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    response::Json,
    Extension,
};
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use coffee_shared_lib::{
    auth::jwt_handler::Claims,
    database::connection_pool::DatabaseManager,
    models::tenant::{Tenant, CreateTenantRequest, UpdateTenantRequest, TenantStatus},
};

#[derive(Debug, Serialize)]
pub struct TenantResponse {
    pub id: Uuid,
    pub name: String,
    pub display_name: String,
    pub domain: Option<String>,
    pub status: TenantStatus,
    pub plan: String,
    pub features: Vec<String>,
    pub max_users: Option<i32>,
    pub max_stores: Option<i32>,
    pub storage_quota_gb: Option<i32>,
    pub created_at: chrono::DateTime<chrono::Utc>,
    pub trial_ends_at: Option<chrono::DateTime<chrono::Utc>>,
}

#[derive(Debug, Deserialize)]
pub struct ListTenantsQuery {
    pub status: Option<TenantStatus>,
    pub plan: Option<String>,
    pub page: Option<u32>,
    pub limit: Option<u32>,
}

#[derive(Debug, Deserialize)]
pub struct CreateTenantApiRequest {
    pub name: String,
    pub display_name: String,
    pub admin_email: String,
    pub admin_password: String,
    pub plan: String,
    pub domain: Option<String>,
    pub trial_days: Option<i32>,
}

/// 創建新租戶
pub async fn create_tenant(
    State(db_manager): State<DatabaseManager>,
    Extension(claims): Extension<Claims>,
    Json(request): Json<CreateTenantApiRequest>,
) -> Result<Json<TenantResponse>, StatusCode> {
    // 檢查超級管理員權限
    if claims.role != "super_admin" {
        return Err(StatusCode::FORBIDDEN);
    }

    // 驗證請求資料
    coffee_shared_lib::utils::validation::Validator::validate_email(&request.admin_email)
        .map_err(|_| StatusCode::BAD_REQUEST)?;
    
    coffee_shared_lib::utils::validation::Validator::validate_password(&request.admin_password)
        .map_err(|_| StatusCode::BAD_REQUEST)?;

    // 檢查租戶名稱唯一性
    let platform_pool = db_manager.get_platform_pool().await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let existing_tenant = Tenant::find_by_name(&platform_pool, &request.name).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    
    if existing_tenant.is_some() {
        return Err(StatusCode::CONFLICT);
    }

    // 創建租戶
    let tenant_id = Uuid::new_v4();
    let create_request = CreateTenantRequest {
        id: tenant_id,
        name: request.name,
        display_name: request.display_name,
        plan: request.plan,
        domain: request.domain,
        trial_days: request.trial_days,
    };

    let tenant = Tenant::create(&platform_pool, create_request).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    // 初始化租戶資料庫
    db_manager.initialize_tenant_database(&tenant.id.to_string()).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    // 創建租戶管理員帳號
    let tenant_pool = db_manager.get_tenant_pool(&tenant.id.to_string()).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let admin_request = coffee_shared_lib::models::user::CreateUserRequest {
        username: "admin".to_string(),
        email: request.admin_email,
        password: request.admin_password,
        display_name: Some("Administrator".to_string()),
        role: Some("tenant_admin".to_string()),
    };

    coffee_shared_lib::models::user::User::create(&tenant_pool, admin_request, &tenant.id.to_string()).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    // 創建預設商品分類和樣例資料
    setup_tenant_defaults(&tenant_pool, &tenant.id.to_string()).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    // 發送歡迎郵件
    send_welcome_email(&request.admin_email, &tenant.display_name, &tenant.name).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok(Json(TenantResponse {
        id: tenant.id,
        name: tenant.name,
        display_name: tenant.display_name,
        domain: tenant.domain,
        status: tenant.status,
        plan: tenant.plan,
        features: tenant.features,
        max_users: tenant.max_users,
        max_stores: tenant.max_stores,
        storage_quota_gb: tenant.storage_quota_gb,
        created_at: tenant.created_at,
        trial_ends_at: tenant.trial_ends_at,
    }))
}

/// 獲取租戶列表
pub async fn list_tenants(
    State(db_manager): State<DatabaseManager>,
    Extension(claims): Extension<Claims>,
    Query(query): Query<ListTenantsQuery>,
) -> Result<Json<PaginatedResponse<TenantResponse>>, StatusCode> {
    // 檢查權限
    if claims.role != "super_admin" {
        return Err(StatusCode::FORBIDDEN);
    }

    let platform_pool = db_manager.get_platform_pool().await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let page = query.page.unwrap_or(1);
    let limit = query.limit.unwrap_or(20).min(100);

    let (tenants, total) = Tenant::list(
        &platform_pool,
        query.status,
        query.plan,
        page,
        limit,
    ).await.map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let tenant_responses: Vec<TenantResponse> = tenants
        .into_iter()
        .map(|tenant| TenantResponse {
            id: tenant.id,
            name: tenant.name,
            display_name: tenant.display_name,
            domain: tenant.domain,
            status: tenant.status,
            plan: tenant.plan,
            features: tenant.features,
            max_users: tenant.max_users,
            max_stores: tenant.max_stores,
            storage_quota_gb: tenant.storage_quota_gb,
            created_at: tenant.created_at,
            trial_ends_at: tenant.trial_ends_at,
        })
        .collect();

    Ok(Json(PaginatedResponse {
        data: tenant_responses,
        total,
        page,
        limit,
        total_pages: (total as f64 / limit as f64).ceil() as u32,
    }))
}

/// 獲取租戶詳情
pub async fn get_tenant(
    Path(tenant_id): Path<Uuid>,
    State(db_manager): State<DatabaseManager>,
    Extension(claims): Extension<Claims>,
) -> Result<Json<TenantDetailResponse>, StatusCode> {
    // 權限檢查
    if claims.role != "super_admin" && claims.tenant_id != tenant_id.to_string() {
        return Err(StatusCode::FORBIDDEN);
    }

    let platform_pool = db_manager.get_platform_pool().await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let tenant = Tenant::find_by_id(&platform_pool, tenant_id).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?
        .ok_or(StatusCode::NOT_FOUND)?;

    // 獲取租戶統計資料
    let stats = get_tenant_statistics(&db_manager, &tenant_id.to_string()).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok(Json(TenantDetailResponse {
        tenant: TenantResponse {
            id: tenant.id,
            name: tenant.name,
            display_name: tenant.display_name,
            domain: tenant.domain,
            status: tenant.status,
            plan: tenant.plan,
            features: tenant.features,
            max_users: tenant.max_users,
            max_stores: tenant.max_stores,
            storage_quota_gb: tenant.storage_quota_gb,
            created_at: tenant.created_at,
            trial_ends_at: tenant.trial_ends_at,
        },
        statistics: stats,
    }))
}

/// 更新租戶
pub async fn update_tenant(
    Path(tenant_id): Path<Uuid>,
    State(db_manager): State<DatabaseManager>,
    Extension(claims): Extension<Claims>,
    Json(request): Json<UpdateTenantRequest>,
) -> Result<Json<TenantResponse>, StatusCode> {
    // 權限檢查
    if claims.role != "super_admin" {
        return Err(StatusCode::FORBIDDEN);
    }

    let platform_pool = db_manager.get_platform_pool().await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let mut tenant = Tenant::find_by_id(&platform_pool, tenant_id).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?
        .ok_or(StatusCode::NOT_FOUND)?;

    tenant.update(&platform_pool, request).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok(Json(TenantResponse {
        id: tenant.id,
        name: tenant.name,
        display_name: tenant.display_name,
        domain: tenant.domain,
        status: tenant.status,
        plan: tenant.plan,
        features: tenant.features,
        max_users: tenant.max_users,
        max_stores: tenant.max_stores,
        storage_quota_gb: tenant.storage_quota_gb,
        created_at: tenant.created_at,
        trial_ends_at: tenant.trial_ends_at,
    }))
}

/// 停用租戶
pub async fn deactivate_tenant(
    Path(tenant_id): Path<Uuid>,
    State(db_manager): State<DatabaseManager>,
    Extension(claims): Extension<Claims>,
) -> Result<StatusCode, StatusCode> {
    // 權限檢查
    if claims.role != "super_admin" {
        return Err(StatusCode::FORBIDDEN);
    }

    let platform_pool = db_manager.get_platform_pool().await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let mut tenant = Tenant::find_by_id(&platform_pool, tenant_id).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?
        .ok_or(StatusCode::NOT_FOUND)?;

    tenant.deactivate(&platform_pool).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    // 停用租戶的所有使用者
    let tenant_pool = db_manager.get_tenant_pool(&tenant_id.to_string()).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    coffee_shared_lib::models::user::User::deactivate_all(&tenant_pool, &tenant_id.to_string()).await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok(StatusCode::NO_CONTENT)
}

/// 租戶詳情回應
#[derive(Debug, Serialize)]
pub struct TenantDetailResponse {
    #[serde(flatten)]
    pub tenant: TenantResponse,
    pub statistics: TenantStatistics,
}

#[derive(Debug, Serialize)]
pub struct TenantStatistics {
    pub total_users: i64,
    pub active_users_30d: i64,
    pub total_orders: i64,
    pub total_revenue: rust_decimal::Decimal,
    pub storage_used_mb: i64,
    pub api_calls_24h: i64,
}

#[derive(Debug, Serialize)]
pub struct PaginatedResponse<T> {
    pub data: Vec<T>,
    pub total: i64,
    pub page: u32,
    pub limit: u32,
    pub total_pages: u32,
}

/// 設置租戶預設資料
async fn setup_tenant_defaults(
    tenant_pool: &sqlx::PgPool,
    tenant_id: &str,
) -> Result<(), sqlx::Error> {
    // 設置租戶搜索路徑
    sqlx::query(&format!("SET search_path TO {}, public", tenant_id))
        .execute(tenant_pool)
        .await?;

    // 創建預設商品分類
    let categories = vec![
        ("咖啡", "各式現煮咖啡"),
        ("茶飲", "茶類飲品"),
        ("點心", "蛋糕甜點"),
        ("輕食", "三明治沙拉"),
        ("周邊商品", "咖啡豆等商品"),
    ];

    for (name, description) in categories {
        sqlx::query(
            "INSERT INTO product_categories (id, name, description) VALUES ($1, $2, $3)"
        )
        .bind(Uuid::new_v4())
        .bind(name)
        .bind(description)
        .execute(tenant_pool)
        .await?;
    }

    // 創建範例商品
    let sample_products = vec![
        ("美式咖啡", "經典美式黑咖啡", "80.00", "咖啡"),
        ("拿鐵咖啡", "香濃牛奶咖啡", "100.00", "咖啡"),
        ("卡布奇諾", "泡沫豐富的咖啡", "95.00", "咖啡"),
        ("提拉米蘇", "經典義式甜點", "120.00", "點心"),
        ("起司蛋糕", "濃郁起司蛋糕", "110.00", "點心"),
    ];

    for (name, description, price, category) in sample_products {
        // 找到分類ID
        let category_id: Uuid = sqlx::query_scalar(
            "SELECT id FROM product_categories WHERE name = $1"
        )
        .bind(category)
        .fetch_one(tenant_pool)
        .await?;

        let product_id = Uuid::new_v4();

        // 創建商品
        sqlx::query(
            r#"
            INSERT INTO products (id, name, description, price, category_id, is_active)
            VALUES ($1, $2, $3, $4, $5, true)
            "#
        )
        .bind(product_id)
        .bind(name)
        .bind(description)
        .bind(rust_decimal::Decimal::from_str_exact(price).unwrap())
        .bind(category_id)
        .execute(tenant_pool)
        .await?;

        // 創建初始庫存
        sqlx::query(
            "INSERT INTO inventory (product_id, current_stock, min_stock_level) VALUES ($1, 100, 10)"
        )
        .bind(product_id)
        .execute(tenant_pool)
        .await?;
    }

    Ok(())
}

/// 獲取租戶統計資料
async fn get_tenant_statistics(
    db_manager: &DatabaseManager,
    tenant_id: &str,
) -> Result<TenantStatistics, Box<dyn std::error::Error>> {
    let tenant_pool = db_manager.get_tenant_pool(tenant_id).await?;

    // 設置搜索路徑
    sqlx::query(&format!("SET search_path TO {}, public", tenant_id))
        .execute(&tenant_pool)
        .await?;

    // 獲取統計資料
    let total_users: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM users")
        .fetch_one(&tenant_pool)
        .await?;

    let active_users_30d: i64 = sqlx::query_scalar(
        "SELECT COUNT(*) FROM users WHERE last_login_at > NOW() - INTERVAL '30 days'"
    )
    .fetch_one(&tenant_pool)
    .await?;

    let total_orders: i64 = sqlx::query_scalar("SELECT COUNT(*) FROM orders")
        .fetch_one(&tenant_pool)
        .await?;

    let total_revenue: Option<rust_decimal::Decimal> = sqlx::query_scalar(
        "SELECT SUM(final_amount) FROM orders WHERE payment_status = 'completed'"
    )
    .fetch_one(&tenant_pool)
    .await?;

    let api_calls_24h: i64 = 0; // 從Redis或監控系統獲取

    Ok(TenantStatistics {
        total_users,
        active_users_30d,
        total_orders,
        total_revenue: total_revenue.unwrap_or_else(|| rust_decimal::Decimal::new(0, 2)),
        storage_used_mb: 0, // 從檔案系統統計
        api_calls_24h,
    })
}

/// 發送歡迎郵件
async fn send_welcome_email(
    email: &str,
    tenant_display_name: &str,
    tenant_name: &str,
) -> Result<(), Box<dyn std::error::Error>> {
    // 實際實現需要整合郵件服務 (SendGrid, AWS SES等)
    println!("發送歡迎郵件到 {} 給租戶 {}", email, tenant_display_name);
    
    // 這裡可以整合郵件模板引擎
    let email_content = format!(
        r#"
        親愛的 {} 管理員，

        歡迎加入 Coffee 管理系統！

        您的租戶資訊：
        - 租戶名稱：{}
        - 管理後台：https://{}.coffee.justit.cc
        - 管理員帳號：admin
        
        請使用您設定的密碼登入系統開始使用。

        如有任何問題，請聯絡我們的支援團隊。

        Coffee 團隊
        "#,
        tenant_display_name,
        tenant_display_name,
        tenant_name
    );

    // 實際郵件發送邏輯
    // mail_service.send_email(email, "歡迎加入 Coffee 系統", &email_content).await?;

    Ok(())
}
```

### **租戶資料模型**
```rust
// coffee-shared-lib/src/models/tenant.rs
use serde::{Deserialize, Serialize};
use sqlx::{FromRow, PgPool};
use uuid::Uuid;
use chrono::{DateTime, Utc};

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "tenant_status", rename_all = "lowercase")]
pub enum TenantStatus {
    Active,
    Suspended,
    Trial,
    Expired,
    Cancelled,
}

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Tenant {
    pub id: Uuid,
    pub name: String,                    // 租戶唯一標識 (URL用)
    pub display_name: String,           // 顯示名稱
    pub domain: Option<String>,         // 自定義域名
    pub status: TenantStatus,
    pub plan: String,                   // 方案: starter, pro, enterprise
    pub features: Vec<String>,          // 啟用功能列表
    pub max_users: Option<i32>,         // 最大使用者數
    pub max_stores: Option<i32>,        // 最大門店數
    pub storage_quota_gb: Option<i32>,  // 儲存配額 (GB)
    pub api_rate_limit: Option<i32>,    // API呼叫限制 (每小時)
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub trial_ends_at: Option<DateTime<Utc>>,
    pub billing_email: Option<String>,
    pub billing_address: Option<String>,
}

#[derive(Debug, Deserialize)]
pub struct CreateTenantRequest {
    pub id: Uuid,
    pub name: String,
    pub display_name: String,
    pub plan: String,
    pub domain: Option<String>,
    pub trial_days: Option<i32>,
}

#[derive(Debug, Deserialize)]
pub struct UpdateTenantRequest {
    pub display_name: Option<String>,
    pub domain: Option<String>,
    pub status: Option<TenantStatus>,
    pub plan: Option<String>,
    pub max_users: Option<i32>,
    pub max_stores: Option<i32>,
    pub storage_quota_gb: Option<i32>,
    pub billing_email: Option<String>,
    pub billing_address: Option<String>,
}

impl Tenant {
    /// 創建新租戶
    pub async fn create(
        pool: &PgPool,
        request: CreateTenantRequest,
    ) -> Result<Tenant, sqlx::Error> {
        let trial_ends_at = request.trial_days.map(|days| {
            Utc::now() + chrono::Duration::days(days as i64)
        });

        let features = Self::get_plan_features(&request.plan);
        let (max_users, max_stores, storage_quota_gb, api_rate_limit) = Self::get_plan_limits(&request.plan);

        let tenant = sqlx::query_as::<_, Tenant>(
            r#"
            INSERT INTO tenants (
                id, name, display_name, domain, status, plan, features,
                max_users, max_stores, storage_quota_gb, api_rate_limit, trial_ends_at
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12)
            RETURNING *
            "#,
        )
        .bind(request.id)
        .bind(request.name)
        .bind(request.display_name)
        .bind(request.domain)
        .bind(if trial_ends_at.is_some() { TenantStatus::Trial } else { TenantStatus::Active })
        .bind(request.plan)
        .bind(&features)
        .bind(max_users)
        .bind(max_stores)
        .bind(storage_quota_gb)
        .bind(api_rate_limit)
        .bind(trial_ends_at)
        .fetch_one(pool)
        .await?;

        Ok(tenant)
    }

    /// 根據ID查找租戶
    pub async fn find_by_id(
        pool: &PgPool,
        tenant_id: Uuid,
    ) -> Result<Option<Tenant>, sqlx::Error> {
        let tenant = sqlx::query_as::<_, Tenant>(
            "SELECT * FROM tenants WHERE id = $1"
        )
        .bind(tenant_id)
        .fetch_optional(pool)
        .await?;

        Ok(tenant)
    }

    /// 根據名稱查找租戶
    pub async fn find_by_name(
        pool: &PgPool,
        name: &str,
    ) -> Result<Option<Tenant>, sqlx::Error> {
        let tenant = sqlx::query_as::<_, Tenant>(
            "SELECT * FROM tenants WHERE name = $1"
        )
        .bind(name)
        .fetch_optional(pool)
        .await?;

        Ok(tenant)
    }

    /// 分頁查詢租戶
    pub async fn list(
        pool: &PgPool,
        status: Option<TenantStatus>,
        plan: Option<String>,
        page: u32,
        limit: u32,
    ) -> Result<(Vec<Tenant>, i64), sqlx::Error> {
        let offset = (page - 1) * limit;

        let mut query = "SELECT * FROM tenants WHERE 1=1".to_string();
        let mut count_query = "SELECT COUNT(*) FROM tenants WHERE 1=1".to_string();

        let mut params = vec![];
        let mut param_count = 0;

        if let Some(tenant_status) = status {
            param_count += 1;
            query.push_str(&format!(" AND status = ${}", param_count));
            count_query.push_str(&format!(" AND status = ${}", param_count));
            params.push(tenant_status.to_string());
        }

        if let Some(tenant_plan) = plan {
            param_count += 1;
            query.push_str(&format!(" AND plan = ${}", param_count));
            count_query.push_str(&format!(" AND plan = ${}", param_count));
            params.push(tenant_plan);
        }

        // 獲取總數
        let total: i64 = sqlx::query_scalar(&count_query)
            .fetch_one(pool)
            .await?;

        // 添加排序和分頁
        param_count += 1;
        query.push_str(&format!(" ORDER BY created_at DESC LIMIT ${}", param_count));
        params.push(limit.to_string());

        param_count += 1;
        query.push_str(&format!(" OFFSET ${}", param_count));
        params.push(offset.to_string());

        let tenants = sqlx::query_as::<_, Tenant>(&query)
            .fetch_all(pool)
            .await?;

        Ok((tenants, total))
    }

    /// 更新租戶
    pub async fn update(
        &mut self,
        pool: &PgPool,
        request: UpdateTenantRequest,
    ) -> Result<(), sqlx::Error> {
        let now = Utc::now();

        if let Some(display_name) = &request.display_name {
            self.display_name = display_name.clone();
        }
        if request.domain.is_some() {
            self.domain = request.domain.clone();
        }
        if let Some(status) = &request.status {
            self.status = status.clone();
        }
        if let Some(plan) = &request.plan {
            self.plan = plan.clone();
            // 更新方案對應的功能和限制
            self.features = Self::get_plan_features(plan);
            let (max_users, max_stores, storage_quota_gb, api_rate_limit) = Self::get_plan_limits(plan);
            self.max_users = max_users;
            self.max_stores = max_stores;
            self.storage_quota_gb = storage_quota_gb;
            self.api_rate_limit = api_rate_limit;
        }
        if let Some(max_users) = request.max_users {
            self.max_users = Some(max_users);
        }
        if let Some(max_stores) = request.max_stores {
            self.max_stores = Some(max_stores);
        }
        if let Some(storage_quota_gb) = request.storage_quota_gb {
            self.storage_quota_gb = Some(storage_quota_gb);
        }
        if request.billing_email.is_some() {
            self.billing_email = request.billing_email.clone();
        }
        if request.billing_address.is_some() {
            self.billing_address = request.billing_address.clone();
        }

        self.updated_at = now;

        sqlx::query(
            r#"
            UPDATE tenants 
            SET display_name = $1, domain = $2, status = $3, plan = $4, features = $5,
                max_users = $6, max_stores = $7, storage_quota_gb = $8, api_rate_limit = $9,
                billing_email = $10, billing_address = $11, updated_at = $12
            WHERE id = $13
            "#
        )
        .bind(&self.display_name)
        .bind(&self.domain)
        .bind(&self.status)
        .bind(&self.plan)
        .bind(&self.features)
        .bind(self.max_users)
        .bind(self.max_stores)
        .bind(self.storage_quota_gb)
        .bind(self.api_rate_limit)
        .bind(&self.billing_email)
        .bind(&self.billing_address)
        .bind(now)
        .bind(self.id)
        .execute(pool)
        .await?;

        Ok(())
    }

    /// 停用租戶
    pub async fn deactivate(&mut self, pool: &PgPool) -> Result<(), sqlx::Error> {
        self.status = TenantStatus::Suspended;
        self.updated_at = Utc::now();

        sqlx::query(
            "UPDATE tenants SET status = $1, updated_at = $2 WHERE id = $3"
        )
        .bind(&self.status)
        .bind(self.updated_at)
        .bind(self.id)
        .execute(pool)
        .await?;

        Ok(())
    }

    /// 檢查試用期是否過期
    pub fn is_trial_expired(&self) -> bool {
        if let Some(trial_ends_at) = self.trial_ends_at {
            Utc::now() > trial_ends_at
        } else {
            false
        }
    }

    /// 檢查是否有指定功能
    pub fn has_feature(&self, feature: &str) -> bool {
        self.features.contains(&feature.to_string())
    }

    /// 獲取方案功能
    fn get_plan_features(plan: &str) -> Vec<String> {
        match plan {
            "starter" => vec![
                "pos".to_string(),
                "customers".to_string(),
                "inventory".to_string(),
                "basic_reports".to_string(),
            ],
            "pro" => vec![
                "pos".to_string(),
                "customers".to_string(),
                "inventory".to_string(),
                "basic_reports".to_string(),
                "advanced_reports".to_string(),
                "multi_store".to_string(),
                "loyalty_program".to_string(),
                "api_access".to_string(),
            ],
            "enterprise" => vec![
                "pos".to_string(),
                "customers".to_string(),
                "inventory".to_string(),
                "basic_reports".to_string(),
                "advanced_reports".to_string(),
                "multi_store".to_string(),
                "loyalty_program".to_string(),
                "api_access".to_string(),
                "custom_integrations".to_string(),
                "priority_support".to_string(),
                "white_label".to_string(),
            ],
            _ => vec!["pos".to_string()], // 預設最基本功能
        }
    }

    /// 獲取方案限制
    fn get_plan_limits(plan: &str) -> (Option<i32>, Option<i32>, Option<i32>, Option<i32>) {
        match plan {
            "starter" => (Some(5), Some(1), Some(1), Some(1000)),
            "pro" => (Some(50), Some(10), Some(10), Some(10000)),
            "enterprise" => (None, None, Some(100), Some(100000)), // 無限制
            _ => (Some(1), Some(1), Some(1), Some(100)),
        }
    }
}
```

---

**🏢 租戶管理服務開發完成！**

包含：
- ✅ **完整租戶CRUD**: 創建/查詢/更新/停用
- ✅ **多方案支援**: starter/pro/enterprise不同功能限制
- ✅ **租戶隔離**: 獨立資料庫Schema自動建立
- ✅ **統計儀表板**: 使用者數/訂單數/營收統計
- ✅ **試用機制**: 自動試用期管理
- ✅ **預設資料**: 自動建立範例商品和分類

**下一步**: 繼續開發平台監控系統？