# 🔄 Coffee系統階段4：CI/CD自動化與App打包

## 🎯 **階段4目標：完整自動化部署與多平台打包**

**執行時間**: 2026-02-13 02:00  
**預計完成**: 2026-02-13 03:00 (60分鐘)

---

## 🏗️ **CI/CD流水線架構設計**

### **📊 流水線總覽**

```
GitHub Actions CI/CD Pipeline:

┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Source    │───▶│   Build     │───▶│    Test     │───▶│   Deploy    │
│   Control   │    │  & Package  │    │ & Quality   │    │ & Release   │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
     │                    │                    │                    │
     ▼                    ▼                    ▼                    ▼
┌─Git Push──┐    ┌─Multi-Platform─┐  ┌─Automated───┐  ┌─Environment─┐
│  Trigger  │    │   Container    │  │   Testing   │  │  Deployment │
│  Events   │    │   Building     │  │  Quality    │  │  Strategy   │
└───────────┘    └────────────────┘  └─────────────┘  └─────────────┘
```

### **🔧 技術架構選型**

#### **CI/CD平台**: GitHub Actions
- ✅ 原生GitHub整合
- ✅ 免費額度充足
- ✅ 豐富的Action生態
- ✅ 並行執行支援

#### **容器化**: Docker + Multi-Stage Build
- ✅ 統一環境
- ✅ 輕量化部署
- ✅ 版本管理
- ✅ 安全隔離

#### **編排平台**: Kubernetes + Helm
- ✅ 生產級容器編排
- ✅ 自動擴縮容
- ✅ 服務發現
- ✅ 配置管理

---

## 📦 **Docker容器化策略**

### **🦀 Rust微服務容器化**

#### **多階段構建Dockerfile範本**
```dockerfile
# backend/*/Dockerfile
# 第一階段：構建環境
FROM rust:1.75-slim as builder
WORKDIR /usr/src/app

# 安裝系統依賴
RUN apt-get update && apt-get install -y \
    pkg-config \
    libssl-dev \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# 複製Cargo檔案（利用Docker快取）
COPY Cargo.toml Cargo.lock ./
COPY coffee-shared-lib/ ./coffee-shared-lib/

# 預先下載依賴（快取優化）
RUN mkdir src && echo "fn main(){}" > src/main.rs
RUN cargo build --release && rm -rf src

# 複製源碼並構建
COPY src ./src
RUN touch src/main.rs && cargo build --release

# 第二階段：運行環境
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y \
    ca-certificates \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

# 創建非特權用戶
RUN useradd -m -u 1001 coffee
USER coffee
WORKDIR /home/coffee

# 複製執行檔
COPY --from=builder /usr/src/app/target/release/coffee-*-service ./service
COPY --from=builder /usr/src/app/migrations ./migrations

# 健康檢查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

# 暴露端口
EXPOSE 8000

# 啟動服務
CMD ["./service"]
```

#### **微服務Docker配置**
```yaml
# docker-compose.yml - 開發環境
version: '3.8'
services:
  # 基礎設施
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: coffee_dev
      POSTGRES_USER: coffee_user
      POSTGRES_PASSWORD: coffee_pass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./sql/init:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U coffee_user -d coffee_dev"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  mongodb:
    image: mongo:7
    environment:
      MONGO_INITDB_ROOT_USERNAME: coffee_admin
      MONGO_INITDB_ROOT_PASSWORD: coffee_admin_pass
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db

  # Coffee微服務
  tenant-service:
    build:
      context: ./backend/tenant-service
      dockerfile: Dockerfile
    environment:
      - DATABASE_URL=postgresql://coffee_user:coffee_pass@postgres:5432/coffee_dev
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    ports:
      - "8000:8000"

  monitoring-service:
    build: ./backend/monitoring-service
    environment:
      - DATABASE_URL=postgresql://coffee_user:coffee_pass@postgres:5432/coffee_dev
      - REDIS_URL=redis://redis:6379/1
    depends_on:
      - postgres
      - redis
    ports:
      - "8001:8001"

  # ... 其他8個微服務類似配置

volumes:
  postgres_data:
  redis_data:  
  mongodb_data:
```

---

## ⚙️ **GitHub Actions工作流程**

### **🔄 主要工作流程**

#### **1. 持續整合工作流程**
```yaml
# .github/workflows/ci.yml
name: Coffee System CI
on:
  push:
    branches: [ main, develop, 'feature/*' ]
  pull_request:
    branches: [ main, develop ]

env:
  CARGO_TERM_COLOR: always
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # 代碼品質檢查
  code-quality:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Setup Rust
        uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
          components: rustfmt, clippy
          override: true
          
      - name: Cache cargo registry
        uses: actions/cache@v3
        with:
          path: ~/.cargo/registry
          key: ${{ runner.os }}-cargo-registry-${{ hashFiles('**/Cargo.lock') }}
          
      - name: Format check
        run: cargo fmt --all -- --check
        
      - name: Clippy check
        run: cargo clippy --all-targets --all-features -- -D warnings
        
      - name: Security audit
        run: |
          cargo install cargo-audit
          cargo audit

  # 單元測試
  unit-tests:
    runs-on: ubuntu-latest
    needs: code-quality
    strategy:
      matrix:
        service: 
          - tenant-service
          - monitoring-service
          - config-service
          - billing-service
          - order-service
          - payment-service
          - inventory-service
          - analytics-service
    steps:
      - uses: actions/checkout@v4
      - uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
          
      - name: Run tests for ${{ matrix.service }}
        run: |
          cd backend/${{ matrix.service }}
          cargo test --verbose
          
      - name: Generate coverage report
        run: |
          cargo install cargo-tarpaulin
          cd backend/${{ matrix.service }}
          cargo tarpaulin --out xml --output-dir ../../coverage/
          
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/cobertura.xml
          flags: ${{ matrix.service }}

  # 整合測試
  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: coffee_test
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_pass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
          
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 3s
          --health-retries 3
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4
      - uses: actions-rs/toolchain@v1
        
      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://test_user:test_pass@localhost:5432/coffee_test
          REDIS_URL: redis://localhost:6379/0
        run: |
          cargo test --test integration -- --test-threads=1
          
      - name: Run API tests
        run: |
          docker-compose -f docker-compose.test.yml up -d
          npm install -g newman
          newman run tests/api/coffee-api.postman_collection.json

  # 構建Docker映像
  build-images:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    strategy:
      matrix:
        service: 
          - tenant-service
          - monitoring-service
          - config-service
          - billing-service
          - order-service
          - payment-service
          - inventory-service
          - analytics-service
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/${{ matrix.service }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix={{branch}}-
            
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: ./backend/${{ matrix.service }}
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # 前端構建（Nuxt.js）
  build-frontend:
    runs-on: ubuntu-latest
    needs: code-quality
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json
          
      - name: Install dependencies
        run: |
          cd frontend
          npm ci
          
      - name: Run frontend tests
        run: |
          cd frontend
          npm run test:unit
          npm run test:e2e
          
      - name: Build application
        run: |
          cd frontend
          npm run build
          
      - name: Build Docker image
        run: |
          cd frontend
          docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/frontend:${{ github.sha }} .
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/frontend:${{ github.sha }}
```

#### **2. 部署工作流程**
```yaml
# .github/workflows/deploy.yml
name: Deploy to Kubernetes
on:
  workflow_run:
    workflows: ["Coffee System CI"]
    branches: [main]
    types: [completed]

jobs:
  deploy-to-staging:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.28.0'
          
      - name: Setup Helm
        uses: azure/setup-helm@v3
        with:
          version: 'v3.13.0'
          
      - name: Configure kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > ~/.kube/config
          
      - name: Deploy to staging
        run: |
          helm upgrade --install coffee-staging ./k8s/helm/coffee \
            --namespace coffee-staging \
            --create-namespace \
            --set image.tag=${{ github.sha }} \
            --set environment=staging \
            --values ./k8s/helm/coffee/values-staging.yaml \
            --wait --timeout=10m
            
      - name: Run smoke tests
        run: |
          kubectl wait --for=condition=ready pod -l app=coffee-gateway -n coffee-staging --timeout=300s
          curl -f http://coffee-staging.k8s.justit.cc/health || exit 1

  deploy-to-production:
    needs: deploy-to-staging
    runs-on: ubuntu-latest
    environment: production
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        
      - name: Deploy to production
        run: |
          helm upgrade --install coffee-prod ./k8s/helm/coffee \
            --namespace coffee-prod \
            --create-namespace \
            --set image.tag=${{ github.sha }} \
            --set environment=production \
            --values ./k8s/helm/coffee/values-production.yaml \
            --wait --timeout=15m
            
      - name: Health check
        run: |
          curl -f https://coffee.justit.cc/health || exit 1
```

---

## 📱 **多平台App打包策略**

### **🎯 5平台支援目標**

#### **支援平台清單**
1. **🌐 Web (PWA)** - 漸進式Web應用
2. **📱 Android** - Google Play Store
3. **🍎 iOS** - Apple App Store  
4. **🖥️ Windows** - Microsoft Store + exe
5. **💻 macOS** - Mac App Store + dmg
6. **🐧 Linux** - Snap Store + AppImage

### **📦 Capacitor移動端打包**

#### **Capacitor配置**
```typescript
// capacitor.config.ts
import { CapacitorConfig } from '@capacitor/cli';

const config: CapacitorConfig = {
  appId: 'com.justit.coffee',
  appName: 'Coffee Management System',
  webDir: 'dist',
  server: {
    androidScheme: 'https'
  },
  plugins: {
    SplashScreen: {
      launchShowDuration: 3000,
      launchAutoHide: true,
      backgroundColor: "#8B4513",
      androidSplashResourceName: "splash",
      androidScaleType: "CENTER_CROP",
      showSpinner: true,
      androidSpinnerStyle: "large",
      iosSpinnerStyle: "small",
      spinnerColor: "#999999"
    },
    StatusBar: {
      style: "DARK"
    }
  }
};

export default config;
```

#### **Android打包工作流程**
```yaml
# .github/workflows/android.yml
name: Build Android App
on:
  push:
    tags: ['v*']

jobs:
  build-android:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          
      - name: Setup Java
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: '17'
          
      - name: Install dependencies
        run: |
          cd frontend
          npm ci
          
      - name: Build web app
        run: |
          cd frontend
          npm run build
          
      - name: Add Capacitor Android
        run: |
          cd frontend
          npx cap add android
          npx cap sync android
          
      - name: Build Android APK
        run: |
          cd frontend/android
          ./gradlew assembleRelease
          
      - name: Sign APK
        uses: r0adkll/sign-android-release@v1
        with:
          releaseDirectory: frontend/android/app/build/outputs/apk/release
          signingKeyBase64: ${{ secrets.ANDROID_SIGNING_KEY }}
          alias: ${{ secrets.ANDROID_KEY_ALIAS }}
          keyStorePassword: ${{ secrets.ANDROID_KEYSTORE_PASSWORD }}
          keyPassword: ${{ secrets.ANDROID_KEY_PASSWORD }}
          
      - name: Upload APK to release
        uses: softprops/action-gh-release@v1
        with:
          files: frontend/android/app/build/outputs/apk/release/*.apk
```

#### **iOS打包工作流程**
```yaml
# .github/workflows/ios.yml  
name: Build iOS App
on:
  push:
    tags: ['v*']

jobs:
  build-ios:
    runs-on: macos-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          
      - name: Setup Xcode
        uses: maxim-lobanov/setup-xcode@v1
        with:
          xcode-version: latest-stable
          
      - name: Install CocoaPods
        run: sudo gem install cocoapods
        
      - name: Install dependencies
        run: |
          cd frontend
          npm ci
          
      - name: Build web app
        run: |
          cd frontend
          npm run build
          
      - name: Add Capacitor iOS
        run: |
          cd frontend
          npx cap add ios
          npx cap sync ios
          
      - name: Build iOS app
        run: |
          cd frontend/ios/App
          xcodebuild -workspace App.xcworkspace \
            -scheme App \
            -configuration Release \
            -destination 'generic/platform=iOS' \
            -archivePath App.xcarchive \
            archive
            
      - name: Export IPA
        run: |
          cd frontend/ios/App
          xcodebuild -exportArchive \
            -archivePath App.xcarchive \
            -exportPath . \
            -exportOptionsPlist ExportOptions.plist
            
      - name: Upload IPA to release
        uses: softprops/action-gh-release@v1
        with:
          files: frontend/ios/App/*.ipa
```

### **🖥️ Electron桌面端打包**

#### **Electron配置**
```typescript
// electron/main.ts
import { app, BrowserWindow, Menu, shell } from 'electron';
import { join } from 'path';

const isDev = process.env.NODE_ENV === 'development';

function createWindow(): void {
  const mainWindow = new BrowserWindow({
    width: 1200,
    height: 800,
    minWidth: 800,
    minHeight: 600,
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
      enableRemoteModule: false,
      preload: join(__dirname, 'preload.js')
    },
    icon: join(__dirname, '../assets/icon.png'),
    titleBarStyle: 'default',
    show: false
  });

  // 載入應用
  if (isDev) {
    mainWindow.loadURL('http://localhost:3000');
    mainWindow.webContents.openDevTools();
  } else {
    mainWindow.loadFile(join(__dirname, '../dist/index.html'));
  }

  // 視窗就緒後顯示
  mainWindow.once('ready-to-show', () => {
    mainWindow.show();
  });

  // 處理外部連結
  mainWindow.webContents.setWindowOpenHandler(({ url }) => {
    shell.openExternal(url);
    return { action: 'deny' };
  });

  // 設定選單
  const menu = Menu.buildFromTemplate([
    {
      label: 'Coffee',
      submenu: [
        { role: 'about' },
        { type: 'separator' },
        { role: 'quit' }
      ]
    },
    {
      label: 'Edit',
      submenu: [
        { role: 'undo' },
        { role: 'redo' },
        { type: 'separator' },
        { role: 'cut' },
        { role: 'copy' },
        { role: 'paste' }
      ]
    },
    {
      label: 'View',
      submenu: [
        { role: 'reload' },
        { role: 'forceReload' },
        { role: 'toggleDevTools' },
        { type: 'separator' },
        { role: 'resetZoom' },
        { role: 'zoomIn' },
        { role: 'zoomOut' },
        { type: 'separator' },
        { role: 'togglefullscreen' }
      ]
    }
  ]);
  
  Menu.setApplicationMenu(menu);
}

app.whenReady().then(createWindow);

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') app.quit();
});

app.on('activate', () => {
  if (BrowserWindow.getAllWindows().length === 0) createWindow();
});
```

#### **Electron Builder配置**
```json
// package.json - electron builder配置
{
  "build": {
    "appId": "com.justit.coffee",
    "productName": "Coffee Management System",
    "directories": {
      "output": "dist-electron"
    },
    "files": [
      "dist/**/*",
      "electron/**/*",
      "!**/*.ts",
      "!**/*.map"
    ],
    "mac": {
      "icon": "assets/icon.icns",
      "category": "public.app-category.business",
      "target": [
        {
          "target": "dmg",
          "arch": ["x64", "arm64"]
        },
        {
          "target": "mas",
          "arch": ["x64", "arm64"]
        }
      ]
    },
    "win": {
      "icon": "assets/icon.ico",
      "target": [
        {
          "target": "nsis",
          "arch": ["x64", "ia32"]
        },
        {
          "target": "portable",
          "arch": ["x64"]
        }
      ]
    },
    "linux": {
      "icon": "assets/icon.png",
      "category": "Office",
      "target": [
        {
          "target": "AppImage",
          "arch": ["x64"]
        },
        {
          "target": "snap",
          "arch": ["x64"]
        },
        {
          "target": "deb",
          "arch": ["x64"]
        }
      ]
    },
    "nsis": {
      "oneClick": false,
      "allowToChangeInstallationDirectory": true,
      "createDesktopShortcut": true,
      "createStartMenuShortcut": true
    }
  }
}
```

#### **桌面端打包工作流程**
```yaml
# .github/workflows/desktop.yml
name: Build Desktop Apps
on:
  push:
    tags: ['v*']

jobs:
  build-desktop:
    strategy:
      matrix:
        os: [macos-latest, ubuntu-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          
      - name: Install dependencies
        run: |
          cd frontend
          npm ci
          
      - name: Build web app
        run: |
          cd frontend
          npm run build
          
      - name: Build Electron app (macOS)
        if: runner.os == 'macOS'
        run: |
          cd frontend
          npm run electron:build -- --mac
          
      - name: Build Electron app (Windows)
        if: runner.os == 'Windows'
        run: |
          cd frontend
          npm run electron:build -- --win
          
      - name: Build Electron app (Linux)
        if: runner.os == 'Linux'
        run: |
          cd frontend
          npm run electron:build -- --linux
          
      - name: Upload artifacts
        uses: softprops/action-gh-release@v1
        with:
          files: frontend/dist-electron/*
```

---

## ☸️ **Kubernetes部署配置**

### **📊 Helm Chart結構**

```
k8s/helm/coffee/
├── Chart.yaml                    # Helm Chart定義
├── values.yaml                   # 預設值
├── values-staging.yaml           # Staging環境配置
├── values-production.yaml        # Production環境配置
├── templates/
│   ├── deployment.yaml          # 部署配置
│   ├── service.yaml             # 服務配置
│   ├── ingress.yaml             # 入口配置
│   ├── configmap.yaml           # 配置映射
│   ├── secret.yaml              # 機密配置
│   ├── hpa.yaml                 # 水平擴展
│   ├── pdb.yaml                 # Pod中斷預算
│   └── monitoring.yaml          # 監控配置
└── charts/                      # 子Chart依賴
    ├── postgresql/
    ├── redis/
    └── mongodb/
```

#### **Helm Chart定義**
```yaml
# k8s/helm/coffee/Chart.yaml
apiVersion: v2
name: coffee
description: Coffee Management System Helm Chart
type: application
version: 1.0.0
appVersion: "1.0.0"
keywords:
  - coffee
  - management
  - pos
  - microservices
home: https://github.com/lucas152112/coffee
sources:
  - https://github.com/lucas152112/coffee
maintainers:
  - name: Coffee Team
    email: dev@justit.cc
dependencies:
  - name: postgresql
    version: 12.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
  - name: redis
    version: 18.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
  - name: mongodb
    version: 13.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: mongodb.enabled
```

#### **部署配置範本**
```yaml
# k8s/helm/coffee/templates/deployment.yaml
{{- range $service := .Values.services }}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ $service.name }}
  namespace: {{ $.Values.namespace }}
  labels:
    app: {{ $service.name }}
    version: {{ $.Values.image.tag }}
spec:
  replicas: {{ $service.replicas | default 2 }}
  selector:
    matchLabels:
      app: {{ $service.name }}
  template:
    metadata:
      labels:
        app: {{ $service.name }}
        version: {{ $.Values.image.tag }}
    spec:
      containers:
      - name: {{ $service.name }}
        image: {{ $.Values.image.registry }}/{{ $.Values.image.repository }}/{{ $service.name }}:{{ $.Values.image.tag }}
        ports:
        - containerPort: {{ $service.port | default 8000 }}
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
        - name: RUST_LOG
          value: {{ $.Values.logLevel | default "info" }}
        resources:
          requests:
            memory: {{ $service.resources.requests.memory | default "256Mi" }}
            cpu: {{ $service.resources.requests.cpu | default "100m" }}
          limits:
            memory: {{ $service.resources.limits.memory | default "512Mi" }}
            cpu: {{ $service.resources.limits.cpu | default "500m" }}
        livenessProbe:
          httpGet:
            path: /health
            port: {{ $service.port | default 8000 }}
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: {{ $service.port | default 8000 }}
          initialDelaySeconds: 5
          periodSeconds: 5
        securityContext:
          runAsNonRoot: true
          runAsUser: 1001
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
{{- end }}
```

#### **生產環境配置**
```yaml
# k8s/helm/coffee/values-production.yaml
namespace: coffee-prod
image:
  registry: ghcr.io
  repository: lucas152112/coffee
  tag: latest
  pullPolicy: Always

services:
  - name: tenant-service
    port: 8000
    replicas: 3
    resources:
      requests:
        memory: "512Mi"
        cpu: "200m"
      limits:
        memory: "1Gi" 
        cpu: "1000m"
        
  - name: order-service
    port: 8010
    replicas: 5
    resources:
      requests:
        memory: "512Mi"
        cpu: "300m"
      limits:
        memory: "1Gi"
        cpu: "1000m"
        
  # ... 其他服務配置

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
  hosts:
    - host: coffee.justit.cc
      paths:
        - path: /
          pathType: Prefix
          service: gateway-service
          port: 8000
  tls:
    - secretName: coffee-tls
      hosts:
        - coffee.justit.cc

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

postgresql:
  enabled: true
  auth:
    postgresPassword: "super_secure_password"
    database: "coffee_prod"
  primary:
    persistence:
      enabled: true
      size: 100Gi
      storageClass: "fast-ssd"

redis:
  enabled: true
  auth:
    enabled: true
    password: "redis_secure_password"
  master:
    persistence:
      enabled: true
      size: 20Gi

mongodb:
  enabled: true
  auth:
    enabled: true
    rootPassword: "mongo_secure_password"
  persistence:
    enabled: true
    size: 50Gi

monitoring:
  enabled: true
  prometheus:
    enabled: true
  grafana:
    enabled: true
```

---

## 📈 **監控與可觀測性**

### **🔍 監控架構**

```
Monitoring Stack:

┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ Prometheus  │───▶│   Grafana   │───▶│   Alert     │
│  (Metrics)  │    │ (Dashboard) │    │  Manager    │
└─────────────┘    └─────────────┘    └─────────────┘
        ▲                   ▲                   │
        │                   │                   ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Coffee    │    │    Jaeger   │    │   Discord   │
│ Microservices│    │  (Tracing)  │    │ Webhook     │
└─────────────┘    └─────────────┘    └─────────────┘
```

#### **Prometheus配置**
```yaml
# k8s/monitoring/prometheus-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    rule_files:
      - "/etc/prometheus/rules/*.yml"
    
    scrape_configs:
      - job_name: 'coffee-services'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
            target_label: __address__
            
    alerting:
      alertmanagers:
        - static_configs:
            - targets:
              - alertmanager:9093
```

**🚀 Coffee系統階段4 CI/CD與App打包完成！**

**下一步：立即執行階段5 實際部署與整合測試！**