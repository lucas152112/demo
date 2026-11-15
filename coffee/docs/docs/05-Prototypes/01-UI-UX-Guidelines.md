# UI/UX 設計指南

## 1. 設計理念

### 1.1 核心設計原則
- **簡潔易用**: 介面設計簡潔明了，減少使用者認知負擔
- **一致性**: 整體風格統一，操作邏輯一致
- **效率優先**: 針對高頻操作優化，提升工作效率
- **響應式**: 支援桌面、平板、手機等多種裝置
- **無障礙**: 符合 WCAG 2.1 AA 無障礙標準

### 1.2 目標使用者
- **CRM 系統**: 門店店員、店長
- **ERP 系統**: 總部採購、管理層、系統管理員
- **行動端**: 顧客、會員

### 1.3 使用情境
- **高壓環境**: 門店營業時間忙碌，需要快速操作
- **多工作業**: 同時處理訂單、客戶服務、庫存管理
- **跨裝置**: 需要在不同裝置間切換使用

---

## 2. 視覺設計系統

### 2.1 色彩系統

#### 主要色彩
```css
/* 品牌主色 */
--primary-color: #8B4513;        /* 咖啡棕 */
--primary-light: #D2691E;        /* 淺咖啡 */
--primary-dark: #654321;         /* 深咖啡 */

/* 功能色彩 */
--success-color: #28a745;        /* 成功綠 */
--warning-color: #ffc107;        /* 警告黃 */
--danger-color: #dc3545;         /* 危險紅 */
--info-color: #17a2b8;           /* 資訊藍 */

/* 中性色彩 */
--gray-50: #f9fafb;              /* 背景灰 */
--gray-100: #f3f4f6;             /* 淺灰 */
--gray-300: #d1d5db;             /* 邊框灰 */
--gray-500: #6b7280;             /* 文字灰 */
--gray-700: #374151;             /* 深文字 */
--gray-900: #111827;             /* 標題黑 */

/* 白色與黑色 */
--white: #ffffff;
--black: #000000;
```

#### 色彩使用指引
- **主色**: 用於重要按鈕、連結、品牌元素
- **成功色**: 確認操作、成功狀態
- **警告色**: 需要注意的資訊、待處理項目
- **危險色**: 刪除操作、錯誤狀態
- **資訊色**: 一般提示、說明文字

### 2.2 字體系統

#### 字體家族
```css
/* 主要字體 */
font-family: 'Inter', 'Noto Sans TC', system-ui, sans-serif;

/* 等寬字體 (程式碼、數字) */
font-family: 'JetBrains Mono', 'Source Code Pro', monospace;
```

#### 字體大小
```css
/* 標題字體 */
--text-xs: 12px;       /* 小註解 */
--text-sm: 14px;       /* 一般文字 */
--text-base: 16px;     /* 基準文字 */
--text-lg: 18px;       /* 大文字 */
--text-xl: 20px;       /* 小標題 */
--text-2xl: 24px;      /* 中標題 */
--text-3xl: 30px;      /* 大標題 */
--text-4xl: 36px;      /* 主標題 */
```

#### 字重
```css
--font-normal: 400;    /* 一般文字 */
--font-medium: 500;    /* 重點文字 */
--font-semibold: 600;  /* 副標題 */
--font-bold: 700;      /* 主標題 */
```

### 2.3 間距系統
```css
/* 間距單位 (基於 4px) */
--space-1: 4px;
--space-2: 8px;
--space-3: 12px;
--space-4: 16px;
--space-5: 20px;
--space-6: 24px;
--space-8: 32px;
--space-10: 40px;
--space-12: 48px;
--space-16: 64px;
--space-20: 80px;
--space-24: 96px;
```

### 2.4 圓角與陰影
```css
/* 圓角 */
--radius-sm: 2px;      /* 小元件 */
--radius-md: 4px;      /* 按鈕、輸入框 */
--radius-lg: 8px;      /* 卡片 */
--radius-xl: 12px;     /* 大卡片 */
--radius-full: 9999px; /* 圓形 */

/* 陰影 */
--shadow-sm: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
--shadow-md: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
--shadow-lg: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
--shadow-xl: 0 20px 25px -5px rgba(0, 0, 0, 0.1);
```

---

## 3. 元件設計規範

### 3.1 按鈕設計

#### 主要按鈕 (Primary Button)
```css
.btn-primary {
  background-color: var(--primary-color);
  color: var(--white);
  padding: var(--space-3) var(--space-6);
  border-radius: var(--radius-md);
  font-weight: var(--font-medium);
  transition: all 0.2s ease;
}

.btn-primary:hover {
  background-color: var(--primary-dark);
  transform: translateY(-1px);
}
```

#### 次要按鈕 (Secondary Button)
```css
.btn-secondary {
  background-color: transparent;
  color: var(--primary-color);
  border: 1px solid var(--primary-color);
  padding: var(--space-3) var(--space-6);
  border-radius: var(--radius-md);
  font-weight: var(--font-medium);
}
```

#### 按鈕尺寸
- **小型**: 高度 32px, 間距 12px 16px
- **中型**: 高度 40px, 間距 12px 24px (預設)
- **大型**: 高度 48px, 間距 16px 32px

### 3.2 輸入框設計
```css
.input-field {
  width: 100%;
  padding: var(--space-3) var(--space-4);
  border: 1px solid var(--gray-300);
  border-radius: var(--radius-md);
  font-size: var(--text-base);
  transition: border-color 0.2s ease;
}

.input-field:focus {
  outline: none;
  border-color: var(--primary-color);
  box-shadow: 0 0 0 3px rgba(139, 69, 19, 0.1);
}
```

### 3.3 卡片設計
```css
.card {
  background: var(--white);
  border-radius: var(--radius-lg);
  box-shadow: var(--shadow-sm);
  padding: var(--space-6);
  border: 1px solid var(--gray-100);
}

.card-header {
  border-bottom: 1px solid var(--gray-100);
  padding-bottom: var(--space-4);
  margin-bottom: var(--space-4);
}
```

### 3.4 表格設計
```css
.table {
  width: 100%;
  border-collapse: collapse;
}

.table th {
  background-color: var(--gray-50);
  padding: var(--space-3) var(--space-4);
  text-align: left;
  font-weight: var(--font-semibold);
  border-bottom: 1px solid var(--gray-200);
}

.table td {
  padding: var(--space-3) var(--space-4);
  border-bottom: 1px solid var(--gray-100);
}

.table tr:hover {
  background-color: var(--gray-50);
}
```

---

## 4. 佈局規範

### 4.1 網格系統
採用 12 欄網格系統，支援響應式斷點：

```css
/* 斷點定義 */
--breakpoint-sm: 640px;   /* 手機 */
--breakpoint-md: 768px;   /* 平板 */
--breakpoint-lg: 1024px;  /* 筆電 */
--breakpoint-xl: 1280px;  /* 桌機 */
--breakpoint-2xl: 1536px; /* 大桌機 */
```

### 4.2 頁面佈局

#### 主要佈局結構
```
┌─────────────────────────────────────┐
│                Header               │ 64px
├─────────────┬───────────────────────┤
│   Sidebar   │      Main Content     │
│    240px    │       Flexible       │
│             │                       │
│             │                       │
└─────────────┴───────────────────────┘
```

#### CRM 佈局 (門店系統)
- **頭部導航**: 64px 高度，包含標誌、店名、使用者資訊
- **側邊欄**: 240px 寬度，可摺疊至 64px
- **主要內容區**: 彈性寬度，最小 320px

#### ERP 佈局 (總部系統)
- **頭部導航**: 64px 高度，包含標誌、模組切換、使用者資訊
- **側邊欄**: 280px 寬度，支援多層選單
- **主要內容區**: 彈性寬度，支援分頁標籤

### 4.3 響應式設計

#### 手機端 (< 768px)
- 側邊欄轉為抽屜式選單
- 表格採用卡片式展示
- 按鈕改為全寬顯示

#### 平板端 (768px - 1024px)
- 側邊欄可摺疊
- 保持桌面版功能
- 優化觸控操作

---

## 5. 交互設計規範

### 5.1 動畫與轉場
```css
/* 標準轉場時間 */
--transition-fast: 150ms;
--transition-normal: 300ms;
--transition-slow: 500ms;

/* 緩動函數 */
--ease-in-out: cubic-bezier(0.4, 0, 0.2, 1);
--ease-out: cubic-bezier(0.0, 0, 0.2, 1);
```

### 5.2 載入狀態
- **按鈕載入**: 顯示旋轉圖示，禁用按鈕
- **頁面載入**: 骨架屏或進度條
- **表格載入**: 顯示佔位符行

### 5.3 錯誤處理
- **表單錯誤**: 紅色邊框 + 錯誤文字
- **網路錯誤**: Toast 通知
- **系統錯誤**: 專用錯誤頁面

### 5.4 通知系統
```css
/* Toast 通知位置 */
.toast-container {
  position: fixed;
  top: var(--space-6);
  right: var(--space-6);
  z-index: 1000;
}

/* 通知類型 */
.toast-success { background: var(--success-color); }
.toast-warning { background: var(--warning-color); }
.toast-error { background: var(--danger-color); }
.toast-info { background: var(--info-color); }
```

---

## 6. 可用性指南

### 6.1 鍵盤操作
- **Tab**: 順序導航
- **Enter**: 確認/提交
- **Esc**: 取消/關閉
- **空格**: 選擇/切換
- **方向鍵**: 列表導航

### 6.2 無障礙設計
- **顏色對比**: 至少 4.5:1
- **聚焦指示**: 明確的聚焦框
- **替代文字**: 所有圖片提供 alt 屬性
- **標籤關聯**: 表單元素正確標籤

### 6.3 效能考量
- **圖片優化**: 使用 WebP 格式
- **字體優化**: 字體子集化
- **代碼分割**: 按需載入元件
- **快取策略**: 靜態資源快取

---

## 7. 品牌元素

### 7.1 標誌使用
- **最小尺寸**: 24px (數位), 15mm (印刷)
- **安全距離**: 標誌高度的 1/2
- **背景要求**: 確保對比度

### 7.2 圖示系統
採用 Heroicons 或 Lucide 圖示庫：
- **線條粗細**: 1.5px
- **尺寸規格**: 16px, 20px, 24px, 32px
- **風格**: 簡潔線條，統一風格

### 7.3 插圖風格
- **風格**: 扁平化設計
- **色彩**: 品牌色系為主
- **用途**: 空狀態、引導頁面、錯誤頁面

---

## 8. 開發指南

### 8.1 CSS 架構
採用 BEM (Block Element Modifier) 命名法：
```css
/* Block */
.menu { }

/* Element */
.menu__item { }
.menu__link { }

/* Modifier */
.menu__item--active { }
.menu__link--disabled { }
```

### 8.2 組件結構
```
/components
  /base          # 基礎元件 (Button, Input, Card)
  /form          # 表單元件 (FormField, FormGroup)
  /layout        # 佈局元件 (Header, Sidebar, Footer)
  /business      # 業務元件 (ProductCard, OrderList)
```

### 8.3 設計 Token
使用 CSS 自定義屬性管理設計 token：
```css
:root {
  /* 色彩 */
  --color-primary: #8B4513;
  
  /* 間距 */
  --space-sm: 8px;
  
  /* 字體 */
  --text-base: 16px;
}
```

---

## 9. 設計檢查清單

### 9.1 視覺設計
- [ ] 色彩對比符合無障礙標準
- [ ] 字體大小適中，易於閱讀
- [ ] 間距一致，佈局平衡
- [ ] 元件狀態完整 (正常、懸停、聚焦、禁用)

### 9.2 交互設計
- [ ] 操作流程順暢直觀
- [ ] 錯誤處理友善明確
- [ ] 載入狀態適當反饋
- [ ] 鍵盤操作完整支援

### 9.3 響應式設計
- [ ] 在各種裝置上正常顯示
- [ ] 觸控操作體驗良好
- [ ] 內容適配螢幕大小
- [ ] 效能在移動設備上可接受

---

## 10. 設計資源

### 10.1 設計工具
- **設計軟體**: Figma (推薦)
- **原型工具**: Figma Prototype
- **圖示庫**: Heroicons, Lucide
- **字體**: Google Fonts

### 10.2 參考資源
- **設計系統**: Material Design, Ant Design
- **色彩工具**: Coolors, Adobe Color
- **無障礙檢測**: WAVE, axe DevTools
- **效能檢測**: Lighthouse, PageSpeed Insights

---

**版本**: v1.0  
**建立日期**: 2025-11-16  
**最後更新**: 2025-11-16  
**負責人**: UI/UX 設計團隊