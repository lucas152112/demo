# 資料庫 ERD 設計文件 (MySQL 8.0+)

## 1. 資料庫架構總覽

### 1.1 資料庫結構

```
MySQL 8.0+
│
└── coffee_crm (單一資料庫)
    ├── users               (Auth Service)
    ├── customers_*         (Customers Service + Loyalty Service)  
    ├── orders_*            (Orders Service)
    ├── inventory_*         (Inventory Service)
    └── notifications_*     (Notifications Service)
```

**說明**: MySQL 使用單一資料庫 `coffee_crm`,透過資料表命名前綴區分不同服務的資料。

---

## 2. 完整 ERD 圖

```mermaid
erDiagram
    %% Auth
    users {
        char(36) id PK
        varchar(255) email UK
        varchar(255) password_hash
        varchar(100) name
        varchar(50) role
        char(36) store_id FK
        varchar(20) status
        datetime created_at
        datetime last_login_at
    }
    
    %% Customers
    customers {
        char(36) id PK
        char(36) owner_user_id FK
        varchar(100) name
        varchar(20) phone UK
        varchar(255) email
        date birthday
        varchar(20) member_number UK
        varchar(20) member_level
        decimal(12,2) total_spent
        int total_visits
        datetime created_at
        datetime last_visit_at
    }
    
    customer_tags {
        char(36) id PK
        char(36) customer_id FK
        varchar(50) tag_name
        datetime created_at
    }
    
    loyalty_accounts {
        char(36) customer_id PK_FK
        int available_points
        int total_earned
        int total_redeemed
        datetime updated_at
    }
    
    point_transactions {
        char(36) id PK
        char(36) customer_id FK
        varchar(20) type
        int amount
        char(36) order_id FK
        text description
        datetime created_at
    }
    
    %% Inventory
    stores {
        char(36) id PK
        varchar(20) code UK
        varchar(100) name
        varchar(255) address
        varchar(20) phone
        varchar(100) opening_hours
        decimal(5,4) tax_rate
        varchar(20) status
        datetime created_at
    }
    
    product_categories {
        char(36) id PK
        varchar(100) name
        char(36) parent_id FK
        int sort_order
        varchar(50) icon
        datetime created_at
    }
    
    products {
        char(36) id PK
        varchar(50) sku UK
        varchar(200) name
        char(36) category_id FK
        text description
        int price_cents
        int cost_cents
        varchar(255) image_url
        varchar(20) status
        datetime created_at
    }
    
    product_variants {
        char(36) id PK
        char(36) product_id FK
        varchar(50) type
        varchar(100) name
        int price_adjustment
        int sort_order
    }
    
    stock {
        char(36) store_id PK_FK
        char(36) product_id PK_FK
        int quantity
        int safety_stock
        int max_stock
        datetime updated_at
    }
    
    stock_movements {
        char(36) id PK
        char(36) store_id FK
        char(36) product_id FK
        int delta
        varchar(50) reason
        char(36) reference_id
        char(36) created_by FK
        datetime created_at
    }
    
    suppliers {
        char(36) id PK
        varchar(20) code UK
        varchar(200) name
        varchar(100) contact_person
        varchar(20) phone
        varchar(255) email
        varchar(255) address
        varchar(100) payment_terms
        tinyint rating
        varchar(20) status
        datetime created_at
    }
    
    purchase_orders {
        char(36) id PK
        varchar(30) po_number UK
        char(36) supplier_id FK
        char(36) store_id FK
        varchar(20) status
        int total_cents
        date expected_delivery_date
        char(36) created_by FK
        char(36) approved_by FK
        datetime approved_at
        text notes
        datetime created_at
    }
    
    purchase_order_items {
        char(36) id PK
        char(36) po_id FK
        char(36) product_id FK
        int quantity
        int received_quantity
        int unit_cost_cents
        int subtotal_cents
    }
    
    %% Orders
    orders {
        char(36) id PK
        varchar(30) order_number UK
        char(36) store_id FK
        char(36) user_id FK
        char(36) customer_id FK
        varchar(20) status
        int subtotal_cents
        int tax_cents
        int discount_cents
        int total_cents
        varchar(50) payment_method
        int points_earned
        int points_redeemed
        datetime created_at
        datetime completed_at
    }
    
    order_items {
        char(36) id PK
        char(36) order_id FK
        char(36) product_id FK
        varchar(200) product_name
        int quantity
        int unit_price_cents
        int subtotal_cents
        json variants
    }
    
    payments {
        char(36) id PK
        char(36) order_id FK
        varchar(50) method
        int amount_cents
        int received_cents
        int change_cents
        varchar(100) transaction_id
        varchar(20) status
        datetime paid_at
    }
    
    invoices {
        char(36) id PK
        char(36) order_id FK
        varchar(50) invoice_number UK
        varchar(20) tax_id
        datetime issued_at
        tinyint(1) printed
    }
    
    %% Notifications
    notifications {
        char(36) id PK
        char(36) user_id FK
        varchar(50) type
        varchar(200) title
        text message
        tinyint(1) is_read
        datetime created_at
        datetime read_at
    }
    
    notification_templates {
        char(36) id PK
        varchar(50) template_key UK
        varchar(200) subject
        text body
        varchar(20) type
        datetime created_at
    }
    
    %% Relationships
    
    users ||--o| stores : "works_at"
    
    customers ||--o{ customer_tags : "has"
    customers ||--|| loyalty_accounts : "has"
    customers ||--o{ point_transactions : "has"
    customers ||--o{ orders : "places"
    
    product_categories ||--o{ product_categories : "parent"
    product_categories ||--o{ products : "contains"
    
    products ||--o{ product_variants : "has"
    products ||--o{ stock : "stored_in"
    products ||--o{ stock_movements : "moved"
    products ||--o{ purchase_order_items : "ordered"
    products ||--o{ order_items : "ordered_in"
    
    stores ||--o{ stock : "stores"
    stores ||--o{ stock_movements : "at"
    stores ||--o{ purchase_orders : "receives"
    stores ||--o{ orders : "has"
    
    suppliers ||--o{ purchase_orders : "supplies"
    
    purchase_orders ||--o{ purchase_order_items : "contains"
    
    orders ||--o{ order_items : "contains"
    orders ||--|| payments : "paid_by"
    orders ||--o| invoices : "has"
    orders ||--o{ point_transactions : "earns"
    
    users ||--o{ orders : "creates"
    users ||--o{ stock_movements : "records"
    users ||--o{ purchase_orders : "creates"
    users ||--o{ notifications : "receives"
```

---

## 3. 資料表詳細設計

### 3.1 Auth 模組

#### users (使用者)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 使用者ID (UUID) |
| email | VARCHAR(255) | UK, NOT NULL | 登入帳號 |
| password_hash | VARCHAR(255) | NOT NULL | 密碼雜湊(bcrypt) |
| name | VARCHAR(100) | NOT NULL | 使用者姓名 |
| role | VARCHAR(50) | NOT NULL | 角色(admin/headquarters/store_manager/staff) |
| store_id | CHAR(36) | FK, NULL | 所屬門店(總部人員為NULL) |
| status | VARCHAR(20) | NOT NULL DEFAULT 'active' | 狀態(active/inactive/locked) |
| created_at | DATETIME | NOT NULL DEFAULT CURRENT_TIMESTAMP | 建立時間 |
| last_login_at | DATETIME | NULL | 最後登入時間 |

**索引**:
```sql
CREATE UNIQUE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_store ON users(store_id);
CREATE INDEX idx_users_role ON users(role);
```

**建表語句**:
```sql
CREATE TABLE users (
    id CHAR(36) PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    role VARCHAR(50) NOT NULL,
    store_id CHAR(36),
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_login_at DATETIME,
    INDEX idx_users_store (store_id),
    INDEX idx_users_role (role)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

### 3.2 Customers 模組

#### customers (客戶)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 客戶ID |
| owner_user_id | CHAR(36) | FK, NOT NULL | 建立此客戶的使用者 |
| name | VARCHAR(100) | NOT NULL | 客戶姓名 |
| phone | VARCHAR(20) | UK, NOT NULL | 手機號碼(唯一識別) |
| email | VARCHAR(255) | NULL | Email |
| birthday | DATE | NULL | 生日 |
| member_number | VARCHAR(20) | UK, NOT NULL | 會員編號(MB-YYYYMMDD-XXXX) |
| member_level | VARCHAR(20) | NOT NULL DEFAULT 'regular' | 會員等級(regular/silver/gold/platinum) |
| total_spent | DECIMAL(12,2) | NOT NULL DEFAULT 0 | 累計消費金額 |
| total_visits | INT | NOT NULL DEFAULT 0 | 累計消費次數 |
| created_at | DATETIME | NOT NULL DEFAULT CURRENT_TIMESTAMP | 建立時間 |
| last_visit_at | DATETIME | NULL | 最後消費時間 |

**索引**:
```sql
CREATE UNIQUE INDEX idx_customers_phone ON customers(phone);
CREATE UNIQUE INDEX idx_customers_member_number ON customers(member_number);
CREATE INDEX idx_customers_member_level ON customers(member_level);
CREATE INDEX idx_customers_created ON customers(created_at);
```

**建表語句**:
```sql
CREATE TABLE customers (
    id CHAR(36) PRIMARY KEY,
    owner_user_id CHAR(36) NOT NULL,
    name VARCHAR(100) NOT NULL,
    phone VARCHAR(20) NOT NULL UNIQUE,
    email VARCHAR(255),
    birthday DATE,
    member_number VARCHAR(20) NOT NULL UNIQUE,
    member_level VARCHAR(20) NOT NULL DEFAULT 'regular',
    total_spent DECIMAL(12,2) NOT NULL DEFAULT 0,
    total_visits INT NOT NULL DEFAULT 0,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_visit_at DATETIME,
    INDEX idx_customers_member_level (member_level),
    INDEX idx_customers_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

#### customer_tags (客戶標籤)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 標籤ID |
| customer_id | CHAR(36) | FK, NOT NULL | 客戶ID |
| tag_name | VARCHAR(50) | NOT NULL | 標籤名稱(VIP/常客/潛在客戶) |
| created_at | DATETIME | NOT NULL DEFAULT CURRENT_TIMESTAMP | 建立時間 |

**索引**:
```sql
CREATE INDEX idx_customer_tags_customer ON customer_tags(customer_id);
CREATE INDEX idx_customer_tags_name ON customer_tags(tag_name);
```

---

#### loyalty_accounts (會員點數帳戶)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| customer_id | CHAR(36) | PK, FK | 客戶ID |
| available_points | INT | NOT NULL DEFAULT 0 | 可用點數 |
| total_earned | INT | NOT NULL DEFAULT 0 | 累計獲得點數 |
| total_redeemed | INT | NOT NULL DEFAULT 0 | 累計兌換點數 |
| updated_at | DATETIME | NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 更新時間 |

**約束**:
```sql
ALTER TABLE loyalty_accounts ADD CONSTRAINT chk_available_points CHECK (available_points >= 0);
ALTER TABLE loyalty_accounts ADD CONSTRAINT chk_points_balance CHECK (available_points = total_earned - total_redeemed);
```

---

#### point_transactions (點數異動記錄)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 異動ID |
| customer_id | CHAR(36) | FK, NOT NULL | 客戶ID |
| type | VARCHAR(20) | NOT NULL | 類型(earn/redeem/expire/adjust) |
| amount | INT | NOT NULL | 點數數量(正數=累積,負數=兌換) |
| order_id | CHAR(36) | FK, NULL | 關聯訂單 |
| description | TEXT | NOT NULL | 說明 |
| created_at | DATETIME | NOT NULL DEFAULT CURRENT_TIMESTAMP | 建立時間 |

**索引**:
```sql
CREATE INDEX idx_point_transactions_customer ON point_transactions(customer_id);
CREATE INDEX idx_point_transactions_created ON point_transactions(created_at);
CREATE INDEX idx_point_transactions_type ON point_transactions(type);
```

---

### 3.3 Inventory 模組

#### stores (門店)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 門店ID |
| code | VARCHAR(20) | UK, NOT NULL | 門店代碼(S001) |
| name | VARCHAR(100) | NOT NULL | 門店名稱 |
| address | VARCHAR(255) | NULL | 地址 |
| phone | VARCHAR(20) | NULL | 電話 |
| opening_hours | VARCHAR(100) | NULL | 營業時間 |
| tax_rate | DECIMAL(5,4) | NOT NULL DEFAULT 0.0500 | 稅率(0.05=5%) |
| status | VARCHAR(20) | NOT NULL DEFAULT 'active' | 狀態(active/inactive) |
| created_at | DATETIME | NOT NULL DEFAULT CURRENT_TIMESTAMP | 建立時間 |

**索引**:
```sql
CREATE UNIQUE INDEX idx_stores_code ON stores(code);
CREATE INDEX idx_stores_status ON stores(status);
```

---

#### product_categories (商品分類)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 分類ID |
| name | VARCHAR(100) | NOT NULL | 分類名稱 |
| parent_id | CHAR(36) | FK, NULL | 父分類ID |
| sort_order | INT | NOT NULL DEFAULT 0 | 排序 |
| icon | VARCHAR(50) | NULL | 圖示 |
| created_at | DATETIME | NOT NULL DEFAULT CURRENT_TIMESTAMP | 建立時間 |

**索引**:
```sql
CREATE INDEX idx_categories_parent ON product_categories(parent_id);
CREATE INDEX idx_categories_sort ON product_categories(sort_order);
```

---

#### products (商品)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 商品ID |
| sku | VARCHAR(50) | UK, NOT NULL | 商品編碼 |
| name | VARCHAR(200) | NOT NULL | 商品名稱 |
| category_id | CHAR(36) | FK, NOT NULL | 商品分類 |
| description | TEXT | NULL | 商品描述 |
| price_cents | INT | NOT NULL | 售價(分) |
| cost_cents | INT | NOT NULL | 成本(分) |
| image_url | VARCHAR(255) | NULL | 商品圖片URL |
| status | VARCHAR(20) | NOT NULL DEFAULT 'active' | 狀態(active/inactive) |
| created_at | DATETIME | NOT NULL DEFAULT CURRENT_TIMESTAMP | 建立時間 |

**約束**:
```sql
ALTER TABLE products ADD CONSTRAINT chk_price CHECK (price_cents >= 0);
ALTER TABLE products ADD CONSTRAINT chk_cost CHECK (cost_cents >= 0);
```

**索引**:
```sql
CREATE UNIQUE INDEX idx_products_sku ON products(sku);
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_status ON products(status);
```

---

#### product_variants (商品規格)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 規格ID |
| product_id | CHAR(36) | FK, NOT NULL | 商品ID |
| type | VARCHAR(50) | NOT NULL | 規格類型(size/temperature/sweetness) |
| name | VARCHAR(100) | NOT NULL | 規格名稱(大杯/熱/正常糖) |
| price_adjustment | INT | NOT NULL DEFAULT 0 | 價格調整(分) |
| sort_order | INT | NOT NULL DEFAULT 0 | 排序 |

**索引**:
```sql
CREATE INDEX idx_variants_product ON product_variants(product_id);
```

---

#### stock (庫存)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| store_id | CHAR(36) | PK, FK | 門店ID |
| product_id | CHAR(36) | PK, FK | 商品ID |
| quantity | INT | NOT NULL DEFAULT 0 | 庫存數量 |
| safety_stock | INT | NOT NULL DEFAULT 0 | 安全庫存 |
| max_stock | INT | NOT NULL DEFAULT 0 | 最大庫存 |
| updated_at | DATETIME | NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP | 更新時間 |

**約束**:
```sql
ALTER TABLE stock ADD CONSTRAINT chk_quantity CHECK (quantity >= 0);
ALTER TABLE stock ADD CONSTRAINT chk_safety_stock CHECK (safety_stock >= 0);
```

**索引**:
```sql
CREATE INDEX idx_stock_low ON stock(store_id, product_id, quantity) WHERE quantity < safety_stock;
CREATE INDEX idx_stock_product ON stock(product_id);
```

---

#### stock_movements (庫存異動)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 異動ID |
| store_id | CHAR(36) | FK, NOT NULL | 門店ID |
| product_id | CHAR(36) | FK, NOT NULL | 商品ID |
| delta | INT | NOT NULL | 異動數量(正數=入庫,負數=出庫) |
| reason | VARCHAR(50) | NOT NULL | 原因(purchase/sale/transfer/adjustment/loss) |
| reference_id | CHAR(36) | NULL | 關聯單據ID(訂單/採購單) |
| created_by | CHAR(36) | FK, NOT NULL | 操作人 |
| created_at | DATETIME | NOT NULL DEFAULT CURRENT_TIMESTAMP | 建立時間 |

**索引**:
```sql
CREATE INDEX idx_movements_store_product ON stock_movements(store_id, product_id);
CREATE INDEX idx_movements_created ON stock_movements(created_at);
CREATE INDEX idx_movements_reason ON stock_movements(reason);
```

---

#### suppliers (供應商)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 供應商ID |
| code | VARCHAR(20) | UK, NOT NULL | 供應商代碼 |
| name | VARCHAR(200) | NOT NULL | 供應商名稱 |
| contact_person | VARCHAR(100) | NULL | 聯絡人 |
| phone | VARCHAR(20) | NULL | 電話 |
| email | VARCHAR(255) | NULL | Email |
| address | VARCHAR(255) | NULL | 地址 |
| payment_terms | VARCHAR(100) | NULL | 付款條件 |
| rating | TINYINT | NULL | 評級(1-5) |
| status | VARCHAR(20) | NOT NULL DEFAULT 'active' | 狀態(active/inactive) |
| created_at | DATETIME | NOT NULL DEFAULT CURRENT_TIMESTAMP | 建立時間 |

**約束**:
```sql
ALTER TABLE suppliers ADD CONSTRAINT chk_rating CHECK (rating BETWEEN 1 AND 5);
```

**索引**:
```sql
CREATE UNIQUE INDEX idx_suppliers_code ON suppliers(code);
CREATE INDEX idx_suppliers_status ON suppliers(status);
```

---

#### purchase_orders (採購單)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 採購單ID |
| po_number | VARCHAR(30) | UK, NOT NULL | 採購單號(PO-YYYYMMDD-XXXX) |
| supplier_id | CHAR(36) | FK, NOT NULL | 供應商ID |
| store_id | CHAR(36) | FK, NOT NULL | 目標門店ID |
| status | VARCHAR(20) | NOT NULL DEFAULT 'draft' | 狀態(draft/submitted/approved/rejected/receiving/completed/cancelled) |
| total_cents | INT | NOT NULL DEFAULT 0 | 總金額(分) |
| expected_delivery_date | DATE | NOT NULL | 預計交貨日期 |
| created_by | CHAR(36) | FK, NOT NULL | 建立人 |
| approved_by | CHAR(36) | FK, NULL | 審核人 |
| approved_at | DATETIME | NULL | 審核時間 |
| notes | TEXT | NULL | 備註 |
| created_at | DATETIME | NOT NULL DEFAULT CURRENT_TIMESTAMP | 建立時間 |

**索引**:
```sql
CREATE UNIQUE INDEX idx_po_number ON purchase_orders(po_number);
CREATE INDEX idx_po_status ON purchase_orders(status);
CREATE INDEX idx_po_supplier ON purchase_orders(supplier_id);
CREATE INDEX idx_po_store ON purchase_orders(store_id);
CREATE INDEX idx_po_created ON purchase_orders(created_at);
```

---

#### purchase_order_items (採購項目)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 項目ID |
| po_id | CHAR(36) | FK, NOT NULL | 採購單ID |
| product_id | CHAR(36) | FK, NOT NULL | 商品ID |
| quantity | INT | NOT NULL | 採購數量 |
| received_quantity | INT | NOT NULL DEFAULT 0 | 已入庫數量 |
| unit_cost_cents | INT | NOT NULL | 單位成本(分) |
| subtotal_cents | INT | NOT NULL | 小計(分) |

**約束**:
```sql
ALTER TABLE purchase_order_items ADD CONSTRAINT chk_po_quantity CHECK (quantity > 0);
ALTER TABLE purchase_order_items ADD CONSTRAINT chk_po_received CHECK (received_quantity >= 0 AND received_quantity <= quantity);
```

**索引**:
```sql
CREATE INDEX idx_po_items_po ON purchase_order_items(po_id);
CREATE INDEX idx_po_items_product ON purchase_order_items(product_id);
```

---

### 3.4 Orders 模組

#### orders (訂單)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 訂單ID |
| order_number | VARCHAR(30) | UK, NOT NULL | 訂單編號(ORD-YYYYMMDD-XXXX) |
| store_id | CHAR(36) | FK, NOT NULL | 門店ID |
| user_id | CHAR(36) | FK, NOT NULL | 操作店員ID |
| customer_id | CHAR(36) | FK, NULL | 客戶ID(非會員為NULL) |
| status | VARCHAR(20) | NOT NULL DEFAULT 'draft' | 狀態(draft/completed/cancelled) |
| subtotal_cents | INT | NOT NULL DEFAULT 0 | 小計(分) |
| tax_cents | INT | NOT NULL DEFAULT 0 | 稅額(分) |
| discount_cents | INT | NOT NULL DEFAULT 0 | 折扣(分) |
| total_cents | INT | NOT NULL DEFAULT 0 | 總金額(分) |
| payment_method | VARCHAR(50) | NOT NULL | 付款方式(cash/credit_card/mobile_pay) |
| points_earned | INT | NOT NULL DEFAULT 0 | 本單累積點數 |
| points_redeemed | INT | NOT NULL DEFAULT 0 | 本單兌換點數 |
| created_at | DATETIME | NOT NULL DEFAULT CURRENT_TIMESTAMP | 建立時間 |
| completed_at | DATETIME | NULL | 完成時間 |

**索引**:
```sql
CREATE UNIQUE INDEX idx_orders_number ON orders(order_number);
CREATE INDEX idx_orders_store ON orders(store_id);
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created ON orders(created_at);
```

---

#### order_items (訂單項目)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 項目ID |
| order_id | CHAR(36) | FK, NOT NULL | 訂單ID |
| product_id | CHAR(36) | FK, NOT NULL | 商品ID |
| product_name | VARCHAR(200) | NOT NULL | 商品名稱快照 |
| quantity | INT | NOT NULL | 數量 |
| unit_price_cents | INT | NOT NULL | 單價(分) |
| subtotal_cents | INT | NOT NULL | 小計(分) |
| variants | JSON | NULL | 選擇的規格(JSON) |

**約束**:
```sql
ALTER TABLE order_items ADD CONSTRAINT chk_order_quantity CHECK (quantity > 0);
```

**索引**:
```sql
CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_order_items_product ON order_items(product_id);
```

---

#### payments (付款記錄)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 付款ID |
| order_id | CHAR(36) | FK, NOT NULL | 訂單ID |
| method | VARCHAR(50) | NOT NULL | 付款方式(cash/credit_card/mobile_pay) |
| amount_cents | INT | NOT NULL | 付款金額(分) |
| received_cents | INT | NULL | 收款金額(現金用) |
| change_cents | INT | NULL | 找零金額(現金用) |
| transaction_id | VARCHAR(100) | NULL | 交易編號(第三方支付) |
| status | VARCHAR(20) | NOT NULL DEFAULT 'pending' | 狀態(pending/success/failed) |
| paid_at | DATETIME | NOT NULL DEFAULT CURRENT_TIMESTAMP | 付款時間 |

**索引**:
```sql
CREATE INDEX idx_payments_order ON payments(order_id);
CREATE INDEX idx_payments_status ON payments(status);
```

---

#### invoices (發票)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 發票ID |
| order_id | CHAR(36) | FK, NOT NULL | 訂單ID |
| invoice_number | VARCHAR(50) | UK, NOT NULL | 發票號碼 |
| tax_id | VARCHAR(20) | NULL | 統一編號 |
| issued_at | DATETIME | NOT NULL DEFAULT CURRENT_TIMESTAMP | 開立時間 |
| printed | TINYINT(1) | NOT NULL DEFAULT 0 | 是否已列印 |

**索引**:
```sql
CREATE UNIQUE INDEX idx_invoices_number ON invoices(invoice_number);
CREATE INDEX idx_invoices_order ON invoices(order_id);
```

---

### 3.5 Notifications 模組

#### notifications (通知)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 通知ID |
| user_id | CHAR(36) | FK, NOT NULL | 接收使用者ID |
| type | VARCHAR(50) | NOT NULL | 類型(order/stock/approval/system) |
| title | VARCHAR(200) | NOT NULL | 標題 |
| message | TEXT | NOT NULL | 訊息內容 |
| is_read | TINYINT(1) | NOT NULL DEFAULT 0 | 是否已讀 |
| created_at | DATETIME | NOT NULL DEFAULT CURRENT_TIMESTAMP | 建立時間 |
| read_at | DATETIME | NULL | 已讀時間 |

**索引**:
```sql
CREATE INDEX idx_notifications_user ON notifications(user_id);
CREATE INDEX idx_notifications_unread ON notifications(user_id, is_read) WHERE is_read = 0;
CREATE INDEX idx_notifications_type ON notifications(type);
CREATE INDEX idx_notifications_created ON notifications(created_at);
```

---

#### notification_templates (通知範本)

| 欄位名 | 資料型態 | 約束 | 說明 |
|-------|---------|------|------|
| id | CHAR(36) | PK | 範本ID |
| template_key | VARCHAR(50) | UK, NOT NULL | 範本鍵值 |
| subject | VARCHAR(200) | NOT NULL | 主旨 |
| body | TEXT | NOT NULL | 內容(支援變數) |
| type | VARCHAR(20) | NOT NULL | 類型(email/notification) |
| created_at | DATETIME | NOT NULL DEFAULT CURRENT_TIMESTAMP | 建立時間 |

**索引**:
```sql
CREATE UNIQUE INDEX idx_templates_key ON notification_templates(template_key);
CREATE INDEX idx_templates_type ON notification_templates(type);
```

---

## 4. 外鍵關係

```sql
-- users
ALTER TABLE users ADD CONSTRAINT fk_users_store 
    FOREIGN KEY (store_id) REFERENCES stores(id) ON DELETE SET NULL;

-- customers
ALTER TABLE customers ADD CONSTRAINT fk_customers_owner 
    FOREIGN KEY (owner_user_id) REFERENCES users(id);

ALTER TABLE customer_tags ADD CONSTRAINT fk_tags_customer 
    FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE CASCADE;

ALTER TABLE loyalty_accounts ADD CONSTRAINT fk_loyalty_customer 
    FOREIGN KEY (customer_id) REFERENCES customers(id) ON DELETE CASCADE;

ALTER TABLE point_transactions ADD CONSTRAINT fk_points_customer 
    FOREIGN KEY (customer_id) REFERENCES customers(id);
ALTER TABLE point_transactions ADD CONSTRAINT fk_points_order 
    FOREIGN KEY (order_id) REFERENCES orders(id);

-- products & inventory
ALTER TABLE product_categories ADD CONSTRAINT fk_category_parent 
    FOREIGN KEY (parent_id) REFERENCES product_categories(id);

ALTER TABLE products ADD CONSTRAINT fk_products_category 
    FOREIGN KEY (category_id) REFERENCES product_categories(id);

ALTER TABLE product_variants ADD CONSTRAINT fk_variants_product 
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE;

ALTER TABLE stock ADD CONSTRAINT fk_stock_store 
    FOREIGN KEY (store_id) REFERENCES stores(id) ON DELETE CASCADE;
ALTER TABLE stock ADD CONSTRAINT fk_stock_product 
    FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE;

ALTER TABLE stock_movements ADD CONSTRAINT fk_movements_store 
    FOREIGN KEY (store_id) REFERENCES stores(id);
ALTER TABLE stock_movements ADD CONSTRAINT fk_movements_product 
    FOREIGN KEY (product_id) REFERENCES products(id);
ALTER TABLE stock_movements ADD CONSTRAINT fk_movements_user 
    FOREIGN KEY (created_by) REFERENCES users(id);

-- purchase orders
ALTER TABLE purchase_orders ADD CONSTRAINT fk_po_supplier 
    FOREIGN KEY (supplier_id) REFERENCES suppliers(id);
ALTER TABLE purchase_orders ADD CONSTRAINT fk_po_store 
    FOREIGN KEY (store_id) REFERENCES stores(id);
ALTER TABLE purchase_orders ADD CONSTRAINT fk_po_creator 
    FOREIGN KEY (created_by) REFERENCES users(id);
ALTER TABLE purchase_orders ADD CONSTRAINT fk_po_approver 
    FOREIGN KEY (approved_by) REFERENCES users(id);

ALTER TABLE purchase_order_items ADD CONSTRAINT fk_po_items_po 
    FOREIGN KEY (po_id) REFERENCES purchase_orders(id) ON DELETE CASCADE;
ALTER TABLE purchase_order_items ADD CONSTRAINT fk_po_items_product 
    FOREIGN KEY (product_id) REFERENCES products(id);

-- orders
ALTER TABLE orders ADD CONSTRAINT fk_orders_store 
    FOREIGN KEY (store_id) REFERENCES stores(id);
ALTER TABLE orders ADD CONSTRAINT fk_orders_user 
    FOREIGN KEY (user_id) REFERENCES users(id);
ALTER TABLE orders ADD CONSTRAINT fk_orders_customer 
    FOREIGN KEY (customer_id) REFERENCES customers(id);

ALTER TABLE order_items ADD CONSTRAINT fk_items_order 
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE;
ALTER TABLE order_items ADD CONSTRAINT fk_items_product 
    FOREIGN KEY (product_id) REFERENCES products(id);

ALTER TABLE payments ADD CONSTRAINT fk_payments_order 
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE;

ALTER TABLE invoices ADD CONSTRAINT fk_invoices_order 
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE;

-- notifications
ALTER TABLE notifications ADD CONSTRAINT fk_notifications_user 
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE;
```

---

## 5. 資料庫腳本

完整的建表腳本請參考: `/coffee_crm/db/init.sql`

---

## 6. 資料庫規範

### 6.1 命名規範
- 資料表: 小寫+底線分隔 (snake_case)
- 欄位: 小寫+底線分隔 (snake_case)
- 索引: `idx_{table}_{column(s)}`
- 外鍵: `fk_{table}_{column}`

### 6.2 資料型態規範
- 主鍵: CHAR(36) - UUID 格式
- 金額: INT (以分為單位,避免浮點數誤差)
- 時間: DATETIME (MySQL 8.0+ 支援到微秒)
- 狀態: VARCHAR(20)
- 布林: TINYINT(1) - 0=false, 1=true
- JSON: JSON (MySQL 5.7.8+)

### 6.3 字元集與排序
- 字元集: utf8mb4 (支援 emoji 和完整 Unicode)
- 排序規則: utf8mb4_unicode_ci (不區分大小寫)

### 6.4 儲存引擎
- 使用 InnoDB (支援外鍵、交易、行級鎖定)

### 6.5 必要欄位
每個資料表應包含:
- `id` (CHAR(36), PK) - UUID
- `created_at` (DATETIME, NOT NULL, DEFAULT CURRENT_TIMESTAMP)
- `updated_at` (DATETIME, 依需求) - 使用 ON UPDATE CURRENT_TIMESTAMP

### 6.6 UUID 生成
MySQL 8.0+ 可使用:
```sql
-- 方式 1: MySQL 內建函數
UUID()

-- 方式 2: 有序 UUID (更適合索引)
UUID_TO_BIN(UUID(), 1)

-- 方式 3: 應用層生成 (推薦)
-- Go: github.com/google/uuid
```

### 6.7 索引設計原則
- 主鍵自動建立聚簇索引
- 外鍵欄位建立索引
- 經常用於 WHERE、JOIN、ORDER BY 的欄位
- 複合索引遵循最左前綴原則
- 避免過多索引影響寫入效能

### 6.8 效能優化建議
- 使用 `EXPLAIN` 分析查詢計畫
- 避免 SELECT *,明確指定欄位
- 大表分頁使用 LIMIT + OFFSET 或游標
- 定期執行 `OPTIMIZE TABLE` 整理碎片
- 監控慢查詢日誌

---

**MySQL 8.0+ 資料庫 ERD 設計完成**
