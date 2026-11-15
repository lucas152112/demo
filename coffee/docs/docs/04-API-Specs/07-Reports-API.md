# 報表分析 API 規格 (Reports & Analytics API)

## 1. API 概觀

### 服務名稱
- **服務名稱**: Reports Service
- **Port**: 3007
- **Base URL**: `/api/v1/reports`
- **業務職責**: 業務報表生成、數據分析、統計圖表、營運儀表板

### 核心功能
- 營收報表分析
- 客戶行為分析
- 商品銷售分析
- 庫存分析報表
- 員工績效分析
- 門店營運分析
- 自定義報表

---

## 2. 認證與授權

### 認證方式
- Bearer Token (JWT)
- 所有 API 需要驗證 `Authorization: Bearer <token>`

### 權限等級
- **REPORTS_READ**: 報表查看權限
- **REPORTS_EXPORT**: 報表匯出權限
- **REPORTS_MANAGE**: 報表管理權限
- **REPORTS_ADMIN**: 報表系統管理權限

---

## 3. 資料模型

### 3.1 報表定義 (ReportDefinition)
```json
{
  "report_id": "string",            // 報表ID
  "report_name": "string",          // 報表名稱
  "report_type": "string",          // 報表類型 (sales/customer/inventory/staff)
  "description": "string",          // 描述
  "category": "string",             // 分類
  "data_source": "string",          // 資料源
  "query_config": "object",         // 查詢配置
  "chart_config": "object",         // 圖表配置
  "filters": "array",               // 篩選條件
  "schedule": "object",             // 排程設定
  "permissions": "array",           // 權限設定
  "is_active": "boolean",           // 是否啟用
  "created_by": "string",           // 建立者
  "created_at": "datetime",         // 建立時間
  "updated_at": "datetime"          // 更新時間
}
```

### 3.2 報表執行記錄 (ReportExecution)
```json
{
  "execution_id": "string",         // 執行ID
  "report_id": "string",           // 報表ID
  "parameters": "object",          // 執行參數
  "status": "string",              // 狀態 (running/completed/failed)
  "start_time": "datetime",        // 開始時間
  "end_time": "datetime",         // 結束時間
  "duration": "integer",           // 執行時長 (秒)
  "result_size": "integer",        // 結果大小
  "error_message": "string",       // 錯誤訊息
  "executed_by": "string",         // 執行者
  "created_at": "datetime"         // 建立時間
}
```

### 3.3 儀表板配置 (Dashboard)
```json
{
  "dashboard_id": "string",         // 儀表板ID
  "dashboard_name": "string",       // 儀表板名稱
  "description": "string",          // 描述
  "layout": "object",              // 佈局配置
  "widgets": "array",              // 小工具清單
  "refresh_interval": "integer",    // 刷新間隔 (秒)
  "is_public": "boolean",          // 是否公開
  "permissions": "array",          // 權限設定
  "created_by": "string",          // 建立者
  "created_at": "datetime",        // 建立時間
  "updated_at": "datetime"         // 更新時間
}
```

---

## 4. API 端點

### 4.1 營收報表

#### 4.1.1 日營收報表
```
GET /api/v1/reports/revenue/daily
```

**權限**: `REPORTS_READ`

**Query Parameters:**
- `store_id` (string, optional): 門店ID篩選
- `date_from` (date, required): 開始日期
- `date_to` (date, required): 結束日期
- `group_by` (string, optional): 分組方式 (day/store/payment_method)

**Response:**
```json
{
  "success": true,
  "data": {
    "summary": {
      "total_revenue": 125000.00,
      "total_orders": 850,
      "average_order_value": 147.06,
      "growth_rate": 15.2
    },
    "daily_data": [
      {
        "date": "2024-01-15",
        "revenue": 8500.00,
        "orders": 58,
        "average_order_value": 146.55,
        "peak_hour": "14:00"
      }
    ],
    "chart_data": {
      "labels": ["2024-01-15", "2024-01-16"],
      "datasets": [
        {
          "label": "營收",
          "data": [8500.00, 9200.00],
          "backgroundColor": "#3498db"
        }
      ]
    }
  }
}
```

#### 4.1.2 月營收報表
```
GET /api/v1/reports/revenue/monthly
```

**權限**: `REPORTS_READ`

**Query Parameters:**
- `year` (integer, required): 年份
- `month` (integer, optional): 月份
- `store_id` (string, optional): 門店ID篩選

#### 4.1.3 年度營收報表
```
GET /api/v1/reports/revenue/yearly
```

**權限**: `REPORTS_READ`

#### 4.1.4 營收趨勢分析
```
GET /api/v1/reports/revenue/trends
```

**權限**: `REPORTS_READ`

**Query Parameters:**
- `period` (string, required): 期間 (week/month/quarter/year)
- `compare_period` (boolean, optional): 是否比較同期

### 4.2 客戶分析報表

#### 4.2.1 客戶統計概覽
```
GET /api/v1/reports/customers/overview
```

**權限**: `REPORTS_READ`

**Query Parameters:**
- `date_from` (date, required): 開始日期
- `date_to` (date, required): 結束日期

**Response:**
```json
{
  "success": true,
  "data": {
    "summary": {
      "total_customers": 5000,
      "new_customers": 250,
      "active_customers": 1200,
      "retention_rate": 75.5,
      "churn_rate": 8.2
    },
    "segments": {
      "by_tier": {
        "Bronze": 3000,
        "Silver": 1500,
        "Gold": 400,
        "Platinum": 100
      },
      "by_age_group": {
        "18-25": 1200,
        "26-35": 2000,
        "36-45": 1300,
        "46+": 500
      }
    },
    "acquisition_channels": [
      {
        "channel": "社群媒體",
        "customers": 120,
        "percentage": 48.0
      }
    ]
  }
}
```

#### 4.2.2 客戶行為分析
```
GET /api/v1/reports/customers/behavior
```

**權限**: `REPORTS_READ`

#### 4.2.3 客戶價值分析 (RFM)
```
GET /api/v1/reports/customers/rfm-analysis
```

**權限**: `REPORTS_READ`

**Response:**
```json
{
  "success": true,
  "data": {
    "segments": [
      {
        "segment": "Champions",
        "description": "最佳客戶",
        "count": 150,
        "percentage": 3.0,
        "avg_revenue": 2500.00,
        "characteristics": "高頻次、近期消費、高金額"
      }
    ],
    "rfm_scores": {
      "recency": {
        "avg_score": 3.2,
        "distribution": [10, 15, 25, 30, 20]
      },
      "frequency": {
        "avg_score": 2.8,
        "distribution": [20, 25, 20, 20, 15]
      },
      "monetary": {
        "avg_score": 3.1,
        "distribution": [15, 20, 25, 25, 15]
      }
    }
  }
}
```

#### 4.2.4 客戶流失預測
```
GET /api/v1/reports/customers/churn-prediction
```

**權限**: `REPORTS_READ`

### 4.3 商品銷售分析

#### 4.3.1 商品銷售排行
```
GET /api/v1/reports/products/bestsellers
```

**權限**: `REPORTS_READ`

**Query Parameters:**
- `period` (string, required): 期間 (day/week/month)
- `limit` (integer, optional): 顯示數量 (預設: 20)
- `category` (string, optional): 商品分類篩選
- `store_id` (string, optional): 門店篩選

**Response:**
```json
{
  "success": true,
  "data": {
    "ranking": [
      {
        "rank": 1,
        "product_id": "PROD-001",
        "product_name": "美式咖啡",
        "category": "咖啡",
        "quantity_sold": 450,
        "revenue": 13500.00,
        "profit": 8100.00,
        "profit_margin": 60.0,
        "growth_rate": 12.5
      }
    ],
    "summary": {
      "total_products_sold": 2850,
      "total_revenue": 85500.00,
      "avg_profit_margin": 55.2
    }
  }
}
```

#### 4.3.2 商品分類分析
```
GET /api/v1/reports/products/category-analysis
```

**權限**: `REPORTS_READ`

#### 4.3.3 商品庫存週轉率
```
GET /api/v1/reports/products/inventory-turnover
```

**權限**: `REPORTS_READ`

#### 4.3.4 新商品表現分析
```
GET /api/v1/reports/products/new-product-performance
```

**權限**: `REPORTS_READ`

### 4.4 門店營運分析

#### 4.4.1 門店績效比較
```
GET /api/v1/reports/stores/performance
```

**權限**: `REPORTS_READ`

**Query Parameters:**
- `date_from` (date, required): 開始日期
- `date_to` (date, required): 結束日期
- `metrics` (array, optional): 指標選擇

**Response:**
```json
{
  "success": true,
  "data": {
    "stores": [
      {
        "store_id": "ST-001",
        "store_name": "台北信義店",
        "revenue": 125000.00,
        "orders": 850,
        "customers": 650,
        "avg_order_value": 147.06,
        "customer_satisfaction": 4.5,
        "staff_efficiency": 85.2,
        "ranking": 1
      }
    ],
    "metrics_comparison": {
      "revenue": {
        "best_performer": "ST-001",
        "worst_performer": "ST-005",
        "variance": 45.2
      }
    }
  }
}
```

#### 4.4.2 營業時段分析
```
GET /api/v1/reports/stores/peak-hours
```

**權限**: `REPORTS_READ`

#### 4.4.3 門店客流分析
```
GET /api/v1/reports/stores/traffic-analysis
```

**權限**: `REPORTS_READ`

### 4.5 員工績效分析

#### 4.5.1 員工銷售績效
```
GET /api/v1/reports/staff/sales-performance
```

**權限**: `REPORTS_READ`

**Query Parameters:**
- `store_id` (string, optional): 門店篩選
- `date_from` (date, required): 開始日期
- `date_to` (date, required): 結束日期

**Response:**
```json
{
  "success": true,
  "data": {
    "staff_performance": [
      {
        "staff_id": "STAFF-001",
        "staff_name": "王小美",
        "store_name": "台北信義店",
        "total_sales": 25000.00,
        "orders_handled": 180,
        "avg_order_value": 138.89,
        "customer_rating": 4.7,
        "upsell_rate": 25.5,
        "ranking": 1
      }
    ],
    "summary": {
      "total_staff": 25,
      "avg_sales_per_staff": 18500.00,
      "best_performer": "王小美",
      "improvement_needed": 3
    }
  }
}
```

#### 4.5.2 員工排班效率分析
```
GET /api/v1/reports/staff/schedule-efficiency
```

**權限**: `REPORTS_READ`

#### 4.5.3 員工培訓進度報表
```
GET /api/v1/reports/staff/training-progress
```

**權限**: `REPORTS_READ`

### 4.6 庫存分析報表

#### 4.6.1 庫存週轉分析
```
GET /api/v1/reports/inventory/turnover
```

**權限**: `REPORTS_READ`

**Query Parameters:**
- `store_id` (string, optional): 門店篩選
- `category` (string, optional): 商品分類篩選
- `period` (string, required): 分析期間

#### 4.6.2 庫存成本分析
```
GET /api/v1/reports/inventory/cost-analysis
```

**權限**: `REPORTS_READ`

#### 4.6.3 缺貨損失分析
```
GET /api/v1/reports/inventory/stockout-analysis
```

**權限**: `REPORTS_READ`

### 4.7 自定義報表

#### 4.7.1 查詢報表定義列表
```
GET /api/v1/reports/custom
```

**權限**: `REPORTS_READ`

#### 4.7.2 建立自定義報表
```
POST /api/v1/reports/custom
```

**權限**: `REPORTS_MANAGE`

**Request Body:**
```json
{
  "report_name": "門店月度績效報表",
  "report_type": "custom",
  "description": "每月門店營收與客戶滿意度綜合報表",
  "data_source": "orders,customers,reviews",
  "query_config": {
    "tables": ["orders", "customers"],
    "fields": ["revenue", "order_count", "customer_satisfaction"],
    "filters": [
      {
        "field": "order_date",
        "operator": "between",
        "values": ["{start_date}", "{end_date}"]
      }
    ],
    "group_by": ["store_id", "month"]
  },
  "chart_config": {
    "type": "combo",
    "x_axis": "month",
    "y_axes": [
      {
        "field": "revenue",
        "type": "bar",
        "position": "left"
      },
      {
        "field": "customer_satisfaction",
        "type": "line",
        "position": "right"
      }
    ]
  },
  "parameters": [
    {
      "name": "start_date",
      "type": "date",
      "required": true
    },
    {
      "name": "end_date",
      "type": "date",
      "required": true
    }
  ]
}
```

#### 4.7.3 執行自定義報表
```
POST /api/v1/reports/custom/{report_id}/execute
```

**權限**: `REPORTS_READ`

**Request Body:**
```json
{
  "parameters": {
    "start_date": "2024-01-01",
    "end_date": "2024-01-31",
    "store_id": "ST-001"
  },
  "format": "json"
}
```

### 4.8 儀表板

#### 4.8.1 查詢儀表板列表
```
GET /api/v1/reports/dashboards
```

**權限**: `REPORTS_READ`

#### 4.8.2 查詢儀表板詳情
```
GET /api/v1/reports/dashboards/{dashboard_id}
```

**權限**: `REPORTS_READ`

**Response:**
```json
{
  "success": true,
  "data": {
    "dashboard_id": "DASH-001",
    "dashboard_name": "營運總覽儀表板",
    "description": "主要營運指標監控",
    "widgets": [
      {
        "widget_id": "WIDGET-001",
        "widget_type": "metric_card",
        "title": "今日營收",
        "position": {"x": 0, "y": 0, "w": 3, "h": 2},
        "data_source": "revenue_daily",
        "config": {
          "metric": "total_revenue",
          "format": "currency",
          "comparison": "yesterday"
        }
      }
    ],
    "last_updated": "2024-01-15T10:30:00Z"
  }
}
```

#### 4.8.3 建立儀表板
```
POST /api/v1/reports/dashboards
```

**權限**: `REPORTS_MANAGE`

### 4.9 報表匯出

#### 4.9.1 匯出報表為 Excel
```
POST /api/v1/reports/export/excel
```

**權限**: `REPORTS_EXPORT`

**Request Body:**
```json
{
  "report_type": "revenue_daily",
  "parameters": {
    "date_from": "2024-01-01",
    "date_to": "2024-01-31",
    "store_id": "ST-001"
  },
  "template": "standard",
  "email_to": "manager@coffee.com"
}
```

#### 4.9.2 匯出報表為 PDF
```
POST /api/v1/reports/export/pdf
```

**權限**: `REPORTS_EXPORT`

#### 4.9.3 排程報表匯出
```
POST /api/v1/reports/schedule
```

**權限**: `REPORTS_MANAGE`

**Request Body:**
```json
{
  "report_id": "RPT-001",
  "schedule": {
    "frequency": "weekly",
    "day_of_week": "monday",
    "time": "09:00"
  },
  "recipients": ["manager@coffee.com"],
  "format": "excel",
  "is_active": true
}
```

### 4.10 報表效能監控

#### 4.10.1 查詢報表執行記錄
```
GET /api/v1/reports/execution-logs
```

**權限**: `REPORTS_ADMIN`

#### 4.10.2 查詢系統效能統計
```
GET /api/v1/reports/performance-stats
```

**權限**: `REPORTS_ADMIN`

---

## 5. 錯誤代碼

### 報表相關錯誤
- `REPORT_001`: 報表不存在
- `REPORT_002`: 報表參數不正確
- `REPORT_003`: 資料查詢失敗
- `REPORT_004`: 報表生成超時
- `REPORT_005`: 匯出格式不支援
- `REPORT_006`: 報表權限不足

### 資料相關錯誤
- `DATA_001`: 查詢資料過大
- `DATA_002`: 日期範圍不正確
- `DATA_003`: 資料來源不可用

---

## 6. 業務規則

### 6.1 報表生成規則
- 報表資料更新頻率：即時/小時/日
- 大型報表限制查詢期間（最多1年）
- 複雜報表採用背景處理

### 6.2 權限規則
- 門店員工僅可查看所屬門店報表
- 區域主管可查看管轄區域報表
- 總部主管可查看全部報表

### 6.3 效能規則
- 報表查詢超過30秒自動終止
- 同時執行報表數量限制
- 資料量過大自動分頁處理

---

## 7. 整合介面

### 7.1 內部服務整合
- 訂單服務：取得交易資料
- 客戶服務：取得客戶資料
- 庫存服務：取得庫存資料
- 員工服務：取得員工資料

### 7.2 外部服務整合
- 商業智慧工具 (Power BI, Tableau)
- 會計系統整合
- 第三方分析平台

---

## 8. 性能要求

### 8.1 回應時間
- 簡單報表查詢: < 2秒
- 複雜報表查詢: < 30秒
- 儀表板載入: < 3秒

### 8.2 資料處理
- 支援百萬級資料查詢
- 即時資料更新 (< 5分鐘)
- 歷史資料保留 3 年

---

## 9. 測試案例

### 9.1 功能測試
- 各類報表正確性
- 資料計算準確性
- 圖表顯示功能
- 匯出功能

### 9.2 效能測試
- 大資料量查詢
- 並發報表生成
- 系統負載測試

---

**版本**: v1.0  
**建立日期**: 2025-11-16  
**最後更新**: 2025-11-16  
**負責人**: 開發團隊