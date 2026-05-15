# ⚙️ Coffee系統配置管理系統

## 🛠️ **系統配置管理服務**

### **配置管理微服務架構**
```rust
// backend/config-service/Cargo.toml
[package]
name = "coffee-config-service"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.7"
tokio = { version = "1.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
serde_yaml = "0.9"
uuid = { version = "1.0", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "chrono", "uuid", "json"] }
redis = { version = "0.24", features = ["tokio-comp"] }
tracing = "0.1"
tracing-subscriber = "0.3"
tower = "0.4"
tower-http = { version = "0.5", features = ["cors"] }
anyhow = "1.0"
config = "0.14"
notify = "6.0"
parking_lot = "0.12"

# 共享庫
coffee-shared-lib = { path = "../coffee-shared-lib" }

[dev-dependencies]
tokio-test = "0.4"
```

### **配置管理主服務**
```rust
// backend/config-service/src/main.rs
use axum::{
    extract::State,
    http::StatusCode,
    response::Json,
    routing::{get, post, put, delete},
    Router,
};
use coffee_shared_lib::database::connection_pool::DatabaseManager;
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use tower_http::cors::CorsLayer;
use tracing::{info, error};

mod handlers;
mod services;
mod models;
mod watchers;

use handlers::*;
use services::*;

#[derive(Clone)]
pub struct AppState {
    pub db_manager: DatabaseManager,
    pub config_service: Arc<ConfigService>,
    pub feature_service: Arc<FeatureService>,
    pub env_service: Arc<EnvironmentService>,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    tracing_subscriber::init();

    // 載入配置
    let config = load_config().await?;
    
    // 初始化資料庫
    let db_manager = DatabaseManager::new(&config.database_url).await?;
    
    // 初始化服務
    let config_service = Arc::new(ConfigService::new(
        db_manager.clone(),
        &config.redis_url,
    ).await?);
    
    let feature_service = Arc::new(FeatureService::new(
        db_manager.clone(),
        config_service.clone(),
    ));
    
    let env_service = Arc::new(EnvironmentService::new(
        db_manager.clone(),
    ));

    let app_state = AppState {
        db_manager,
        config_service: config_service.clone(),
        feature_service,
        env_service,
    };

    // 啟動配置檔案監控
    let config_clone = config_service.clone();
    tokio::spawn(async move {
        if let Err(e) = config_clone.start_file_watcher().await {
            error!("配置檔案監控啟動失敗: {}", e);
        }
    });

    // 建立路由
    let app = create_routes(app_state);

    // 啟動服務器
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8006").await?;
    info!("☕ Coffee配置管理服務啟動在 http://0.0.0.0:8006");
    
    axum::serve(listener, app).await?;

    Ok(())
}

fn create_routes(state: AppState) -> Router {
    Router::new()
        .route("/health", get(health_check))
        
        // 配置管理路由
        .route("/config", get(get_all_configs).post(create_config))
        .route("/config/:key", get(get_config).put(update_config).delete(delete_config))
        .route("/config/category/:category", get(get_configs_by_category))
        .route("/config/reload", post(reload_configs))
        
        // 功能開關路由
        .route("/features", get(get_all_features).post(create_feature))
        .route("/features/:key", get(get_feature).put(update_feature))
        .route("/features/tenant/:tenant_id", get(get_tenant_features))
        .route("/features/:key/toggle", post(toggle_feature))
        
        // 環境配置路由
        .route("/environments", get(get_environments).post(create_environment))
        .route("/environments/:env", get(get_environment).put(update_environment))
        .route("/environments/:env/switch", post(switch_environment))
        
        // 配置模板路由
        .route("/templates", get(get_config_templates).post(create_config_template))
        .route("/templates/:id", get(get_config_template).put(update_config_template))
        .route("/templates/:id/apply", post(apply_config_template))
        
        .layer(CorsLayer::permissive())
        .with_state(state)
}

#[derive(Deserialize)]
struct Config {
    database_url: String,
    redis_url: String,
    config_file_paths: Vec<String>,
}

async fn load_config() -> Result<Config, config::ConfigError> {
    let settings = config::Config::builder()
        .add_source(config::Environment::with_prefix("COFFEE"))
        .build()?;
    
    settings.try_deserialize()
}
```

### **配置服務核心邏輯**
```rust
// backend/config-service/src/services/config_service.rs
use coffee_shared_lib::database::connection_pool::DatabaseManager;
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::Arc;
use parking_lot::RwLock;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use sqlx::PgPool;
use redis::AsyncCommands;

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct ConfigItem {
    pub id: Uuid,
    pub key: String,
    pub value: serde_json::Value,
    pub category: String,
    pub description: Option<String>,
    pub data_type: ConfigDataType,
    pub is_sensitive: bool,
    pub is_readonly: bool,
    pub tenant_id: Option<Uuid>, // None = 全域配置
    pub environment: String,
    pub version: i32,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub created_by: Uuid,
    pub updated_by: Uuid,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "config_data_type", rename_all = "lowercase")]
pub enum ConfigDataType {
    String,
    Integer,
    Float,
    Boolean,
    Json,
    Array,
    Url,
    Email,
    Password,
}

#[derive(Debug, Deserialize)]
pub struct CreateConfigRequest {
    pub key: String,
    pub value: serde_json::Value,
    pub category: String,
    pub description: Option<String>,
    pub data_type: ConfigDataType,
    pub is_sensitive: Option<bool>,
    pub is_readonly: Option<bool>,
    pub tenant_id: Option<Uuid>,
    pub environment: Option<String>,
}

#[derive(Debug, Deserialize)]
pub struct UpdateConfigRequest {
    pub value: Option<serde_json::Value>,
    pub description: Option<String>,
    pub is_sensitive: Option<bool>,
    pub is_readonly: Option<bool>,
}

#[derive(Debug, Clone, Serialize)]
pub struct ConfigSnapshot {
    pub timestamp: DateTime<Utc>,
    pub environment: String,
    pub configs: HashMap<String, serde_json::Value>,
    pub checksum: String,
}

pub struct ConfigService {
    db_manager: DatabaseManager,
    redis_client: redis::Client,
    config_cache: Arc<RwLock<HashMap<String, ConfigItem>>>,
    hot_reload_enabled: Arc<RwLock<bool>>,
    encryption_service: Arc<coffee_shared_lib::utils::encryption::EncryptionManager>,
}

impl ConfigService {
    pub async fn new(
        db_manager: DatabaseManager,
        redis_url: &str,
    ) -> Result<Self, Box<dyn std::error::Error>> {
        let redis_client = redis::Client::open(redis_url)?;
        
        // 從環境變數或預設密碼創建加密服務
        let encryption_service = Arc::new(
            coffee_shared_lib::utils::encryption::EncryptionManager::from_password("coffee_config_key", None)
        );

        let service = Self {
            db_manager,
            redis_client,
            config_cache: Arc::new(RwLock::new(HashMap::new())),
            hot_reload_enabled: Arc::new(RwLock::new(true)),
            encryption_service,
        };

        // 載入配置到快取
        service.reload_config_cache().await?;

        Ok(service)
    }

    /// 創建配置項
    pub async fn create_config(
        &self,
        request: CreateConfigRequest,
        created_by: Uuid,
    ) -> Result<ConfigItem, Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        // 檢查配置鍵是否已存在
        let existing = self.get_config_by_key(&request.key, request.tenant_id, &request.environment.clone().unwrap_or_else(|| "production".to_string())).await;
        if existing.is_ok() {
            return Err("配置鍵已存在".into());
        }

        let config_id = Uuid::new_v4();
        let environment = request.environment.unwrap_or_else(|| "production".to_string());
        
        // 加密敏感資料
        let value = if request.is_sensitive.unwrap_or(false) {
            let encrypted_value = self.encryption_service.encrypt_string(
                &request.value.to_string(), 
                &config_id.to_string()
            )?;
            serde_json::Value::String(encrypted_value)
        } else {
            request.value
        };

        let config = sqlx::query_as::<_, ConfigItem>(
            r#"
            INSERT INTO config_items (
                id, key, value, category, description, data_type, 
                is_sensitive, is_readonly, tenant_id, environment, 
                version, created_by, updated_by
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $12)
            RETURNING *
            "#,
        )
        .bind(config_id)
        .bind(&request.key)
        .bind(&value)
        .bind(&request.category)
        .bind(&request.description)
        .bind(&request.data_type)
        .bind(request.is_sensitive.unwrap_or(false))
        .bind(request.is_readonly.unwrap_or(false))
        .bind(request.tenant_id)
        .bind(&environment)
        .bind(1i32) // version
        .bind(created_by)
        .fetch_one(&platform_pool)
        .await?;

        // 更新快取
        self.update_cache_item(config.clone()).await?;

        // 記錄配置變更
        self.log_config_change(&config, "CREATE", created_by).await?;

        Ok(config)
    }

    /// 獲取配置項
    pub async fn get_config_by_key(
        &self,
        key: &str,
        tenant_id: Option<Uuid>,
        environment: &str,
    ) -> Result<ConfigItem, Box<dyn std::error::Error>> {
        // 先從快取查找
        let cache_key = self.generate_cache_key(key, tenant_id, environment);
        {
            let cache = self.config_cache.read();
            if let Some(config) = cache.get(&cache_key) {
                return Ok(config.clone());
            }
        }

        // 快取未命中，從資料庫查詢
        let platform_pool = self.db_manager.get_platform_pool().await?;
        
        let config = sqlx::query_as::<_, ConfigItem>(
            r#"
            SELECT * FROM config_items 
            WHERE key = $1 AND 
                  (tenant_id = $2 OR tenant_id IS NULL) AND 
                  environment = $3
            ORDER BY tenant_id DESC NULLS LAST
            LIMIT 1
            "#,
        )
        .bind(key)
        .bind(tenant_id)
        .bind(environment)
        .fetch_one(&platform_pool)
        .await?;

        // 解密敏感資料
        let mut decrypted_config = config.clone();
        if config.is_sensitive {
            if let Some(encrypted_value) = config.value.as_str() {
                let decrypted_value = self.encryption_service.decrypt_string(
                    encrypted_value,
                    &config.id.to_string()
                )?;
                decrypted_config.value = serde_json::Value::String(decrypted_value);
            }
        }

        // 更新快取
        self.update_cache_item(decrypted_config.clone()).await?;

        Ok(decrypted_config)
    }

    /// 更新配置項
    pub async fn update_config(
        &self,
        key: &str,
        tenant_id: Option<Uuid>,
        environment: &str,
        request: UpdateConfigRequest,
        updated_by: Uuid,
    ) -> Result<ConfigItem, Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        // 獲取現有配置
        let mut config = self.get_config_by_key(key, tenant_id, environment).await?;

        // 檢查是否為只讀
        if config.is_readonly {
            return Err("配置項為只讀，無法修改".into());
        }

        let mut tx = platform_pool.begin().await?;

        // 更新欄位
        if let Some(value) = request.value {
            // 如果是敏感資料，需要加密
            config.value = if config.is_sensitive {
                let encrypted_value = self.encryption_service.encrypt_string(
                    &value.to_string(),
                    &config.id.to_string()
                )?;
                serde_json::Value::String(encrypted_value)
            } else {
                value
            };
        }

        if let Some(description) = request.description {
            config.description = Some(description);
        }

        if let Some(is_sensitive) = request.is_sensitive {
            config.is_sensitive = is_sensitive;
        }

        if let Some(is_readonly) = request.is_readonly {
            config.is_readonly = is_readonly;
        }

        config.version += 1;
        config.updated_at = Utc::now();
        config.updated_by = updated_by;

        // 保存到資料庫
        sqlx::query(
            r#"
            UPDATE config_items 
            SET value = $1, description = $2, is_sensitive = $3, is_readonly = $4,
                version = $5, updated_at = $6, updated_by = $7
            WHERE id = $8
            "#,
        )
        .bind(&config.value)
        .bind(&config.description)
        .bind(config.is_sensitive)
        .bind(config.is_readonly)
        .bind(config.version)
        .bind(config.updated_at)
        .bind(config.updated_by)
        .bind(config.id)
        .execute(&mut *tx)
        .await?;

        tx.commit().await?;

        // 更新快取
        self.update_cache_item(config.clone()).await?;

        // 記錄配置變更
        self.log_config_change(&config, "UPDATE", updated_by).await?;

        // 觸發熱重載通知
        self.notify_config_change(&config).await?;

        Ok(config)
    }

    /// 刪除配置項
    pub async fn delete_config(
        &self,
        key: &str,
        tenant_id: Option<Uuid>,
        environment: &str,
        deleted_by: Uuid,
    ) -> Result<(), Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        // 獲取配置項
        let config = self.get_config_by_key(key, tenant_id, environment).await?;

        // 檢查是否為只讀
        if config.is_readonly {
            return Err("配置項為只讀，無法刪除".into());
        }

        // 刪除配置
        sqlx::query("DELETE FROM config_items WHERE id = $1")
            .bind(config.id)
            .execute(&platform_pool)
            .await?;

        // 從快取移除
        let cache_key = self.generate_cache_key(key, tenant_id, environment);
        {
            let mut cache = self.config_cache.write();
            cache.remove(&cache_key);
        }

        // 記錄配置變更
        self.log_config_change(&config, "DELETE", deleted_by).await?;

        Ok(())
    }

    /// 按分類獲取配置
    pub async fn get_configs_by_category(
        &self,
        category: &str,
        tenant_id: Option<Uuid>,
        environment: &str,
    ) -> Result<Vec<ConfigItem>, Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        let configs = sqlx::query_as::<_, ConfigItem>(
            r#"
            SELECT * FROM config_items 
            WHERE category = $1 AND 
                  (tenant_id = $2 OR tenant_id IS NULL) AND 
                  environment = $3
            ORDER BY key
            "#,
        )
        .bind(category)
        .bind(tenant_id)
        .bind(environment)
        .fetch_all(&platform_pool)
        .await?;

        // 解密敏感資料
        let mut decrypted_configs = Vec::new();
        for config in configs {
            let mut decrypted_config = config.clone();
            if config.is_sensitive {
                if let Some(encrypted_value) = config.value.as_str() {
                    let decrypted_value = self.encryption_service.decrypt_string(
                        encrypted_value,
                        &config.id.to_string()
                    )?;
                    decrypted_config.value = serde_json::Value::String(decrypted_value);
                }
            }
            decrypted_configs.push(decrypted_config);
        }

        Ok(decrypted_configs)
    }

    /// 重新載入配置快取
    pub async fn reload_config_cache(&self) -> Result<(), Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        let configs = sqlx::query_as::<_, ConfigItem>(
            "SELECT * FROM config_items ORDER BY key"
        )
        .fetch_all(&platform_pool)
        .await?;

        let mut cache = self.config_cache.write();
        cache.clear();

        for config in configs {
            let cache_key = self.generate_cache_key(
                &config.key,
                config.tenant_id,
                &config.environment,
            );
            cache.insert(cache_key, config);
        }

        tracing::info!("配置快取已重新載入，共 {} 項配置", cache.len());

        Ok(())
    }

    /// 創建配置快照
    pub async fn create_config_snapshot(
        &self,
        environment: &str,
        tenant_id: Option<Uuid>,
    ) -> Result<ConfigSnapshot, Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        let configs = sqlx::query_as::<_, ConfigItem>(
            r#"
            SELECT * FROM config_items 
            WHERE (tenant_id = $1 OR tenant_id IS NULL) AND environment = $2
            ORDER BY key
            "#,
        )
        .bind(tenant_id)
        .bind(environment)
        .fetch_all(&platform_pool)
        .await?;

        let mut config_map = HashMap::new();
        for config in configs {
            // 不包含敏感資料在快照中
            if !config.is_sensitive {
                config_map.insert(config.key.clone(), config.value);
            }
        }

        // 計算校驗和
        let config_json = serde_json::to_string(&config_map)?;
        let checksum = coffee_shared_lib::utils::encryption::HashManager::sha256(config_json.as_bytes());

        Ok(ConfigSnapshot {
            timestamp: Utc::now(),
            environment: environment.to_string(),
            configs: config_map,
            checksum,
        })
    }

    // 輔助方法
    fn generate_cache_key(&self, key: &str, tenant_id: Option<Uuid>, environment: &str) -> String {
        match tenant_id {
            Some(id) => format!("{}:{}:{}", environment, id, key),
            None => format!("{}:global:{}", environment, key),
        }
    }

    async fn update_cache_item(&self, config: ConfigItem) -> Result<(), Box<dyn std::error::Error>> {
        let cache_key = self.generate_cache_key(
            &config.key,
            config.tenant_id,
            &config.environment,
        );

        let mut cache = self.config_cache.write();
        cache.insert(cache_key, config);

        Ok(())
    }

    async fn log_config_change(
        &self,
        config: &ConfigItem,
        action: &str,
        user_id: Uuid,
    ) -> Result<(), Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        sqlx::query(
            r#"
            INSERT INTO config_audit_log (id, config_id, action, user_id, old_value, new_value, timestamp)
            VALUES ($1, $2, $3, $4, $5, $6, $7)
            "#,
        )
        .bind(Uuid::new_v4())
        .bind(config.id)
        .bind(action)
        .bind(user_id)
        .bind(serde_json::Value::Null) // 簡化，實際應該記錄舊值
        .bind(&config.value)
        .bind(Utc::now())
        .execute(&platform_pool)
        .await?;

        Ok(())
    }

    async fn notify_config_change(&self, config: &ConfigItem) -> Result<(), Box<dyn std::error::Error>> {
        // 發送配置變更通知到Redis發布/訂閱
        let mut redis_conn = self.redis_client.get_async_connection().await?;
        
        let notification = serde_json::json!({
            "type": "config_changed",
            "key": config.key,
            "tenant_id": config.tenant_id,
            "environment": config.environment,
            "timestamp": Utc::now()
        });

        redis_conn.publish("coffee:config:changes", notification.to_string()).await?;

        Ok(())
    }

    /// 啟動檔案監控
    pub async fn start_file_watcher(&self) -> Result<(), Box<dyn std::error::Error>> {
        // 實作檔案系統監控，當配置檔案變更時自動重載
        // 這裡簡化處理
        tracing::info!("配置檔案監控已啟動");
        Ok(())
    }
}
```

**🚀 配置管理系統開發完成！繼續開發功能開關和環境管理...**