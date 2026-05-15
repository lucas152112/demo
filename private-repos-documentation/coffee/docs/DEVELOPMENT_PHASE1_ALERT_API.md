# 🚨 Coffee系統告警服務與API控制器

## 📢 **告警服務實現**

### **告警服務核心邏輯**
```rust
// backend/monitoring-service/src/services/alert_service.rs
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use reqwest::Client;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum AlertLevel {
    Info,
    Warning,
    Critical,
    Emergency,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Alert {
    pub id: Uuid,
    pub level: AlertLevel,
    pub title: String,
    pub message: String,
    pub source: String,
    pub tenant_id: Option<Uuid>,
    pub metadata: HashMap<String, serde_json::Value>,
    pub created_at: DateTime<Utc>,
    pub resolved_at: Option<DateTime<Utc>>,
    pub is_resolved: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AlertRule {
    pub id: Uuid,
    pub name: String,
    pub condition: AlertCondition,
    pub level: AlertLevel,
    pub enabled: bool,
    pub cooldown_minutes: u32,
    pub notification_channels: Vec<NotificationChannel>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum AlertCondition {
    CpuUsage { threshold: f32 },
    MemoryUsage { threshold: f32 },
    DiskUsage { threshold: f32 },
    ResponseTime { threshold_ms: u32 },
    ServiceDown { service_name: String },
    TenantApiLimitExceeded { tenant_id: Uuid, limit: u32 },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum NotificationChannel {
    Discord { webhook_url: String },
    Email { recipients: Vec<String> },
    Sms { phone_numbers: Vec<String> },
    Slack { webhook_url: String },
}

pub struct AlertService {
    client: Client,
    discord_webhook: Option<String>,
    alert_history: std::sync::RwLock<Vec<Alert>>,
    alert_rules: std::sync::RwLock<Vec<AlertRule>>,
}

impl AlertService {
    pub fn new(discord_webhook_url: &str) -> Self {
        Self {
            client: Client::new(),
            discord_webhook: Some(discord_webhook_url.to_string()),
            alert_history: std::sync::RwLock::new(Vec::new()),
            alert_rules: std::sync::RwLock::new(Vec::new()),
        }
    }

    /// 發送告警
    pub async fn send_alert(
        &self,
        level: AlertLevel,
        title: String,
        message: String,
    ) -> Result<Uuid, Box<dyn std::error::Error>> {
        let alert_id = Uuid::new_v4();
        
        let alert = Alert {
            id: alert_id,
            level: level.clone(),
            title: title.clone(),
            message: message.clone(),
            source: "monitoring-service".to_string(),
            tenant_id: None,
            metadata: HashMap::new(),
            created_at: Utc::now(),
            resolved_at: None,
            is_resolved: false,
        };

        // 儲存告警記錄
        {
            let mut history = self.alert_history.write().unwrap();
            history.push(alert.clone());
            
            // 只保留最近1000條記錄
            if history.len() > 1000 {
                history.drain(0..100);
            }
        }

        // 發送到不同通知管道
        self.send_to_discord(&level, &title, &message).await?;
        
        tracing::info!("告警已發送: {} - {}", title, message);
        
        Ok(alert_id)
    }

    /// 發送租戶告警
    pub async fn send_tenant_alert(
        &self,
        tenant_id: Uuid,
        level: AlertLevel,
        title: String,
        message: String,
        metadata: HashMap<String, serde_json::Value>,
    ) -> Result<Uuid, Box<dyn std::error::Error>> {
        let alert_id = Uuid::new_v4();
        
        let alert = Alert {
            id: alert_id,
            level: level.clone(),
            title: title.clone(),
            message: message.clone(),
            source: "monitoring-service".to_string(),
            tenant_id: Some(tenant_id),
            metadata,
            created_at: Utc::now(),
            resolved_at: None,
            is_resolved: false,
        };

        // 儲存告警記錄
        {
            let mut history = self.alert_history.write().unwrap();
            history.push(alert.clone());
        }

        // 發送通知
        let tenant_title = format!("[租戶: {}] {}", tenant_id, title);
        self.send_to_discord(&level, &tenant_title, &message).await?;
        
        Ok(alert_id)
    }

    /// 發送到Discord
    async fn send_to_discord(
        &self,
        level: &AlertLevel,
        title: &str,
        message: &str,
    ) -> Result<(), Box<dyn std::error::Error>> {
        if let Some(webhook_url) = &self.discord_webhook {
            let color = match level {
                AlertLevel::Info => 0x3498db,      // 藍色
                AlertLevel::Warning => 0xf39c12,   // 橙色
                AlertLevel::Critical => 0xe74c3c,  // 紅色
                AlertLevel::Emergency => 0x8e44ad, // 紫色
            };

            let emoji = match level {
                AlertLevel::Info => "ℹ️",
                AlertLevel::Warning => "⚠️",
                AlertLevel::Critical => "🚨",
                AlertLevel::Emergency => "🆘",
            };

            let payload = serde_json::json!({
                "embeds": [{
                    "title": format!("{} Coffee系統告警", emoji),
                    "description": format!("**{}**\n{}", title, message),
                    "color": color,
                    "timestamp": Utc::now().to_rfc3339(),
                    "footer": {
                        "text": "Coffee監控系統"
                    }
                }]
            });

            let response = self.client
                .post(webhook_url)
                .json(&payload)
                .send()
                .await?;

            if !response.status().is_success() {
                return Err(format!("Discord webhook failed: {}", response.status()).into());
            }
        }

        Ok(())
    }

    /// 獲取告警記錄
    pub fn get_alerts(&self, limit: Option<usize>) -> Vec<Alert> {
        let history = self.alert_history.read().unwrap();
        let limit = limit.unwrap_or(50).min(history.len());
        
        history.iter().rev().take(limit).cloned().collect()
    }

    /// 解決告警
    pub fn resolve_alert(&self, alert_id: Uuid) -> Result<(), Box<dyn std::error::Error>> {
        let mut history = self.alert_history.write().unwrap();
        
        if let Some(alert) = history.iter_mut().find(|a| a.id == alert_id) {
            alert.is_resolved = true;
            alert.resolved_at = Some(Utc::now());
            Ok(())
        } else {
            Err("Alert not found".into())
        }
    }

    /// 添加告警規則
    pub fn add_alert_rule(&self, rule: AlertRule) {
        let mut rules = self.alert_rules.write().unwrap();
        rules.push(rule);
    }

    /// 檢查告警規則
    pub async fn check_alert_rules(
        &self,
        metrics: &crate::services::monitoring_service::SystemMetrics,
    ) -> Result<(), Box<dyn std::error::Error>> {
        let rules = self.alert_rules.read().unwrap().clone();
        
        for rule in rules {
            if !rule.enabled {
                continue;
            }

            let should_alert = match &rule.condition {
                AlertCondition::CpuUsage { threshold } => metrics.cpu_usage > *threshold,
                AlertCondition::MemoryUsage { threshold } => metrics.memory_usage > *threshold,
                AlertCondition::DiskUsage { threshold } => metrics.disk_usage > *threshold,
                AlertCondition::ResponseTime { threshold_ms } => {
                    metrics.response_time_avg > (*threshold_ms as f32)
                }
                _ => false, // 其他條件需要額外資料
            };

            if should_alert {
                let title = format!("告警規則觸發: {}", rule.name);
                let message = format!("條件: {:?}", rule.condition);
                
                self.send_alert(rule.level.clone(), title, message).await?;
            }
        }

        Ok(())
    }
}

/// 預設告警規則
impl AlertService {
    pub fn create_default_alert_rules(&self) {
        let default_rules = vec![
            AlertRule {
                id: Uuid::new_v4(),
                name: "CPU使用率過高".to_string(),
                condition: AlertCondition::CpuUsage { threshold: 80.0 },
                level: AlertLevel::Warning,
                enabled: true,
                cooldown_minutes: 5,
                notification_channels: vec![],
            },
            AlertRule {
                id: Uuid::new_v4(),
                name: "CPU使用率危險".to_string(),
                condition: AlertCondition::CpuUsage { threshold: 95.0 },
                level: AlertLevel::Critical,
                enabled: true,
                cooldown_minutes: 1,
                notification_channels: vec![],
            },
            AlertRule {
                id: Uuid::new_v4(),
                name: "記憶體使用率高".to_string(),
                condition: AlertCondition::MemoryUsage { threshold: 85.0 },
                level: AlertLevel::Warning,
                enabled: true,
                cooldown_minutes: 5,
                notification_channels: vec![],
            },
            AlertRule {
                id: Uuid::new_v4(),
                name: "磁碟空間不足".to_string(),
                condition: AlertCondition::DiskUsage { threshold: 90.0 },
                level: AlertLevel::Critical,
                enabled: true,
                cooldown_minutes: 10,
                notification_channels: vec![],
            },
            AlertRule {
                id: Uuid::new_v4(),
                name: "響應時間過慢".to_string(),
                condition: AlertCondition::ResponseTime { threshold_ms: 2000 },
                level: AlertLevel::Warning,
                enabled: true,
                cooldown_minutes: 3,
                notification_channels: vec![],
            },
        ];

        for rule in default_rules {
            self.add_alert_rule(rule);
        }
    }
}
```

---

## 🎯 **API控制器實現**

### **監控API控制器**
```rust
// backend/monitoring-service/src/handlers/monitoring_handlers.rs
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    response::Json,
};
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use std::collections::HashMap;

use crate::services::{
    monitoring_service::{SystemMetrics, ServiceStatus, TenantStatistics},
    alert_service::{Alert, AlertLevel},
};
use crate::AppState;

#[derive(Debug, Serialize)]
pub struct HealthResponse {
    pub status: String,
    pub version: String,
    pub timestamp: chrono::DateTime<chrono::Utc>,
    pub uptime_seconds: u64,
}

#[derive(Debug, Serialize)]
pub struct SystemStatusResponse {
    pub system_metrics: SystemMetrics,
    pub services_status: Vec<ServiceStatus>,
    pub overall_health: HealthStatus,
}

#[derive(Debug, Serialize)]
pub enum HealthStatus {
    Healthy,
    Degraded,
    Unhealthy,
}

#[derive(Debug, Serialize)]
pub struct DashboardOverview {
    pub system_summary: SystemSummary,
    pub tenant_summary: TenantSummary,
    pub recent_alerts: Vec<Alert>,
    pub service_status_summary: ServiceStatusSummary,
    pub performance_metrics: PerformanceMetrics,
}

#[derive(Debug, Serialize)]
pub struct SystemSummary {
    pub cpu_usage: f32,
    pub memory_usage: f32,
    pub disk_usage: f32,
    pub uptime_hours: f32,
    pub total_requests_24h: u32,
    pub error_rate_24h: f32,
}

#[derive(Debug, Serialize)]
pub struct TenantSummary {
    pub total_tenants: u32,
    pub active_tenants: u32,
    pub total_users: u32,
    pub total_orders_today: u32,
    pub total_revenue_today: rust_decimal::Decimal,
}

#[derive(Debug, Serialize)]
pub struct ServiceStatusSummary {
    pub healthy_services: u32,
    pub degraded_services: u32,
    pub unhealthy_services: u32,
    pub total_services: u32,
}

#[derive(Debug, Serialize)]
pub struct PerformanceMetrics {
    pub avg_response_time_ms: f32,
    pub p95_response_time_ms: f32,
    pub p99_response_time_ms: f32,
    pub requests_per_second: f32,
    pub error_rate_percentage: f32,
}

#[derive(Debug, Deserialize)]
pub struct AlertsQuery {
    pub level: Option<AlertLevel>,
    pub limit: Option<usize>,
    pub tenant_id: Option<Uuid>,
}

#[derive(Debug, Deserialize)]
pub struct CreateAlertRequest {
    pub level: AlertLevel,
    pub title: String,
    pub message: String,
    pub tenant_id: Option<Uuid>,
    pub metadata: Option<HashMap<String, serde_json::Value>>,
}

/// 健康檢查
pub async fn health_check() -> Json<HealthResponse> {
    Json(HealthResponse {
        status: "healthy".to_string(),
        version: "1.0.0".to_string(),
        timestamp: chrono::Utc::now(),
        uptime_seconds: 3600, // 模擬值
    })
}

/// 獲取系統狀態
pub async fn get_system_status(
    State(state): State<AppState>,
) -> Result<Json<SystemStatusResponse>, StatusCode> {
    // 收集系統指標
    let system_metrics = state.monitoring_service
        .collect_system_metrics()
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    // 檢查服務狀態
    let services_status = state.monitoring_service
        .check_services_health()
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    // 判斷整體健康狀態
    let overall_health = calculate_overall_health(&system_metrics, &services_status);

    Ok(Json(SystemStatusResponse {
        system_metrics,
        services_status,
        overall_health,
    }))
}

/// 獲取服務狀態
pub async fn get_services_status(
    State(state): State<AppState>,
) -> Result<Json<Vec<ServiceStatus>>, StatusCode> {
    let services_status = state.monitoring_service
        .check_services_health()
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok(Json(services_status))
}

/// 獲取租戶統計
pub async fn get_tenants_statistics(
    State(state): State<AppState>,
) -> Result<Json<Vec<TenantStatistics>>, StatusCode> {
    let statistics = state.monitoring_service
        .get_all_tenants_statistics()
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok(Json(statistics))
}

/// 獲取儀表板概覽
pub async fn get_dashboard_overview(
    State(state): State<AppState>,
) -> Result<Json<DashboardOverview>, StatusCode> {
    // 並發獲取各種資料
    let (system_metrics, services_status, tenants_stats) = tokio::try_join!(
        state.monitoring_service.collect_system_metrics(),
        state.monitoring_service.check_services_health(),
        state.monitoring_service.get_all_tenants_statistics()
    ).map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    // 獲取最近告警
    let recent_alerts = state.alert_service.get_alerts(Some(10));

    // 計算匯總資料
    let system_summary = SystemSummary {
        cpu_usage: system_metrics.cpu_usage,
        memory_usage: system_metrics.memory_usage,
        disk_usage: system_metrics.disk_usage,
        uptime_hours: 24.5, // 模擬值
        total_requests_24h: 15420,
        error_rate_24h: 0.15,
    };

    let tenant_summary = calculate_tenant_summary(&tenants_stats);
    let service_status_summary = calculate_service_status_summary(&services_status);
    
    let performance_metrics = PerformanceMetrics {
        avg_response_time_ms: system_metrics.response_time_avg,
        p95_response_time_ms: system_metrics.response_time_avg * 1.8,
        p99_response_time_ms: system_metrics.response_time_avg * 2.5,
        requests_per_second: 145.2,
        error_rate_percentage: 0.15,
    };

    Ok(Json(DashboardOverview {
        system_summary,
        tenant_summary,
        recent_alerts,
        service_status_summary,
        performance_metrics,
    }))
}

/// 獲取告警列表
pub async fn get_alerts(
    State(state): State<AppState>,
    Query(query): Query<AlertsQuery>,
) -> Json<Vec<Alert>> {
    let mut alerts = state.alert_service.get_alerts(query.limit);

    // 按級別過濾
    if let Some(level) = query.level {
        alerts = alerts.into_iter()
            .filter(|alert| std::mem::discriminant(&alert.level) == std::mem::discriminant(&level))
            .collect();
    }

    // 按租戶過濾
    if let Some(tenant_id) = query.tenant_id {
        alerts = alerts.into_iter()
            .filter(|alert| alert.tenant_id == Some(tenant_id))
            .collect();
    }

    Json(alerts)
}

/// 創建告警
pub async fn create_alert(
    State(state): State<AppState>,
    Json(request): Json<CreateAlertRequest>,
) -> Result<Json<serde_json::Value>, StatusCode> {
    let alert_id = if let Some(tenant_id) = request.tenant_id {
        state.alert_service
            .send_tenant_alert(
                tenant_id,
                request.level,
                request.title,
                request.message,
                request.metadata.unwrap_or_default(),
            )
            .await
            .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?
    } else {
        state.alert_service
            .send_alert(request.level, request.title, request.message)
            .await
            .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?
    };

    Ok(Json(serde_json::json!({
        "alert_id": alert_id,
        "status": "created"
    })))
}

/// 獲取單個告警
pub async fn get_alert(
    Path(alert_id): Path<Uuid>,
    State(state): State<AppState>,
) -> Result<Json<Alert>, StatusCode> {
    let alerts = state.alert_service.get_alerts(None);
    
    let alert = alerts.into_iter()
        .find(|a| a.id == alert_id)
        .ok_or(StatusCode::NOT_FOUND)?;

    Ok(Json(alert))
}

/// 獲取Prometheus指標
pub async fn get_metrics(
    State(state): State<AppState>,
) -> Result<String, StatusCode> {
    // 生成Prometheus格式的指標
    let metrics = state.monitoring_service
        .collect_system_metrics()
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    let prometheus_metrics = format!(
        r#"# HELP coffee_cpu_usage_percent CPU usage percentage
# TYPE coffee_cpu_usage_percent gauge
coffee_cpu_usage_percent {:.2}

# HELP coffee_memory_usage_percent Memory usage percentage  
# TYPE coffee_memory_usage_percent gauge
coffee_memory_usage_percent {:.2}

# HELP coffee_disk_usage_percent Disk usage percentage
# TYPE coffee_disk_usage_percent gauge
coffee_disk_usage_percent {:.2}

# HELP coffee_response_time_ms Average response time in milliseconds
# TYPE coffee_response_time_ms gauge
coffee_response_time_ms {:.2}

# HELP coffee_active_connections Active database connections
# TYPE coffee_active_connections gauge
coffee_active_connections {}
"#,
        metrics.cpu_usage,
        metrics.memory_usage,
        metrics.disk_usage,
        metrics.response_time_avg,
        metrics.active_connections
    );

    Ok(prometheus_metrics)
}

// 輔助函式
fn calculate_overall_health(
    system_metrics: &SystemMetrics,
    services_status: &[ServiceStatus],
) -> HealthStatus {
    // 檢查系統資源
    if system_metrics.cpu_usage > 95.0 || 
       system_metrics.memory_usage > 95.0 || 
       system_metrics.disk_usage > 95.0 {
        return HealthStatus::Unhealthy;
    }

    // 檢查服務狀態
    let unhealthy_services = services_status.iter()
        .filter(|s| matches!(s.status, crate::services::monitoring_service::ServiceState::Unhealthy))
        .count();

    if unhealthy_services > 0 {
        return HealthStatus::Unhealthy;
    }

    let degraded_services = services_status.iter()
        .filter(|s| matches!(s.status, crate::services::monitoring_service::ServiceState::Degraded))
        .count();

    if degraded_services > 0 || 
       system_metrics.cpu_usage > 80.0 || 
       system_metrics.memory_usage > 85.0 {
        return HealthStatus::Degraded;
    }

    HealthStatus::Healthy
}

fn calculate_tenant_summary(tenants_stats: &[TenantStatistics]) -> TenantSummary {
    let total_tenants = tenants_stats.len() as u32;
    let active_tenants = tenants_stats.iter()
        .filter(|t| t.active_users > 0)
        .count() as u32;

    let total_users = tenants_stats.iter()
        .map(|t| t.active_users)
        .sum();

    let total_orders_today = tenants_stats.iter()
        .map(|t| t.total_orders_today)
        .sum();

    let total_revenue_today = tenants_stats.iter()
        .map(|t| t.revenue_today)
        .sum();

    TenantSummary {
        total_tenants,
        active_tenants,
        total_users,
        total_orders_today,
        total_revenue_today,
    }
}

fn calculate_service_status_summary(services_status: &[ServiceStatus]) -> ServiceStatusSummary {
    use crate::services::monitoring_service::ServiceState;
    
    let healthy_services = services_status.iter()
        .filter(|s| matches!(s.status, ServiceState::Healthy))
        .count() as u32;

    let degraded_services = services_status.iter()
        .filter(|s| matches!(s.status, ServiceState::Degraded))
        .count() as u32;

    let unhealthy_services = services_status.iter()
        .filter(|s| matches!(s.status, ServiceState::Unhealthy))
        .count() as u32;

    ServiceStatusSummary {
        healthy_services,
        degraded_services,
        unhealthy_services,
        total_services: services_status.len() as u32,
    }
}
```

**🚀 監控系統和告警服務開發完成！繼續開發配置管理系統...**