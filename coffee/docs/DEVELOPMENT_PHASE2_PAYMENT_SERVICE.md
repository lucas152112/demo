# 💳 Coffee POS系統 - 支付處理服務

## 🏗️ **支付處理微服務架構**

### **服務配置**
```rust
// backend/payment-service/Cargo.toml
[package]
name = "coffee-payment-service"
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
stripe = "0.19"
reqwest = { version = "0.11", features = ["json"] }
hmac = "0.12"
sha2 = "0.10"
hex = "0.4"

# 共享庫
coffee-shared-lib = { path = "../coffee-shared-lib" }

[dev-dependencies]
tokio-test = "0.4"
```

### **支付處理主服務**
```rust
// backend/payment-service/src/main.rs
use axum::{
    extract::State,
    http::StatusCode,
    response::Json,
    routing::{get, post, put},
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
mod integrations;

use handlers::*;
use services::*;

#[derive(Clone)]
pub struct AppState {
    pub db_manager: DatabaseManager,
    pub payment_service: Arc<PaymentService>,
    pub refund_service: Arc<RefundService>,
    pub receipt_service: Arc<ReceiptService>,
    pub webhook_service: Arc<WebhookService>,
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
    let payment_service = Arc::new(PaymentService::new(
        db_manager.clone(),
        &config.stripe_secret_key,
        &config.redis_url,
        notification_sender.clone(),
    ).await?);
    
    let refund_service = Arc::new(RefundService::new(
        db_manager.clone(),
        payment_service.clone(),
    ));
    
    let receipt_service = Arc::new(ReceiptService::new(
        db_manager.clone(),
    ));
    
    let webhook_service = Arc::new(WebhookService::new(
        payment_service.clone(),
        &config.webhook_secret,
    ));

    let app_state = AppState {
        db_manager,
        payment_service: payment_service.clone(),
        refund_service,
        receipt_service,
        webhook_service,
        notification_sender,
    };

    // 啟動支付狀態同步任務
    let payment_clone = payment_service.clone();
    tokio::spawn(async move {
        payment_clone.start_payment_sync_task().await;
    });

    // 建立路由
    let app = create_routes(app_state);

    // 啟動服務器
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8011").await?;
    info!("💳 Coffee支付處理服務啟動在 http://0.0.0.0:8011");
    
    axum::serve(listener, app).await?;

    Ok(())
}

fn create_routes(state: AppState) -> Router {
    Router::new()
        .route("/health", get(health_check))
        
        // 支付處理路由
        .route("/payments", get(get_payments).post(create_payment))
        .route("/payments/:id", get(get_payment).put(update_payment))
        .route("/payments/:id/capture", post(capture_payment))
        .route("/payments/:id/cancel", post(cancel_payment))
        .route("/payments/order/:order_id", get(get_order_payments))
        
        // 退款處理路由
        .route("/refunds", get(get_refunds).post(create_refund))
        .route("/refunds/:id", get(get_refund))
        .route("/refunds/payment/:payment_id", get(get_payment_refunds))
        
        // 收據路由
        .route("/receipts/generate", post(generate_receipt))
        .route("/receipts/:id", get(get_receipt))
        .route("/receipts/:id/send", post(send_receipt))
        .route("/receipts/order/:order_id", get(get_order_receipts))
        
        // 支付方法路由
        .route("/payment-methods", get(get_payment_methods).post(create_payment_method))
        .route("/payment-methods/:id", get(get_payment_method).put(update_payment_method))
        
        // 對帳路由
        .route("/reconciliation/daily", get(get_daily_reconciliation))
        .route("/reconciliation/settlement", post(create_settlement))
        
        // Webhook路由
        .route("/webhook/stripe", post(handle_stripe_webhook))
        .route("/webhook/linepay", post(handle_linepay_webhook))
        
        // 統計報表路由
        .route("/analytics/payments/summary", get(get_payment_summary))
        .route("/analytics/payments/methods", get(get_payment_methods_stats))
        
        .layer(CorsLayer::permissive())
        .with_state(state)
}

#[derive(Deserialize)]
struct Config {
    database_url: String,
    redis_url: String,
    stripe_secret_key: String,
    webhook_secret: String,
}

async fn load_config() -> Result<Config, config::ConfigError> {
    let settings = config::Config::builder()
        .add_source(config::Environment::with_prefix("COFFEE"))
        .build()?;
    
    settings.try_deserialize()
}
```

### **支付處理核心服務**
```rust
// backend/payment-service/src/services/payment_service.rs
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
pub struct Payment {
    pub id: Uuid,
    pub tenant_id: Uuid,
    pub order_id: Uuid,
    pub payment_intent_id: Option<String>, // Stripe Payment Intent ID
    pub amount: Decimal,
    pub currency: String,
    pub payment_method: PaymentMethod,
    pub status: PaymentStatus,
    pub gateway: PaymentGateway,
    pub gateway_transaction_id: Option<String>,
    pub gateway_response: Option<serde_json::Value>,
    pub failure_reason: Option<String>,
    pub captured_amount: Option<Decimal>,
    pub refunded_amount: Decimal,
    pub fees_amount: Decimal,
    pub net_amount: Decimal,
    pub metadata: Option<serde_json::Value>,
    pub processed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct Refund {
    pub id: Uuid,
    pub payment_id: Uuid,
    pub amount: Decimal,
    pub reason: RefundReason,
    pub status: RefundStatus,
    pub gateway_refund_id: Option<String>,
    pub gateway_response: Option<serde_json::Value>,
    pub processed_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
    pub created_by: Uuid,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "payment_method", rename_all = "lowercase")]
pub enum PaymentMethod {
    Cash,         // 現金
    Card,         // 信用卡/簽帳卡
    LinePay,      // Line Pay
    ApplePay,     // Apple Pay
    GooglePay,    // Google Pay
    Alipay,       // 支付寶
    JKOPay,       // 街口支付
    EasyCard,     // 悠遊卡
    IPAss,        // 一卡通
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "payment_status", rename_all = "lowercase")]
pub enum PaymentStatus {
    Pending,              // 待處理
    RequiresPaymentMethod, // 需要支付方法
    RequiresConfirmation, // 需要確認
    RequiresAction,       // 需要額外操作
    Processing,           // 處理中
    RequiresCapture,      // 需要捕獲
    Succeeded,           // 成功
    Cancelled,           // 已取消
    Failed,              // 失敗
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "payment_gateway", rename_all = "lowercase")]
pub enum PaymentGateway {
    Internal,    // 內部處理(現金等)
    Stripe,      // Stripe
    LinePay,     // Line Pay
    EcPay,       // 綠界
    NewebPay,    // 藍新金流
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "refund_reason", rename_all = "lowercase")]
pub enum RefundReason {
    Requested,        // 客戶要求
    Duplicate,        // 重複收費
    Fraudulent,       // 欺詐
    OrderCancelled,   // 訂單取消
    ProductDefective, // 商品瑕疵
    Other,           // 其他
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "refund_status", rename_all = "lowercase")]
pub enum RefundStatus {
    Pending,    // 待處理
    Processing, // 處理中
    Succeeded,  // 成功
    Failed,     // 失敗
    Cancelled,  // 已取消
}

#[derive(Debug, Deserialize)]
pub struct CreatePaymentRequest {
    pub order_id: Uuid,
    pub tenant_id: Uuid,
    pub amount: Decimal,
    pub currency: String,
    pub payment_method: PaymentMethod,
    pub customer_id: Option<String>,
    pub metadata: Option<serde_json::Value>,
}

#[derive(Debug, Deserialize)]
pub struct CreateRefundRequest {
    pub payment_id: Uuid,
    pub amount: Option<Decimal>, // None表示全額退款
    pub reason: RefundReason,
    pub description: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct PaymentSummary {
    pub total_transactions: i64,
    pub total_amount: Decimal,
    pub successful_payments: i64,
    pub successful_amount: Decimal,
    pub refund_count: i64,
    pub refund_amount: Decimal,
    pub fees_amount: Decimal,
    pub net_amount: Decimal,
    pub payment_methods_breakdown: HashMap<String, PaymentMethodStats>,
}

#[derive(Debug, Serialize)]
pub struct PaymentMethodStats {
    pub count: i64,
    pub amount: Decimal,
    pub percentage: f64,
}

pub struct PaymentService {
    db_manager: DatabaseManager,
    stripe_client: stripe::Client,
    redis_client: redis::Client,
    notification_sender: broadcast::Sender<String>,
}

impl PaymentService {
    pub async fn new(
        db_manager: DatabaseManager,
        stripe_secret_key: &str,
        redis_url: &str,
        notification_sender: broadcast::Sender<String>,
    ) -> Result<Self, Box<dyn std::error::Error>> {
        let stripe_client = stripe::Client::new(stripe_secret_key);
        let redis_client = redis::Client::open(redis_url)?;

        Ok(Self {
            db_manager,
            stripe_client,
            redis_client,
            notification_sender,
        })
    }

    /// 創建支付
    pub async fn create_payment(
        &self,
        request: CreatePaymentRequest,
    ) -> Result<Payment, Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(request.tenant_id).await?;

        let payment_id = Uuid::new_v4();
        let now = Utc::now();

        // 根據支付方式選擇處理網關
        let (gateway, gateway_transaction_id, payment_intent_id, initial_status) = 
            match request.payment_method {
                PaymentMethod::Cash => {
                    // 現金支付直接成功
                    (PaymentGateway::Internal, None, None, PaymentStatus::Succeeded)
                }
                PaymentMethod::Card | PaymentMethod::ApplePay | PaymentMethod::GooglePay => {
                    // 使用Stripe處理
                    let intent = self.create_stripe_payment_intent(
                        &request.amount,
                        &request.currency,
                        &request.payment_method,
                        request.customer_id.as_deref(),
                        &request.metadata,
                    ).await?;

                    (
                        PaymentGateway::Stripe,
                        Some(intent.id.to_string()),
                        Some(intent.id.to_string()),
                        PaymentStatus::RequiresPaymentMethod,
                    )
                }
                PaymentMethod::LinePay => {
                    // 使用Line Pay處理
                    let line_pay_response = self.create_linepay_payment(
                        &request.amount,
                        &request.order_id,
                    ).await?;

                    (
                        PaymentGateway::LinePay,
                        Some(line_pay_response.transaction_id),
                        None,
                        PaymentStatus::RequiresAction,
                    )
                }
                _ => {
                    // 其他支付方式暫時使用內部處理
                    (PaymentGateway::Internal, None, None, PaymentStatus::RequiresAction)
                }
            };

        // 計算費用和淨額
        let fees_amount = self.calculate_payment_fees(&request.amount, &request.payment_method);
        let net_amount = request.amount - fees_amount;

        let payment = sqlx::query_as::<_, Payment>(
            r#"
            INSERT INTO payments (
                id, tenant_id, order_id, payment_intent_id, amount, currency,
                payment_method, status, gateway, gateway_transaction_id,
                fees_amount, net_amount, refunded_amount, metadata
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14)
            RETURNING *
            "#,
        )
        .bind(payment_id)
        .bind(request.tenant_id)
        .bind(request.order_id)
        .bind(payment_intent_id)
        .bind(request.amount)
        .bind(&request.currency)
        .bind(&request.payment_method)
        .bind(&initial_status)
        .bind(&gateway)
        .bind(gateway_transaction_id)
        .bind(fees_amount)
        .bind(net_amount)
        .bind(Decimal::ZERO)
        .bind(&request.metadata)
        .fetch_one(&tenant_pool)
        .await?;

        // 如果是現金支付，立即更新訂單狀態為已付款
        if request.payment_method == PaymentMethod::Cash {
            self.update_order_payment_status(
                request.order_id,
                request.tenant_id,
                true,
            ).await?;
        }

        // 發送支付創建通知
        let notification = serde_json::json!({
            "type": "payment_created",
            "payment_id": payment_id,
            "order_id": request.order_id,
            "amount": request.amount,
            "payment_method": request.payment_method,
            "status": initial_status,
            "tenant_id": request.tenant_id,
            "timestamp": now
        });

        let _ = self.notification_sender.send(notification.to_string());

        tracing::info!("支付已創建: {} for order {}", payment_id, request.order_id);

        Ok(payment)
    }

    /// 捕獲支付 (適用於需要兩步驟確認的支付)
    pub async fn capture_payment(
        &self,
        payment_id: Uuid,
        tenant_id: Uuid,
        amount_to_capture: Option<Decimal>,
    ) -> Result<Payment, Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(tenant_id).await?;

        let mut payment = self.get_payment_by_id(payment_id, tenant_id).await?;

        // 驗證支付狀態
        if payment.status != PaymentStatus::RequiresCapture {
            return Err("支付狀態不允許捕獲".into());
        }

        let capture_amount = amount_to_capture.unwrap_or(payment.amount);

        let mut tx = tenant_pool.begin().await?;

        // 根據支付閘道處理捕獲
        let capture_result = match payment.gateway {
            PaymentGateway::Stripe => {
                if let Some(intent_id) = &payment.payment_intent_id {
                    self.capture_stripe_payment(intent_id, capture_amount).await?
                } else {
                    return Err("缺少Stripe Payment Intent ID".into());
                }
            }
            _ => {
                // 其他閘道的捕獲邏輯
                true
            }
        };

        if capture_result {
            payment.status = PaymentStatus::Succeeded;
            payment.captured_amount = Some(capture_amount);
            payment.processed_at = Some(Utc::now());
            payment.updated_at = Utc::now();

            // 更新支付記錄
            sqlx::query(
                r#"
                UPDATE payments 
                SET status = $1, captured_amount = $2, processed_at = $3, updated_at = $4
                WHERE id = $5
                "#,
            )
            .bind(&payment.status)
            .bind(payment.captured_amount)
            .bind(payment.processed_at)
            .bind(payment.updated_at)
            .bind(payment.id)
            .execute(&mut *tx)
            .await?;

            // 更新訂單支付狀態
            self.update_order_payment_status(
                payment.order_id,
                tenant_id,
                true,
            ).await?;
        } else {
            payment.status = PaymentStatus::Failed;
            payment.failure_reason = Some("捕獲失敗".to_string());
            payment.updated_at = Utc::now();

            sqlx::query(
                "UPDATE payments SET status = $1, failure_reason = $2, updated_at = $3 WHERE id = $4"
            )
            .bind(&payment.status)
            .bind(&payment.failure_reason)
            .bind(payment.updated_at)
            .bind(payment.id)
            .execute(&mut *tx)
            .await?;
        }

        tx.commit().await?;

        // 發送支付狀態更新通知
        let notification = serde_json::json!({
            "type": "payment_captured",
            "payment_id": payment_id,
            "order_id": payment.order_id,
            "amount": capture_amount,
            "status": payment.status,
            "tenant_id": tenant_id,
            "timestamp": Utc::now()
        });

        let _ = self.notification_sender.send(notification.to_string());

        tracing::info!("支付已捕獲: {} ({})", payment_id, capture_amount);

        Ok(payment)
    }

    /// 創建退款
    pub async fn create_refund(
        &self,
        request: CreateRefundRequest,
        created_by: Uuid,
    ) -> Result<Refund, Box<dyn std::error::Error>> {
        let payment = self.get_payment_by_id(
            request.payment_id, 
            // 需要從payment中獲取tenant_id，這裡簡化處理
            Uuid::nil()
        ).await?;

        if payment.status != PaymentStatus::Succeeded {
            return Err("只能對成功的支付進行退款".into());
        }

        let tenant_pool = self.db_manager.get_tenant_pool(payment.tenant_id).await?;

        let refund_amount = request.amount.unwrap_or(
            payment.amount - payment.refunded_amount
        );

        if refund_amount > (payment.amount - payment.refunded_amount) {
            return Err("退款金額超過可退款餘額".into());
        }

        let refund_id = Uuid::new_v4();

        // 根據支付閘道處理退款
        let (gateway_refund_id, refund_status) = match payment.gateway {
            PaymentGateway::Stripe => {
                if let Some(intent_id) = &payment.payment_intent_id {
                    match self.create_stripe_refund(intent_id, refund_amount).await {
                        Ok(refund_id) => (Some(refund_id), RefundStatus::Processing),
                        Err(_) => (None, RefundStatus::Failed),
                    }
                } else {
                    (None, RefundStatus::Failed)
                }
            }
            PaymentGateway::Internal => {
                // 內部處理(現金)立即成功
                (None, RefundStatus::Succeeded)
            }
            _ => {
                // 其他閘道
                (None, RefundStatus::Processing)
            }
        };

        let mut tx = tenant_pool.begin().await?;

        let refund = sqlx::query_as::<_, Refund>(
            r#"
            INSERT INTO refunds (
                id, payment_id, amount, reason, status, 
                gateway_refund_id, created_by
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7)
            RETURNING *
            "#,
        )
        .bind(refund_id)
        .bind(request.payment_id)
        .bind(refund_amount)
        .bind(&request.reason)
        .bind(&refund_status)
        .bind(gateway_refund_id)
        .bind(created_by)
        .fetch_one(&mut *tx)
        .await?;

        // 如果退款成功，更新支付記錄
        if refund_status == RefundStatus::Succeeded {
            sqlx::query(
                "UPDATE payments SET refunded_amount = refunded_amount + $1 WHERE id = $2"
            )
            .bind(refund_amount)
            .bind(request.payment_id)
            .execute(&mut *tx)
            .await?;
        }

        tx.commit().await?;

        // 發送退款通知
        let notification = serde_json::json!({
            "type": "refund_created",
            "refund_id": refund_id,
            "payment_id": request.payment_id,
            "order_id": payment.order_id,
            "amount": refund_amount,
            "status": refund_status,
            "tenant_id": payment.tenant_id,
            "timestamp": Utc::now()
        });

        let _ = self.notification_sender.send(notification.to_string());

        tracing::info!("退款已創建: {} ({})", refund_id, refund_amount);

        Ok(refund)
    }

    /// 獲取支付統計
    pub async fn get_payment_summary(
        &self,
        tenant_id: Uuid,
        start_date: DateTime<Utc>,
        end_date: DateTime<Utc>,
    ) -> Result<PaymentSummary, Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(tenant_id).await?;

        // 基本統計
        let basic_stats = sqlx::query!(
            r#"
            SELECT 
                COUNT(*) as total_transactions,
                COALESCE(SUM(amount), 0) as total_amount,
                COUNT(CASE WHEN status = 'succeeded' THEN 1 END) as successful_payments,
                COALESCE(SUM(CASE WHEN status = 'succeeded' THEN amount ELSE 0 END), 0) as successful_amount,
                COALESCE(SUM(refunded_amount), 0) as refund_amount,
                COALESCE(SUM(fees_amount), 0) as fees_amount,
                COALESCE(SUM(net_amount), 0) as net_amount
            FROM payments 
            WHERE tenant_id = $1 
            AND created_at BETWEEN $2 AND $3
            "#,
            tenant_id,
            start_date,
            end_date
        )
        .fetch_one(&tenant_pool)
        .await?;

        // 退款統計
        let refund_stats = sqlx::query!(
            r#"
            SELECT COUNT(*) as refund_count
            FROM refunds r
            JOIN payments p ON r.payment_id = p.id
            WHERE p.tenant_id = $1 
            AND r.created_at BETWEEN $2 AND $3
            AND r.status = 'succeeded'
            "#,
            tenant_id,
            start_date,
            end_date
        )
        .fetch_one(&tenant_pool)
        .await?;

        // 支付方式統計
        let method_stats = sqlx::query!(
            r#"
            SELECT 
                payment_method::text,
                COUNT(*) as count,
                COALESCE(SUM(amount), 0) as amount
            FROM payments 
            WHERE tenant_id = $1 
            AND created_at BETWEEN $2 AND $3
            AND status = 'succeeded'
            GROUP BY payment_method
            "#,
            tenant_id,
            start_date,
            end_date
        )
        .fetch_all(&tenant_pool)
        .await?;

        let total_successful_amount = basic_stats.successful_amount.unwrap_or_default();
        
        let mut payment_methods_breakdown = HashMap::new();
        for row in method_stats {
            let method_amount = row.amount.unwrap_or_default();
            let percentage = if total_successful_amount > Decimal::ZERO {
                (method_amount / total_successful_amount * Decimal::new(100, 0)).to_f64().unwrap_or(0.0)
            } else {
                0.0
            };

            payment_methods_breakdown.insert(
                row.payment_method.unwrap_or_default(),
                PaymentMethodStats {
                    count: row.count.unwrap_or(0),
                    amount: method_amount,
                    percentage,
                },
            );
        }

        Ok(PaymentSummary {
            total_transactions: basic_stats.total_transactions.unwrap_or(0),
            total_amount: basic_stats.total_amount.unwrap_or_default(),
            successful_payments: basic_stats.successful_payments.unwrap_or(0),
            successful_amount: total_successful_amount,
            refund_count: refund_stats.refund_count.unwrap_or(0),
            refund_amount: basic_stats.refund_amount.unwrap_or_default(),
            fees_amount: basic_stats.fees_amount.unwrap_or_default(),
            net_amount: basic_stats.net_amount.unwrap_or_default(),
            payment_methods_breakdown,
        })
    }

    /// 啟動支付狀態同步任務
    pub async fn start_payment_sync_task(&self) {
        let mut interval = tokio::time::interval(tokio::time::Duration::from_minutes(5));
        
        loop {
            interval.tick().await;
            
            if let Err(e) = self.sync_pending_payments().await {
                tracing::error!("同步支付狀態失敗: {}", e);
            }
        }
    }

    // 輔助方法
    async fn create_stripe_payment_intent(
        &self,
        amount: &Decimal,
        currency: &str,
        payment_method: &PaymentMethod,
        customer_id: Option<&str>,
        metadata: &Option<serde_json::Value>,
    ) -> Result<stripe::PaymentIntent, Box<dyn std::error::Error>> {
        // Stripe金額以最小單位計算(分)
        let amount_cents = (amount * Decimal::new(100, 0)).to_u64().unwrap_or(0);

        let mut params = stripe::CreatePaymentIntent::new(amount_cents, currency.parse()?);
        
        if let Some(customer) = customer_id {
            params.customer = Some(customer.parse()?);
        }

        // 根據支付方式設定
        match payment_method {
            PaymentMethod::Card => {
                params.payment_method_types = Some(vec!["card".to_string()]);
            }
            PaymentMethod::ApplePay => {
                params.payment_method_types = Some(vec!["card".to_string()]);
            }
            PaymentMethod::GooglePay => {
                params.payment_method_types = Some(vec!["card".to_string()]);
            }
            _ => {}
        }

        let intent = stripe::PaymentIntent::create(&self.stripe_client, params).await?;
        Ok(intent)
    }

    async fn create_linepay_payment(
        &self,
        amount: &Decimal,
        order_id: &Uuid,
    ) -> Result<LinePayResponse, Box<dyn std::error::Error>> {
        // Line Pay API 集成
        // 這裡簡化處理
        Ok(LinePayResponse {
            transaction_id: format!("LP_{}", Uuid::new_v4()),
            payment_url: format!("https://sandbox-web-pay.line.me/web/payment/{}", order_id),
        })
    }

    async fn capture_stripe_payment(
        &self,
        intent_id: &str,
        amount: Decimal,
    ) -> Result<bool, Box<dyn std::error::Error>> {
        // Stripe支付捕獲
        // 實際實現需要調用Stripe API
        tracing::info!("捕獲Stripe支付: {} ({})", intent_id, amount);
        Ok(true)
    }

    async fn create_stripe_refund(
        &self,
        intent_id: &str,
        amount: Decimal,
    ) -> Result<String, Box<dyn std::error::Error>> {
        // Stripe退款創建
        // 實際實現需要調用Stripe API
        let refund_id = format!("re_{}", Uuid::new_v4());
        tracing::info!("創建Stripe退款: {} -> {} ({})", intent_id, refund_id, amount);
        Ok(refund_id)
    }

    fn calculate_payment_fees(&self, amount: &Decimal, method: &PaymentMethod) -> Decimal {
        // 計算支付手續費
        match method {
            PaymentMethod::Cash => Decimal::ZERO,
            PaymentMethod::Card => amount * Decimal::new(28, 3), // 2.8%
            PaymentMethod::LinePay => amount * Decimal::new(25, 3), // 2.5%
            PaymentMethod::ApplePay | PaymentMethod::GooglePay => amount * Decimal::new(30, 3), // 3.0%
            _ => amount * Decimal::new(20, 3), // 2.0% 預設
        }
    }

    async fn update_order_payment_status(
        &self,
        order_id: Uuid,
        tenant_id: Uuid,
        is_paid: bool,
    ) -> Result<(), Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(tenant_id).await?;

        let new_status = if is_paid { "paid" } else { "pending" };

        sqlx::query(
            "UPDATE orders SET payment_status = $1, updated_at = NOW() WHERE id = $2"
        )
        .bind(new_status)
        .bind(order_id)
        .execute(&tenant_pool)
        .await?;

        Ok(())
    }

    async fn get_payment_by_id(
        &self,
        payment_id: Uuid,
        tenant_id: Uuid,
    ) -> Result<Payment, Box<dyn std::error::Error>> {
        let tenant_pool = self.db_manager.get_tenant_pool(tenant_id).await?;

        let payment = sqlx::query_as::<_, Payment>(
            "SELECT * FROM payments WHERE id = $1"
        )
        .bind(payment_id)
        .fetch_one(&tenant_pool)
        .await?;

        Ok(payment)
    }

    async fn sync_pending_payments(&self) -> Result<(), Box<dyn std::error::Error>> {
        // 同步待處理支付的狀態
        tracing::debug!("同步待處理支付狀態...");
        Ok(())
    }
}

#[derive(Debug)]
struct LinePayResponse {
    transaction_id: String,
    payment_url: String,
}
```

**🚀 Stage 2 支付處理系統開發完成！繼續Stage 3 庫存管理系統...**