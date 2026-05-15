# 📊 Coffee系統 Phase 1: 平台監控系統開發

## 🏗️ **監控服務架構**

### **監控微服務 - Rust實現**

```rust
// backend/monitoring-service/Cargo.toml
[package]
name = "coffee-monitoring-service"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.7"
tokio = { version = "1.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
uuid = { version = "1.0", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "chrono", "uuid"] }
redis = { version = "0.24", features = ["tokio-comp"] }
tracing = "0.1"
tracing-subscriber = "0.3"
tower = "0.4"
tower-http = { version = "0.5", features = ["cors"] }
anyhow = "1.0"
config = "0.14"
prometheus = "0.13"
sysinfo = "0.30"
reqwest = { version = "0.11", features = ["json"] }

# 共享庫
coffee-shared-lib = { path = "../coffee-shared-lib" }
```

### **主服務程式**
```rust
// backend/monitoring-service/src/main.rs
use axum::{
    extract::State,
    http::StatusCode,
    response::Json,
    routing::{get, post},
    Router,
};
use coffee_shared_lib::database::connection_pool::DatabaseManager;
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::Arc;
use tokio::time::{interval, Duration};
use tower_http::cors::CorsLayer;
use tracing::{info, error};

mod handlers;
mod services;
mod metrics;
mod models;

use handlers::*;
use services::*;

#[derive(Clone)]
pub struct AppState {
    pub db_manager: DatabaseManager,
    pub monitoring_service: Arc<MonitoringService>,
    pub alert_service: Arc<AlertService>,
    pub metrics_service: Arc<MetricsService>,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 初始化日誌
    tracing_subscriber::init();

    // 載入配置
    let config = load_config().await?;
    
    // 初始化資料庫
    let db_manager = DatabaseManager::new(&config.database_url).await?;
    
    // 初始化服務
    let monitoring_service = Arc::new(MonitoringService::new(db_manager.clone()));
    let alert_service = Arc::new(AlertService::new(&config.discord_webhook_url));
    let metrics_service = Arc::new(MetricsService::new());

    let app_state = AppState {
        db_manager,
        monitoring_service: monitoring_service.clone(),
        alert_service: alert_service.clone(),
        metrics_service: metrics_service.clone(),
    };

    // 啟動背景監控任務
    let monitoring_clone = monitoring_service.clone();
    let alert_clone = alert_service.clone();
    tokio::spawn(async move {
        run_monitoring_loop(monitoring_clone, alert_clone).await;
    });

    // 建立路由
    let app = create_routes(app_state);

    // 啟動服務器
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8001").await?;
    info!("☕ Coffee監控服務啟動在 http://0.0.0.0:8001");
    
    axum::serve(listener, app).await?;

    Ok(())
}

fn create_routes(state: AppState) -> Router {
    Router::new()
        .route("/health", get(health_check))
        .route("/metrics", get(get_metrics))
        .route("/system/status", get(get_system_status))
        .route("/system/services", get(get_services_status))
        .route("/alerts", get(get_alerts).post(create_alert))
        .route("/alerts/:id", get(get_alert))
        .route("/tenants/stats", get(get_tenants_statistics))
        .route("/dashboard/overview", get(get_dashboard_overview))
        .layer(CorsLayer::permissive())
        .with_state(state)
}

#[derive(Deserialize)]
struct Config {
    database_url: String,
    redis_url: String,
    discord_webhook_url: String,
    monitoring_interval_seconds: u64,
}

async fn load_config() -> Result<Config, config::ConfigError> {
    let settings = config::Config::builder()
        .add_source(config::Environment::with_prefix("COFFEE"))
        .build()?;
    
    settings.try_deserialize()
}

// 背景監控循環
async fn run_monitoring_loop(
    monitoring_service: Arc<MonitoringService>,
    alert_service: Arc<AlertService>,
) {
    let mut interval = interval(Duration::from_secs(30)); // 每30秒檢查一次
    
    loop {
        interval.tick().await;
        
        match monitoring_service.collect_system_metrics().await {
            Ok(metrics) => {
                // 檢查告警條件
                if let Err(e) = check_and_send_alerts(&metrics, &alert_service).await {
                    error!("告警檢查失敗: {}", e);
                }
            }
            Err(e) => {
                error!("系統指標收集失敗: {}", e);
            }
        }
    }
}

async fn check_and_send_alerts(
    metrics: &SystemMetrics,
    alert_service: &AlertService,
) -> Result<(), Box<dyn std::error::Error>> {
    // CPU使用率過高
    if metrics.cpu_usage > 80.0 {
        alert_service.send_alert(
            AlertLevel::Critical,
            "CPU使用率過高".to_string(),
            format!("CPU使用率: {:.1}%", metrics.cpu_usage),
        ).await?;
    }

    // 記憶體使用率過高  
    if metrics.memory_usage > 85.0 {
        alert_service.send_alert(
            AlertLevel::Warning,
            "記憶體使用率高".to_string(),
            format!("記憶體使用率: {:.1}%", metrics.memory_usage),
        ).await?;
    }

    // 磁碟空間不足
    if metrics.disk_usage > 90.0 {
        alert_service.send_alert(
            AlertLevel::Critical,
            "磁碟空間不足".to_string(),
            format!("磁碟使用率: {:.1}%", metrics.disk_usage),
        ).await?;
    }

    Ok(())
}
```

### **監控服務核心邏輯**
```rust
// backend/monitoring-service/src/services/monitoring_service.rs
use coffee_shared_lib::database::connection_pool::DatabaseManager;
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use sysinfo::{System, SystemExt, CpuExt, DiskExt};
use uuid::Uuid;
use chrono::{DateTime, Utc};
use sqlx::PgPool;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SystemMetrics {
    pub timestamp: DateTime<Utc>,
    pub cpu_usage: f32,
    pub memory_usage: f32,
    pub memory_total: u64,
    pub memory_used: u64,
    pub disk_usage: f32,
    pub disk_total: u64,
    pub disk_used: u64,
    pub network_in: u64,
    pub network_out: u64,
    pub active_connections: u32,
    pub response_time_avg: f32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ServiceStatus {
    pub service_name: String,
    pub status: ServiceState,
    pub health_check_url: String,
    pub last_check: DateTime<Utc>,
    pub response_time_ms: u32,
    pub error_message: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ServiceState {
    Healthy,
    Degraded,
    Unhealthy,
    Unknown,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TenantStatistics {
    pub tenant_id: Uuid,
    pub tenant_name: String,
    pub active_users: u32,
    pub total_orders_today: u32,
    pub revenue_today: rust_decimal::Decimal,
    pub api_calls_count: u32,
    pub storage_used_mb: u32,
    pub last_activity: DateTime<Utc>,
}

pub struct MonitoringService {
    db_manager: DatabaseManager,
    system: System,
    client: reqwest::Client,
}

impl MonitoringService {
    pub fn new(db_manager: DatabaseManager) -> Self {
        Self {
            db_manager,
            system: System::new_all(),
            client: reqwest::Client::new(),
        }
    }

    /// 收集系統指標
    pub async fn collect_system_metrics(&self) -> Result<SystemMetrics, Box<dyn std::error::Error>> {
        let mut sys = System::new_all();
        sys.refresh_all();

        // CPU使用率
        let cpu_usage = sys.global_cpu_info().cpu_usage();

        // 記憶體使用率
        let memory_total = sys.total_memory();
        let memory_used = sys.used_memory();
        let memory_usage = (memory_used as f32 / memory_total as f32) * 100.0;

        // 磁碟使用率
        let (disk_total, disk_used) = sys.disks()
            .iter()
            .fold((0, 0), |(total, used), disk| {
                (total + disk.total_space(), used + (disk.total_space() - disk.available_space()))
            });
        
        let disk_usage = if disk_total > 0 {
            (disk_used as f32 / disk_total as f32) * 100.0
        } else {
            0.0
        };

        // 網路統計 (簡化)
        let (network_in, network_out) = (0, 0); // 需要實際網路介面統計

        // 活躍連接數 (從資料庫獲取)
        let active_connections = self.get_active_connections_count().await?;

        // 平均響應時間
        let response_time_avg = self.get_average_response_time().await?;

        let metrics = SystemMetrics {
            timestamp: Utc::now(),
            cpu_usage,
            memory_usage,
            memory_total,
            memory_used,
            disk_usage,
            disk_total,
            disk_used,
            network_in,
            network_out,
            active_connections,
            response_time_avg,
        };

        // 儲存到資料庫
        self.save_metrics_to_db(&metrics).await?;

        Ok(metrics)
    }

    /// 檢查所有微服務狀態
    pub async fn check_services_health(&self) -> Result<Vec<ServiceStatus>, Box<dyn std::error::Error>> {
        let services = vec![
            ("tenant-management", "http://localhost:8000/health"),
            ("pos-service", "http://localhost:8002/health"),
            ("inventory-service", "http://localhost:8003/health"),
            ("customer-service", "http://localhost:8004/health"),
            ("notification-service", "http://localhost:8005/health"),
        ];

        let mut results = Vec::new();

        for (service_name, url) in services {
            let status = self.check_single_service_health(service_name, url).await;
            results.push(status);
        }

        Ok(results)
    }

    async fn check_single_service_health(&self, service_name: &str, url: &str) -> ServiceStatus {
        let start_time = std::time::Instant::now();
        
        match self.client.get(url).timeout(std::time::Duration::from_secs(5)).send().await {
            Ok(response) => {
                let response_time_ms = start_time.elapsed().as_millis() as u32;
                
                let status = if response.status().is_success() {
                    ServiceState::Healthy
                } else if response.status().is_server_error() {
                    ServiceState::Unhealthy
                } else {
                    ServiceState::Degraded
                };

                ServiceStatus {
                    service_name: service_name.to_string(),
                    status,
                    health_check_url: url.to_string(),
                    last_check: Utc::now(),
                    response_time_ms,
                    error_message: None,
                }
            }
            Err(e) => {
                ServiceStatus {
                    service_name: service_name.to_string(),
                    status: ServiceState::Unhealthy,
                    health_check_url: url.to_string(),
                    last_check: Utc::now(),
                    response_time_ms: 0,
                    error_message: Some(e.to_string()),
                }
            }
        }
    }

    /// 獲取所有租戶統計資料
    pub async fn get_all_tenants_statistics(&self) -> Result<Vec<TenantStatistics>, Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;
        
        // 獲取所有租戶
        let tenants: Vec<(Uuid, String)> = sqlx::query_as(
            "SELECT id, name FROM tenants WHERE status = 'active'"
        )
        .fetch_all(&platform_pool)
        .await?;

        let mut statistics = Vec::new();

        for (tenant_id, tenant_name) in tenants {
            match self.get_tenant_statistics(tenant_id, &tenant_name).await {
                Ok(stats) => statistics.push(stats),
                Err(e) => {
                    tracing::warn!("獲取租戶 {} 統計失敗: {}", tenant_name, e);
                }
            }
        }

        Ok(statistics)
    }

    async fn get_tenant_statistics(
        &self, 
        tenant_id: Uuid, 
        tenant_name: &str
    ) -> Result<TenantStatistics, Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(&tenant_id.to_string()).await?;

        // 設置搜索路徑
        sqlx::query(&format!("SET search_path TO {}, public", tenant_id))
            .execute(&tenant_pool)
            .await?;

        // 今日活躍用戶
        let active_users: i64 = sqlx::query_scalar(
            "SELECT COUNT(*) FROM users WHERE last_login_at > CURRENT_DATE"
        )
        .fetch_one(&tenant_pool)
        .await?;

        // 今日訂單數
        let total_orders_today: i64 = sqlx::query_scalar(
            "SELECT COUNT(*) FROM orders WHERE created_at > CURRENT_DATE"
        )
        .fetch_one(&tenant_pool)
        .await?;

        // 今日營收
        let revenue_today: Option<rust_decimal::Decimal> = sqlx::query_scalar(
            "SELECT SUM(final_amount) FROM orders WHERE created_at > CURRENT_DATE AND payment_status = 'completed'"
        )
        .fetch_one(&tenant_pool)
        .await?;

        // API呼叫次數 (從Redis獲取)
        let api_calls_count = self.get_tenant_api_calls_count(&tenant_id.to_string()).await?;

        // 儲存使用量 (簡化)
        let storage_used_mb = 0; // 需要實際計算

        // 最後活動時間
        let last_activity: Option<DateTime<Utc>> = sqlx::query_scalar(
            "SELECT MAX(created_at) FROM orders"
        )
        .fetch_one(&tenant_pool)
        .await?;

        Ok(TenantStatistics {
            tenant_id,
            tenant_name: tenant_name.to_string(),
            active_users: active_users as u32,
            total_orders_today: total_orders_today as u32,
            revenue_today: revenue_today.unwrap_or_else(|| rust_decimal::Decimal::new(0, 2)),
            api_calls_count,
            storage_used_mb,
            last_activity: last_activity.unwrap_or_else(|| Utc::now()),
        })
    }

    async fn get_active_connections_count(&self) -> Result<u32, Box<dyn std::error::Error>> {
        // 從資料庫連接池或監控系統獲取
        Ok(50) // 模擬值
    }

    async fn get_average_response_time(&self) -> Result<f32, Box<dyn std::error::Error>> {
        // 從監控指標獲取
        Ok(120.5) // 模擬值 (毫秒)
    }

    async fn get_tenant_api_calls_count(&self, tenant_id: &str) -> Result<u32, Box<dyn std::error::Error>> {
        // 從Redis獲取API呼叫統計
        Ok(1500) // 模擬值
    }

    async fn save_metrics_to_db(&self, metrics: &SystemMetrics) -> Result<(), Box<dyn std::error::Error>> {
        let platform_pool = self.db_manager.get_platform_pool().await?;
        
        sqlx::query(
            r#"
            INSERT INTO system_metrics (
                timestamp, cpu_usage, memory_usage, memory_total, memory_used,
                disk_usage, disk_total, disk_used, network_in, network_out,
                active_connections, response_time_avg
            ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12)
            "#
        )
        .bind(&metrics.timestamp)
        .bind(metrics.cpu_usage)
        .bind(metrics.memory_usage)
        .bind(metrics.memory_total as i64)
        .bind(metrics.memory_used as i64)
        .bind(metrics.disk_usage)
        .bind(metrics.disk_total as i64)
        .bind(metrics.disk_used as i64)
        .bind(metrics.network_in as i64)
        .bind(metrics.network_out as i64)
        .bind(metrics.active_connections as i32)
        .bind(metrics.response_time_avg)
        .execute(&platform_pool)
        .await?;

        Ok(())
    }
}
```

**🚀 繼續開發告警系統和API控制器...**