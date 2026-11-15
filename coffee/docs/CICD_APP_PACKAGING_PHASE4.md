# 🔄 Coffee系統 - 階段4: CI/CD與App打包

## 🚀 **CI/CD流水線設計**

### **自動化部署流程架構**
```yaml
觸發條件:
  - 程式碼推送到分支
  - Pull Request合併
  - 手動觸發部署
  - 定時建構 (每日)

多環境部署:
  - develop分支 → dev環境自動部署
  - env/test分支 → test環境自動部署
  - env/pp分支 → pp環境自動部署
  - env/prod分支 → prod環境手動部署
```

### **GitHub Actions工作流程**

#### **前端CI/CD流水線**
```yaml
# .github/workflows/frontend-ci-cd.yml
name: Frontend CI/CD

on:
  push:
    branches: [develop, env/test, env/pp, env/prod]
    paths: ['frontend/**']
  pull_request:
    branches: [develop]
    paths: ['frontend/**']

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'
          
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
        
      - name: Run linting
        run: pnpm lint
        
      - name: Run type checking
        run: pnpm type-check
        
      - name: Run unit tests
        run: pnpm test:unit
        
      - name: Run component tests
        run: pnpm test:component
        
      - name: Build application
        run: pnpm build
        
      - name: Run E2E tests
        run: pnpm test:e2e

  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    
    strategy:
      matrix:
        platform: [web, android, ios, windows, macos, linux]
        
    steps:
      - uses: actions/checkout@v4
      
      - name: Determine environment
        id: env
        run: |
          if [[ ${{ github.ref }} == 'refs/heads/develop' ]]; then
            echo "environment=dev" >> $GITHUB_OUTPUT
          elif [[ ${{ github.ref }} == 'refs/heads/env/test' ]]; then
            echo "environment=test" >> $GITHUB_OUTPUT
          elif [[ ${{ github.ref }} == 'refs/heads/env/pp' ]]; then
            echo "environment=pp" >> $GITHUB_OUTPUT
          elif [[ ${{ github.ref }} == 'refs/heads/env/prod' ]]; then
            echo "environment=prod" >> $GITHUB_OUTPUT
          fi
          
      - name: Build for ${{ matrix.platform }}
        run: |
          case ${{ matrix.platform }} in
            web)
              pnpm build:web --mode ${{ steps.env.outputs.environment }}
              ;;
            android)
              pnpm build:android --mode ${{ steps.env.outputs.environment }}
              ;;
            ios)
              pnpm build:ios --mode ${{ steps.env.outputs.environment }}
              ;;
            windows)
              pnpm build:electron --platform win32 --mode ${{ steps.env.outputs.environment }}
              ;;
            macos)
              pnpm build:electron --platform darwin --mode ${{ steps.env.outputs.environment }}
              ;;
            linux)
              pnpm build:electron --platform linux --mode ${{ steps.env.outputs.environment }}
              ;;
          esac
          
      - name: Deploy to ${{ steps.env.outputs.environment }}
        if: matrix.platform == 'web'
        run: |
          kubectl apply -f k8s/frontend-${{ steps.env.outputs.environment }}.yaml
          kubectl rollout status deployment/coffee-frontend-${{ steps.env.outputs.environment }}
```

#### **後端CI/CD流水線**
```yaml
# .github/workflows/backend-ci-cd.yml
name: Backend CI/CD

on:
  push:
    branches: [develop, env/test, env/pp, env/prod]
    paths: ['backend/**']
  pull_request:
    branches: [develop]
    paths: ['backend/**']

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
          
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
          
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy
          
      - name: Cache Cargo
        uses: actions/cache@v3
        with:
          path: |
            ~/.cargo/bin/
            ~/.cargo/registry/index/
            ~/.cargo/registry/cache/
            ~/.cargo/git/db/
            target/
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
          
      - name: Format check
        run: cargo fmt --all -- --check
        
      - name: Lint check
        run: cargo clippy --all-targets --all-features -- -D warnings
        
      - name: Run tests
        run: cargo test --all-features
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/coffee_test
          REDIS_URL: redis://localhost:6379/0
          
      - name: Security audit
        run: cargo audit
        
      - name: Build release
        run: cargo build --release

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    
    strategy:
      matrix:
        service: 
          - auth-service
          - customer-service
          - sales-service
          - inventory-service
          - purchase-service
          - finance-service
          - supplier-service
          - report-service
          - notification-service
          - audit-service
          - file-service
          - workflow-service
          - ai-chat-service
          - gateway-service
          - tenant-management
          - platform-ops
          
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}/${{ matrix.service }}
          
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: ./backend/${{ matrix.service }}
          file: ./backend/${{ matrix.service }}/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Determine environment
        id: env
        run: |
          if [[ ${{ github.ref }} == 'refs/heads/develop' ]]; then
            echo "environment=dev" >> $GITHUB_OUTPUT
          elif [[ ${{ github.ref }} == 'refs/heads/env/test' ]]; then
            echo "environment=test" >> $GITHUB_OUTPUT
          elif [[ ${{ github.ref }} == 'refs/heads/env/pp' ]]; then
            echo "environment=pp" >> $GITHUB_OUTPUT
          elif [[ ${{ github.ref }} == 'refs/heads/env/prod' ]]; then
            echo "environment=prod" >> $GITHUB_OUTPUT
          fi
          
      - name: Deploy to Kubernetes
        run: |
          kubectl apply -f k8s/${{ steps.env.outputs.environment }}/
          kubectl rollout status deployment/coffee-backend-${{ steps.env.outputs.environment }}
```

---

## 📱 **跨平台App打包配置**

### **1. Web應用 (PWA)**

#### **Nuxt.js PWA配置**
```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  modules: [
    '@vite-pwa/nuxt'
  ],
  
  pwa: {
    registerType: 'autoUpdate',
    workbox: {
      globPatterns: ['**/*.{js,css,html,png,svg,ico}'],
      navigateFallback: '/offline',
      runtimeCaching: [
        {
          urlPattern: /^https:\/\/api\.coffee\.local\/.*/i,
          handler: 'CacheFirst',
          options: {
            cacheName: 'coffee-api-cache',
            expiration: {
              maxEntries: 10,
              maxAgeSeconds: 60 * 60 * 24 * 365 // 1 year
            },
            cacheableResponse: {
              statuses: [0, 200]
            }
          }
        }
      ]
    },
    manifest: {
      name: 'Coffee Management System',
      short_name: 'Coffee CRM',
      description: '多租戶咖啡館管理系統',
      theme_color: '#8B4513',
      background_color: '#F5F5DC',
      display: 'standalone',
      icons: [
        {
          src: 'icon-192x192.png',
          sizes: '192x192',
          type: 'image/png'
        },
        {
          src: 'icon-512x512.png',
          sizes: '512x512',
          type: 'image/png'
        }
      ]
    }
  }
})
```

#### **離線功能實現**
```typescript
// plugins/offline.client.ts
export default defineNuxtPlugin(() => {
  if (process.client) {
    // 監聽網路狀態
    window.addEventListener('online', () => {
      // 同步離線期間的資料
      syncOfflineData()
    })
    
    window.addEventListener('offline', () => {
      // 啟用離線模式
      enableOfflineMode()
    })
  }
})

// composables/useOffline.ts
export const useOffline = () => {
  const isOnline = ref(true)
  const offlineQueue = ref([])
  
  const syncOfflineData = async () => {
    for (const item of offlineQueue.value) {
      try {
        await $fetch(item.url, item.options)
        offlineQueue.value = offlineQueue.value.filter(i => i.id !== item.id)
      } catch (error) {
        console.error('同步失敗:', error)
      }
    }
  }
  
  const addToOfflineQueue = (request) => {
    offlineQueue.value.push({
      id: Date.now(),
      ...request
    })
  }
  
  return {
    isOnline: readonly(isOnline),
    syncOfflineData,
    addToOfflineQueue
  }
}
```

### **2. 移動端App (Capacitor)**

#### **Capacitor配置**
```json
// capacitor.config.ts
import { CapacitorConfig } from '@capacitor/cli'

const config: CapacitorConfig = {
  appId: 'cc.justit.coffee',
  appName: 'Coffee Management',
  webDir: '.output/public',
  bundledWebRuntime: false,
  
  server: {
    androidScheme: 'https'
  },
  
  plugins: {
    SplashScreen: {
      launchShowDuration: 2000,
      backgroundColor: '#8B4513',
      androidScaleType: 'CENTER_CROP',
      showSpinner: true,
      spinnerColor: '#F5F5DC'
    },
    
    PushNotifications: {
      presentationOptions: ['badge', 'sound', 'alert']
    },
    
    LocalNotifications: {
      smallIcon: 'ic_stat_icon_config_sample',
      iconColor: '#8B4513'
    },
    
    Camera: {
      permissions: ['camera', 'photos']
    },
    
    Geolocation: {
      permissions: ['location']
    }
  }
}

export default config
```

#### **Android特定配置**
```json
// android/app/build.gradle (部分)
android {
    compileSdkVersion 34
    defaultConfig {
        applicationId "cc.justit.coffee"
        minSdkVersion 22
        targetSdkVersion 34
        versionCode 1
        versionName "1.0.0"
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
    }
    
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
            signingConfig signingConfigs.release
        }
        debug {
            applicationIdSuffix '.debug'
        }
    }
    
    flavorDimensions "version"
    productFlavors {
        dev {
            dimension "version"
            applicationIdSuffix ".dev"
            versionNameSuffix "-dev"
        }
        staging {
            dimension "version"
            applicationIdSuffix ".staging"
            versionNameSuffix "-staging"
        }
        prod {
            dimension "version"
        }
    }
}
```

#### **iOS特定配置**
```xml
<!-- ios/App/App/Info.plist -->
<dict>
    <key>CFBundleDevelopmentRegion</key>
    <string>zh_TW</string>
    <key>CFBundleDisplayName</key>
    <string>Coffee Management</string>
    <key>CFBundleExecutable</key>
    <string>$(EXECUTABLE_NAME)</string>
    <key>CFBundleIdentifier</key>
    <string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
    <key>CFBundleInfoDictionaryVersion</key>
    <string>6.0</string>
    <key>CFBundleName</key>
    <string>$(PRODUCT_NAME)</string>
    <key>CFBundlePackageType</key>
    <string>APPL</string>
    <key>CFBundleShortVersionString</key>
    <string>$(MARKETING_VERSION)</string>
    <key>CFBundleVersion</key>
    <string>$(CURRENT_PROJECT_VERSION)</string>
    
    <!-- 權限設定 -->
    <key>NSCameraUsageDescription</key>
    <string>掃描QR碼和拍攝商品照片</string>
    <key>NSLocationWhenInUseUsageDescription</key>
    <string>獲取門店位置資訊</string>
    <key>NSMicrophoneUsageDescription</key>
    <string>AI語音輸入功能</string>
</dict>
```

### **3. 桌面端App (Electron)**

#### **Electron主進程配置**
```typescript
// electron/main.ts
import { app, BrowserWindow, Menu, shell, ipcMain } from 'electron'
import { join } from 'path'
import { autoUpdater } from 'electron-updater'

class CoffeeApp {
  private mainWindow: BrowserWindow | null = null
  
  constructor() {
    app.whenReady().then(() => {
      this.createWindow()
      this.setupMenu()
      this.setupAutoUpdater()
      this.setupIPC()
    })
    
    app.on('window-all-closed', () => {
      if (process.platform !== 'darwin') {
        app.quit()
      }
    })
    
    app.on('activate', () => {
      if (BrowserWindow.getAllWindows().length === 0) {
        this.createWindow()
      }
    })
  }
  
  private createWindow() {
    this.mainWindow = new BrowserWindow({
      width: 1200,
      height: 800,
      minWidth: 800,
      minHeight: 600,
      webPreferences: {
        nodeIntegration: false,
        contextIsolation: true,
        preload: join(__dirname, 'preload.js')
      },
      icon: join(__dirname, '../assets/icon.png'),
      titleBarStyle: 'hiddenInset',
      show: false
    })
    
    // 載入應用
    if (process.env.NODE_ENV === 'development') {
      this.mainWindow.loadURL('http://localhost:3000')
      this.mainWindow.webContents.openDevTools()
    } else {
      this.mainWindow.loadFile(join(__dirname, '../dist/index.html'))
    }
    
    this.mainWindow.once('ready-to-show', () => {
      this.mainWindow?.show()
    })
    
    // 外部連結用瀏覽器開啟
    this.mainWindow.webContents.setWindowOpenHandler(({ url }) => {
      shell.openExternal(url)
      return { action: 'deny' }
    })
  }
  
  private setupMenu() {
    const template: Electron.MenuItemConstructorOptions[] = [
      {
        label: 'Coffee Management',
        submenu: [
          { label: '關於', role: 'about' },
          { type: 'separator' },
          { label: '服務', role: 'services' },
          { type: 'separator' },
          { label: '隱藏', role: 'hide' },
          { label: '隱藏其他', role: 'hideOthers' },
          { label: '全部顯示', role: 'unhide' },
          { type: 'separator' },
          { label: '退出', role: 'quit' }
        ]
      },
      {
        label: '檔案',
        submenu: [
          { label: '新增', accelerator: 'CmdOrCtrl+N' },
          { label: '開啟', accelerator: 'CmdOrCtrl+O' },
          { label: '儲存', accelerator: 'CmdOrCtrl+S' },
          { type: 'separator' },
          { label: '列印', accelerator: 'CmdOrCtrl+P' }
        ]
      },
      {
        label: '檢視',
        submenu: [
          { label: '重新載入', role: 'reload' },
          { label: '強制重新載入', role: 'forceReload' },
          { label: '開發者工具', role: 'toggleDevTools' },
          { type: 'separator' },
          { label: '實際大小', role: 'resetZoom' },
          { label: '放大', role: 'zoomIn' },
          { label: '縮小', role: 'zoomOut' },
          { type: 'separator' },
          { label: '全螢幕', role: 'togglefullscreen' }
        ]
      },
      {
        label: '視窗',
        submenu: [
          { label: '最小化', role: 'minimize' },
          { label: '關閉', role: 'close' }
        ]
      }
    ]
    
    Menu.setApplicationMenu(Menu.buildFromTemplate(template))
  }
  
  private setupAutoUpdater() {
    autoUpdater.checkForUpdatesAndNotify()
    
    autoUpdater.on('update-available', () => {
      console.log('有可用更新')
    })
    
    autoUpdater.on('update-downloaded', () => {
      console.log('更新已下載')
      autoUpdater.quitAndInstall()
    })
  }
  
  private setupIPC() {
    ipcMain.handle('get-app-version', () => {
      return app.getVersion()
    })
    
    ipcMain.handle('show-save-dialog', async () => {
      const { dialog } = await import('electron')
      return dialog.showSaveDialog(this.mainWindow!, {
        filters: [
          { name: 'CSV', extensions: ['csv'] },
          { name: 'PDF', extensions: ['pdf'] }
        ]
      })
    })
  }
}

new CoffeeApp()
```

#### **Electron打包配置**
```json
// electron-builder.json
{
  "appId": "cc.justit.coffee",
  "productName": "Coffee Management System",
  "directories": {
    "output": "dist-electron"
  },
  "files": [
    "dist/**/*",
    "electron/**/*",
    "!node_modules/**/*",
    "node_modules/electron/**/*"
  ],
  
  "mac": {
    "category": "public.app-category.business",
    "icon": "build/icon.icns",
    "hardenedRuntime": true,
    "entitlements": "build/entitlements.mac.plist",
    "entitlementsInherit": "build/entitlements.mac.plist",
    "target": [
      {
        "target": "dmg",
        "arch": ["x64", "arm64"]
      }
    ]
  },
  
  "win": {
    "target": [
      {
        "target": "nsis",
        "arch": ["x64", "ia32"]
      }
    ],
    "icon": "build/icon.ico"
  },
  
  "linux": {
    "target": [
      {
        "target": "AppImage",
        "arch": ["x64"]
      },
      {
        "target": "deb",
        "arch": ["x64"]
      }
    ],
    "icon": "build/icon.png",
    "category": "Office"
  },
  
  "nsis": {
    "oneClick": false,
    "allowToChangeInstallationDirectory": true,
    "installerIcon": "build/icon.ico",
    "uninstallerIcon": "build/icon.ico",
    "installerHeaderIcon": "build/icon.ico",
    "createDesktopShortcut": true,
    "createStartMenuShortcut": true
  }
}
```

---

## 🏗️ **Docker容器化配置**

### **前端Dockerfile**
```dockerfile
# Dockerfile.frontend
FROM node:20-alpine AS builder

WORKDIR /app

# 安裝pnpm
RUN npm install -g pnpm

# 複製依賴文件
COPY package.json pnpm-lock.yaml ./
COPY shared/ ./shared/
COPY frontend/ ./frontend/

# 安裝依賴
RUN pnpm install --frozen-lockfile

# 建置應用
WORKDIR /app/frontend
RUN pnpm build

# 生產階段
FROM nginx:alpine

# 複製建置文件
COPY --from=builder /app/frontend/.output/public /usr/share/nginx/html

# 複製nginx配置
COPY nginx.conf /etc/nginx/nginx.conf

# 健康檢查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/ || exit 1

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### **後端Dockerfile**
```dockerfile
# Dockerfile.backend
FROM rust:1.75-alpine AS builder

RUN apk add --no-cache musl-dev pkgconfig openssl-dev

WORKDIR /app

# 複製Cargo文件
COPY Cargo.toml Cargo.lock ./
COPY shared/ ./shared/
COPY backend/ ./backend/

# 建置應用
WORKDIR /app/backend
RUN cargo build --release

# 生產階段
FROM alpine:latest

RUN apk add --no-cache ca-certificates tzdata

WORKDIR /app

# 創建應用用戶
RUN addgroup -g 1000 coffee && \
    adduser -D -s /bin/sh -u 1000 -G coffee coffee

# 複製二進制文件
COPY --from=builder /app/backend/target/release/coffee-backend ./

# 設置權限
RUN chown coffee:coffee coffee-backend

USER coffee

# 健康檢查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080

CMD ["./coffee-backend"]
```

---

## ☸️ **Kubernetes部署配置**

### **命名空間配置**
```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: coffee-dev
  labels:
    name: coffee-dev
    environment: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: coffee-test
  labels:
    name: coffee-test
    environment: test
---
apiVersion: v1
kind: Namespace
metadata:
  name: coffee-pp
  labels:
    name: coffee-pp
    environment: pp
---
apiVersion: v1
kind: Namespace
metadata:
  name: coffee-prod
  labels:
    name: coffee-prod
    environment: prod
```

### **前端部署配置**
```yaml
# k8s/frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coffee-frontend
  namespace: coffee-dev
  labels:
    app: coffee-frontend
    component: frontend
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: coffee-frontend
  template:
    metadata:
      labels:
        app: coffee-frontend
        component: frontend
        version: v1
    spec:
      containers:
      - name: frontend
        image: ghcr.io/lucas152112/coffee/frontend:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: coffee-frontend-service
  namespace: coffee-dev
spec:
  selector:
    app: coffee-frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

### **後端微服務部署配置**
```yaml
# k8s/backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coffee-backend
  namespace: coffee-dev
  labels:
    app: coffee-backend
    component: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: coffee-backend
  template:
    metadata:
      labels:
        app: coffee-backend
        component: backend
    spec:
      containers:
      - name: auth-service
        image: ghcr.io/lucas152112/coffee/auth-service:latest
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: coffee-secrets
              key: database-url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: coffee-secrets
              key: redis-url
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: coffee-backend-service
  namespace: coffee-dev
spec:
  selector:
    app: coffee-backend
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
  type: ClusterIP
```

---

## 📊 **監控與日誌配置**

### **Prometheus監控配置**
```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "coffee-alerts.yml"

scrape_configs:
  - job_name: 'coffee-frontend'
    static_configs:
      - targets: ['coffee-frontend-service:80']
    metrics_path: /metrics
    scrape_interval: 30s

  - job_name: 'coffee-backend'
    static_configs:
      - targets: ['coffee-backend-service:8080']
    metrics_path: /metrics
    scrape_interval: 15s

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

### **告警規則配置**
```yaml
# monitoring/coffee-alerts.yml
groups:
  - name: coffee-system
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "高錯誤率告警"
          description: "{{ $labels.instance }} 錯誤率超過 10%"

      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 0.2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "高延遲告警"
          description: "{{ $labels.instance }} P95延遲超過 200ms"

      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "服務離線告警"
          description: "{{ $labels.instance }} 服務已離線"
```

---

## ✅ **階段4交付物檢查清單**

### **CI/CD流水線** 
- [ ] GitHub Actions工作流程配置完成
- [ ] 多環境自動部署流程建立
- [ ] 容器映像自動建置與推送
- [ ] 自動化測試整合完成

### **跨平台App打包**
- [ ] Web PWA離線功能完成
- [ ] Android APK自動打包
- [ ] iOS App自動建置
- [ ] Windows/macOS/Linux桌面版打包
- [ ] 應用商店發布準備完成

### **容器化部署**
- [ ] 所有服務Docker化完成
- [ ] Kubernetes部署配置完成
- [ ] 多環境部署腳本完成
- [ ] 監控日誌系統整合完成

### **自動化測試**
- [ ] 單元測試自動化執行
- [ ] 整合測試自動化執行  
- [ ] 端到端測試自動化執行
- [ ] 效能測試自動化執行

---

**📋 階段4完成**: 芊芊AI助手  
**📅 完成日期**: 2026-02-12 15:05  
**🎯 符合標準**: 系統開發基本規範階段4  
**⚠️ 審核狀態**: 待老大審核確認進入階段5

**🌱 老大，階段4 CI/CD與App打包配置已完成！包含完整的自動化部署流程和6平台App打包方案。請審核是否可以進入階段5實際開發？**