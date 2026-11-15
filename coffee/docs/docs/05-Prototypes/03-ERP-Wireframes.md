# ERP 後台管理系統原型 (總部管理系統)

## 1. 系統概述

### 1.1 設計目標
ERP 系統供總部管理人員使用，包含系統管理員、區域經理、總經理等角色。設計重點在於：
- **數據導向**: 提供全面的數據分析和洞察
- **決策支援**: 協助管理層制定營運策略
- **效率優化**: 簡化複雜的管理流程
- **權限控制**: 嚴格的角色權限管理

### 1.2 核心功能模組
- 總部報表分析
- 多店管理
- 庫存管理
- 人員管理
- 財務管理
- 系統設定

---

## 2. 整體佈局設計

### 2.1 主要佈局結構
```
┌─────────────────────────────────────┐
│          Header (72px)              │
│  Logo | Navigation | User Menu      │
├─────────────┬───────────────────────┤
│   Sidebar   │      Main Content     │
│   (280px)   │       Flexible       │
│             │                       │
│  Feature    │    Content Area       │
│  Navigation │                       │
│             │                       │
│  Settings   │                       │
│  & Tools    │                       │
└─────────────┴───────────────────────┘
```

### 2.2 頭部導航 (Header)
```html
<header class="erp-header">
  <div class="header-left">
    <img src="logo.svg" alt="Coffee Chain ERP" class="logo">
    <nav class="main-nav">
      <ul class="nav-list">
        <li><a href="/dashboard" class="nav-link active">總覽</a></li>
        <li><a href="/stores" class="nav-link">門店管理</a></li>
        <li><a href="/reports" class="nav-link">營運報表</a></li>
        <li><a href="/finance" class="nav-link">財務管理</a></li>
        <li><a href="/system" class="nav-link">系統管理</a></li>
      </ul>
    </nav>
  </div>
  
  <div class="header-right">
    <div class="header-tools">
      <button class="notification-btn">
        🔔 <span class="badge">3</span>
      </button>
      <button class="search-btn">🔍</button>
    </div>
    <div class="user-menu">
      <img src="admin-avatar.jpg" alt="Admin" class="avatar">
      <div class="user-info">
        <span class="username">張總經理</span>
        <span class="user-role">總經理</span>
      </div>
      <button class="dropdown-toggle">⌄</button>
    </div>
  </div>
</header>
```

### 2.3 側邊欄導航 (Sidebar)
```html
<aside class="erp-sidebar">
  <div class="sidebar-section">
    <h4 class="section-title">核心功能</h4>
    <nav class="sidebar-nav">
      <ul class="nav-list">
        <li class="nav-item">
          <a href="/dashboard" class="nav-link active">
            <icon name="chart-line" />
            <span>營運總覽</span>
          </a>
        </li>
        <li class="nav-item">
          <a href="/stores" class="nav-link">
            <icon name="store" />
            <span>門店管理</span>
          </a>
        </li>
        <li class="nav-item">
          <a href="/inventory" class="nav-link">
            <icon name="boxes" />
            <span>庫存管理</span>
          </a>
        </li>
        <li class="nav-item">
          <a href="/products" class="nav-link">
            <icon name="coffee" />
            <span>商品管理</span>
          </a>
        </li>
        <li class="nav-item">
          <a href="/customers" class="nav-link">
            <icon name="users" />
            <span>客戶分析</span>
          </a>
        </li>
      </ul>
    </nav>
  </div>
  
  <div class="sidebar-section">
    <h4 class="section-title">營運分析</h4>
    <nav class="sidebar-nav">
      <ul class="nav-list">
        <li class="nav-item">
          <a href="/reports/sales" class="nav-link">
            <icon name="chart-bar" />
            <span>銷售報表</span>
          </a>
        </li>
        <li class="nav-item">
          <a href="/reports/finance" class="nav-link">
            <icon name="dollar-sign" />
            <span>財務報表</span>
          </a>
        </li>
        <li class="nav-item">
          <a href="/reports/inventory" class="nav-link">
            <icon name="warehouse" />
            <span>庫存報表</span>
          </a>
        </li>
      </ul>
    </nav>
  </div>
  
  <div class="sidebar-section">
    <h4 class="section-title">系統管理</h4>
    <nav class="sidebar-nav">
      <ul class="nav-list">
        <li class="nav-item">
          <a href="/admin/users" class="nav-link">
            <icon name="user-cog" />
            <span>用戶管理</span>
          </a>
        </li>
        <li class="nav-item">
          <a href="/admin/settings" class="nav-link">
            <icon name="cog" />
            <span>系統設定</span>
          </a>
        </li>
      </ul>
    </nav>
  </div>
</aside>
```

---

## 3. 營運總覽儀表板

### 3.1 儀表板主頁面
```html
<div class="dashboard-page">
  <div class="page-header">
    <h1>營運總覽</h1>
    <div class="header-actions">
      <select class="time-selector">
        <option value="today">今日</option>
        <option value="week">本週</option>
        <option value="month" selected>本月</option>
        <option value="quarter">本季</option>
      </select>
      <button class="btn btn-primary export-report">匯出報表</button>
    </div>
  </div>
  
  <!-- KPI 指標區 -->
  <div class="kpi-section">
    <div class="kpi-grid">
      <div class="kpi-card revenue">
        <div class="kpi-header">
          <h3>總營收</h3>
          <span class="kpi-period">本月</span>
        </div>
        <div class="kpi-value">$2,850,000</div>
        <div class="kpi-change positive">
          <span class="change-icon">📈</span>
          <span class="change-text">+12.5% vs 上月</span>
        </div>
        <div class="kpi-chart">
          <canvas id="revenueSparkline" width="100" height="30"></canvas>
        </div>
      </div>
      
      <div class="kpi-card orders">
        <div class="kpi-header">
          <h3>訂單數量</h3>
          <span class="kpi-period">本月</span>
        </div>
        <div class="kpi-value">18,500</div>
        <div class="kpi-change positive">
          <span class="change-icon">📈</span>
          <span class="change-text">+8.2% vs 上月</span>
        </div>
        <div class="kpi-chart">
          <canvas id="ordersSparkline" width="100" height="30"></canvas>
        </div>
      </div>
      
      <div class="kpi-card customers">
        <div class="kpi-header">
          <h3>活躍會員</h3>
          <span class="kpi-period">本月</span>
        </div>
        <div class="kpi-value">12,300</div>
        <div class="kpi-change positive">
          <span class="change-icon">📈</span>
          <span class="change-text">+15.8% vs 上月</span>
        </div>
        <div class="kpi-chart">
          <canvas id="customersSparkline" width="100" height="30"></canvas>
        </div>
      </div>
      
      <div class="kpi-card aov">
        <div class="kpi-header">
          <h3>平均客單價</h3>
          <span class="kpi-period">本月</span>
        </div>
        <div class="kpi-value">$154</div>
        <div class="kpi-change negative">
          <span class="change-icon">📉</span>
          <span class="change-text">-2.1% vs 上月</span>
        </div>
        <div class="kpi-chart">
          <canvas id="aovSparkline" width="100" height="30"></canvas>
        </div>
      </div>
    </div>
  </div>
  
  <!-- 圖表區 -->
  <div class="charts-section">
    <div class="charts-grid">
      <div class="chart-card large">
        <div class="chart-header">
          <h3>營收趨勢分析</h3>
          <div class="chart-controls">
            <button class="chart-btn active" data-period="7">7天</button>
            <button class="chart-btn" data-period="30">30天</button>
            <button class="chart-btn" data-period="90">90天</button>
          </div>
        </div>
        <div class="chart-body">
          <canvas id="revenueChart" width="600" height="300"></canvas>
        </div>
      </div>
      
      <div class="chart-card">
        <div class="chart-header">
          <h3>門店業績排行</h3>
        </div>
        <div class="chart-body">
          <div class="ranking-list">
            <div class="ranking-item">
              <div class="rank-number">1</div>
              <div class="store-info">
                <span class="store-name">台北信義店</span>
                <span class="store-location">台北市信義區</span>
              </div>
              <div class="store-revenue">$185,000</div>
              <div class="store-growth positive">+18%</div>
            </div>
            
            <div class="ranking-item">
              <div class="rank-number">2</div>
              <div class="store-info">
                <span class="store-name">台中西屯店</span>
                <span class="store-location">台中市西屯區</span>
              </div>
              <div class="store-revenue">$168,000</div>
              <div class="store-growth positive">+12%</div>
            </div>
            
            <div class="ranking-item">
              <div class="rank-number">3</div>
              <div class="store-info">
                <span class="store-name">高雄前鎮店</span>
                <span class="store-location">高雄市前鎮區</span>
              </div>
              <div class="store-revenue">$152,000</div>
              <div class="store-growth negative">-3%</div>
            </div>
          </div>
        </div>
      </div>
    </div>
    
    <div class="charts-grid">
      <div class="chart-card">
        <div class="chart-header">
          <h3>商品銷售分佈</h3>
        </div>
        <div class="chart-body">
          <canvas id="productChart" width="300" height="300"></canvas>
        </div>
      </div>
      
      <div class="chart-card">
        <div class="chart-header">
          <h3>會員等級分佈</h3>
        </div>
        <div class="chart-body">
          <canvas id="memberChart" width="300" height="200"></canvas>
        </div>
      </div>
    </div>
  </div>
  
  <!-- 快速操作區 -->
  <div class="quick-actions-section">
    <h3>快速操作</h3>
    <div class="actions-grid">
      <button class="action-card">
        <icon name="plus-circle" />
        <span>新增門店</span>
      </button>
      <button class="action-card">
        <icon name="package" />
        <span>庫存進貨</span>
      </button>
      <button class="action-card">
        <icon name="user-plus" />
        <span>員工入職</span>
      </button>
      <button class="action-card">
        <icon name="bell" />
        <span>發送通知</span>
      </button>
    </div>
  </div>
</div>
```

---

## 4. 門店管理介面

### 4.1 門店清單頁面
```html
<div class="stores-page">
  <div class="page-header">
    <h1>門店管理</h1>
    <div class="header-actions">
      <button class="btn btn-secondary import-stores">匯入門店</button>
      <button class="btn btn-primary add-store">新增門店</button>
    </div>
  </div>
  
  <div class="stores-filters">
    <div class="filter-row">
      <div class="filter-group">
        <label>地區</label>
        <select class="filter-select">
          <option value="">全部地區</option>
          <option value="north">北部</option>
          <option value="central">中部</option>
          <option value="south">南部</option>
        </select>
      </div>
      
      <div class="filter-group">
        <label>營業狀態</label>
        <select class="filter-select">
          <option value="">全部狀態</option>
          <option value="active">營業中</option>
          <option value="maintenance">維護中</option>
          <option value="closed">暫停營業</option>
        </select>
      </div>
      
      <div class="filter-group">
        <label>業績表現</label>
        <select class="filter-select">
          <option value="">全部</option>
          <option value="high">高於平均</option>
          <option value="average">平均水準</option>
          <option value="low">低於平均</option>
        </select>
      </div>
      
      <div class="search-group">
        <input type="text" placeholder="搜尋門店名稱..." class="search-input">
        <button class="btn btn-secondary search">搜尋</button>
      </div>
    </div>
  </div>
  
  <div class="stores-grid">
    <div class="store-card high-performance">
      <div class="store-header">
        <h3>台北信義店</h3>
        <span class="store-status active">營業中</span>
      </div>
      
      <div class="store-info">
        <div class="info-row">
          <span class="label">地址：</span>
          <span class="value">台北市信義區信義路五段150號</span>
        </div>
        <div class="info-row">
          <span class="label">電話：</span>
          <span class="value">02-2345-6789</span>
        </div>
        <div class="info-row">
          <span class="label">店長：</span>
          <span class="value">林小明</span>
        </div>
        <div class="info-row">
          <span class="label">開店時間：</span>
          <span class="value">07:00 - 22:00</span>
        </div>
      </div>
      
      <div class="store-metrics">
        <div class="metric">
          <span class="metric-label">本月營收</span>
          <span class="metric-value">$185,000</span>
          <span class="metric-trend positive">+18%</span>
        </div>
        <div class="metric">
          <span class="metric-label">本月訂單</span>
          <span class="metric-value">1,250</span>
          <span class="metric-trend positive">+12%</span>
        </div>
        <div class="metric">
          <span class="metric-label">員工數</span>
          <span class="metric-value">8人</span>
        </div>
      </div>
      
      <div class="store-actions">
        <button class="btn btn-sm btn-secondary view-details">查看詳情</button>
        <button class="btn btn-sm btn-secondary edit-store">編輯</button>
        <button class="btn btn-sm btn-secondary view-reports">報表</button>
      </div>
    </div>
    
    <!-- 更多門店卡片... -->
  </div>
  
  <div class="pagination">
    <button class="page-btn prev">上一頁</button>
    <span class="page-numbers">
      <button class="page-btn active">1</button>
      <button class="page-btn">2</button>
      <button class="page-btn">3</button>
    </span>
    <button class="page-btn next">下一頁</button>
  </div>
</div>
```

### 4.2 門店詳情頁面
```html
<div class="store-detail-page">
  <div class="page-header">
    <button class="back-btn">← 返回門店列表</button>
    <h1>台北信義店</h1>
    <div class="header-actions">
      <button class="btn btn-secondary edit-store">編輯門店</button>
      <button class="btn btn-secondary store-settings">設定</button>
    </div>
  </div>
  
  <div class="store-overview">
    <div class="overview-grid">
      <div class="info-card">
        <h3>基本資訊</h3>
        <div class="info-content">
          <div class="info-item">
            <span class="label">門店編號：</span>
            <span class="value">STR-001</span>
          </div>
          <div class="info-item">
            <span class="label">開業日期：</span>
            <span class="value">2020-05-15</span>
          </div>
          <div class="info-item">
            <span class="label">店面面積：</span>
            <span class="value">180 平方米</span>
          </div>
          <div class="info-item">
            <span class="label">座位數：</span>
            <span class="value">45 個</span>
          </div>
        </div>
      </div>
      
      <div class="performance-card">
        <h3>業績表現</h3>
        <div class="performance-metrics">
          <div class="metric-row">
            <span class="label">本月營收：</span>
            <span class="value highlight">$185,000</span>
            <span class="trend positive">+18%</span>
          </div>
          <div class="metric-row">
            <span class="label">本月訂單：</span>
            <span class="value">1,250 筆</span>
            <span class="trend positive">+12%</span>
          </div>
          <div class="metric-row">
            <span class="label">平均客單價：</span>
            <span class="value">$148</span>
            <span class="trend neutral">-2%</span>
          </div>
        </div>
      </div>
      
      <div class="staff-card">
        <h3>員工配置</h3>
        <div class="staff-list">
          <div class="staff-item manager">
            <img src="manager-avatar.jpg" alt="Manager" class="staff-avatar">
            <div class="staff-info">
              <span class="staff-name">林小明</span>
              <span class="staff-role">店長</span>
            </div>
            <span class="staff-status online">在線</span>
          </div>
          
          <div class="staff-item">
            <img src="staff-avatar.jpg" alt="Staff" class="staff-avatar">
            <div class="staff-info">
              <span class="staff-name">張小華</span>
              <span class="staff-role">店員</span>
            </div>
            <span class="staff-status online">在線</span>
          </div>
          
          <div class="staff-summary">
            <span>總計 8 名員工</span>
            <button class="btn btn-sm btn-secondary manage-staff">管理員工</button>
          </div>
        </div>
      </div>
    </div>
  </div>
  
  <div class="tabs-container">
    <div class="tabs">
      <button class="tab active" data-tab="analytics">數據分析</button>
      <button class="tab" data-tab="inventory">庫存狀態</button>
      <button class="tab" data-tab="orders">訂單記錄</button>
      <button class="tab" data-tab="reviews">評價管理</button>
    </div>
    
    <div class="tab-content active" data-content="analytics">
      <div class="analytics-section">
        <div class="chart-row">
          <div class="chart-card">
            <h4>營收趨勢 (近30天)</h4>
            <canvas id="storeRevenueChart" width="500" height="200"></canvas>
          </div>
          
          <div class="chart-card">
            <h4>熱銷商品排行</h4>
            <div class="product-ranking">
              <div class="rank-item">
                <span class="rank">#1</span>
                <span class="product">美式咖啡</span>
                <span class="sales">285 杯</span>
                <span class="revenue">$8,550</span>
              </div>
              <!-- 更多排行... -->
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
```

---

## 5. 庫存管理介面

### 5.1 庫存總覽頁面
```html
<div class="inventory-page">
  <div class="page-header">
    <h1>庫存管理</h1>
    <div class="header-actions">
      <button class="btn btn-secondary import-inventory">匯入庫存</button>
      <button class="btn btn-secondary generate-order">生成進貨單</button>
      <button class="btn btn-primary add-product">新增商品</button>
    </div>
  </div>
  
  <div class="inventory-summary">
    <div class="summary-cards">
      <div class="summary-card">
        <div class="card-icon">📦</div>
        <div class="card-content">
          <h3>庫存商品</h3>
          <div class="card-value">156 種</div>
        </div>
      </div>
      
      <div class="summary-card warning">
        <div class="card-icon">⚠️</div>
        <div class="card-content">
          <h3>庫存不足</h3>
          <div class="card-value">12 種</div>
        </div>
      </div>
      
      <div class="summary-card danger">
        <div class="card-icon">🚨</div>
        <div class="card-content">
          <h3>即將過期</h3>
          <div class="card-value">3 種</div>
        </div>
      </div>
      
      <div class="summary-card">
        <div class="card-icon">💰</div>
        <div class="card-content">
          <h3>庫存總值</h3>
          <div class="card-value">$285,000</div>
        </div>
      </div>
    </div>
  </div>
  
  <div class="inventory-filters">
    <div class="filter-row">
      <div class="filter-group">
        <label>商品分類</label>
        <select class="filter-select">
          <option value="">全部分類</option>
          <option value="coffee">咖啡類</option>
          <option value="tea">茶飲類</option>
          <option value="food">輕食類</option>
          <option value="material">原物料</option>
        </select>
      </div>
      
      <div class="filter-group">
        <label>庫存狀態</label>
        <select class="filter-select">
          <option value="">全部狀態</option>
          <option value="sufficient">充足</option>
          <option value="low">不足</option>
          <option value="out">缺貨</option>
          <option value="expiring">即將過期</option>
        </select>
      </div>
      
      <div class="filter-group">
        <label>門店</label>
        <select class="filter-select">
          <option value="">全部門店</option>
          <option value="STR-001">台北信義店</option>
          <option value="STR-002">台中西屯店</option>
          <option value="STR-003">高雄前鎮店</option>
        </select>
      </div>
      
      <div class="search-group">
        <input type="text" placeholder="搜尋商品名稱或編號..." class="search-input">
        <button class="btn btn-secondary search">搜尋</button>
      </div>
    </div>
  </div>
  
  <div class="inventory-table">
    <table class="table">
      <thead>
        <tr>
          <th>
            <input type="checkbox" class="select-all">
          </th>
          <th>商品資訊</th>
          <th>商品編號</th>
          <th>分類</th>
          <th>當前庫存</th>
          <th>安全庫存</th>
          <th>單位成本</th>
          <th>庫存價值</th>
          <th>最後進貨</th>
          <th>狀態</th>
          <th>操作</th>
        </tr>
      </thead>
      <tbody>
        <tr class="inventory-row low-stock">
          <td>
            <input type="checkbox" class="row-select">
          </td>
          <td>
            <div class="product-info">
              <img src="coffee-beans.jpg" alt="咖啡豆" class="product-image">
              <div class="product-details">
                <span class="product-name">阿拉比卡咖啡豆</span>
                <span class="product-spec">1kg/包</span>
              </div>
            </div>
          </td>
          <td>PROD-001</td>
          <td>原物料</td>
          <td>
            <div class="stock-info">
              <span class="current-stock">25</span>
              <span class="stock-unit">包</span>
            </div>
          </td>
          <td>50 包</td>
          <td>$180</td>
          <td>$4,500</td>
          <td>2024-01-10</td>
          <td>
            <span class="status-badge low">庫存不足</span>
          </td>
          <td>
            <div class="action-buttons">
              <button class="btn-icon edit" title="編輯">✏️</button>
              <button class="btn-icon restock" title="進貨">📦</button>
              <button class="btn-icon history" title="記錄">📋</button>
            </div>
          </td>
        </tr>
        
        <tr class="inventory-row expiring">
          <td>
            <input type="checkbox" class="row-select">
          </td>
          <td>
            <div class="product-info">
              <img src="milk.jpg" alt="牛奶" class="product-image">
              <div class="product-details">
                <span class="product-name">鮮奶</span>
                <span class="product-spec">1L/瓶</span>
              </div>
            </div>
          </td>
          <td>PROD-025</td>
          <td>原物料</td>
          <td>
            <div class="stock-info">
              <span class="current-stock">15</span>
              <span class="stock-unit">瓶</span>
            </div>
          </td>
          <td>30 瓶</td>
          <td>$65</td>
          <td>$975</td>
          <td>2024-01-12</td>
          <td>
            <span class="status-badge expiring">即將過期</span>
          </td>
          <td>
            <div class="action-buttons">
              <button class="btn-icon edit" title="編輯">✏️</button>
              <button class="btn-icon restock" title="進貨">📦</button>
              <button class="btn-icon history" title="記錄">📋</button>
            </div>
          </td>
        </tr>
      </tbody>
    </table>
  </div>
  
  <div class="bulk-actions">
    <div class="selected-info">
      已選擇 <span class="selected-count">0</span> 項商品
    </div>
    <div class="bulk-buttons">
      <button class="btn btn-secondary bulk-restock" disabled>批量進貨</button>
      <button class="btn btn-secondary bulk-adjust" disabled>調整庫存</button>
      <button class="btn btn-danger bulk-delete" disabled>批量刪除</button>
    </div>
  </div>
</div>
```

### 5.2 進貨管理介面
```html
<div class="purchase-page">
  <div class="page-header">
    <h1>進貨管理</h1>
    <div class="header-actions">
      <button class="btn btn-secondary import-order">匯入訂單</button>
      <button class="btn btn-primary create-order">建立進貨單</button>
    </div>
  </div>
  
  <div class="purchase-orders">
    <div class="orders-filters">
      <select class="filter-select">
        <option value="">全部狀態</option>
        <option value="draft">草稿</option>
        <option value="pending">待審核</option>
        <option value="approved">已審核</option>
        <option value="shipped">已出貨</option>
        <option value="received">已收貨</option>
      </select>
    </div>
    
    <div class="orders-table">
      <table class="table">
        <thead>
          <tr>
            <th>訂單編號</th>
            <th>供應商</th>
            <th>訂購日期</th>
            <th>預計到貨</th>
            <th>總金額</th>
            <th>狀態</th>
            <th>操作</th>
          </tr>
        </thead>
        <tbody>
          <tr>
            <td>PO-2024-001</td>
            <td>優質咖啡供應商</td>
            <td>2024-01-15</td>
            <td>2024-01-18</td>
            <td>$25,000</td>
            <td>
              <span class="status-badge approved">已審核</span>
            </td>
            <td>
              <div class="action-buttons">
                <button class="btn btn-sm btn-secondary view-order">查看</button>
                <button class="btn btn-sm btn-primary receive-order">收貨</button>
              </div>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</div>
```

---

## 6. 營運報表介面

### 6.1 財務報表頁面
```html
<div class="financial-reports-page">
  <div class="page-header">
    <h1>財務報表</h1>
    <div class="header-actions">
      <select class="period-selector">
        <option value="daily">日報</option>
        <option value="weekly">週報</option>
        <option value="monthly" selected>月報</option>
        <option value="quarterly">季報</option>
        <option value="yearly">年報</option>
      </select>
      <input type="month" class="date-picker" value="2024-01">
      <button class="btn btn-primary export-excel">匯出 Excel</button>
    </div>
  </div>
  
  <div class="financial-overview">
    <div class="overview-grid">
      <div class="financial-card revenue">
        <h3>營業收入</h3>
        <div class="amount">$2,850,000</div>
        <div class="change positive">+12.5%</div>
      </div>
      
      <div class="financial-card cost">
        <h3>營業成本</h3>
        <div class="amount">$1,140,000</div>
        <div class="change neutral">+2.1%</div>
      </div>
      
      <div class="financial-card profit">
        <h3>毛利潤</h3>
        <div class="amount">$1,710,000</div>
        <div class="change positive">+18.2%</div>
      </div>
      
      <div class="financial-card margin">
        <h3>毛利率</h3>
        <div class="amount">60%</div>
        <div class="change positive">+3.2%</div>
      </div>
    </div>
  </div>
  
  <div class="financial-charts">
    <div class="chart-section">
      <div class="chart-card large">
        <h3>收入與成本分析</h3>
        <canvas id="revenueVsCostChart" width="800" height="400"></canvas>
      </div>
    </div>
    
    <div class="charts-row">
      <div class="chart-card">
        <h3>收入來源分佈</h3>
        <canvas id="revenueSourceChart" width="350" height="300"></canvas>
      </div>
      
      <div class="chart-card">
        <h3>成本結構分析</h3>
        <canvas id="costStructureChart" width="350" height="300"></canvas>
      </div>
    </div>
  </div>
  
  <div class="financial-details">
    <div class="details-tabs">
      <button class="tab active" data-tab="income">損益表</button>
      <button class="tab" data-tab="cashflow">現金流量</button>
      <button class="tab" data-tab="balance">資產負債</button>
    </div>
    
    <div class="tab-content active" data-content="income">
      <div class="income-statement">
        <table class="financial-table">
          <thead>
            <tr>
              <th>項目</th>
              <th>本月</th>
              <th>上月</th>
              <th>變化</th>
              <th>年累計</th>
            </tr>
          </thead>
          <tbody>
            <tr class="revenue-row">
              <td class="item-name">營業收入</td>
              <td class="amount">$2,850,000</td>
              <td class="amount">$2,534,000</td>
              <td class="change positive">+12.5%</td>
              <td class="amount">$28,500,000</td>
            </tr>
            <tr class="cost-row">
              <td class="item-name">營業成本</td>
              <td class="amount">$1,140,000</td>
              <td class="amount">$1,117,000</td>
              <td class="change negative">+2.1%</td>
              <td class="amount">$11,400,000</td>
            </tr>
            <tr class="profit-row">
              <td class="item-name">毛利潤</td>
              <td class="amount highlight">$1,710,000</td>
              <td class="amount">$1,417,000</td>
              <td class="change positive">+20.7%</td>
              <td class="amount">$17,100,000</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</div>
```

---

## 7. 系統管理介面

### 7.1 用戶管理頁面
```html
<div class="user-management-page">
  <div class="page-header">
    <h1>用戶管理</h1>
    <div class="header-actions">
      <button class="btn btn-secondary import-users">批量匯入</button>
      <button class="btn btn-primary add-user">新增用戶</button>
    </div>
  </div>
  
  <div class="users-filters">
    <div class="filter-row">
      <div class="filter-group">
        <label>角色</label>
        <select class="filter-select">
          <option value="">全部角色</option>
          <option value="admin">系統管理員</option>
          <option value="manager">區域經理</option>
          <option value="store_manager">店長</option>
          <option value="staff">店員</option>
        </select>
      </div>
      
      <div class="filter-group">
        <label>狀態</label>
        <select class="filter-select">
          <option value="">全部狀態</option>
          <option value="active">啟用</option>
          <option value="inactive">停用</option>
          <option value="pending">待審核</option>
        </select>
      </div>
      
      <div class="filter-group">
        <label>門店</label>
        <select class="filter-select">
          <option value="">全部門店</option>
          <option value="STR-001">台北信義店</option>
          <option value="STR-002">台中西屯店</option>
        </select>
      </div>
      
      <div class="search-group">
        <input type="text" placeholder="搜尋用戶名稱或 Email..." class="search-input">
        <button class="btn btn-secondary search">搜尋</button>
      </div>
    </div>
  </div>
  
  <div class="users-table">
    <table class="table">
      <thead>
        <tr>
          <th>
            <input type="checkbox" class="select-all">
          </th>
          <th>用戶資訊</th>
          <th>角色</th>
          <th>門店</th>
          <th>最後登入</th>
          <th>狀態</th>
          <th>操作</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>
            <input type="checkbox" class="row-select">
          </td>
          <td>
            <div class="user-info">
              <img src="user-avatar.jpg" alt="User" class="user-avatar">
              <div class="user-details">
                <span class="user-name">張店長</span>
                <span class="user-email">zhang@coffeeshop.com</span>
                <span class="user-phone">0912-345-678</span>
              </div>
            </div>
          </td>
          <td>
            <span class="role-badge manager">店長</span>
          </td>
          <td>台北信義店</td>
          <td>2024-01-15 14:30</td>
          <td>
            <span class="status-badge active">啟用</span>
          </td>
          <td>
            <div class="action-buttons">
              <button class="btn-icon edit" title="編輯">✏️</button>
              <button class="btn-icon permissions" title="權限">🔐</button>
              <button class="btn-icon disable" title="停用">⏸️</button>
            </div>
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</div>
```

### 7.2 權限管理對話框
```html
<div class="permissions-modal" id="permissionsModal">
  <div class="modal-content large">
    <div class="modal-header">
      <h2>權限管理 - 張店長</h2>
      <button class="close-btn">×</button>
    </div>
    
    <div class="modal-body">
      <div class="permissions-section">
        <h4>系統模組權限</h4>
        <div class="permissions-grid">
          <div class="permission-group">
            <h5>POS 系統</h5>
            <div class="permission-items">
              <label class="permission-item">
                <input type="checkbox" checked>
                <span>查看 POS</span>
              </label>
              <label class="permission-item">
                <input type="checkbox" checked>
                <span>操作 POS</span>
              </label>
              <label class="permission-item">
                <input type="checkbox">
                <span>退款處理</span>
              </label>
            </div>
          </div>
          
          <div class="permission-group">
            <h5>客戶管理</h5>
            <div class="permission-items">
              <label class="permission-item">
                <input type="checkbox" checked>
                <span>查看客戶</span>
              </label>
              <label class="permission-item">
                <input type="checkbox" checked>
                <span>編輯客戶</span>
              </label>
              <label class="permission-item">
                <input type="checkbox">
                <span>刪除客戶</span>
              </label>
            </div>
          </div>
          
          <div class="permission-group">
            <h5>報表查看</h5>
            <div class="permission-items">
              <label class="permission-item">
                <input type="checkbox" checked>
                <span>門店報表</span>
              </label>
              <label class="permission-item">
                <input type="checkbox">
                <span>財務報表</span>
              </label>
              <label class="permission-item">
                <input type="checkbox">
                <span>庫存報表</span>
              </label>
            </div>
          </div>
        </div>
      </div>
      
      <div class="data-permissions-section">
        <h4>數據權限範圍</h4>
        <div class="data-scope">
          <label class="scope-option">
            <input type="radio" name="dataScope" value="own" checked>
            <span>僅限本門店</span>
          </label>
          <label class="scope-option">
            <input type="radio" name="dataScope" value="region">
            <span>區域門店</span>
          </label>
          <label class="scope-option">
            <input type="radio" name="dataScope" value="all">
            <span>全部門店</span>
          </label>
        </div>
      </div>
    </div>
    
    <div class="modal-footer">
      <button class="btn btn-secondary cancel">取消</button>
      <button class="btn btn-primary save-permissions">儲存權限</button>
    </div>
  </div>
</div>
```

---

## 8. 響應式設計 (RWD)

### 8.1 桌面版佈局 (>= 1200px)
- 完整的三欄式佈局
- 豐富的數據視覺化圖表
- 詳細的數據表格展示

### 8.2 平板版佈局 (768px - 1199px)
```css
@media (max-width: 1199px) {
  .erp-sidebar {
    width: 200px;
  }
  
  .charts-grid {
    grid-template-columns: 1fr;
  }
  
  .overview-grid {
    grid-template-columns: repeat(2, 1fr);
  }
  
  .table {
    font-size: 14px;
  }
  
  .table th,
  .table td {
    padding: 8px;
  }
}
```

### 8.3 手機版佈局 (<= 767px)
```css
@media (max-width: 767px) {
  .erp-header {
    padding: 0 16px;
  }
  
  .main-nav {
    display: none;
  }
  
  .mobile-menu-btn {
    display: block;
  }
  
  .erp-sidebar {
    position: fixed;
    top: 72px;
    left: -280px;
    width: 280px;
    height: calc(100vh - 72px);
    transition: left 0.3s ease;
    z-index: 1000;
  }
  
  .erp-sidebar.open {
    left: 0;
  }
  
  .overview-grid {
    grid-template-columns: 1fr;
    gap: 12px;
  }
  
  .kpi-grid {
    grid-template-columns: repeat(2, 1fr);
  }
  
  .stores-grid {
    grid-template-columns: 1fr;
  }
  
  .table-responsive {
    overflow-x: auto;
  }
  
  .table {
    min-width: 600px;
  }
}
```

---

## 9. 主題與樣式

### 9.1 ERP 專用色彩系統
```css
:root {
  /* 主要色彩 */
  --erp-primary: #1a365d;
  --erp-secondary: #2d3748;
  --erp-accent: #3182ce;
  
  /* 功能色彩 */
  --success: #38a169;
  --warning: #d69e2e;
  --danger: #e53e3e;
  --info: #3182ce;
  
  /* 背景色彩 */
  --bg-primary: #f7fafc;
  --bg-secondary: #edf2f7;
  --bg-dark: #2d3748;
  
  /* 文字色彩 */
  --text-primary: #1a202c;
  --text-secondary: #4a5568;
  --text-muted: #718096;
}
```

### 9.2 卡片與容器樣式
```css
.erp-card {
  background: white;
  border-radius: 8px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  border: 1px solid var(--gray-200);
}

.metric-card {
  padding: 24px;
  border-left: 4px solid var(--erp-accent);
}

.chart-card {
  padding: 20px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}
```

---

## 10. 互動與動畫

### 10.1 數據更新動畫
```css
.metric-value {
  transition: all 0.3s ease;
}

.metric-value.updating {
  transform: scale(1.05);
  color: var(--erp-accent);
}

@keyframes countUp {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

.animated-number {
  animation: countUp 0.5s ease-out;
}
```

### 10.2 圖表載入效果
```css
.chart-loading {
  position: relative;
}

.chart-loading::after {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 30px;
  height: 30px;
  margin: -15px 0 0 -15px;
  border: 3px solid var(--gray-300);
  border-top: 3px solid var(--erp-accent);
  border-radius: 50%;
  animation: spin 1s linear infinite;
}
```

---

**版本**: v1.0  
**建立日期**: 2025-01-16  
**最後更新**: 2025-01-16  
**負責人**: UI/UX 設計團隊