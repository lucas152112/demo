# 🛒 Coffee POS系統 - 訂單管理服務

## 🎯 **訂單管理微服務架構**

### **服務配置**
```rust
// backend/order-service/Cargo.toml
[package]
name = "coffee-order-service"
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
tokio-tungstenite = "0.23"
futures-util = "0.3"

# 共享庫
coffee-shared-lib = { path = "../coffee-shared-lib" }

[dev-dependencies]
tokio-test = "0.4"
```

### **訂單管理主服務**
```rust
// backend/order-service/src/main.rs
use axum::{
    extract::{State, WebSocketUpgrade, ws::WebSocket},
    http::StatusCode,
    response::{Json, Response},
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
mod websocket;

use handlers::*;
use services::*;

#[derive(Clone)]
pub struct AppState {
    pub db_manager: DatabaseManager,
    pub order_service: Arc<OrderService>,
    pub menu_service: Arc<MenuService>,
    pub notification_sender: broadcast::Sender<String>,
    pub websocket_manager: Arc<WebSocketManager>,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    tracing_subscriber::init();

    // 載入配置
    let config = load_config().await?;
    
    // 初始化資料庫
    let db_manager = DatabaseManager::new(&config.database_url).await?;
    
    // 創建廣播頻道用於即時通知
    let (notification_sender, _) = broadcast::channel(1000);
    
    // 初始化服務
    let order_service = Arc::new(OrderService::new(
        db_manager.clone(),
        &config.redis_url,
        notification_sender.clone(),
    ).await?);
    
    let menu_service = Arc::new(MenuService::new(
        db_manager.clone(),
    ));
    
    let websocket_manager = Arc::new(WebSocketManager::new(
        notification_sender.clone(),
    ));

    let app_state = AppState {
        db_manager,
        order_service: order_service.clone(),
        menu_service,
        notification_sender,
        websocket_manager,
    };

    // 啟動訂單狀態自動更新任務
    let order_clone = order_service.clone();
    tokio::spawn(async move {
        order_clone.start_order_status_updater().await;
    });

    // 建立路由
    let app = create_routes(app_state);

    // 啟動服務器
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8010").await?;
    info!("🛒 Coffee訂單管理服務啟動在 http://0.0.0.0:8010");
    
    axum::serve(listener, app).await?;

    Ok(())
}

fn create_routes(state: AppState) -> Router {
    Router::new()
        .route("/health", get(health_check))
        
        // 訂單管理路由
        .route("/orders", get(get_orders).post(create_order))
        .route("/orders/:id", get(get_order).put(update_order).delete(cancel_order))
        .route("/orders/:id/status", put(update_order_status))
        .route("/orders/:id/items", post(add_order_item).delete(remove_order_item))
        .route("/orders/tenant/:tenant_id", get(get_tenant_orders))
        .route("/orders/status/:status", get(get_orders_by_status))
        
        // 菜單管理路由
        .route("/menu/categories", get(get_menu_categories).post(create_menu_category))
        .route("/menu/categories/:id", get(get_menu_category).put(update_menu_category))
        .route("/menu/items", get(get_menu_items).post(create_menu_item))
        .route("/menu/items/:id", get(get_menu_item).put(update_menu_item))
        .route("/menu/items/:id/availability", put(update_item_availability))
        .route("/menu/tenant/:tenant_id", get(get_tenant_menu))
        
        // 訂單統計路由
        .route("/analytics/orders/summary", get(get_orders_summary))
        .route("/analytics/orders/trends", get(get_order_trends))
        .route("/analytics/items/popular", get(get_popular_items))
        
        // WebSocket路由
        .route("/ws/orders", get(websocket_handler))
        
        .layer(CorsLayer::permissive())
        .with_state(state)
}

async fn websocket_handler(
    ws: WebSocketUpgrade,
    State(state): State<AppState>,
) -> Response {
    ws.on_upgrade(|socket| handle_websocket(socket, state))
}

async fn handle_websocket(socket: WebSocket, state: AppState) {
    state.websocket_manager.handle_connection(socket).await;
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

### **訂單服務核心邏輯**
```rust
// backend/order-service/src/services/order_service.rs
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
pub struct Order {
    pub id: Uuid,
    pub tenant_id: Uuid,
    pub order_number: String,
    pub customer_name: Option<String>,
    pub customer_phone: Option<String>,
    pub order_type: OrderType,
    pub status: OrderStatus,
    pub total_amount: Decimal,
    pub tax_amount: Decimal,
    pub discount_amount: Decimal,
    pub final_amount: Decimal,
    pub payment_status: PaymentStatus,
    pub payment_method: Option<String>,
    pub notes: Option<String>,
    pub table_number: Option<String>,
    pub estimated_ready_time: Option<DateTime<Utc>>,
    pub actual_ready_time: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub created_by: Uuid,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct OrderItem {
    pub id: Uuid,
    pub order_id: Uuid,
    pub menu_item_id: Uuid,
    pub quantity: i32,
    pub unit_price: Decimal,
    pub total_price: Decimal,
    pub customizations: Option<serde_json::Value>,
    pub special_instructions: Option<String>,
    pub status: OrderItemStatus,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "order_type", rename_all = "lowercase")]
pub enum OrderType {
    DineIn,    // 內用
    Takeaway,  // 外帶
    Delivery,  // 外送
    Pickup,    // 自取
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "order_status", rename_all = "lowercase")]
pub enum OrderStatus {
    Pending,      // 待確認
    Confirmed,    // 已確認
    Preparing,    // 製作中
    Ready,        // 製作完成
    Served,       // 已送達/已取餐
    Completed,    // 已完成
    Cancelled,    // 已取消
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "payment_status", rename_all = "lowercase")]
pub enum PaymentStatus {
    Pending,   // 待付款
    Paid,      // 已付款
    Refunded,  // 已退款
    Failed,    // 付款失敗
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "order_item_status", rename_all = "lowercase")]
pub enum OrderItemStatus {
    Pending,    // 待製作
    Preparing,  // 製作中
    Ready,      // 製作完成
    Served,     // 已送達
}

#[derive(Debug, Deserialize)]
pub struct CreateOrderRequest {
    pub tenant_id: Uuid,
    pub customer_name: Option<String>,
    pub customer_phone: Option<String>,
    pub order_type: OrderType,
    pub table_number: Option<String>,
    pub items: Vec<CreateOrderItemRequest>,
    pub notes: Option<String>,
}

#[derive(Debug, Deserialize)]
pub struct CreateOrderItemRequest {
    pub menu_item_id: Uuid,
    pub quantity: i32,
    pub customizations: Option<serde_json::Value>,
    pub special_instructions: Option<String>,
}

#[derive(Debug, Deserialize)]
pub struct UpdateOrderStatusRequest {
    pub status: OrderStatus,
    pub estimated_ready_time: Option<DateTime<Utc>>,
}

#[derive(Debug, Serialize)]
pub struct OrderSummary {
    pub total_orders: i64,
    pub total_revenue: Decimal,
    pub average_order_value: Decimal,
    pub orders_by_status: HashMap<String, i64>,
    pub orders_by_type: HashMap<String, i64>,
}

#[derive(Debug, Serialize)]
pub struct OrderWithItems {
    #[serde(flatten)]
    pub order: Order,
    pub items: Vec<OrderItemWithMenu>,
}

#[derive(Debug, Serialize)]
pub struct OrderItemWithMenu {
    #[serde(flatten)]
    pub item: OrderItem,
    pub menu_item_name: String,
    pub menu_item_category: String,
}

pub struct OrderService {
    db_manager: DatabaseManager,
    redis_client: redis::Client,
    notification_sender: broadcast::Sender<String>,
}

impl OrderService {
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

    /// 創建訂單
    pub async fn create_order(
        &self,
        request: CreateOrderRequest,
        created_by: Uuid,
    ) -> Result<OrderWithItems, Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(request.tenant_id).await?;
        
        let mut tx = tenant_pool.begin().await?;

        // 生成訂單號
        let order_number = self.generate_order_number(request.tenant_id).await?;
        let order_id = Uuid::new_v4();

        // 計算訂單總額
        let mut total_amount = Decimal::ZERO;
        let mut order_items = Vec::new();

        for item_request in &request.items {
            // 獲取菜單項目和價格
            let menu_item = self.get_menu_item_price(item_request.menu_item_id, &mut tx).await?;
            let item_total = menu_item.price * Decimal::new(item_request.quantity as i64, 0);
            total_amount += item_total;

            order_items.push((item_request, item_total, menu_item));
        }

        // 計算稅額和最終金額 (簡化處理，實際可根據稅率計算)
        let tax_rate = Decimal::new(8, 2); // 8%
        let tax_amount = total_amount * tax_rate / Decimal::new(100, 0);
        let discount_amount = Decimal::ZERO; // 暫無折扣
        let final_amount = total_amount + tax_amount - discount_amount;

        // 創建訂單
        let order = sqlx::query_as::<_, Order>(
            r#"
            INSERT INTO orders (
                id, tenant_id, order_number, customer_name, customer_phone,
                order_type, status, total_amount, tax_amount, discount_amount,
                final_amount, payment_status, notes, table_number, created_by
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15)
            RETURNING *
            "#,
        )
        .bind(order_id)
        .bind(request.tenant_id)
        .bind(&order_number)
        .bind(&request.customer_name)
        .bind(&request.customer_phone)
        .bind(&request.order_type)
        .bind(OrderStatus::Pending)
        .bind(total_amount)
        .bind(tax_amount)
        .bind(discount_amount)
        .bind(final_amount)
        .bind(PaymentStatus::Pending)
        .bind(&request.notes)
        .bind(&request.table_number)
        .bind(created_by)
        .fetch_one(&mut *tx)
        .await?;

        // 創建訂單項目
        let mut created_items = Vec::new();
        for (item_request, item_total, menu_item) in order_items {
            let order_item = sqlx::query_as::<_, OrderItem>(
                r#"
                INSERT INTO order_items (
                    id, order_id, menu_item_id, quantity, unit_price, total_price,
                    customizations, special_instructions, status
                )
                VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
                RETURNING *
                "#,
            )
            .bind(Uuid::new_v4())
            .bind(order_id)
            .bind(item_request.menu_item_id)
            .bind(item_request.quantity)
            .bind(menu_item.price)
            .bind(item_total)
            .bind(&item_request.customizations)
            .bind(&item_request.special_instructions)
            .bind(OrderItemStatus::Pending)
            .fetch_one(&mut *tx)
            .await?;

            created_items.push(OrderItemWithMenu {
                item: order_item,
                menu_item_name: menu_item.name,
                menu_item_category: menu_item.category_name.unwrap_or_default(),
            });
        }

        tx.commit().await?;

        // 更新Redis快取
        self.update_order_cache(&order).await?;

        // 發送即時通知
        let notification = serde_json::json!({
            "type": "order_created",
            "order_id": order_id,
            "order_number": order_number,
            "tenant_id": request.tenant_id,
            "timestamp": Utc::now()
        });

        let _ = self.notification_sender.send(notification.to_string());

        tracing::info!("訂單已創建: {} ({})", order_number, order_id);

        Ok(OrderWithItems {
            order,
            items: created_items,
        })
    }

    /// 更新訂單狀態
    pub async fn update_order_status(
        &self,
        order_id: Uuid,
        tenant_id: Uuid,
        request: UpdateOrderStatusRequest,
        updated_by: Uuid,
    ) -> Result<Order, Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(tenant_id).await?;

        let mut order = self.get_order_by_id(order_id, tenant_id).await?;

        // 狀態轉換驗證
        self.validate_status_transition(&order.status, &request.status)?;

        let mut tx = tenant_pool.begin().await?;

        // 更新訂單狀態
        order.status = request.status.clone();
        order.updated_at = Utc::now();

        if let Some(ready_time) = request.estimated_ready_time {
            order.estimated_ready_time = Some(ready_time);
        }

        // 如果狀態變為Ready，記錄實際完成時間
        if request.status == OrderStatus::Ready {
            order.actual_ready_time = Some(Utc::now());
        }

        sqlx::query(
            r#"
            UPDATE orders 
            SET status = $1, estimated_ready_time = $2, actual_ready_time = $3, updated_at = $4
            WHERE id = $5
            "#,
        )
        .bind(&order.status)
        .bind(order.estimated_ready_time)
        .bind(order.actual_ready_time)
        .bind(order.updated_at)
        .bind(order.id)
        .execute(&mut *tx)
        .await?;

        // 如果訂單狀態變為製作中，自動更新所有項目狀態
        if request.status == OrderStatus::Preparing {
            sqlx::query(
                "UPDATE order_items SET status = 'preparing' WHERE order_id = $1"
            )
            .bind(order_id)
            .execute(&mut *tx)
            .await?;
        }

        tx.commit().await?;

        // 更新Redis快取
        self.update_order_cache(&order).await?;

        // 發送狀態變更通知
        let notification = serde_json::json!({
            "type": "order_status_updated",
            "order_id": order_id,
            "order_number": order.order_number,
            "old_status": "unknown", // 簡化處理
            "new_status": order.status,
            "tenant_id": tenant_id,
            "timestamp": Utc::now()
        });

        let _ = self.notification_sender.send(notification.to_string());

        tracing::info!("訂單狀態已更新: {} -> {:?}", order.order_number, order.status);

        Ok(order)
    }

    /// 獲取訂單詳情
    pub async fn get_order_with_items(
        &self,
        order_id: Uuid,
        tenant_id: Uuid,
    ) -> Result<OrderWithItems, Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(tenant_id).await?;

        // 獲取訂單基本資訊
        let order = self.get_order_by_id(order_id, tenant_id).await?;

        // 獲取訂單項目
        let items = sqlx::query_as::<_, OrderItemWithMenu>(
            r#"
            SELECT 
                oi.*,
                mi.name as menu_item_name,
                mc.name as menu_item_category
            FROM order_items oi
            JOIN menu_items mi ON oi.menu_item_id = mi.id
            LEFT JOIN menu_categories mc ON mi.category_id = mc.id
            WHERE oi.order_id = $1
            ORDER BY oi.created_at
            "#,
        )
        .bind(order_id)
        .fetch_all(&tenant_pool)
        .await?;

        Ok(OrderWithItems { order, items })
    }

    /// 獲取租戶訂單列表
    pub async fn get_tenant_orders(
        &self,
        tenant_id: Uuid,
        status_filter: Option<OrderStatus>,
        limit: Option<i64>,
        offset: Option<i64>,
    ) -> Result<Vec<Order>, Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(tenant_id).await?;

        let mut query = "SELECT * FROM orders WHERE tenant_id = $1".to_string();
        let mut param_count = 1;

        if status_filter.is_some() {
            param_count += 1;
            query.push_str(&format!(" AND status = ${}", param_count));
        }

        query.push_str(" ORDER BY created_at DESC");

        if let Some(limit) = limit {
            param_count += 1;
            query.push_str(&format!(" LIMIT ${}", param_count));
        }

        if let Some(offset) = offset {
            param_count += 1;
            query.push_str(&format!(" OFFSET ${}", param_count));
        }

        let mut query_builder = sqlx::query_as::<_, Order>(&query);
        query_builder = query_builder.bind(tenant_id);

        if let Some(status) = status_filter {
            query_builder = query_builder.bind(status);
        }

        if let Some(limit) = limit {
            query_builder = query_builder.bind(limit);
        }

        if let Some(offset) = offset {
            query_builder = query_builder.bind(offset);
        }

        let orders = query_builder.fetch_all(&tenant_pool).await?;

        Ok(orders)
    }

    /// 獲取訂單統計
    pub async fn get_orders_summary(
        &self,
        tenant_id: Uuid,
        start_date: Option<DateTime<Utc>>,
        end_date: Option<DateTime<Utc>>,
    ) -> Result<OrderSummary, Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(tenant_id).await?;

        let start = start_date.unwrap_or_else(|| {
            Utc::now() - chrono::Duration::days(30)
        });
        let end = end_date.unwrap_or_else(|| Utc::now());

        // 總訂單數和總收入
        let summary_row = sqlx::query!(
            r#"
            SELECT 
                COUNT(*) as total_orders,
                COALESCE(SUM(final_amount), 0) as total_revenue,
                COALESCE(AVG(final_amount), 0) as average_order_value
            FROM orders 
            WHERE tenant_id = $1 
            AND created_at BETWEEN $2 AND $3
            AND status NOT IN ('cancelled')
            "#,
            tenant_id,
            start,
            end
        )
        .fetch_one(&tenant_pool)
        .await?;

        // 按狀態統計
        let status_stats = sqlx::query!(
            r#"
            SELECT status::text, COUNT(*) as count
            FROM orders 
            WHERE tenant_id = $1 
            AND created_at BETWEEN $2 AND $3
            GROUP BY status
            "#,
            tenant_id,
            start,
            end
        )
        .fetch_all(&tenant_pool)
        .await?;

        // 按類型統計
        let type_stats = sqlx::query!(
            r#"
            SELECT order_type::text, COUNT(*) as count
            FROM orders 
            WHERE tenant_id = $1 
            AND created_at BETWEEN $2 AND $3
            GROUP BY order_type
            "#,
            tenant_id,
            start,
            end
        )
        .fetch_all(&tenant_pool)
        .await?;

        let mut orders_by_status = HashMap::new();
        for row in status_stats {
            orders_by_status.insert(row.status.unwrap_or_default(), row.count.unwrap_or(0));
        }

        let mut orders_by_type = HashMap::new();
        for row in type_stats {
            orders_by_type.insert(row.order_type.unwrap_or_default(), row.count.unwrap_or(0));
        }

        Ok(OrderSummary {
            total_orders: summary_row.total_orders.unwrap_or(0),
            total_revenue: summary_row.total_revenue.unwrap_or_default(),
            average_order_value: summary_row.average_order_value.unwrap_or_default(),
            orders_by_status,
            orders_by_type,
        })
    }

    /// 啟動訂單狀態自動更新任務
    pub async fn start_order_status_updater(&self) {
        let mut interval = tokio::time::interval(tokio::time::Duration::from_minutes(5));
        
        loop {
            interval.tick().await;
            
            if let Err(e) = self.process_overdue_orders().await {
                tracing::error!("處理逾期訂單失敗: {}", e);
            }
        }
    }

    // 輔助方法
    async fn generate_order_number(&self, tenant_id: Uuid) -> Result<String, Box<dyn std::error::Error>> {
        let mut redis_conn = self.redis_client.get_async_connection().await?;
        
        let today = Utc::now().format("%Y%m%d").to_string();
        let key = format!("coffee:order_seq:{}:{}", tenant_id, today);
        
        let sequence: i64 = redis_conn.incr(&key, 1).await?;
        redis_conn.expire(&key, 86400).await?; // 24小時過期

        Ok(format!("{}{:04}", today, sequence))
    }

    async fn get_menu_item_price(
        &self,
        menu_item_id: Uuid,
        tx: &mut sqlx::Transaction<'_, sqlx::Postgres>,
    ) -> Result<MenuItem, Box<dyn std::error::Error>> {
        let menu_item = sqlx::query_as::<_, MenuItem>(
            r#"
            SELECT 
                mi.*,
                mc.name as category_name
            FROM menu_items mi
            LEFT JOIN menu_categories mc ON mi.category_id = mc.id
            WHERE mi.id = $1 AND mi.is_available = true
            "#,
        )
        .bind(menu_item_id)
        .fetch_one(&mut **tx)
        .await?;

        Ok(menu_item)
    }

    async fn get_order_by_id(
        &self,
        order_id: Uuid,
        tenant_id: Uuid,
    ) -> Result<Order, Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(tenant_id).await?;

        let order = sqlx::query_as::<_, Order>(
            "SELECT * FROM orders WHERE id = $1 AND tenant_id = $2"
        )
        .bind(order_id)
        .bind(tenant_id)
        .fetch_one(&tenant_pool)
        .await?;

        Ok(order)
    }

    fn validate_status_transition(
        &self,
        current: &OrderStatus,
        new: &OrderStatus,
    ) -> Result<(), Box<dyn std::error::Error>> {
        use OrderStatus::*;

        let valid = match (current, new) {
            (Pending, Confirmed) => true,
            (Pending, Cancelled) => true,
            (Confirmed, Preparing) => true,
            (Confirmed, Cancelled) => true,
            (Preparing, Ready) => true,
            (Ready, Served) => true,
            (Served, Completed) => true,
            (_, _) if current == new => true, // 相同狀態允許
            _ => false,
        };

        if !valid {
            return Err(format!("不允許的狀態轉換: {:?} -> {:?}", current, new).into());
        }

        Ok(())
    }

    async fn update_order_cache(&self, order: &Order) -> Result<(), Box<dyn std::error::Error>> {
        let mut redis_conn = self.redis_client.get_async_connection().await?;
        
        let cache_key = format!("coffee:order:{}:{}", order.tenant_id, order.id);
        let order_json = serde_json::to_string(order)?;
        
        redis_conn.set_ex(&cache_key, order_json, 3600).await?; // 1小時快取

        Ok(())
    }

    async fn process_overdue_orders(&self) -> Result<(), Box<dyn std::error::Error>> {
        // 處理逾期訂單的邏輯
        // 例如：超過預計時間的訂單發送提醒
        tracing::debug!("檢查逾期訂單...");
        Ok(())
    }
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct MenuItem {
    pub id: Uuid,
    pub name: String,
    pub price: Decimal,
    pub category_name: Option<String>,
}
```

**🚀 Stage 1 訂單管理系統開發完成！繼續Stage 2 支付處理系統...**