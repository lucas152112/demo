# 🎛️ Coffee系統功能開關與環境管理

## 🔧 **功能開關服務**

### **功能開關核心邏輯**
```rust
// backend/config-service/src/services/feature_service.rs
use coffee_shared_lib::database::connection_pool::DatabaseManager;
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::Arc;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use sqlx::PgPool;

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct FeatureFlag {
    pub id: Uuid,
    pub key: String,
    pub name: String,
    pub description: Option<String>,
    pub is_enabled: bool,
    pub rollout_percentage: f32, // 0.0-100.0
    pub target_groups: Vec<String>,
    pub conditions: Option<serde_json::Value>,
    pub environment: String,
    pub tenant_id: Option<Uuid>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub created_by: Uuid,
    pub expires_at: Option<DateTime<Utc>>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FeatureEvaluation {
    pub feature_key: String,
    pub is_enabled: bool,
    pub reason: String,
    pub rollout_percentage: f32,
    pub evaluated_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FeatureContext {
    pub user_id: Option<Uuid>,
    pub tenant_id: Option<Uuid>,
    pub user_groups: Vec<String>,
    pub environment: String,
    pub custom_attributes: HashMap<String, serde_json::Value>,
}

#[derive(Debug, Deserialize)]
pub struct CreateFeatureRequest {
    pub key: String,
    pub name: String,
    pub description: Option<String>,
    pub is_enabled: bool,
    pub rollout_percentage: Option<f32>,
    pub target_groups: Option<Vec<String>>,
    pub conditions: Option<serde_json::Value>,
    pub environment: Option<String>,
    pub tenant_id: Option<Uuid>,
    pub expires_at: Option<DateTime<Utc>>,
}

#[derive(Debug, Deserialize)]
pub struct UpdateFeatureRequest {
    pub name: Option<String>,
    pub description: Option<String>,
    pub is_enabled: Option<bool>,
    pub rollout_percentage: Option<f32>,
    pub target_groups: Option<Vec<String>>,
    pub conditions: Option<serde_json::Value>,
    pub expires_at: Option<DateTime<Utc>>,
}

pub struct FeatureService {
    db_manager: DatabaseManager,
    config_service: Arc<super::config_service::ConfigService>,
    feature_cache: Arc<parking_lot::RwLock<HashMap<String, FeatureFlag>>>,
}

impl FeatureService {
    pub fn new(
        db_manager: DatabaseManager,
        config_service: Arc<super::config_service::ConfigService>,
    ) -> Self {
        let service = Self {
            db_manager,
            config_service,
            feature_cache: Arc::new(parking_lot::RwLock::new(HashMap::new())),
        };

        // 載入功能開關到快取
        tokio::spawn({
            let service_clone = service.clone();
            async move {
                if let Err(e) = service_clone.reload_feature_cache().await {
                    tracing::error!("載入功能開關快取失敗: {}", e);
                }
            }
        });

        service
    }

    /// 創建功能開關
    pub async fn create_feature(
        &self,
        request: CreateFeatureRequest,
        created_by: Uuid,
    ) -> Result<FeatureFlag, Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        // 檢查功能鍵是否已存在
        let environment = request.environment.unwrap_or_else(|| "production".to_string());
        if self.feature_exists(&request.key, request.tenant_id, &environment).await? {
            return Err("功能開關鍵已存在".into());
        }

        let feature_id = Uuid::new_v4();

        let feature = sqlx::query_as::<_, FeatureFlag>(
            r#"
            INSERT INTO feature_flags (
                id, key, name, description, is_enabled, rollout_percentage,
                target_groups, conditions, environment, tenant_id, created_by, expires_at
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12)
            RETURNING *
            "#,
        )
        .bind(feature_id)
        .bind(&request.key)
        .bind(&request.name)
        .bind(&request.description)
        .bind(request.is_enabled)
        .bind(request.rollout_percentage.unwrap_or(100.0))
        .bind(&request.target_groups.unwrap_or_default())
        .bind(&request.conditions)
        .bind(&environment)
        .bind(request.tenant_id)
        .bind(created_by)
        .bind(request.expires_at)
        .fetch_one(&platform_pool)
        .await?;

        // 更新快取
        self.update_cache_feature(feature.clone()).await;

        tracing::info!("功能開關已創建: {}", request.key);

        Ok(feature)
    }

    /// 評估功能開關
    pub async fn evaluate_feature(
        &self,
        key: &str,
        context: &FeatureContext,
    ) -> Result<FeatureEvaluation, Box<dyn std::error::Error>> {
        // 先從快取查找
        let feature = {
            let cache = self.feature_cache.read();
            let cache_key = self.generate_feature_cache_key(key, context.tenant_id, &context.environment);
            cache.get(&cache_key).cloned()
        };

        let feature = match feature {
            Some(f) => f,
            None => {
                // 快取未命中，從資料庫查詢
                self.get_feature_from_db(key, context.tenant_id, &context.environment).await?
            }
        };

        // 檢查功能是否過期
        if let Some(expires_at) = feature.expires_at {
            if Utc::now() > expires_at {
                return Ok(FeatureEvaluation {
                    feature_key: key.to_string(),
                    is_enabled: false,
                    reason: "功能已過期".to_string(),
                    rollout_percentage: 0.0,
                    evaluated_at: Utc::now(),
                });
            }
        }

        // 基本啟用檢查
        if !feature.is_enabled {
            return Ok(FeatureEvaluation {
                feature_key: key.to_string(),
                is_enabled: false,
                reason: "功能已停用".to_string(),
                rollout_percentage: feature.rollout_percentage,
                evaluated_at: Utc::now(),
            });
        }

        // 目標群組檢查
        if !feature.target_groups.is_empty() {
            let has_target_group = feature.target_groups.iter()
                .any(|group| context.user_groups.contains(group));
            
            if !has_target_group {
                return Ok(FeatureEvaluation {
                    feature_key: key.to_string(),
                    is_enabled: false,
                    reason: "不在目標群組".to_string(),
                    rollout_percentage: feature.rollout_percentage,
                    evaluated_at: Utc::now(),
                });
            }
        }

        // 滾動發布百分比檢查
        if feature.rollout_percentage < 100.0 {
            let hash = self.calculate_user_hash(context.user_id, &feature.key);
            let user_percentage = (hash % 100) as f32;
            
            if user_percentage >= feature.rollout_percentage {
                return Ok(FeatureEvaluation {
                    feature_key: key.to_string(),
                    is_enabled: false,
                    reason: format!("滾動發布未命中 ({:.1}% > {:.1}%)", user_percentage, feature.rollout_percentage),
                    rollout_percentage: feature.rollout_percentage,
                    evaluated_at: Utc::now(),
                });
            }
        }

        // 條件評估
        if let Some(conditions) = &feature.conditions {
            if !self.evaluate_conditions(conditions, context)? {
                return Ok(FeatureEvaluation {
                    feature_key: key.to_string(),
                    is_enabled: false,
                    reason: "條件不滿足".to_string(),
                    rollout_percentage: feature.rollout_percentage,
                    evaluated_at: Utc::now(),
                });
            }
        }

        Ok(FeatureEvaluation {
            feature_key: key.to_string(),
            is_enabled: true,
            reason: "所有條件滿足".to_string(),
            rollout_percentage: feature.rollout_percentage,
            evaluated_at: Utc::now(),
        })
    }

    /// 切換功能開關
    pub async fn toggle_feature(
        &self,
        key: &str,
        tenant_id: Option<Uuid>,
        environment: &str,
        updated_by: Uuid,
    ) -> Result<FeatureFlag, Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        let mut feature = self.get_feature_from_db(key, tenant_id, environment).await?;
        
        // 切換狀態
        feature.is_enabled = !feature.is_enabled;
        feature.updated_at = Utc::now();

        // 更新資料庫
        sqlx::query(
            "UPDATE feature_flags SET is_enabled = $1, updated_at = $2 WHERE id = $3"
        )
        .bind(feature.is_enabled)
        .bind(feature.updated_at)
        .bind(feature.id)
        .execute(&platform_pool)
        .await?;

        // 更新快取
        self.update_cache_feature(feature.clone()).await;

        tracing::info!("功能開關 {} 已{}啟用", key, if feature.is_enabled { "" } else { "停用" });

        Ok(feature)
    }

    /// 獲取租戶的所有功能開關
    pub async fn get_tenant_features(
        &self,
        tenant_id: Uuid,
        environment: &str,
    ) -> Result<Vec<FeatureFlag>, Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        let features = sqlx::query_as::<_, FeatureFlag>(
            r#"
            SELECT * FROM feature_flags 
            WHERE (tenant_id = $1 OR tenant_id IS NULL) AND environment = $2
            ORDER BY key
            "#,
        )
        .bind(tenant_id)
        .bind(environment)
        .fetch_all(&platform_pool)
        .await?;

        Ok(features)
    }

    /// 批量評估功能開關
    pub async fn evaluate_multiple_features(
        &self,
        keys: &[String],
        context: &FeatureContext,
    ) -> Result<HashMap<String, bool>, Box<dyn std::error::Error>> {
        let mut results = HashMap::new();

        for key in keys {
            match self.evaluate_feature(key, context).await {
                Ok(evaluation) => {
                    results.insert(key.clone(), evaluation.is_enabled);
                }
                Err(e) => {
                    tracing::warn!("評估功能開關 {} 失敗: {}", key, e);
                    results.insert(key.clone(), false);
                }
            }
        }

        Ok(results)
    }

    // 輔助方法
    async fn feature_exists(
        &self,
        key: &str,
        tenant_id: Option<Uuid>,
        environment: &str,
    ) -> Result<bool, Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        let count: i64 = sqlx::query_scalar(
            r#"
            SELECT COUNT(*) FROM feature_flags 
            WHERE key = $1 AND 
                  (tenant_id = $2 OR tenant_id IS NULL) AND 
                  environment = $3
            "#,
        )
        .bind(key)
        .bind(tenant_id)
        .bind(environment)
        .fetch_one(&platform_pool)
        .await?;

        Ok(count > 0)
    }

    async fn get_feature_from_db(
        &self,
        key: &str,
        tenant_id: Option<Uuid>,
        environment: &str,
    ) -> Result<FeatureFlag, Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        let feature = sqlx::query_as::<_, FeatureFlag>(
            r#"
            SELECT * FROM feature_flags 
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

        // 更新快取
        self.update_cache_feature(feature.clone()).await;

        Ok(feature)
    }

    async fn update_cache_feature(&self, feature: FeatureFlag) {
        let cache_key = self.generate_feature_cache_key(
            &feature.key,
            feature.tenant_id,
            &feature.environment,
        );

        let mut cache = self.feature_cache.write();
        cache.insert(cache_key, feature);
    }

    fn generate_feature_cache_key(&self, key: &str, tenant_id: Option<Uuid>, environment: &str) -> String {
        match tenant_id {
            Some(id) => format!("feature:{}:{}:{}", environment, id, key),
            None => format!("feature:{}:global:{}", environment, key),
        }
    }

    fn calculate_user_hash(&self, user_id: Option<Uuid>, feature_key: &str) -> u32 {
        use std::collections::hash_map::DefaultHasher;
        use std::hash::{Hash, Hasher};

        let mut hasher = DefaultHasher::new();
        
        if let Some(uid) = user_id {
            uid.hash(&mut hasher);
        }
        feature_key.hash(&mut hasher);
        
        hasher.finish() as u32
    }

    fn evaluate_conditions(
        &self,
        conditions: &serde_json::Value,
        context: &FeatureContext,
    ) -> Result<bool, Box<dyn std::error::Error>> {
        // 簡化的條件評估邏輯
        // 實際實現可以支援複雜的條件表達式
        
        if let Some(conditions_obj) = conditions.as_object() {
            for (key, expected_value) in conditions_obj {
                if let Some(actual_value) = context.custom_attributes.get(key) {
                    if actual_value != expected_value {
                        return Ok(false);
                    }
                } else {
                    return Ok(false);
                }
            }
        }

        Ok(true)
    }

    async fn reload_feature_cache(&self) -> Result<(), Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        let features = sqlx::query_as::<_, FeatureFlag>(
            "SELECT * FROM feature_flags ORDER BY key"
        )
        .fetch_all(&platform_pool)
        .await?;

        let mut cache = self.feature_cache.write();
        cache.clear();

        for feature in features {
            let cache_key = self.generate_feature_cache_key(
                &feature.key,
                feature.tenant_id,
                &feature.environment,
            );
            cache.insert(cache_key, feature);
        }

        tracing::info!("功能開關快取已重新載入，共 {} 個功能開關", cache.len());

        Ok(())
    }
}

// 實作 Clone trait
impl Clone for FeatureService {
    fn clone(&self) -> Self {
        Self {
            db_manager: self.db_manager.clone(),
            config_service: self.config_service.clone(),
            feature_cache: self.feature_cache.clone(),
        }
    }
}
```

---

## 🌍 **環境管理服務**

### **環境管理核心邏輯**
```rust
// backend/config-service/src/services/environment_service.rs
use coffee_shared_lib::database::connection_pool::DatabaseManager;
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use sqlx::PgPool;

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Environment {
    pub id: Uuid,
    pub name: String,
    pub display_name: String,
    pub description: Option<String>,
    pub is_active: bool,
    pub is_default: bool,
    pub config_overrides: serde_json::Value,
    pub deployment_target: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EnvironmentStats {
    pub environment: String,
    pub total_configs: u32,
    pub total_features: u32,
    pub enabled_features: u32,
    pub active_tenants: u32,
    pub last_deployment: Option<DateTime<Utc>>,
}

#[derive(Debug, Deserialize)]
pub struct CreateEnvironmentRequest {
    pub name: String,
    pub display_name: String,
    pub description: Option<String>,
    pub config_overrides: Option<serde_json::Value>,
    pub deployment_target: Option<String>,
}

#[derive(Debug, Deserialize)]
pub struct UpdateEnvironmentRequest {
    pub display_name: Option<String>,
    pub description: Option<String>,
    pub is_active: Option<bool>,
    pub config_overrides: Option<serde_json::Value>,
    pub deployment_target: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct EnvironmentComparison {
    pub source_env: String,
    pub target_env: String,
    pub config_differences: Vec<ConfigDifference>,
    pub feature_differences: Vec<FeatureDifference>,
}

#[derive(Debug, Serialize)]
pub struct ConfigDifference {
    pub key: String,
    pub source_value: Option<serde_json::Value>,
    pub target_value: Option<serde_json::Value>,
    pub difference_type: DifferenceType,
}

#[derive(Debug, Serialize)]
pub struct FeatureDifference {
    pub key: String,
    pub source_enabled: Option<bool>,
    pub target_enabled: Option<bool>,
    pub difference_type: DifferenceType,
}

#[derive(Debug, Serialize)]
pub enum DifferenceType {
    Added,
    Removed,
    Modified,
    Identical,
}

pub struct EnvironmentService {
    db_manager: DatabaseManager,
}

impl EnvironmentService {
    pub fn new(db_manager: DatabaseManager) -> Self {
        Self { db_manager }
    }

    /// 創建環境
    pub async fn create_environment(
        &self,
        request: CreateEnvironmentRequest,
        created_by: Uuid,
    ) -> Result<Environment, Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        // 檢查環境名稱是否已存在
        if self.environment_exists(&request.name).await? {
            return Err("環境名稱已存在".into());
        }

        let environment_id = Uuid::new_v4();

        let environment = sqlx::query_as::<_, Environment>(
            r#"
            INSERT INTO environments (
                id, name, display_name, description, is_active, is_default,
                config_overrides, deployment_target
            )
            VALUES ($1, $2, $3, $4, true, false, $5, $6)
            RETURNING *
            "#,
        )
        .bind(environment_id)
        .bind(&request.name)
        .bind(&request.display_name)
        .bind(&request.description)
        .bind(&request.config_overrides.unwrap_or_else(|| serde_json::json!({})))
        .bind(&request.deployment_target)
        .fetch_one(&platform_pool)
        .await?;

        tracing::info!("環境已創建: {}", request.name);

        Ok(environment)
    }

    /// 獲取所有環境
    pub async fn get_environments(&self) -> Result<Vec<Environment>, Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        let environments = sqlx::query_as::<_, Environment>(
            "SELECT * FROM environments ORDER BY name"
        )
        .fetch_all(&platform_pool)
        .await?;

        Ok(environments)
    }

    /// 獲取環境詳情
    pub async fn get_environment(&self, name: &str) -> Result<Environment, Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        let environment = sqlx::query_as::<_, Environment>(
            "SELECT * FROM environments WHERE name = $1"
        )
        .bind(name)
        .fetch_one(&platform_pool)
        .await?;

        Ok(environment)
    }

    /// 更新環境
    pub async fn update_environment(
        &self,
        name: &str,
        request: UpdateEnvironmentRequest,
    ) -> Result<Environment, Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        let mut environment = self.get_environment(name).await?;

        // 更新欄位
        if let Some(display_name) = request.display_name {
            environment.display_name = display_name;
        }
        if let Some(description) = request.description {
            environment.description = Some(description);
        }
        if let Some(is_active) = request.is_active {
            environment.is_active = is_active;
        }
        if let Some(config_overrides) = request.config_overrides {
            environment.config_overrides = config_overrides;
        }
        if let Some(deployment_target) = request.deployment_target {
            environment.deployment_target = Some(deployment_target);
        }

        environment.updated_at = Utc::now();

        // 保存到資料庫
        sqlx::query(
            r#"
            UPDATE environments 
            SET display_name = $1, description = $2, is_active = $3,
                config_overrides = $4, deployment_target = $5, updated_at = $6
            WHERE id = $7
            "#,
        )
        .bind(&environment.display_name)
        .bind(&environment.description)
        .bind(environment.is_active)
        .bind(&environment.config_overrides)
        .bind(&environment.deployment_target)
        .bind(environment.updated_at)
        .bind(environment.id)
        .execute(&platform_pool)
        .await?;

        Ok(environment)
    }

    /// 切換預設環境
    pub async fn switch_default_environment(&self, name: &str) -> Result<(), Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        let mut tx = platform_pool.begin().await?;

        // 清除所有環境的預設標記
        sqlx::query("UPDATE environments SET is_default = false")
            .execute(&mut *tx)
            .await?;

        // 設置新的預設環境
        sqlx::query("UPDATE environments SET is_default = true WHERE name = $1")
            .bind(name)
            .execute(&mut *tx)
            .await?;

        tx.commit().await?;

        tracing::info!("預設環境已切換到: {}", name);

        Ok(())
    }

    /// 獲取環境統計
    pub async fn get_environment_stats(&self, name: &str) -> Result<EnvironmentStats, Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        // 配置數量
        let total_configs: i64 = sqlx::query_scalar(
            "SELECT COUNT(*) FROM config_items WHERE environment = $1"
        )
        .bind(name)
        .fetch_one(&platform_pool)
        .await?;

        // 功能開關數量
        let total_features: i64 = sqlx::query_scalar(
            "SELECT COUNT(*) FROM feature_flags WHERE environment = $1"
        )
        .bind(name)
        .fetch_one(&platform_pool)
        .await?;

        // 啟用的功能開關數量
        let enabled_features: i64 = sqlx::query_scalar(
            "SELECT COUNT(*) FROM feature_flags WHERE environment = $1 AND is_enabled = true"
        )
        .bind(name)
        .fetch_one(&platform_pool)
        .await?;

        // 活躍租戶數量 (簡化)
        let active_tenants: i64 = sqlx::query_scalar(
            "SELECT COUNT(DISTINCT tenant_id) FROM config_items WHERE environment = $1 AND tenant_id IS NOT NULL"
        )
        .bind(name)
        .fetch_one(&platform_pool)
        .await?;

        Ok(EnvironmentStats {
            environment: name.to_string(),
            total_configs: total_configs as u32,
            total_features: total_features as u32,
            enabled_features: enabled_features as u32,
            active_tenants: active_tenants as u32,
            last_deployment: None, // 需要從部署記錄獲取
        })
    }

    /// 比較環境差異
    pub async fn compare_environments(
        &self,
        source_env: &str,
        target_env: &str,
    ) -> Result<EnvironmentComparison, Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        // 比較配置差異
        let config_differences = self.compare_configs(source_env, target_env, &platform_pool).await?;

        // 比較功能開關差異
        let feature_differences = self.compare_features(source_env, target_env, &platform_pool).await?;

        Ok(EnvironmentComparison {
            source_env: source_env.to_string(),
            target_env: target_env.to_string(),
            config_differences,
            feature_differences,
        })
    }

    /// 複製環境配置
    pub async fn copy_environment_config(
        &self,
        source_env: &str,
        target_env: &str,
        copy_configs: bool,
        copy_features: bool,
    ) -> Result<(), Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        let mut tx = platform_pool.begin().await?;

        if copy_configs {
            // 複製配置項
            sqlx::query(
                r#"
                INSERT INTO config_items (
                    id, key, value, category, description, data_type, 
                    is_sensitive, is_readonly, tenant_id, environment, 
                    version, created_by, updated_by
                )
                SELECT 
                    gen_random_uuid(), key, value, category, description, data_type,
                    is_sensitive, is_readonly, tenant_id, $1,
                    version, created_by, updated_by
                FROM config_items 
                WHERE environment = $2
                ON CONFLICT (key, COALESCE(tenant_id, '00000000-0000-0000-0000-000000000000'::uuid), environment) 
                DO UPDATE SET 
                    value = EXCLUDED.value,
                    updated_at = NOW()
                "#,
            )
            .bind(target_env)
            .bind(source_env)
            .execute(&mut *tx)
            .await?;
        }

        if copy_features {
            // 複製功能開關
            sqlx::query(
                r#"
                INSERT INTO feature_flags (
                    id, key, name, description, is_enabled, rollout_percentage,
                    target_groups, conditions, environment, tenant_id, 
                    created_by, expires_at
                )
                SELECT 
                    gen_random_uuid(), key, name, description, is_enabled, rollout_percentage,
                    target_groups, conditions, $1, tenant_id,
                    created_by, expires_at
                FROM feature_flags 
                WHERE environment = $2
                ON CONFLICT (key, COALESCE(tenant_id, '00000000-0000-0000-0000-000000000000'::uuid), environment) 
                DO UPDATE SET 
                    is_enabled = EXCLUDED.is_enabled,
                    rollout_percentage = EXCLUDED.rollout_percentage,
                    updated_at = NOW()
                "#,
            )
            .bind(target_env)
            .bind(source_env)
            .execute(&mut *tx)
            .await?;
        }

        tx.commit().await?;

        tracing::info!("環境配置已從 {} 複製到 {}", source_env, target_env);

        Ok(())
    }

    // 輔助方法
    async fn environment_exists(&self, name: &str) -> Result<bool, Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;

        let count: i64 = sqlx::query_scalar(
            "SELECT COUNT(*) FROM environments WHERE name = $1"
        )
        .bind(name)
        .fetch_one(&platform_pool)
        .await?;

        Ok(count > 0)
    }

    async fn compare_configs(
        &self,
        source_env: &str,
        target_env: &str,
        pool: &PgPool,
    ) -> Result<Vec<ConfigDifference>, Box<dyn std::error::Error>> {
        let configs = sqlx::query!(
            r#"
            WITH source_configs AS (
                SELECT key, value FROM config_items WHERE environment = $1
            ),
            target_configs AS (
                SELECT key, value FROM config_items WHERE environment = $2
            )
            SELECT 
                COALESCE(s.key, t.key) as key,
                s.value as source_value,
                t.value as target_value
            FROM source_configs s
            FULL OUTER JOIN target_configs t ON s.key = t.key
            ORDER BY key
            "#,
            source_env,
            target_env
        )
        .fetch_all(pool)
        .await?;

        let mut differences = Vec::new();

        for record in configs {
            let difference_type = match (&record.source_value, &record.target_value) {
                (Some(_), None) => DifferenceType::Removed,
                (None, Some(_)) => DifferenceType::Added,
                (Some(s), Some(t)) if s == t => DifferenceType::Identical,
                (Some(_), Some(_)) => DifferenceType::Modified,
                (None, None) => continue, // 不應該發生
            };

            differences.push(ConfigDifference {
                key: record.key.unwrap(),
                source_value: record.source_value,
                target_value: record.target_value,
                difference_type,
            });
        }

        Ok(differences)
    }

    async fn compare_features(
        &self,
        source_env: &str,
        target_env: &str,
        pool: &PgPool,
    ) -> Result<Vec<FeatureDifference>, Box<dyn std::error::Error>> {
        let features = sqlx::query!(
            r#"
            WITH source_features AS (
                SELECT key, is_enabled FROM feature_flags WHERE environment = $1
            ),
            target_features AS (
                SELECT key, is_enabled FROM feature_flags WHERE environment = $2
            )
            SELECT 
                COALESCE(s.key, t.key) as key,
                s.is_enabled as source_enabled,
                t.is_enabled as target_enabled
            FROM source_features s
            FULL OUTER JOIN target_features t ON s.key = t.key
            ORDER BY key
            "#,
            source_env,
            target_env
        )
        .fetch_all(pool)
        .await?;

        let mut differences = Vec::new();

        for record in features {
            let difference_type = match (&record.source_enabled, &record.target_enabled) {
                (Some(_), None) => DifferenceType::Removed,
                (None, Some(_)) => DifferenceType::Added,
                (Some(s), Some(t)) if s == t => DifferenceType::Identical,
                (Some(_), Some(_)) => DifferenceType::Modified,
                (None, None) => continue,
            };

            differences.push(FeatureDifference {
                key: record.key.unwrap(),
                source_enabled: record.source_enabled,
                target_enabled: record.target_enabled,
                difference_type,
            });
        }

        Ok(differences)
    }
}
```

**🚀 功能開關和環境管理系統開發完成！繼續開發計費管理系統...**