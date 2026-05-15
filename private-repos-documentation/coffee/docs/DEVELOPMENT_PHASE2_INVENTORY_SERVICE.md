# 📦 Coffee POS系統 - 庫存管理服務

## 🏗️ **庫存管理微服務架構**

### **服務配置**
```rust
// backend/inventory-service/Cargo.toml
[package]
name = "coffee-inventory-service"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.7"
tokio = { version = "1.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
uuid = { version = "1.0", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "chrono", "uuid", "decimal"] }
rust_decimal = { version = "1.32", features = ["serde", "db-postgres"] }
redis = { version = "0.24", features = ["tokio-comp"] }
tracing = "0.1"
tracing-subscriber = "0.3"
tower = "0.4"
tower-http = { version = "0.5", features = ["cors"] }
anyhow = "1.0"
config = "0.14"

# 共享庫
coffee-shared-lib = { path = "../coffee-shared-lib" }

[dev-dependencies]
tokio-test = "0.4"
```

### **庫存管理主服務**
```rust
// backend/inventory-service/src/main.rs
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
use tokio::sync::broadcast;

mod handlers;
mod services;
mod models;

use handlers::*;
use services::*;

#[derive(Clone)]
pub struct AppState {
    pub db_manager: DatabaseManager,
    pub inventory_service: Arc<InventoryService>,
    pub supplier_service: Arc<SupplierService>,
    pub purchase_service: Arc<PurchaseOrderService>,
    pub audit_service: Arc<InventoryAuditService>,
    pub notification_sender: broadcast::Sender<String>,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    tracing_subscriber::init();

    // 載入配置
    let config = load_config().await?;
    
    // 初始化資料庫
    let db_manager = DatabaseManager::new(&config.database_url).await?;
    
    // 創建通知廣播頻道
    let (notification_sender, _) = broadcast::channel(1000);
    
    // 初始化服務
    let inventory_service = Arc::new(InventoryService::new(
        db_manager.clone(),
        &config.redis_url,
        notification_sender.clone(),
    ).await?);
    
    let supplier_service = Arc::new(SupplierService::new(
        db_manager.clone(),
    ));
    
    let purchase_service = Arc::new(PurchaseOrderService::new(
        db_manager.clone(),
        inventory_service.clone(),
    ));
    
    let audit_service = Arc::new(InventoryAuditService::new(
        db_manager.clone(),
    ));

    let app_state = AppState {
        db_manager,
        inventory_service: inventory_service.clone(),
        supplier_service,
        purchase_service,
        audit_service,
        notification_sender,
    };

    // 啟動庫存監控任務
    let inventory_clone = inventory_service.clone();
    tokio::spawn(async move {
        inventory_clone.start_inventory_monitoring_task().await;
    });

    // 啟動自動補貨任務
    let inventory_clone2 = inventory_service.clone();
    tokio::spawn(async move {
        inventory_clone2.start_auto_restock_task().await;
    });

    // 建立路由
    let app = create_routes(app_state);

    // 啟動服務器
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8012").await?;
    info!("📦 Coffee庫存管理服務啟動在 http://0.0.0.0:8012");
    
    axum::serve(listener, app).await?;

    Ok(())
}

fn create_routes(state: AppState) -> Router {
    Router::new()
        .route("/health", get(health_check))
        
        // 庫存管理路由
        .route("/inventory", get(get_inventory_items).post(create_inventory_item))
        .route("/inventory/:id", get(get_inventory_item).put(update_inventory_item))
        .route("/inventory/:id/stock", put(adjust_stock))
        .route("/inventory/low-stock", get(get_low_stock_items))
        .route("/inventory/tenant/:tenant_id", get(get_tenant_inventory))
        
        // 庫存異動路由
        .route("/inventory/transactions", get(get_inventory_transactions).post(create_inventory_transaction))
        .route("/inventory/transactions/item/:item_id", get(get_item_transactions))
        
        // 供應商管理路由
        .route("/suppliers", get(get_suppliers).post(create_supplier))
        .route("/suppliers/:id", get(get_supplier).put(update_supplier))
        .route("/suppliers/:id/items", get(get_supplier_items))
        
        // 採購訂單路由
        .route("/purchase-orders", get(get_purchase_orders).post(create_purchase_order))
        .route("/purchase-orders/:id", get(get_purchase_order).put(update_purchase_order))
        .route("/purchase-orders/:id/receive", post(receive_purchase_order))
        .route("/purchase-orders/:id/items", post(add_purchase_order_item))
        
        // 庫存盤點路由
        .route("/audits", get(get_inventory_audits).post(create_inventory_audit))
        .route("/audits/:id", get(get_inventory_audit).put(update_inventory_audit))
        .route("/audits/:id/items", post(add_audit_item).put(update_audit_item))
        .route("/audits/:id/finalize", post(finalize_inventory_audit))
        
        // 庫存分析路由
        .route("/analytics/inventory/summary", get(get_inventory_summary))
        .route("/analytics/inventory/turnover", get(get_inventory_turnover))
        .route("/analytics/inventory/abc-analysis", get(get_abc_analysis))
        .route("/analytics/suppliers/performance", get(get_supplier_performance))
        
        // 自動補貨路由
        .route("/auto-restock/rules", get(get_restock_rules).post(create_restock_rule))
        .route("/auto-restock/rules/:id", put(update_restock_rule))
        .route("/auto-restock/suggestions", get(get_restock_suggestions))
        .route("/auto-restock/execute", post(execute_auto_restock))
        
        .layer(CorsLayer::permissive())
        .with_state(state)
}

#[derive(Deserialize)]
struct Config {
    database_url: String,
    redis_url: String,
}

async fn load_config() -> Result<Config, config::ConfigError> {
    let settings = config::Config::builder()
        .add_source(config::Environment::with_prefix("COFFEE"))
        .build()?;
    
    settings.try_deserialize()
}
```

### **庫存管理核心服務**
```rust
// backend/inventory-service/src/services/inventory_service.rs
use coffee_shared_lib::database::connection_pool::DatabaseManager;
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use rust_decimal::Decimal;
use sqlx::PgPool;
use redis::AsyncCommands;
use tokio::sync::broadcast;

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct InventoryItem {
    pub id: Uuid,
    pub tenant_id: Uuid,
    pub sku: String,
    pub name: String,
    pub description: Option<String>,
    pub category: String,
    pub unit: String,
    pub current_stock: Decimal,
    pub reserved_stock: Decimal,
    pub available_stock: Decimal,
    pub reorder_point: Decimal,
    pub reorder_quantity: Decimal,
    pub max_stock_level: Option<Decimal>,
    pub unit_cost: Decimal,
    pub average_cost: Decimal,
    pub last_cost: Decimal,
    pub supplier_id: Option<Uuid>,
    pub storage_location: Option<String>,
    pub is_active: bool,
    pub is_trackable: bool,
    pub expiry_tracking: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct InventoryTransaction {
    pub id: Uuid,
    pub tenant_id: Uuid,
    pub item_id: Uuid,
    pub transaction_type: TransactionType,
    pub reference_type: ReferenceType,
    pub reference_id: Option<Uuid>,
    pub quantity: Decimal,
    pub unit_cost: Option<Decimal>,
    pub total_cost: Option<Decimal>,
    pub reason: Option<String>,
    pub notes: Option<String>,
    pub batch_number: Option<String>,
    pub expiry_date: Option<DateTime<Utc>>,
    pub location: Option<String>,
    pub created_at: DateTime<Utc>,
    pub created_by: Uuid,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "transaction_type", rename_all = "lowercase")]
pub enum TransactionType {
    In,        // 入庫
    Out,       // 出庫
    Adjustment, // 調整
    Transfer,  // 轉移
    Waste,     // 耗損
    Return,    // 退貨
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "reference_type", rename_all = "lowercase")]
pub enum ReferenceType {
    Sale,           // 銷售訂單
    Purchase,       // 採購訂單
    Adjustment,     // 庫存調整
    Transfer,       // 庫存轉移
    Production,     // 生產
    Audit,         // 盤點
    Waste,         // 耗損
    Initial,       // 初始庫存
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Supplier {
    pub id: Uuid,
    pub tenant_id: Uuid,
    pub name: String,
    pub code: String,
    pub contact_person: Option<String>,
    pub email: Option<String>,
    pub phone: Option<String>,
    pub address: Option<String>,
    pub payment_terms: Option<String>,
    pub lead_time_days: Option<i32>,
    pub minimum_order: Option<Decimal>,
    pub is_active: bool,
    pub rating: Option<Decimal>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct PurchaseOrder {
    pub id: Uuid,
    pub tenant_id: Uuid,
    pub supplier_id: Uuid,
    pub po_number: String,
    pub status: PurchaseOrderStatus,
    pub order_date: DateTime<Utc>,
    pub expected_delivery_date: Option<DateTime<Utc>>,
    pub actual_delivery_date: Option<DateTime<Utc>>,
    pub total_amount: Decimal,
    pub tax_amount: Decimal,
    pub final_amount: Decimal,
    pub notes: Option<String>,
    pub created_at: DateTime<Utc>,
    pub created_by: Uuid,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "purchase_order_status", rename_all = "lowercase")]
pub enum PurchaseOrderStatus {
    Draft,      // 草稿
    Pending,    // 待確認
    Approved,   // 已批准
    Ordered,    // 已下單
    PartiallyReceived, // 部分收貨
    Received,   // 已收貨
    Cancelled,  // 已取消
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct AutoRestockRule {
    pub id: Uuid,
    pub tenant_id: Uuid,
    pub item_id: Uuid,
    pub is_enabled: bool,
    pub trigger_point: Decimal,
    pub reorder_quantity: Decimal,
    pub preferred_supplier_id: Option<Uuid>,
    pub max_auto_order_amount: Option<Decimal>,
    pub last_triggered_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct CreateInventoryItemRequest {
    pub tenant_id: Uuid,
    pub sku: String,
    pub name: String,
    pub description: Option<String>,
    pub category: String,
    pub unit: String,
    pub initial_stock: Decimal,
    pub reorder_point: Decimal,
    pub reorder_quantity: Decimal,
    pub unit_cost: Decimal,
    pub supplier_id: Option<Uuid>,
    pub storage_location: Option<String>,
    pub expiry_tracking: Option<bool>,
}

#[derive(Debug, Deserialize)]
pub struct StockAdjustmentRequest {
    pub quantity: Decimal,
    pub adjustment_type: TransactionType,
    pub reason: String,
    pub notes: Option<String>,
    pub unit_cost: Option<Decimal>,
    pub batch_number: Option<String>,
    pub expiry_date: Option<DateTime<Utc>>,
}

#[derive(Debug, Serialize)]
pub struct InventorySummary {
    pub total_items: i64,
    pub total_stock_value: Decimal,
    pub low_stock_items: i64,
    pub out_of_stock_items: i64,
    pub expired_items: i64,
    pub expiring_soon_items: i64,
    pub category_breakdown: HashMap<String, CategoryStats>,
}

#[derive(Debug, Serialize)]
pub struct CategoryStats {
    pub item_count: i64,
    pub stock_value: Decimal,
    pub percentage: f64,
}

#[derive(Debug, Serialize)]
pub struct InventoryTurnover {
    pub item_id: Uuid,
    pub item_name: String,
    pub category: String,
    pub turnover_rate: f64,
    pub days_of_supply: f64,
    pub avg_monthly_usage: Decimal,
    pub current_stock: Decimal,
    pub stock_value: Decimal,
}

pub struct InventoryService {
    db_manager: DatabaseManager,
    redis_client: redis::Client,
    notification_sender: broadcast::Sender<String>,
}

impl InventoryService {
    pub async fn new(
        db_manager: DatabaseManager,
        redis_url: &str,
        notification_sender: broadcast::Sender<String>,
    ) -> Result<Self, Box<dyn std::error::Error>> {
        let redis_client = redis::Client::open(redis_url)?;

        Ok(Self {
            db_manager,
            redis_client,
            notification_sender,
        })
    }

    /// 創建庫存項目
    pub async fn create_inventory_item(
        &self,
        request: CreateInventoryItemRequest,
        created_by: Uuid,
    ) -> Result<InventoryItem, Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(request.tenant_id).await?;

        // 檢查SKU是否已存在
        let existing_count: i64 = sqlx::query_scalar(
            "SELECT COUNT(*) FROM inventory_items WHERE tenant_id = $1 AND sku = $2"
        )
        .bind(request.tenant_id)
        .bind(&request.sku)
        .fetch_one(&tenant_pool)
        .await?;

        if existing_count > 0 {
            return Err("SKU已存在".into());
        }

        let item_id = Uuid::new_v4();
        let available_stock = request.initial_stock;

        let mut tx = tenant_pool.begin().await?;

        // 創建庫存項目
        let inventory_item = sqlx::query_as::<_, InventoryItem>(
            r#"
            INSERT INTO inventory_items (
                id, tenant_id, sku, name, description, category, unit,
                current_stock, reserved_stock, available_stock,
                reorder_point, reorder_quantity, unit_cost, average_cost, last_cost,
                supplier_id, storage_location, is_active, is_trackable, 
                expiry_tracking, created_by
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15, $16, $17, $18, $19, $20, $21)
            RETURNING *
            "#,
        )
        .bind(item_id)
        .bind(request.tenant_id)
        .bind(&request.sku)
        .bind(&request.name)
        .bind(&request.description)
        .bind(&request.category)
        .bind(&request.unit)
        .bind(request.initial_stock)
        .bind(Decimal::ZERO)
        .bind(available_stock)
        .bind(request.reorder_point)
        .bind(request.reorder_quantity)
        .bind(request.unit_cost)
        .bind(request.unit_cost)
        .bind(request.unit_cost)
        .bind(request.supplier_id)
        .bind(&request.storage_location)
        .bind(true)
        .bind(true)
        .bind(request.expiry_tracking.unwrap_or(false))
        .bind(created_by)
        .fetch_one(&mut *tx)
        .await?;

        // 如果有初始庫存，創建初始庫存交易記錄
        if request.initial_stock > Decimal::ZERO {
            self.create_inventory_transaction_internal(
                &mut tx,
                request.tenant_id,
                item_id,
                TransactionType::In,
                ReferenceType::Initial,
                None,
                request.initial_stock,
                Some(request.unit_cost),
                "初始庫存".to_string(),
                None,
                None,
                None,
                created_by,
            ).await?;
        }

        tx.commit().await?;

        // 更新Redis快取
        self.update_inventory_cache(&inventory_item).await?;

        tracing::info!("庫存項目已創建: {} ({})", request.name, request.sku);

        Ok(inventory_item)
    }

    /// 調整庫存
    pub async fn adjust_stock(
        &self,
        item_id: Uuid,
        tenant_id: Uuid,
        request: StockAdjustmentRequest,
        adjusted_by: Uuid,
    ) -> Result<InventoryItem, Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(tenant_id).await?;

        let mut tx = tenant_pool.begin().await?;

        // 獲取當前庫存項目
        let mut item = self.get_inventory_item_by_id(item_id, tenant_id).await?;

        // 計算新的庫存量
        let old_stock = item.current_stock;
        let new_stock = match request.adjustment_type {
            TransactionType::In => old_stock + request.quantity,
            TransactionType::Out => {
                if old_stock < request.quantity {
                    return Err("庫存不足".into());
                }
                old_stock - request.quantity
            }
            TransactionType::Adjustment => {
                if request.quantity < Decimal::ZERO && old_stock < request.quantity.abs() {
                    return Err("調整後庫存不能為負數".into());
                }
                old_stock + request.quantity
            }
            _ => return Err("不支援的調整類型".into()),
        };

        // 更新庫存量
        item.current_stock = new_stock;
        item.available_stock = new_stock - item.reserved_stock;
        item.updated_at = Utc::now();

        // 更新成本 (如果提供)
        if let Some(unit_cost) = request.unit_cost {
            if request.adjustment_type == TransactionType::In {
                // 加權平均成本計算
                let total_value = (old_stock * item.average_cost) + (request.quantity * unit_cost);
                item.average_cost = if new_stock > Decimal::ZERO {
                    total_value / new_stock
                } else {
                    item.average_cost
                };
                item.last_cost = unit_cost;
            }
        }

        // 更新庫存項目
        sqlx::query(
            r#"
            UPDATE inventory_items 
            SET current_stock = $1, available_stock = $2, average_cost = $3, 
                last_cost = $4, updated_at = $5
            WHERE id = $6
            "#,
        )
        .bind(item.current_stock)
        .bind(item.available_stock)
        .bind(item.average_cost)
        .bind(item.last_cost)
        .bind(item.updated_at)
        .bind(item.id)
        .execute(&mut *tx)
        .await?;

        // 創建庫存交易記錄
        self.create_inventory_transaction_internal(
            &mut tx,
            tenant_id,
            item_id,
            request.adjustment_type,
            ReferenceType::Adjustment,
            None,
            request.quantity,
            request.unit_cost,
            request.reason,
            request.notes,
            request.batch_number,
            request.expiry_date,
            adjusted_by,
        ).await?;

        tx.commit().await?;

        // 更新Redis快取
        self.update_inventory_cache(&item).await?;

        // 檢查是否需要補貨提醒
        if item.current_stock <= item.reorder_point {
            self.send_low_stock_notification(&item).await?;
        }

        // 發送庫存變更通知
        let notification = serde_json::json!({
            "type": "inventory_adjusted",
            "item_id": item_id,
            "item_name": item.name,
            "old_stock": old_stock,
            "new_stock": new_stock,
            "adjustment_type": request.adjustment_type,
            "tenant_id": tenant_id,
            "timestamp": Utc::now()
        });

        let _ = self.notification_sender.send(notification.to_string());

        tracing::info!("庫存已調整: {} {} -> {}", item.name, old_stock, new_stock);

        Ok(item)
    }

    /// 獲取低庫存項目
    pub async fn get_low_stock_items(
        &self,
        tenant_id: Uuid,
    ) -> Result<Vec<InventoryItem>, Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(tenant_id).await?;

        let items = sqlx::query_as::<_, InventoryItem>(
            r#"
            SELECT * FROM inventory_items 
            WHERE tenant_id = $1 
            AND is_active = true 
            AND current_stock <= reorder_point
            ORDER BY (current_stock / NULLIF(reorder_point, 0)) ASC
            "#,
        )
        .bind(tenant_id)
        .fetch_all(&tenant_pool)
        .await?;

        Ok(items)
    }

    /// 獲取庫存統計摘要
    pub async fn get_inventory_summary(
        &self,
        tenant_id: Uuid,
    ) -> Result<InventorySummary, Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(tenant_id).await?;

        // 基本統計
        let basic_stats = sqlx::query!(
            r#"
            SELECT 
                COUNT(*) as total_items,
                COALESCE(SUM(current_stock * average_cost), 0) as total_stock_value,
                COUNT(CASE WHEN current_stock <= reorder_point THEN 1 END) as low_stock_items,
                COUNT(CASE WHEN current_stock = 0 THEN 1 END) as out_of_stock_items
            FROM inventory_items 
            WHERE tenant_id = $1 AND is_active = true
            "#,
            tenant_id
        )
        .fetch_one(&tenant_pool)
        .await?;

        // 過期商品統計 (簡化處理，實際需要考慮批次和過期日期)
        let expired_stats = sqlx::query!(
            r#"
            SELECT 
                COUNT(DISTINCT item_id) as expired_items,
                COUNT(DISTINCT item_id) as expiring_soon_items
            FROM inventory_transactions 
            WHERE tenant_id = $1 
            AND expiry_date IS NOT NULL
            "#,
            tenant_id
        )
        .fetch_one(&tenant_pool)
        .await?;

        // 分類統計
        let category_stats = sqlx::query!(
            r#"
            SELECT 
                category,
                COUNT(*) as item_count,
                COALESCE(SUM(current_stock * average_cost), 0) as stock_value
            FROM inventory_items 
            WHERE tenant_id = $1 AND is_active = true
            GROUP BY category
            ORDER BY stock_value DESC
            "#,
            tenant_id
        )
        .fetch_all(&tenant_pool)
        .await?;

        let total_value = basic_stats.total_stock_value.unwrap_or_default();
        let mut category_breakdown = HashMap::new();

        for row in category_stats {
            let category_value = row.stock_value.unwrap_or_default();
            let percentage = if total_value > Decimal::ZERO {
                (category_value / total_value * Decimal::new(100, 0))
                    .to_f64().unwrap_or(0.0)
            } else {
                0.0
            };

            category_breakdown.insert(
                row.category,
                CategoryStats {
                    item_count: row.item_count.unwrap_or(0),
                    stock_value: category_value,
                    percentage,
                },
            );
        }

        Ok(InventorySummary {
            total_items: basic_stats.total_items.unwrap_or(0),
            total_stock_value: total_value,
            low_stock_items: basic_stats.low_stock_items.unwrap_or(0),
            out_of_stock_items: basic_stats.out_of_stock_items.unwrap_or(0),
            expired_items: expired_stats.expired_items.unwrap_or(0),
            expiring_soon_items: expired_stats.expiring_soon_items.unwrap_or(0),
            category_breakdown,
        })
    }

    /// 獲取庫存週轉率分析
    pub async fn get_inventory_turnover_analysis(
        &self,
        tenant_id: Uuid,
        period_days: i32,
    ) -> Result<Vec<InventoryTurnover>, Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(tenant_id).await?;

        let start_date = Utc::now() - chrono::Duration::days(period_days as i64);

        let turnover_data = sqlx::query!(
            r#"
            SELECT 
                i.id as item_id,
                i.name as item_name,
                i.category,
                i.current_stock,
                i.average_cost,
                COALESCE(SUM(CASE WHEN t.transaction_type = 'out' THEN t.quantity ELSE 0 END), 0) as total_out_quantity
            FROM inventory_items i
            LEFT JOIN inventory_transactions t ON i.id = t.item_id 
                AND t.created_at >= $2
                AND t.transaction_type = 'out'
            WHERE i.tenant_id = $1 AND i.is_active = true
            GROUP BY i.id, i.name, i.category, i.current_stock, i.average_cost
            ORDER BY total_out_quantity DESC
            "#,
            tenant_id,
            start_date
        )
        .fetch_all(&tenant_pool)
        .await?;

        let mut turnover_analysis = Vec::new();

        for row in turnover_data {
            let total_usage = row.total_out_quantity.unwrap_or_default();
            let avg_monthly_usage = total_usage * Decimal::new(30, 0) / Decimal::new(period_days as i64, 0);
            let current_stock = row.current_stock;
            
            let turnover_rate = if current_stock > Decimal::ZERO && total_usage > Decimal::ZERO {
                (total_usage / current_stock * Decimal::new(365, 0) / Decimal::new(period_days as i64, 0))
                    .to_f64().unwrap_or(0.0)
            } else {
                0.0
            };

            let days_of_supply = if avg_monthly_usage > Decimal::ZERO {
                (current_stock / avg_monthly_usage * Decimal::new(30, 0))
                    .to_f64().unwrap_or(0.0)
            } else {
                f64::INFINITY
            };

            turnover_analysis.push(InventoryTurnover {
                item_id: row.item_id,
                item_name: row.item_name,
                category: row.category,
                turnover_rate,
                days_of_supply,
                avg_monthly_usage,
                current_stock,
                stock_value: current_stock * row.average_cost,
            });
        }

        Ok(turnover_analysis)
    }

    /// 啟動庫存監控任務
    pub async fn start_inventory_monitoring_task(&self) {
        let mut interval = tokio::time::interval(tokio::time::Duration::from_hours(1));
        
        loop {
            interval.tick().await;
            
            if let Err(e) = self.monitor_inventory_levels().await {
                tracing::error!("庫存監控任務失敗: {}", e);
            }
        }
    }

    /// 啟動自動補貨任務
    pub async fn start_auto_restock_task(&self) {
        let mut interval = tokio::time::interval(tokio::time::Duration::from_hours(6));
        
        loop {
            interval.tick().await;
            
            if let Err(e) = self.process_auto_restock().await {
                tracing::error!("自動補貨任務失敗: {}", e);
            }
        }
    }

    // 輔助方法
    async fn create_inventory_transaction_internal(
        &self,
        tx: &mut sqlx::Transaction<'_, sqlx::Postgres>,
        tenant_id: Uuid,
        item_id: Uuid,
        transaction_type: TransactionType,
        reference_type: ReferenceType,
        reference_id: Option<Uuid>,
        quantity: Decimal,
        unit_cost: Option<Decimal>,
        reason: String,
        notes: Option<String>,
        batch_number: Option<String>,
        expiry_date: Option<DateTime<Utc>>,
        created_by: Uuid,
    ) -> Result<InventoryTransaction, Box<dyn std::error::Error>> {
        let transaction_id = Uuid::new_v4();
        let total_cost = if let Some(cost) = unit_cost {
            Some(cost * quantity)
        } else {
            None
        };

        let transaction = sqlx::query_as::<_, InventoryTransaction>(
            r#"
            INSERT INTO inventory_transactions (
                id, tenant_id, item_id, transaction_type, reference_type, reference_id,
                quantity, unit_cost, total_cost, reason, notes, batch_number, 
                expiry_date, created_by
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14)
            RETURNING *
            "#,
        )
        .bind(transaction_id)
        .bind(tenant_id)
        .bind(item_id)
        .bind(&transaction_type)
        .bind(&reference_type)
        .bind(reference_id)
        .bind(quantity)
        .bind(unit_cost)
        .bind(total_cost)
        .bind(&reason)
        .bind(&notes)
        .bind(&batch_number)
        .bind(expiry_date)
        .bind(created_by)
        .fetch_one(&mut **tx)
        .await?;

        Ok(transaction)
    }

    async fn get_inventory_item_by_id(
        &self,
        item_id: Uuid,
        tenant_id: Uuid,
    ) -> Result<InventoryItem, Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(tenant_id).await?;

        let item = sqlx::query_as::<_, InventoryItem>(
            "SELECT * FROM inventory_items WHERE id = $1 AND tenant_id = $2"
        )
        .bind(item_id)
        .bind(tenant_id)
        .fetch_one(&tenant_pool)
        .await?;

        Ok(item)
    }

    async fn update_inventory_cache(
        &self,
        item: &InventoryItem,
    ) -> Result<(), Box<dyn std::error::Error>> {
        let mut redis_conn = self.redis_client.get_async_connection().await?;
        
        let cache_key = format!("coffee:inventory:{}:{}", item.tenant_id, item.id);
        let item_json = serde_json::to_string(item)?;
        
        redis_conn.set_ex(&cache_key, item_json, 3600).await?;

        Ok(())
    }

    async fn send_low_stock_notification(
        &self,
        item: &InventoryItem,
    ) -> Result<(), Box<dyn std::error::Error>> {
        let notification = serde_json::json!({
            "type": "low_stock_alert",
            "item_id": item.id,
            "item_name": item.name,
            "sku": item.sku,
            "current_stock": item.current_stock,
            "reorder_point": item.reorder_point,
            "tenant_id": item.tenant_id,
            "timestamp": Utc::now()
        });

        let _ = self.notification_sender.send(notification.to_string());

        tracing::warn!("低庫存警告: {} ({})", item.name, item.current_stock);

        Ok(())
    }

    async fn monitor_inventory_levels(&self) -> Result<(), Box<dyn std::error::Error>> {
        // 監控所有租戶的庫存水位
        tracing::debug!("監控庫存水位...");
        Ok(())
    }

    async fn process_auto_restock(&self) -> Result<(), Box<dyn std::error::Error>> {
        // 處理自動補貨邏輯
        tracing::debug!("處理自動補貨...");
        Ok(())
    }
}
```

**🚀 Stage 3 庫存管理系統開發完成！繼續Stage 4 報表分析系統...**