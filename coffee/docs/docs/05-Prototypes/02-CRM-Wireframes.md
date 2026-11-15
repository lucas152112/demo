# CRM 前端介面原型 (門店管理系統)

## 1. 系統概述

### 1.1 設計目標
CRM 系統主要供門店員工使用，包含店員和店長兩種角色。設計重點在於：
- **操作效率**: 支援快速點餐和結帳流程
- **簡單易用**: 降低學習成本，減少操作錯誤
- **響應迅速**: 適應門店忙碌環境
- **觸控友好**: 支援觸控式螢幕操作

### 1.2 核心功能模組
- POS 點餐系統
- 客戶管理
- 會員服務
- 訂單管理
- 門店報表
- 系統設定

---

## 2. 整體佈局設計

### 2.1 主要佈局結構
```
┌─────────────────────────────────────┐
│          Header (64px)              │
│  Logo  |  Store Info  |  User Menu  │
├─────────────┬───────────────────────┤
│   Sidebar   │      Main Content     │
│   (240px)   │       Flexible       │
│             │                       │
│  Navigation │    Content Area       │
│   Modules   │                       │
│             │                       │
│             │                       │
└─────────────┴───────────────────────┘
```

### 2.2 頭部導航 (Header)
```html
<header class="header">
  <div class="header-left">
    <img src="logo.svg" alt="Coffee Chain" class="logo">
    <div class="store-info">
      <span class="store-name">台北信義店</span>
      <span class="store-status online">營業中</span>
    </div>
  </div>
  
  <div class="header-right">
    <div class="clock">15:30</div>
    <div class="user-menu">
      <img src="avatar.jpg" alt="User" class="avatar">
      <span class="username">王小美</span>
      <button class="dropdown-toggle">⌄</button>
    </div>
  </div>
</header>
```

### 2.3 側邊欄導航 (Sidebar)
```html
<aside class="sidebar">
  <nav class="nav-menu">
    <ul class="nav-list">
      <li class="nav-item active">
        <a href="/pos" class="nav-link">
          <icon name="shopping-cart" />
          <span>POS 點餐</span>
        </a>
      </li>
      <li class="nav-item">
        <a href="/customers" class="nav-link">
          <icon name="users" />
          <span>客戶管理</span>
        </a>
      </li>
      <li class="nav-item">
        <a href="/orders" class="nav-link">
          <icon name="receipt" />
          <span>訂單管理</span>
        </a>
      </li>
      <li class="nav-item">
        <a href="/reports" class="nav-link">
          <icon name="chart-bar" />
          <span>門店報表</span>
        </a>
      </li>
      <li class="nav-item">
        <a href="/settings" class="nav-link">
          <icon name="cog" />
          <span>系統設定</span>
        </a>
      </li>
    </ul>
  </nav>
</aside>
```

---

## 3. POS 點餐系統

### 3.1 POS 主介面佈局
```
┌─────────────────┬───────────────────────┐
│   商品分類區     │      購物車區         │
│    (40%)        │       (30%)          │
│                 │                       │
├─────────────────┤                       │
│   商品選擇區     │                       │
│    (40%)        │                       │
│                 │                       │
│                 ├───────────────────────┤
│                 │      操作區           │
│                 │       (30%)          │
└─────────────────┴───────────────────────┘
```

### 3.2 商品分類區
```html
<div class="category-section">
  <div class="category-tabs">
    <button class="category-tab active" data-category="coffee">
      咖啡 ☕
    </button>
    <button class="category-tab" data-category="tea">
      茶飲 🍵
    </button>
    <button class="category-tab" data-category="food">
      輕食 🥪
    </button>
    <button class="category-tab" data-category="dessert">
      甜點 🍰
    </button>
  </div>
</div>
```

### 3.3 商品選擇區
```html
<div class="products-grid">
  <div class="product-card" data-product="PROD-001">
    <img src="americano.jpg" alt="美式咖啡" class="product-image">
    <h3 class="product-name">美式咖啡</h3>
    <div class="product-prices">
      <span class="price-small">小杯 $30</span>
      <span class="price-medium">中杯 $35</span>
      <span class="price-large">大杯 $40</span>
    </div>
    <div class="product-stock">
      <span class="stock-status available">供應中</span>
    </div>
  </div>
  
  <div class="product-card out-of-stock" data-product="PROD-002">
    <img src="latte.jpg" alt="拿鐵咖啡" class="product-image">
    <h3 class="product-name">拿鐵咖啡</h3>
    <div class="product-prices">
      <span class="price-small">小杯 $45</span>
      <span class="price-medium">中杯 $50</span>
      <span class="price-large">大杯 $55</span>
    </div>
    <div class="product-stock">
      <span class="stock-status out-of-stock">缺貨</span>
    </div>
  </div>
</div>
```

### 3.4 商品選項對話框
```html
<div class="product-modal" id="productModal">
  <div class="modal-content">
    <div class="modal-header">
      <h2>美式咖啡</h2>
      <button class="close-btn">×</button>
    </div>
    
    <div class="modal-body">
      <!-- 尺寸選擇 -->
      <div class="option-group">
        <label class="option-label">尺寸</label>
        <div class="option-buttons">
          <button class="option-btn active" data-size="small">
            小杯 (+$0)
          </button>
          <button class="option-btn" data-size="medium">
            中杯 (+$5)
          </button>
          <button class="option-btn" data-size="large">
            大杯 (+$10)
          </button>
        </div>
      </div>
      
      <!-- 溫度選擇 -->
      <div class="option-group">
        <label class="option-label">溫度</label>
        <div class="option-buttons">
          <button class="option-btn active" data-temp="hot">熱</button>
          <button class="option-btn" data-temp="ice">冰</button>
        </div>
      </div>
      
      <!-- 數量選擇 -->
      <div class="quantity-group">
        <label class="option-label">數量</label>
        <div class="quantity-controls">
          <button class="quantity-btn decrease">-</button>
          <input type="number" class="quantity-input" value="1" min="1">
          <button class="quantity-btn increase">+</button>
        </div>
      </div>
    </div>
    
    <div class="modal-footer">
      <div class="total-price">小計：$30</div>
      <button class="btn btn-primary add-to-cart">
        加入購物車
      </button>
    </div>
  </div>
</div>
```

### 3.5 購物車區
```html
<div class="cart-section">
  <div class="cart-header">
    <h3>購物車</h3>
    <button class="clear-cart">清空</button>
  </div>
  
  <div class="cart-items">
    <div class="cart-item">
      <div class="item-info">
        <h4>美式咖啡</h4>
        <p>中杯 / 熱</p>
      </div>
      <div class="item-controls">
        <button class="quantity-btn">-</button>
        <span class="quantity">2</span>
        <button class="quantity-btn">+</button>
        <span class="item-price">$70</span>
        <button class="remove-item">×</button>
      </div>
    </div>
    
    <div class="cart-item">
      <div class="item-info">
        <h4>拿鐵咖啡</h4>
        <p>大杯 / 冰</p>
      </div>
      <div class="item-controls">
        <button class="quantity-btn">-</button>
        <span class="quantity">1</span>
        <button class="quantity-btn">+</button>
        <span class="item-price">$55</span>
        <button class="remove-item">×</button>
      </div>
    </div>
  </div>
  
  <div class="cart-summary">
    <div class="summary-line">
      <span>小計</span>
      <span>$125</span>
    </div>
    <div class="summary-line total">
      <span>總計</span>
      <span>$125</span>
    </div>
  </div>
  
  <div class="cart-actions">
    <button class="btn btn-secondary member-check">
      會員查詢
    </button>
    <button class="btn btn-primary checkout">
      結帳
    </button>
  </div>
</div>
```

---

## 4. 客戶管理介面

### 4.1 客戶清單頁面
```html
<div class="customers-page">
  <div class="page-header">
    <h1>客戶管理</h1>
    <button class="btn btn-primary add-customer">
      新增客戶
    </button>
  </div>
  
  <div class="filters-bar">
    <div class="search-box">
      <input type="text" placeholder="搜尋客戶姓名或手機號碼" class="search-input">
      <button class="search-btn">🔍</button>
    </div>
    
    <div class="filter-options">
      <select class="filter-select" name="tier">
        <option value="">全部等級</option>
        <option value="bronze">Bronze</option>
        <option value="silver">Silver</option>
        <option value="gold">Gold</option>
      </select>
      
      <select class="filter-select" name="status">
        <option value="">全部狀態</option>
        <option value="active">活躍</option>
        <option value="inactive">非活躍</option>
      </select>
    </div>
  </div>
  
  <div class="customers-table">
    <table class="table">
      <thead>
        <tr>
          <th>客戶資訊</th>
          <th>會員等級</th>
          <th>累積點數</th>
          <th>最後消費</th>
          <th>操作</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>
            <div class="customer-info">
              <div class="customer-name">王小明</div>
              <div class="customer-phone">0912-345-678</div>
            </div>
          </td>
          <td>
            <span class="badge gold">Gold</span>
          </td>
          <td>1,250 點</td>
          <td>2024-01-15</td>
          <td>
            <button class="btn-icon view-customer">👁️</button>
            <button class="btn-icon edit-customer">✏️</button>
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</div>
```

### 4.2 客戶詳情頁面
```html
<div class="customer-detail-page">
  <div class="page-header">
    <button class="back-btn">← 返回</button>
    <h1>客戶詳情</h1>
    <button class="btn btn-secondary edit-customer">編輯</button>
  </div>
  
  <div class="customer-overview">
    <div class="customer-card">
      <div class="customer-avatar">
        <img src="avatar-placeholder.png" alt="Customer">
      </div>
      <div class="customer-info">
        <h2>王小明</h2>
        <p>手機：0912-345-678</p>
        <p>Email：wang@example.com</p>
        <p>生日：1985-03-15</p>
      </div>
      <div class="customer-stats">
        <div class="stat-item">
          <span class="stat-label">會員等級</span>
          <span class="badge gold">Gold</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">累積點數</span>
          <span class="stat-value">1,250</span>
        </div>
        <div class="stat-item">
          <span class="stat-label">累積消費</span>
          <span class="stat-value">$12,500</span>
        </div>
      </div>
    </div>
  </div>
  
  <div class="tabs-container">
    <div class="tabs">
      <button class="tab active" data-tab="orders">消費記錄</button>
      <button class="tab" data-tab="points">點數記錄</button>
      <button class="tab" data-tab="preferences">偏好設定</button>
    </div>
    
    <div class="tab-content active" data-content="orders">
      <div class="orders-history">
        <div class="order-item">
          <div class="order-header">
            <span class="order-id">#ORD-001</span>
            <span class="order-date">2024-01-15 14:30</span>
            <span class="order-amount">$85</span>
          </div>
          <div class="order-items">
            <p>美式咖啡 x2, 拿鐵咖啡 x1</p>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
```

---

## 5. 會員服務介面

### 5.1 會員查詢對話框
```html
<div class="member-modal" id="memberModal">
  <div class="modal-content">
    <div class="modal-header">
      <h2>會員查詢</h2>
      <button class="close-btn">×</button>
    </div>
    
    <div class="modal-body">
      <div class="member-search">
        <input type="text" placeholder="請輸入手機號碼" class="member-input">
        <button class="btn btn-primary search-member">查詢</button>
      </div>
      
      <div class="member-result" style="display: none;">
        <div class="member-info">
          <h3>王小明</h3>
          <div class="member-tier">
            <span class="badge gold">Gold 會員</span>
          </div>
          <div class="member-points">
            <span class="points-label">可用點數：</span>
            <span class="points-value">1,250 點</span>
          </div>
        </div>
        
        <div class="member-actions">
          <button class="btn btn-secondary use-points">使用點數</button>
          <button class="btn btn-primary apply-member">套用會員</button>
        </div>
      </div>
    </div>
  </div>
</div>
```

### 5.2 點數使用對話框
```html
<div class="points-modal" id="pointsModal">
  <div class="modal-content">
    <div class="modal-header">
      <h2>點數兌換</h2>
      <button class="close-btn">×</button>
    </div>
    
    <div class="modal-body">
      <div class="current-points">
        <span class="label">目前點數：</span>
        <span class="value">1,250 點</span>
      </div>
      
      <div class="points-options">
        <div class="option-item">
          <input type="radio" id="points-100" name="points" value="100">
          <label for="points-100">
            <span class="points">100 點</span>
            <span class="discount">= $1 折扣</span>
          </label>
        </div>
        
        <div class="option-item">
          <input type="radio" id="points-500" name="points" value="500">
          <label for="points-500">
            <span class="points">500 點</span>
            <span class="discount">= $5 折扣</span>
          </label>
        </div>
        
        <div class="option-item">
          <input type="radio" id="points-custom" name="points" value="custom">
          <label for="points-custom">自訂點數：</label>
          <input type="number" class="custom-points" placeholder="輸入點數" min="100" step="100">
        </div>
      </div>
    </div>
    
    <div class="modal-footer">
      <button class="btn btn-secondary cancel">取消</button>
      <button class="btn btn-primary apply-points">確認使用</button>
    </div>
  </div>
</div>
```

---

## 6. 結帳介面

### 6.1 結帳對話框
```html
<div class="checkout-modal" id="checkoutModal">
  <div class="modal-content large">
    <div class="modal-header">
      <h2>結帳</h2>
      <button class="close-btn">×</button>
    </div>
    
    <div class="modal-body">
      <div class="checkout-layout">
        <div class="order-summary">
          <h3>訂單摘要</h3>
          <div class="order-items">
            <div class="item">
              <span>美式咖啡 x2</span>
              <span>$70</span>
            </div>
            <div class="item">
              <span>拿鐵咖啡 x1</span>
              <span>$55</span>
            </div>
          </div>
          
          <div class="order-totals">
            <div class="total-line">
              <span>小計</span>
              <span>$125</span>
            </div>
            <div class="total-line discount" style="display: none;">
              <span>會員折扣</span>
              <span>-$12</span>
            </div>
            <div class="total-line points-discount" style="display: none;">
              <span>點數折抵</span>
              <span>-$5</span>
            </div>
            <div class="total-line final">
              <span>總計</span>
              <span>$125</span>
            </div>
          </div>
        </div>
        
        <div class="payment-section">
          <div class="member-section">
            <h4>會員資訊</h4>
            <div class="member-display">
              <span class="no-member">非會員訂單</span>
              <button class="btn btn-secondary search-member">查詢會員</button>
            </div>
          </div>
          
          <div class="payment-method">
            <h4>付款方式</h4>
            <div class="payment-options">
              <label class="payment-option">
                <input type="radio" name="payment" value="cash" checked>
                <div class="option-content">
                  <span class="option-icon">💵</span>
                  <span class="option-text">現金</span>
                </div>
              </label>
              
              <label class="payment-option">
                <input type="radio" name="payment" value="card">
                <div class="option-content">
                  <span class="option-icon">💳</span>
                  <span class="option-text">信用卡</span>
                </div>
              </label>
              
              <label class="payment-option">
                <input type="radio" name="payment" value="mobile">
                <div class="option-content">
                  <span class="option-icon">📱</span>
                  <span class="option-text">行動支付</span>
                </div>
              </label>
            </div>
          </div>
          
          <div class="cash-section">
            <h4>收款金額</h4>
            <div class="cash-buttons">
              <button class="cash-btn" data-amount="125">$125</button>
              <button class="cash-btn" data-amount="130">$130</button>
              <button class="cash-btn" data-amount="150">$150</button>
              <button class="cash-btn" data-amount="200">$200</button>
            </div>
            <input type="number" class="cash-input" placeholder="輸入收款金額">
            <div class="change-amount">
              <span class="label">找零：</span>
              <span class="value">$0</span>
            </div>
          </div>
        </div>
      </div>
    </div>
    
    <div class="modal-footer">
      <button class="btn btn-secondary cancel">取消</button>
      <button class="btn btn-large btn-primary process-payment">
        確認收款
      </button>
    </div>
  </div>
</div>
```

---

## 7. 訂單管理介面

### 7.1 訂單清單頁面
```html
<div class="orders-page">
  <div class="page-header">
    <h1>訂單管理</h1>
    <div class="date-selector">
      <input type="date" class="date-input" value="2024-01-15">
      <button class="btn btn-secondary today">今日</button>
    </div>
  </div>
  
  <div class="orders-stats">
    <div class="stat-card">
      <div class="stat-number">58</div>
      <div class="stat-label">今日訂單</div>
    </div>
    <div class="stat-card">
      <div class="stat-number">$8,750</div>
      <div class="stat-label">今日營收</div>
    </div>
    <div class="stat-card">
      <div class="stat-number">$151</div>
      <div class="stat-label">平均客單價</div>
    </div>
  </div>
  
  <div class="orders-table">
    <table class="table">
      <thead>
        <tr>
          <th>訂單編號</th>
          <th>時間</th>
          <th>客戶</th>
          <th>商品</th>
          <th>金額</th>
          <th>付款方式</th>
          <th>操作</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>#ORD-001</td>
          <td>14:30</td>
          <td>王小明</td>
          <td>美式咖啡 x2, 拿鐵 x1</td>
          <td>$125</td>
          <td>現金</td>
          <td>
            <button class="btn-icon view-order">👁️</button>
            <button class="btn-icon print-receipt">🖨️</button>
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</div>
```

---

## 8. 門店報表介面

### 8.1 報表總覽頁面
```html
<div class="reports-page">
  <div class="page-header">
    <h1>門店報表</h1>
    <div class="date-range">
      <input type="date" class="date-input" value="2024-01-01">
      <span>至</span>
      <input type="date" class="date-input" value="2024-01-31">
      <button class="btn btn-primary generate-report">產生報表</button>
    </div>
  </div>
  
  <div class="report-dashboard">
    <div class="dashboard-row">
      <div class="metric-card">
        <h3>營收總額</h3>
        <div class="metric-value">$125,000</div>
        <div class="metric-trend up">+15.2%</div>
      </div>
      
      <div class="metric-card">
        <h3>訂單數量</h3>
        <div class="metric-value">850</div>
        <div class="metric-trend up">+8.5%</div>
      </div>
      
      <div class="metric-card">
        <h3>平均客單價</h3>
        <div class="metric-value">$147</div>
        <div class="metric-trend down">-2.1%</div>
      </div>
      
      <div class="metric-card">
        <h3>新會員數</h3>
        <div class="metric-value">45</div>
        <div class="metric-trend up">+22.3%</div>
      </div>
    </div>
    
    <div class="dashboard-row">
      <div class="chart-card">
        <h3>每日營收趨勢</h3>
        <canvas id="revenueChart" width="400" height="200"></canvas>
      </div>
      
      <div class="chart-card">
        <h3>熱銷商品排行</h3>
        <div class="ranking-list">
          <div class="ranking-item">
            <span class="rank">#1</span>
            <span class="product">美式咖啡</span>
            <span class="quantity">120 杯</span>
          </div>
          <div class="ranking-item">
            <span class="rank">#2</span>
            <span class="product">拿鐵咖啡</span>
            <span class="quantity">95 杯</span>
          </div>
          <div class="ranking-item">
            <span class="rank">#3</span>
            <span class="product">卡布奇諾</span>
            <span class="quantity">78 杯</span>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
```

---

## 9. 響應式設計

### 9.1 平板適配 (768px - 1024px)
```css
@media (max-width: 1024px) {
  .sidebar {
    width: 64px;
    transition: width 0.3s ease;
  }
  
  .sidebar:hover {
    width: 240px;
  }
  
  .sidebar .nav-link span {
    opacity: 0;
    transition: opacity 0.3s ease;
  }
  
  .sidebar:hover .nav-link span {
    opacity: 1;
  }
}
```

### 9.2 手機適配 (< 768px)
```css
@media (max-width: 768px) {
  .sidebar {
    transform: translateX(-100%);
    position: fixed;
    top: 64px;
    left: 0;
    height: calc(100vh - 64px);
    z-index: 1000;
  }
  
  .sidebar.open {
    transform: translateX(0);
  }
  
  .header-left::before {
    content: '☰';
    font-size: 20px;
    margin-right: 16px;
    cursor: pointer;
  }
  
  /* POS 系統手機版佈局 */
  .pos-layout {
    flex-direction: column;
  }
  
  .products-grid {
    grid-template-columns: repeat(2, 1fr);
  }
  
  .cart-section {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    background: white;
    border-top: 1px solid var(--gray-200);
    padding: 16px;
  }
}
```

---

## 10. 互動效果

### 10.1 按鈕點擊效果
```css
.btn {
  transition: all 0.2s ease;
  position: relative;
  overflow: hidden;
}

.btn:active {
  transform: translateY(1px);
}

.btn::after {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 0;
  height: 0;
  background: rgba(255, 255, 255, 0.3);
  border-radius: 50%;
  transform: translate(-50%, -50%);
  transition: width 0.6s ease, height 0.6s ease;
}

.btn:active::after {
  width: 300px;
  height: 300px;
}
```

### 10.2 載入動畫
```html
<div class="loading-overlay" style="display: none;">
  <div class="loading-spinner">
    <div class="spinner"></div>
    <p>處理中...</p>
  </div>
</div>
```

```css
.loading-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9999;
}

.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid var(--gray-200);
  border-top: 4px solid var(--primary-color);
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
```

---

## 11. 快速鍵設計

### 11.1 POS 系統快速鍵
- **F1-F8**: 商品分類快速切換
- **Enter**: 確認加入購物車
- **Delete**: 刪除購物車項目
- **F9**: 會員查詢
- **F10**: 結帳
- **Esc**: 取消當前操作

### 11.2 通用快速鍵
- **Ctrl + F**: 搜尋
- **Ctrl + N**: 新增
- **Ctrl + S**: 儲存
- **Alt + ←**: 返回上頁

---

**版本**: v1.0  
**建立日期**: 2025-11-16  
**最後更新**: 2025-11-16  
**負責人**: UI/UX 設計團隊