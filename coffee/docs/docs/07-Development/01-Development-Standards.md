# 開發標準文檔 (Development Standards)

## 1. 總體開發標準

### 1.1 開發原則
- **代碼品質**: 優先考慮代碼的可讀性、可維護性和可擴展性
- **性能優先**: 確保系統響應時間和吞吐量符合業務需求
- **安全第一**: 所有開發都必須考慮安全性和資料保護
- **測試驅動**: 採用 TDD 方法，確保代碼品質和功能完整性

### 1.2 技術棧標準
```yaml
後端技術棧:
  語言: Go 1.21+
  框架: Gin 1.9+
  資料庫: MySQL 8.0+
  快取: Redis 7.0+
  消息隊列: NATS 2.9+
  容器化: Docker 24.0+

前端技術棧:
  語言: TypeScript 5.0+
  框架: Vue 3.3+ / Nuxt 3.7+
  UI框架: Element Plus / Tailwind CSS
  狀態管理: Pinia
  建置工具: Vite 4.4+

開發工具:
  代碼管理: Git
  CI/CD: GitHub Actions
  代碼品質: SonarQube
  API文檔: Swagger/OpenAPI 3.0
  監控: Prometheus + Grafana
```

---

## 2. Go 後端開發標準

### 2.1 專案結構標準
```
service-name/
├── cmd/
│   └── server/
│       └── main.go              # 應用程式入口
├── internal/
│   ├── config/                  # 配置管理
│   ├── handler/                 # HTTP 處理器
│   ├── service/                 # 業務邏輯
│   ├── repository/              # 數據存取層
│   ├── model/                   # 數據模型
│   ├── middleware/              # 中間件
│   └── utils/                   # 工具函數
├── pkg/                         # 可共享的包
├── api/                         # API 定義文件
├── scripts/                     # 腳本文件
├── deployments/                 # 部署配置
├── docs/                        # 文檔
├── go.mod
├── go.sum
├── Dockerfile
└── README.md
```

### 2.2 代碼規範
```go
// 包命名：使用小寫，避免下劃線
package handler

// 常量命名：使用大駝峰
const (
    DefaultTimeout = 30 * time.Second
    MaxRetryCount  = 3
)

// 變數命名：使用小駝峰
var (
    dbConnection *sql.DB
    configPath   string
)

// 函數命名：公開函數使用大駝峰，私有函數使用小駝峰
func CreateCustomer(customer *model.Customer) error {
    return createCustomerInDB(customer)
}

func createCustomerInDB(customer *model.Customer) error {
    // 實作
}

// 結構體命名：使用大駝峰
type CustomerService struct {
    repo CustomerRepository
    log  *logrus.Logger
}

// 介面命名：使用大駝峰，通常以 -er 結尾
type CustomerRepository interface {
    Create(customer *model.Customer) error
    GetByID(id string) (*model.Customer, error)
    Update(customer *model.Customer) error
    Delete(id string) error
}
```

### 2.3 錯誤處理標準
```go
// 定義業務錯誤
package errors

import "errors"

var (
    ErrCustomerNotFound     = errors.New("customer not found")
    ErrInvalidPhoneNumber   = errors.New("invalid phone number")
    ErrDuplicateCustomer    = errors.New("customer already exists")
)

// 錯誤包裝
func (s *CustomerService) GetCustomer(id string) (*model.Customer, error) {
    customer, err := s.repo.GetByID(id)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, ErrCustomerNotFound
        }
        return nil, fmt.Errorf("failed to get customer: %w", err)
    }
    return customer, nil
}

// HTTP 錯誤回應
func (h *CustomerHandler) GetCustomer(c *gin.Context) {
    id := c.Param("id")
    customer, err := h.service.GetCustomer(id)
    if err != nil {
        switch {
        case errors.Is(err, ErrCustomerNotFound):
            c.JSON(http.StatusNotFound, gin.H{"error": "Customer not found"})
        default:
            h.log.Error("Failed to get customer", "error", err)
            c.JSON(http.StatusInternalServerError, gin.H{"error": "Internal server error"})
        }
        return
    }
    c.JSON(http.StatusOK, customer)
}
```

### 2.4 資料庫操作標準
```go
// Repository 實作範例
type customerRepository struct {
    db *sql.DB
}

func (r *customerRepository) Create(customer *model.Customer) error {
    query := `
        INSERT INTO customers (id, name, phone, email, created_at) 
        VALUES (?, ?, ?, ?, ?)
    `
    
    _, err := r.db.Exec(query, 
        customer.ID, 
        customer.Name, 
        customer.Phone, 
        customer.Email, 
        customer.CreatedAt,
    )
    if err != nil {
        return fmt.Errorf("failed to create customer: %w", err)
    }
    
    return nil
}

func (r *customerRepository) GetByID(id string) (*model.Customer, error) {
    query := `
        SELECT id, name, phone, email, created_at, updated_at 
        FROM customers 
        WHERE id = ?
    `
    
    row := r.db.QueryRow(query, id)
    
    var customer model.Customer
    err := row.Scan(
        &customer.ID,
        &customer.Name,
        &customer.Phone,
        &customer.Email,
        &customer.CreatedAt,
        &customer.UpdatedAt,
    )
    if err != nil {
        return nil, err
    }
    
    return &customer, nil
}

// 事務處理
func (s *CustomerService) TransferLoyaltyPoints(fromID, toID string, points int) error {
    tx, err := s.db.Begin()
    if err != nil {
        return fmt.Errorf("failed to begin transaction: %w", err)
    }
    defer tx.Rollback()

    // 扣除來源客戶點數
    if err := s.deductPoints(tx, fromID, points); err != nil {
        return err
    }

    // 增加目標客戶點數
    if err := s.addPoints(tx, toID, points); err != nil {
        return err
    }

    if err := tx.Commit(); err != nil {
        return fmt.Errorf("failed to commit transaction: %w", err)
    }

    return nil
}
```

### 2.5 配置管理標準
```go
// config/config.go
type Config struct {
    Server   ServerConfig   `yaml:"server"`
    Database DatabaseConfig `yaml:"database"`
    Redis    RedisConfig    `yaml:"redis"`
    JWT      JWTConfig      `yaml:"jwt"`
}

type ServerConfig struct {
    Port         int           `yaml:"port"`
    ReadTimeout  time.Duration `yaml:"read_timeout"`
    WriteTimeout time.Duration `yaml:"write_timeout"`
}

type DatabaseConfig struct {
    Host         string        `yaml:"host"`
    Port         int           `yaml:"port"`
    Username     string        `yaml:"username"`
    Password     string        `yaml:"password"`
    Database     string        `yaml:"database"`
    MaxOpenConns int           `yaml:"max_open_conns"`
    MaxIdleConns int           `yaml:"max_idle_conns"`
    MaxLifetime  time.Duration `yaml:"max_lifetime"`
}

func LoadConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("failed to read config file: %w", err)
    }

    var config Config
    if err := yaml.Unmarshal(data, &config); err != nil {
        return nil, fmt.Errorf("failed to unmarshal config: %w", err)
    }

    return &config, nil
}
```

### 2.6 日誌標準
```go
// 使用結構化日誌
import "github.com/sirupsen/logrus"

func setupLogger() *logrus.Logger {
    log := logrus.New()
    log.SetFormatter(&logrus.JSONFormatter{})
    log.SetLevel(logrus.InfoLevel)
    
    return log
}

// 日誌使用範例
func (h *CustomerHandler) CreateCustomer(c *gin.Context) {
    var req CreateCustomerRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        h.log.WithFields(logrus.Fields{
            "error":   err.Error(),
            "request": req,
        }).Error("Invalid request body")
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid request"})
        return
    }

    customer, err := h.service.CreateCustomer(&req)
    if err != nil {
        h.log.WithFields(logrus.Fields{
            "error":   err.Error(),
            "request": req,
        }).Error("Failed to create customer")
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Internal server error"})
        return
    }

    h.log.WithFields(logrus.Fields{
        "customer_id": customer.ID,
        "name":        customer.Name,
    }).Info("Customer created successfully")

    c.JSON(http.StatusCreated, customer)
}
```

---

## 3. Vue.js 前端開發標準

### 3.1 專案結構標準
```
webapp/
├── assets/                      # 靜態資源
│   ├── css/
│   ├── images/
│   └── fonts/
├── components/                  # 共用元件
│   ├── common/                  # 基礎元件
│   ├── forms/                   # 表單元件
│   └── charts/                  # 圖表元件
├── composables/                 # 組合式函數
├── layouts/                     # 佈局元件
├── middleware/                  # 中間件
├── pages/                       # 頁面元件
├── plugins/                     # 插件
├── server/                      # 伺服器端邏輯
├── stores/                      # 狀態管理
├── types/                       # TypeScript 型別定義
├── utils/                       # 工具函數
├── nuxt.config.ts              # Nuxt 配置
├── package.json
└── tsconfig.json
```

### 3.2 元件開發標準
```vue
<!-- CustomerCard.vue -->
<template>
  <div class="customer-card">
    <div class="customer-card__header">
      <h3 class="customer-card__name">{{ customer.name }}</h3>
      <CustomerTierBadge :tier="customer.tier" />
    </div>
    
    <div class="customer-card__info">
      <p class="customer-card__phone">{{ customer.phone }}</p>
      <p class="customer-card__email">{{ customer.email }}</p>
    </div>
    
    <div class="customer-card__actions">
      <BaseButton 
        variant="primary" 
        @click="$emit('edit', customer)"
      >
        編輯
      </BaseButton>
      <BaseButton 
        variant="secondary" 
        @click="$emit('view', customer)"
      >
        查看詳情
      </BaseButton>
    </div>
  </div>
</template>

<script setup lang="ts">
import type { Customer } from '~/types/customer'

// Props 定義
interface Props {
  customer: Customer
}

const props = defineProps<Props>()

// Events 定義
interface Emits {
  edit: [customer: Customer]
  view: [customer: Customer]
}

const emit = defineEmits<Emits>()
</script>

<style lang="scss" scoped>
.customer-card {
  @apply bg-white rounded-lg shadow-md p-4 border border-gray-200;
  
  &__header {
    @apply flex justify-between items-center mb-3;
  }
  
  &__name {
    @apply text-lg font-semibold text-gray-900;
  }
  
  &__info {
    @apply space-y-2 mb-4;
  }
  
  &__phone,
  &__email {
    @apply text-sm text-gray-600;
  }
  
  &__actions {
    @apply flex gap-2 justify-end;
  }
}
</style>
```

### 3.3 狀態管理標準
```typescript
// stores/customer.ts
import { defineStore } from 'pinia'
import type { Customer, CreateCustomerRequest, UpdateCustomerRequest } from '~/types/customer'

interface CustomerState {
  customers: Customer[]
  currentCustomer: Customer | null
  loading: boolean
  error: string | null
}

export const useCustomerStore = defineStore('customer', {
  state: (): CustomerState => ({
    customers: [],
    currentCustomer: null,
    loading: false,
    error: null,
  }),

  getters: {
    getCustomerById: (state) => (id: string) => {
      return state.customers.find(customer => customer.id === id)
    },

    totalCustomers: (state) => state.customers.length,

    activeCustomers: (state) => {
      return state.customers.filter(customer => customer.status === 'active')
    },
  },

  actions: {
    async fetchCustomers() {
      this.loading = true
      this.error = null

      try {
        const { data } = await $fetch<{ data: Customer[] }>('/api/customers')
        this.customers = data
      } catch (error) {
        this.error = 'Failed to fetch customers'
        console.error('Error fetching customers:', error)
      } finally {
        this.loading = false
      }
    },

    async createCustomer(customerData: CreateCustomerRequest) {
      this.loading = true
      this.error = null

      try {
        const { data } = await $fetch<{ data: Customer }>('/api/customers', {
          method: 'POST',
          body: customerData,
        })
        
        this.customers.push(data)
        return data
      } catch (error) {
        this.error = 'Failed to create customer'
        throw error
      } finally {
        this.loading = false
      }
    },

    async updateCustomer(id: string, customerData: UpdateCustomerRequest) {
      this.loading = true
      this.error = null

      try {
        const { data } = await $fetch<{ data: Customer }>(`/api/customers/${id}`, {
          method: 'PUT',
          body: customerData,
        })

        const index = this.customers.findIndex(c => c.id === id)
        if (index !== -1) {
          this.customers[index] = data
        }

        if (this.currentCustomer?.id === id) {
          this.currentCustomer = data
        }

        return data
      } catch (error) {
        this.error = 'Failed to update customer'
        throw error
      } finally {
        this.loading = false
      }
    },
  },
})
```

### 3.4 Composables 標準
```typescript
// composables/useApi.ts
interface ApiOptions {
  method?: 'GET' | 'POST' | 'PUT' | 'DELETE'
  body?: any
  headers?: Record<string, string>
}

interface ApiResponse<T> {
  data: T
  message?: string
  status: number
}

export const useApi = () => {
  const loading = ref(false)
  const error = ref<string | null>(null)

  const execute = async <T>(url: string, options: ApiOptions = {}): Promise<T> => {
    loading.value = true
    error.value = null

    try {
      const response = await $fetch<ApiResponse<T>>(url, {
        method: options.method || 'GET',
        body: options.body,
        headers: {
          'Content-Type': 'application/json',
          ...options.headers,
        },
      })

      return response.data
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : 'Unknown error'
      error.value = errorMessage
      throw new Error(errorMessage)
    } finally {
      loading.value = false
    }
  }

  return {
    loading: readonly(loading),
    error: readonly(error),
    execute,
  }
}

// composables/useCustomer.ts
export const useCustomer = () => {
  const customerStore = useCustomerStore()
  const { execute, loading, error } = useApi()

  const customers = computed(() => customerStore.customers)
  const currentCustomer = computed(() => customerStore.currentCustomer)

  const fetchCustomers = async () => {
    return await customerStore.fetchCustomers()
  }

  const createCustomer = async (customerData: CreateCustomerRequest) => {
    return await customerStore.createCustomer(customerData)
  }

  const updateCustomer = async (id: string, customerData: UpdateCustomerRequest) => {
    return await customerStore.updateCustomer(id, customerData)
  }

  const searchCustomers = async (query: string) => {
    return await execute<Customer[]>('/api/customers/search', {
      method: 'POST',
      body: { query },
    })
  }

  return {
    customers,
    currentCustomer,
    loading,
    error,
    fetchCustomers,
    createCustomer,
    updateCustomer,
    searchCustomers,
  }
}
```

### 3.5 表單驗證標準
```typescript
// composables/useFormValidation.ts
import { z } from 'zod'

const customerSchema = z.object({
  name: z.string()
    .min(2, '姓名至少需要 2 個字符')
    .max(50, '姓名不能超過 50 個字符'),
  
  phone: z.string()
    .regex(/^09\d{8}$/, '請輸入有效的手機號碼'),
  
  email: z.string()
    .email('請輸入有效的 Email 地址')
    .optional()
    .or(z.literal('')),
  
  birthday: z.string()
    .optional()
    .refine(
      (date) => !date || isValid(new Date(date)),
      '請輸入有效的日期'
    ),
})

export const useCustomerForm = () => {
  const formData = ref({
    name: '',
    phone: '',
    email: '',
    birthday: '',
  })

  const errors = ref<Record<string, string>>({})

  const validate = () => {
    try {
      customerSchema.parse(formData.value)
      errors.value = {}
      return true
    } catch (error) {
      if (error instanceof z.ZodError) {
        errors.value = error.issues.reduce((acc, issue) => {
          acc[issue.path[0]] = issue.message
          return acc
        }, {} as Record<string, string>)
      }
      return false
    }
  }

  const reset = () => {
    formData.value = {
      name: '',
      phone: '',
      email: '',
      birthday: '',
    }
    errors.value = {}
  }

  return {
    formData,
    errors,
    validate,
    reset,
  }
}
```

---

## 4. 測試標準

### 4.1 Go 單元測試標準
```go
// customer_service_test.go
package service

import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
)

// Mock Repository
type MockCustomerRepository struct {
    mock.Mock
}

func (m *MockCustomerRepository) Create(customer *model.Customer) error {
    args := m.Called(customer)
    return args.Error(0)
}

func (m *MockCustomerRepository) GetByID(id string) (*model.Customer, error) {
    args := m.Called(id)
    return args.Get(0).(*model.Customer), args.Error(1)
}

func TestCustomerService_CreateCustomer(t *testing.T) {
    // Arrange
    mockRepo := new(MockCustomerRepository)
    service := NewCustomerService(mockRepo)
    
    customer := &model.Customer{
        Name:  "測試客戶",
        Phone: "0912345678",
        Email: "test@example.com",
    }
    
    mockRepo.On("Create", mock.MatchedBy(func(c *model.Customer) bool {
        return c.Name == customer.Name && c.Phone == customer.Phone
    })).Return(nil)
    
    // Act
    result, err := service.CreateCustomer(customer)
    
    // Assert
    assert.NoError(t, err)
    assert.NotEmpty(t, result.ID)
    assert.Equal(t, customer.Name, result.Name)
    assert.Equal(t, customer.Phone, result.Phone)
    mockRepo.AssertExpectations(t)
}

func TestCustomerService_GetCustomer_NotFound(t *testing.T) {
    // Arrange
    mockRepo := new(MockCustomerRepository)
    service := NewCustomerService(mockRepo)
    
    customerID := "non-existent-id"
    mockRepo.On("GetByID", customerID).Return((*model.Customer)(nil), ErrCustomerNotFound)
    
    // Act
    result, err := service.GetCustomer(customerID)
    
    // Assert
    assert.Error(t, err)
    assert.Nil(t, result)
    assert.Equal(t, ErrCustomerNotFound, err)
    mockRepo.AssertExpectations(t)
}
```

### 4.2 前端測試標準
```typescript
// CustomerCard.spec.ts
import { describe, it, expect, vi } from 'vitest'
import { mount } from '@vue/test-utils'
import CustomerCard from '~/components/CustomerCard.vue'
import type { Customer } from '~/types/customer'

describe('CustomerCard', () => {
  const mockCustomer: Customer = {
    id: '1',
    name: '張小明',
    phone: '0912345678',
    email: 'zhang@example.com',
    tier: 'gold',
    points: 1250,
    status: 'active',
    createdAt: '2024-01-15T10:30:00Z',
    updatedAt: '2024-01-15T10:30:00Z',
  }

  it('renders customer information correctly', () => {
    const wrapper = mount(CustomerCard, {
      props: { customer: mockCustomer },
    })

    expect(wrapper.text()).toContain(mockCustomer.name)
    expect(wrapper.text()).toContain(mockCustomer.phone)
    expect(wrapper.text()).toContain(mockCustomer.email)
  })

  it('emits edit event when edit button is clicked', async () => {
    const wrapper = mount(CustomerCard, {
      props: { customer: mockCustomer },
    })

    const editButton = wrapper.find('[data-testid="edit-button"]')
    await editButton.trigger('click')

    expect(wrapper.emitted()).toHaveProperty('edit')
    expect(wrapper.emitted('edit')?.[0]).toEqual([mockCustomer])
  })

  it('emits view event when view button is clicked', async () => {
    const wrapper = mount(CustomerCard, {
      props: { customer: mockCustomer },
    })

    const viewButton = wrapper.find('[data-testid="view-button"]')
    await viewButton.trigger('click')

    expect(wrapper.emitted()).toHaveProperty('view')
    expect(wrapper.emitted('view')?.[0]).toEqual([mockCustomer])
  })
})

// useCustomer.spec.ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { useCustomer } from '~/composables/useCustomer'

// Mock fetch
global.$fetch = vi.fn()

describe('useCustomer', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    vi.clearAllMocks()
  })

  it('fetches customers successfully', async () => {
    const mockCustomers = [
      { id: '1', name: '張小明', phone: '0912345678' },
      { id: '2', name: '李小華', phone: '0987654321' },
    ]

    vi.mocked($fetch).mockResolvedValueOnce({ data: mockCustomers })

    const { customers, fetchCustomers } = useCustomer()
    await fetchCustomers()

    expect(customers.value).toEqual(mockCustomers)
    expect($fetch).toHaveBeenCalledWith('/api/customers')
  })

  it('handles fetch error gracefully', async () => {
    const errorMessage = 'Network error'
    vi.mocked($fetch).mockRejectedValueOnce(new Error(errorMessage))

    const { error, fetchCustomers } = useCustomer()

    await expect(fetchCustomers()).rejects.toThrow()
    expect(error.value).toBe('Failed to fetch customers')
  })
})
```

---

## 5. Git 工作流程標準

### 5.1 分支策略
```
main            # 主分支，用於生產環境
├── develop     # 開發分支，用於整合功能
├── feature/    # 功能分支
│   ├── feature/auth-system
│   ├── feature/pos-integration
│   └── feature/customer-management
├── release/    # 發布分支
│   └── release/v1.0.0
└── hotfix/     # 熱修復分支
    └── hotfix/critical-bug-fix
```

### 5.2 提交訊息規範
```
類型(範圍): 簡短描述

詳細描述（可選）

相關 Issue（可選）

類型:
- feat: 新功能
- fix: 修復 bug
- docs: 文檔更新
- style: 代碼格式調整
- refactor: 代碼重構
- test: 測試相關
- chore: 建置或工具更新

範例:
feat(auth): add JWT token validation middleware

Implement JWT token validation middleware for API authentication.
- Add token parsing and validation
- Handle expired tokens
- Add error handling for invalid tokens

Closes #123
```

### 5.3 程式碼審查標準
- **功能性**: 代碼是否實現了預期的功能
- **可讀性**: 代碼是否清晰易懂
- **性能**: 是否有性能問題或優化機會
- **安全性**: 是否存在安全漏洞
- **測試**: 是否有適當的測試覆蓋
- **文檔**: 是否有必要的註釋和文檔

---

## 6. 部署與維運標準

### 6.1 Dockerfile 標準
```dockerfile
# Go 服務 Dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app

# 複製依賴文件
COPY go.mod go.sum ./
RUN go mod download

# 複製源碼
COPY . .

# 建構應用程式
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main cmd/server/main.go

# 最終映像
FROM alpine:latest

RUN apk --no-cache add ca-certificates tzdata
WORKDIR /root/

# 複製執行文件
COPY --from=builder /app/main .
COPY --from=builder /app/config config/

# 設定時區
ENV TZ=Asia/Taipei

EXPOSE 8080

CMD ["./main"]
```

### 6.2 Docker Compose 配置
```yaml
version: '3.8'

services:
  # API Gateway
  gateway:
    build:
      context: ./gateway
    ports:
      - "8080:8080"
    environment:
      - ENV=production
      - DB_HOST=mysql
      - REDIS_HOST=redis
    depends_on:
      - mysql
      - redis
    networks:
      - coffee-network

  # 認證服務
  auth-service:
    build:
      context: ./services/auth
    environment:
      - ENV=production
      - DB_HOST=mysql
      - REDIS_HOST=redis
    depends_on:
      - mysql
      - redis
    networks:
      - coffee-network

  # 資料庫
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: CoffeeMySQL2026!
      MYSQL_DATABASE: coffee_crm
    volumes:
      - mysql_data:/var/lib/mysql
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - coffee-network

  # 快取
  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    networks:
      - coffee-network

volumes:
  mysql_data:
  redis_data:

networks:
  coffee-network:
    driver: bridge
```

### 6.3 環境配置標準
```yaml
# config/production.yaml
server:
  port: 8080
  read_timeout: 30s
  write_timeout: 30s

database:
  host: mysql
  port: 3306
  username: coffee_user
  password: ${DB_PASSWORD}
  database: coffee_crm
  max_open_conns: 25
  max_idle_conns: 10
  max_lifetime: 5m

redis:
  host: redis
  port: 6379
  password: ${REDIS_PASSWORD}
  db: 0
  pool_size: 10

jwt:
  secret: ${JWT_SECRET}
  expire_hours: 24

logging:
  level: info
  format: json
```

---

## 7. 監控與日誌標準

### 7.1 應用程式監控
```go
// metrics/metrics.go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    // HTTP 請求計數器
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )

    // HTTP 請求持續時間
    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "http_request_duration_seconds",
            Help: "HTTP request duration in seconds",
        },
        []string{"method", "endpoint"},
    )

    // 資料庫連接池
    dbConnectionsInUse = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "db_connections_in_use",
            Help: "Number of database connections currently in use",
        },
    )
)

func RecordHTTPRequest(method, endpoint, status string, duration float64) {
    httpRequestsTotal.WithLabelValues(method, endpoint, status).Inc()
    httpRequestDuration.WithLabelValues(method, endpoint).Observe(duration)
}

func SetDBConnectionsInUse(count float64) {
    dbConnectionsInUse.Set(count)
}
```

### 7.2 日誌聚合配置
```yaml
# docker-compose.logging.yml
version: '3.8'

services:
  # ELK Stack for logging
  elasticsearch:
    image: elasticsearch:8.5.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data

  logstash:
    image: logstash:8.5.0
    volumes:
      - ./config/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    depends_on:
      - elasticsearch

  kibana:
    image: kibana:8.5.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

volumes:
  elasticsearch_data:
```

---

## 8. 安全性標準

### 8.1 API 安全
```go
// middleware/security.go
package middleware

func SecurityHeaders() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("X-Content-Type-Options", "nosniff")
        c.Header("X-Frame-Options", "DENY")
        c.Header("X-XSS-Protection", "1; mode=block")
        c.Header("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        c.Header("Content-Security-Policy", "default-src 'self'")
        c.Next()
    }
}

func RateLimit() gin.HandlerFunc {
    limiter := rate.NewLimiter(rate.Limit(100), 200) // 100 requests per second, burst of 200
    
    return func(c *gin.Context) {
        if !limiter.Allow() {
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "error": "Rate limit exceeded",
            })
            return
        }
        c.Next()
    }
}

func CORS() gin.HandlerFunc {
    return cors.New(cors.Config{
        AllowOrigins:     []string{"https://your-frontend-domain.com"},
        AllowMethods:     []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
        AllowHeaders:     []string{"Origin", "Content-Type", "Authorization"},
        ExposeHeaders:    []string{"Content-Length"},
        AllowCredentials: true,
        MaxAge:          12 * time.Hour,
    })
}
```

### 8.2 輸入驗證
```go
// validation/validator.go
package validation

import (
    "github.com/go-playground/validator/v10"
)

type CustomerRequest struct {
    Name  string `json:"name" validate:"required,min=2,max=50"`
    Phone string `json:"phone" validate:"required,phone"`
    Email string `json:"email" validate:"omitempty,email"`
}

func ValidateCustomerRequest(req *CustomerRequest) error {
    validate := validator.New()
    
    // 註冊自定義驗證規則
    validate.RegisterValidation("phone", validatePhone)
    
    return validate.Struct(req)
}

func validatePhone(fl validator.FieldLevel) bool {
    phone := fl.Field().String()
    matched, _ := regexp.MatchString(`^09\d{8}$`, phone)
    return matched
}
```

---

**版本**: v1.0  
**建立日期**: 2025-01-16  
**最後更新**: 2025-01-16  
**負責人**: 技術架構團隊