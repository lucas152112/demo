# 🚀 Coffee系統階段5: 開發編譯測試部署 - 執行開始

## 📋 **Phase 0: 共享基礎架構建設** (第1-3週)

### **正在執行**: 共享元件庫開發

---

## 🎨 **1. 共享前端元件庫建設**

### **基礎元件庫架構**
```typescript
// packages/shared-components/index.ts
export * from './components/base'
export * from './components/business'  
export * from './components/layout'
export * from './components/ai'
export * from './utils'
export * from './composables'
```

### **基礎元件開發**

#### **CoffeeButton 統一按鈕元件**
```vue
<!-- packages/shared-components/components/base/CoffeeButton.vue -->
<template>
  <a-button 
    :type="buttonType"
    :size="size"
    :loading="loading"
    :disabled="disabled"
    :danger="danger"
    :ghost="ghost"
    :shape="shape"
    :icon="icon"
    :block="block"
    :class="buttonClass"
    @click="handleClick"
  >
    <slot />
  </a-button>
</template>

<script setup lang="ts">
interface Props {
  type?: 'primary' | 'default' | 'dashed' | 'text' | 'link'
  size?: 'large' | 'middle' | 'small'
  loading?: boolean
  disabled?: boolean
  danger?: boolean
  ghost?: boolean
  shape?: 'default' | 'circle' | 'round'
  icon?: string
  block?: boolean
  variant?: 'coffee' | 'success' | 'warning' | 'error'
}

const props = withDefaults(defineProps<Props>(), {
  type: 'default',
  size: 'middle',
  loading: false,
  disabled: false,
  danger: false,
  ghost: false,
  shape: 'default',
  block: false,
  variant: 'coffee'
})

const emit = defineEmits<{
  click: [event: MouseEvent]
}>()

const buttonType = computed(() => {
  if (props.danger) return 'primary'
  return props.type
})

const buttonClass = computed(() => {
  return [
    'coffee-button',
    `coffee-button--${props.variant}`,
    {
      'coffee-button--danger': props.danger
    }
  ]
})

const handleClick = (event: MouseEvent) => {
  if (!props.disabled && !props.loading) {
    emit('click', event)
  }
}
</script>

<style scoped>
.coffee-button {
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}

.coffee-button--coffee {
  --primary-color: #8B4513;
  --primary-hover: #A0522D;
  --primary-active: #654321;
}

.coffee-button--success {
  --primary-color: #52c41a;
  --primary-hover: #73d13d;
  --primary-active: #389e0d;
}

.coffee-button--warning {
  --primary-color: #faad14;
  --primary-hover: #ffc53d;
  --primary-active: #d48806;
}

.coffee-button--error {
  --primary-color: #ff4d4f;
  --primary-hover: #ff7875;
  --primary-active: #d9363e;
}

.coffee-button--danger {
  background-color: var(--primary-color);
  border-color: var(--primary-color);
  color: white;
}

.coffee-button--danger:hover {
  background-color: var(--primary-hover);
  border-color: var(--primary-hover);
}

.coffee-button--danger:active {
  background-color: var(--primary-active);
  border-color: var(--primary-active);
}
</style>
```

#### **CoffeeTable 統一表格元件**
```vue
<!-- packages/shared-components/components/base/CoffeeTable.vue -->
<template>
  <div class="coffee-table-wrapper">
    <div v-if="showHeader" class="coffee-table-header">
      <div class="coffee-table-title">
        <slot name="title">{{ title }}</slot>
      </div>
      <div class="coffee-table-actions">
        <slot name="actions" />
      </div>
    </div>
    
    <a-table
      :columns="tableColumns"
      :data-source="dataSource"
      :loading="loading"
      :pagination="tablePagination"
      :scroll="scroll"
      :size="size"
      :bordered="bordered"
      :row-selection="rowSelection"
      :row-key="rowKey"
      :class="tableClass"
      @change="handleTableChange"
    >
      <template v-for="(_, name) in $slots" :key="name" #[name]="slotData">
        <slot :name="name" v-bind="slotData" />
      </template>
    </a-table>
  </div>
</template>

<script setup lang="ts">
import type { TableColumnsType, TablePaginationConfig } from 'ant-design-vue'

interface Props {
  columns: TableColumnsType
  dataSource: any[]
  loading?: boolean
  title?: string
  showHeader?: boolean
  pagination?: TablePaginationConfig | false
  scroll?: { x?: number; y?: number }
  size?: 'large' | 'middle' | 'small'
  bordered?: boolean
  rowSelection?: any
  rowKey?: string | ((record: any) => string)
  variant?: 'default' | 'compact' | 'card'
}

const props = withDefaults(defineProps<Props>(), {
  loading: false,
  showHeader: true,
  pagination: true,
  size: 'middle',
  bordered: false,
  rowKey: 'id',
  variant: 'default'
})

const emit = defineEmits<{
  change: [pagination: any, filters: any, sorter: any]
}>()

const tableColumns = computed(() => {
  return props.columns.map(col => ({
    ...col,
    className: `coffee-table-column ${col.className || ''}`
  }))
})

const tablePagination = computed(() => {
  if (props.pagination === false) return false
  
  return {
    showSizeChanger: true,
    showQuickJumper: true,
    showTotal: (total: number, range: [number, number]) => 
      `第 ${range[0]}-${range[1]} 項，共 ${total} 項`,
    ...props.pagination
  }
})

const tableClass = computed(() => {
  return [
    'coffee-table',
    `coffee-table--${props.variant}`
  ]
})

const handleTableChange = (pagination: any, filters: any, sorter: any) => {
  emit('change', pagination, filters, sorter)
}
</script>

<style scoped>
.coffee-table-wrapper {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.coffee-table-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 16px 20px;
  border-bottom: 1px solid #f0f0f0;
}

.coffee-table-title {
  font-size: 16px;
  font-weight: 600;
  color: #262626;
}

.coffee-table-actions {
  display: flex;
  gap: 8px;
}

.coffee-table {
  margin: 0;
}

.coffee-table--compact {
  font-size: 12px;
}

.coffee-table--card {
  border: 1px solid #f0f0f0;
  border-radius: 8px;
}

:deep(.coffee-table-column) {
  text-align: center;
}

:deep(.ant-table-thead > tr > th) {
  background: #fafafa;
  font-weight: 600;
  color: #262626;
}

:deep(.ant-table-tbody > tr:hover > td) {
  background: #f5f5f5;
}
</style>
```

#### **CoffeeForm 統一表單元件**
```vue
<!-- packages/shared-components/components/base/CoffeeForm.vue -->
<template>
  <a-form
    :model="formModel"
    :rules="formRules"
    :label-col="labelCol"
    :wrapper-col="wrapperCol"
    :layout="layout"
    :size="size"
    :class="formClass"
    @finish="handleSubmit"
    @finishFailed="handleSubmitFailed"
    ref="formRef"
  >
    <slot />
    
    <a-form-item v-if="showActions" :wrapper-col="actionCol">
      <div class="coffee-form-actions">
        <coffee-button 
          v-if="showReset"
          @click="handleReset"
        >
          重置
        </coffee-button>
        <coffee-button 
          type="primary"
          :loading="submitting"
          html-type="submit"
          variant="coffee"
        >
          {{ submitText }}
        </coffee-button>
      </div>
    </a-form-item>
  </a-form>
</template>

<script setup lang="ts">
import type { FormInstance } from 'ant-design-vue'

interface Props {
  model: Record<string, any>
  rules?: Record<string, any>
  labelCol?: object
  wrapperCol?: object
  layout?: 'horizontal' | 'vertical' | 'inline'
  size?: 'large' | 'middle' | 'small'
  showActions?: boolean
  showReset?: boolean
  submitText?: string
  submitting?: boolean
  variant?: 'default' | 'card' | 'modal'
}

const props = withDefaults(defineProps<Props>(), {
  layout: 'horizontal',
  size: 'middle',
  showActions: true,
  showReset: true,
  submitText: '提交',
  submitting: false,
  variant: 'default'
})

const emit = defineEmits<{
  submit: [values: any]
  submitFailed: [errorInfo: any]
  reset: []
}>()

const formRef = ref<FormInstance>()
const formModel = toRef(props, 'model')
const formRules = toRef(props, 'rules')

const actionCol = computed(() => {
  if (props.layout === 'vertical') {
    return { span: 24 }
  }
  return {
    offset: props.labelCol?.span || 6,
    span: props.wrapperCol?.span || 18
  }
})

const formClass = computed(() => {
  return [
    'coffee-form',
    `coffee-form--${props.variant}`
  ]
})

const handleSubmit = (values: any) => {
  emit('submit', values)
}

const handleSubmitFailed = (errorInfo: any) => {
  emit('submitFailed', errorInfo)
}

const handleReset = () => {
  formRef.value?.resetFields()
  emit('reset')
}

defineExpose({
  validate: () => formRef.value?.validate(),
  resetFields: () => formRef.value?.resetFields(),
  clearValidate: () => formRef.value?.clearValidate()
})
</script>

<style scoped>
.coffee-form {
  max-width: 600px;
}

.coffee-form--card {
  padding: 24px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.coffee-form--modal {
  padding: 0;
}

.coffee-form-actions {
  display: flex;
  justify-content: flex-end;
  gap: 12px;
}

:deep(.ant-form-item-label > label) {
  font-weight: 500;
  color: #262626;
}
</style>
```

### **業務元件開發**

#### **ProductSelector 商品選擇器**
```vue
<!-- packages/shared-components/components/business/ProductSelector.vue -->
<template>
  <div class="product-selector">
    <div class="product-search">
      <a-input-search
        v-model:value="searchKeyword"
        placeholder="搜尋商品..."
        @search="handleSearch"
      />
    </div>
    
    <div class="product-categories">
      <a-radio-group 
        v-model:value="selectedCategory"
        button-style="solid"
        @change="handleCategoryChange"
      >
        <a-radio-button value="">全部</a-radio-button>
        <a-radio-button 
          v-for="category in categories"
          :key="category.id"
          :value="category.id"
        >
          {{ category.name }}
        </a-radio-button>
      </a-radio-group>
    </div>
    
    <div class="product-grid">
      <div
        v-for="product in filteredProducts"
        :key="product.id"
        :class="['product-item', { 'selected': isSelected(product.id) }]"
        @click="handleProductSelect(product)"
      >
        <div class="product-image">
          <img :src="product.image || '/default-product.png'" :alt="product.name" />
        </div>
        <div class="product-info">
          <h4 class="product-name">{{ product.name }}</h4>
          <p class="product-price">NT$ {{ product.price }}</p>
          <p class="product-stock" :class="{ 'low-stock': product.stock < 10 }">
            庫存: {{ product.stock }}
          </p>
        </div>
        <div v-if="isSelected(product.id)" class="product-quantity">
          <a-input-number
            v-model:value="getQuantity(product.id)"
            :min="0"
            :max="product.stock"
            @change="(value) => updateQuantity(product.id, value)"
            @click.stop
          />
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
interface Product {
  id: string
  name: string
  price: number
  stock: number
  image?: string
  categoryId: string
}

interface Category {
  id: string
  name: string
}

interface SelectedProduct {
  id: string
  quantity: number
}

interface Props {
  products: Product[]
  categories: Category[]
  selectedProducts?: SelectedProduct[]
  maxSelections?: number
}

const props = withDefaults(defineProps<Props>(), {
  selectedProducts: () => [],
  maxSelections: 0
})

const emit = defineEmits<{
  select: [product: Product, quantity: number]
  remove: [productId: string]
  change: [selectedProducts: SelectedProduct[]]
}>()

const searchKeyword = ref('')
const selectedCategory = ref('')
const selections = ref<Map<string, number>>(new Map())

// 初始化已選商品
watchEffect(() => {
  const newSelections = new Map<string, number>()
  props.selectedProducts.forEach(item => {
    newSelections.set(item.id, item.quantity)
  })
  selections.value = newSelections
})

const filteredProducts = computed(() => {
  let result = props.products
  
  // 分類篩選
  if (selectedCategory.value) {
    result = result.filter(p => p.categoryId === selectedCategory.value)
  }
  
  // 搜尋篩選
  if (searchKeyword.value) {
    const keyword = searchKeyword.value.toLowerCase()
    result = result.filter(p => 
      p.name.toLowerCase().includes(keyword)
    )
  }
  
  return result
})

const isSelected = (productId: string) => {
  return selections.value.has(productId)
}

const getQuantity = (productId: string) => {
  return selections.value.get(productId) || 0
}

const handleProductSelect = (product: Product) => {
  if (isSelected(product.id)) {
    // 如果已選中，移除選擇
    selections.value.delete(product.id)
    emit('remove', product.id)
  } else {
    // 檢查是否超過最大選擇數量
    if (props.maxSelections > 0 && selections.value.size >= props.maxSelections) {
      return
    }
    
    // 新增選擇
    selections.value.set(product.id, 1)
    emit('select', product, 1)
  }
  
  emitChange()
}

const updateQuantity = (productId: string, quantity: number | null) => {
  if (quantity === null || quantity <= 0) {
    selections.value.delete(productId)
    emit('remove', productId)
  } else {
    selections.value.set(productId, quantity)
    const product = props.products.find(p => p.id === productId)
    if (product) {
      emit('select', product, quantity)
    }
  }
  
  emitChange()
}

const handleSearch = () => {
  // 搜尋邏輯已在 computed 中處理
}

const handleCategoryChange = () => {
  // 分類變更邏輯已在 computed 中處理
}

const emitChange = () => {
  const selectedProducts: SelectedProduct[] = Array.from(selections.value.entries())
    .map(([id, quantity]) => ({ id, quantity }))
  emit('change', selectedProducts)
}
</script>

<style scoped>
.product-selector {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.product-categories {
  padding: 8px 0;
}

.product-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 16px;
  max-height: 400px;
  overflow-y: auto;
}

.product-item {
  display: flex;
  flex-direction: column;
  padding: 12px;
  border: 2px solid #f0f0f0;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  background: white;
}

.product-item:hover {
  border-color: #8B4513;
  box-shadow: 0 4px 12px rgba(139, 69, 19, 0.1);
}

.product-item.selected {
  border-color: #8B4513;
  background: #f6f6f6;
}

.product-image {
  width: 100%;
  height: 120px;
  overflow: hidden;
  border-radius: 6px;
  margin-bottom: 8px;
}

.product-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.product-info {
  flex: 1;
}

.product-name {
  margin: 0 0 4px 0;
  font-size: 14px;
  font-weight: 600;
  color: #262626;
}

.product-price {
  margin: 0 0 4px 0;
  font-size: 16px;
  font-weight: 700;
  color: #8B4513;
}

.product-stock {
  margin: 0;
  font-size: 12px;
  color: #666;
}

.product-stock.low-stock {
  color: #ff4d4f;
  font-weight: 600;
}

.product-quantity {
  margin-top: 8px;
  display: flex;
  justify-content: center;
}
</style>
```

### **AI元件開發**

#### **AIChatButton 全域AI浮動按鈕**
```vue
<!-- packages/shared-components/components/ai/AIChatButton.vue -->
<template>
  <teleport to="body">
    <div class="ai-chat-container">
      <!-- AI浮動按鈕 -->
      <transition name="ai-button">
        <div
          v-show="!chatVisible"
          class="ai-chat-button"
          :class="{ 'pulsing': hasNewMessage }"
          @click="openChat"
        >
          <robot-outlined />
          <span v-if="hasNewMessage" class="notification-dot" />
        </div>
      </transition>
      
      <!-- AI對話視窗 -->
      <transition name="ai-chat">
        <div v-show="chatVisible" class="ai-chat-modal">
          <div class="ai-chat-header">
            <div class="ai-chat-title">
              <robot-outlined />
              <span>AI智能助手</span>
            </div>
            <div class="ai-chat-actions">
              <a-button type="text" size="small" @click="toggleVoiceMode">
                <audio-outlined v-if="!voiceMode" />
                <audio-muted-outlined v-else />
              </a-button>
              <a-button type="text" size="small" @click="closeChat">
                <close-outlined />
              </a-button>
            </div>
          </div>
          
          <div class="ai-chat-body" ref="chatBodyRef">
            <div
              v-for="message in messages"
              :key="message.id"
              :class="['ai-message', `ai-message--${message.type}`]"
            >
              <div v-if="message.type === 'user'" class="ai-message-content">
                {{ message.content }}
              </div>
              <div v-else class="ai-message-content">
                <div v-if="message.loading" class="ai-typing">
                  <span></span><span></span><span></span>
                </div>
                <div v-else v-html="formatAIResponse(message.content)"></div>
              </div>
            </div>
          </div>
          
          <div class="ai-chat-input">
            <div v-if="voiceMode" class="voice-input">
              <a-button
                :type="recording ? 'primary' : 'default'"
                :danger="recording"
                size="large"
                shape="circle"
                @mousedown="startRecording"
                @mouseup="stopRecording"
                @mouseleave="stopRecording"
              >
                <audio-outlined v-if="!recording" />
                <stop-outlined v-else />
              </a-button>
              <p class="voice-hint">
                {{ recording ? '正在錄音...' : '按住說話' }}
              </p>
            </div>
            
            <div v-else class="text-input">
              <a-input
                v-model:value="inputMessage"
                :placeholder="inputPlaceholder"
                :disabled="sending"
                @press-enter="sendMessage"
              >
                <template #suffix>
                  <a-button
                    type="text"
                    :loading="sending"
                    :disabled="!inputMessage.trim()"
                    @click="sendMessage"
                  >
                    <send-outlined />
                  </a-button>
                </template>
              </a-input>
            </div>
          </div>
        </div>
      </transition>
    </div>
  </teleport>
</template>

<script setup lang="ts">
interface Message {
  id: string
  type: 'user' | 'ai'
  content: string
  loading?: boolean
  timestamp: number
}

interface Props {
  position?: 'bottom-right' | 'bottom-left' | 'top-right' | 'top-left'
  tenantId?: string
  userId?: string
  permissions?: string[]
}

const props = withDefaults(defineProps<Props>(), {
  position: 'bottom-right'
})

const emit = defineEmits<{
  message: [message: string]
  response: [response: string]
}>()

// 狀態管理
const chatVisible = ref(false)
const voiceMode = ref(false)
const recording = ref(false)
const sending = ref(false)
const hasNewMessage = ref(false)
const inputMessage = ref('')
const messages = ref<Message[]>([])
const chatBodyRef = ref<HTMLElement>()

// 語音識別
let mediaRecorder: MediaRecorder | null = null
let audioStream: MediaStream | null = null

const inputPlaceholder = computed(() => {
  if (sending.value) return '正在處理...'
  return '輸入問題，例如：今天銷售情況如何？'
})

// 開啟聊天
const openChat = () => {
  chatVisible.value = true
  hasNewMessage.value = false
  
  // 如果沒有歡迎訊息，添加一個
  if (messages.value.length === 0) {
    addWelcomeMessage()
  }
  
  nextTick(() => {
    scrollToBottom()
  })
}

// 關閉聊天
const closeChat = () => {
  chatVisible.value = false
  stopRecording()
}

// 添加歡迎訊息
const addWelcomeMessage = () => {
  const welcomeMessage: Message = {
    id: generateId(),
    type: 'ai',
    content: '您好！我是您的AI智能助手。您可以問我關於系統的任何問題，比如：<br/>• 今天的銷售數據如何？<br/>• 哪些商品庫存不足？<br/>• 本月營收統計？',
    timestamp: Date.now()
  }
  messages.value.push(welcomeMessage)
}

// 切換語音模式
const toggleVoiceMode = () => {
  voiceMode.value = !voiceMode.value
  if (!voiceMode.value) {
    stopRecording()
  }
}

// 開始錄音
const startRecording = async () => {
  if (recording.value) return
  
  try {
    audioStream = await navigator.mediaDevices.getUserMedia({ audio: true })
    mediaRecorder = new MediaRecorder(audioStream)
    
    const audioChunks: Blob[] = []
    
    mediaRecorder.ondataavailable = (event) => {
      audioChunks.push(event.data)
    }
    
    mediaRecorder.onstop = async () => {
      const audioBlob = new Blob(audioChunks, { type: 'audio/wav' })
      await processVoiceInput(audioBlob)
    }
    
    mediaRecorder.start()
    recording.value = true
  } catch (error) {
    console.error('無法啟動錄音:', error)
    // 可以顯示錯誤提示
  }
}

// 停止錄音
const stopRecording = () => {
  if (!recording.value || !mediaRecorder) return
  
  mediaRecorder.stop()
  recording.value = false
  
  if (audioStream) {
    audioStream.getTracks().forEach(track => track.stop())
    audioStream = null
  }
}

// 處理語音輸入
const processVoiceInput = async (audioBlob: Blob) => {
  try {
    // 這裡應該調用語音轉文字服務
    // const text = await speechToText(audioBlob)
    
    // 示例：模擬語音轉文字
    const text = '今天銷售情況如何？' // 模擬結果
    
    if (text.trim()) {
      inputMessage.value = text
      await sendMessage()
    }
  } catch (error) {
    console.error('語音識別失敗:', error)
  }
}

// 發送訊息
const sendMessage = async () => {
  const message = inputMessage.value.trim()
  if (!message || sending.value) return
  
  // 添加用戶訊息
  const userMessage: Message = {
    id: generateId(),
    type: 'user',
    content: message,
    timestamp: Date.now()
  }
  messages.value.push(userMessage)
  
  // 添加AI載入訊息
  const aiMessage: Message = {
    id: generateId(),
    type: 'ai',
    content: '',
    loading: true,
    timestamp: Date.now()
  }
  messages.value.push(aiMessage)
  
  // 清空輸入
  inputMessage.value = ''
  sending.value = true
  
  // 滾動到底部
  nextTick(() => {
    scrollToBottom()
  })
  
  try {
    // 發送到AI服務
    emit('message', message)
    
    // 模擬AI回應 (實際應該調用AI API)
    const response = await mockAIResponse(message)
    
    // 更新AI訊息
    aiMessage.loading = false
    aiMessage.content = response
    
    emit('response', response)
  } catch (error) {
    console.error('AI回應失敗:', error)
    aiMessage.loading = false
    aiMessage.content = '抱歉，AI服務暫時不可用，請稍後再試。'
  } finally {
    sending.value = false
    nextTick(() => {
      scrollToBottom()
    })
  }
}

// 模擬AI回應
const mockAIResponse = async (message: string): Promise<string> => {
  // 模擬API延遲
  await new Promise(resolve => setTimeout(resolve, 1500))
  
  // 根據問題類型返回不同回應
  if (message.includes('銷售') || message.includes('營收')) {
    return `根據今日數據統計：<br/>
    • 總銷售額：NT$ 45,680<br/>
    • 訂單數量：127筆<br/>
    • 平均客單價：NT$ 360<br/>
    • 熱銷商品：美式咖啡、拿鐵咖啡<br/>
    相較昨日成長 12%，表現優異！`
  }
  
  if (message.includes('庫存')) {
    return `庫存狀況檢查：<br/>
    🔴 <strong>庫存不足 (需補貨)</strong>：<br/>
    • 咖啡豆 (衣索比亞)：剩餘 5kg<br/>
    • 牛奶：剩餘 8瓶<br/>
    🟡 <strong>庫存偏低</strong>：<br/>
    • 紙杯 (大)：剩餘 150個<br/>
    • 糖包：剩餘 200包<br/>
    建議盡快安排補貨。`
  }
  
  return `我已收到您的問題："${message}"<br/>
  正在為您查詢相關資料，請稍候...`
}

// 格式化AI回應
const formatAIResponse = (content: string) => {
  return content.replace(/\n/g, '<br/>')
}

// 滾動到底部
const scrollToBottom = () => {
  if (chatBodyRef.value) {
    chatBodyRef.value.scrollTop = chatBodyRef.value.scrollHeight
  }
}

// 生成ID
const generateId = () => {
  return `msg-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`
}

// 清理資源
onUnmounted(() => {
  stopRecording()
})
</script>

<style scoped>
.ai-chat-container {
  position: fixed;
  bottom: 20px;
  right: 20px;
  z-index: 9999;
}

/* AI浮動按鈕 */
.ai-chat-button {
  width: 60px;
  height: 60px;
  background: linear-gradient(135deg, #8B4513, #A0522D);
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-size: 24px;
  cursor: pointer;
  box-shadow: 0 4px 20px rgba(139, 69, 19, 0.3);
  transition: all 0.3s ease;
  position: relative;
}

.ai-chat-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 25px rgba(139, 69, 19, 0.4);
}

.ai-chat-button.pulsing {
  animation: pulse 2s infinite;
}

.notification-dot {
  position: absolute;
  top: 8px;
  right: 8px;
  width: 12px;
  height: 12px;
  background: #ff4d4f;
  border-radius: 50%;
  border: 2px solid white;
}

/* AI對話視窗 */
.ai-chat-modal {
  width: 400px;
  height: 600px;
  background: white;
  border-radius: 12px;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.15);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.ai-chat-header {
  padding: 16px 20px;
  background: linear-gradient(135deg, #8B4513, #A0522D);
  color: white;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.ai-chat-title {
  display: flex;
  align-items: center;
  gap: 8px;
  font-weight: 600;
  font-size: 16px;
}

.ai-chat-actions {
  display: flex;
  gap: 4px;
}

.ai-chat-body {
  flex: 1;
  padding: 16px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

/* 訊息樣式 */
.ai-message {
  display: flex;
  max-width: 80%;
}

.ai-message--user {
  align-self: flex-end;
}

.ai-message--ai {
  align-self: flex-start;
}

.ai-message-content {
  padding: 12px 16px;
  border-radius: 18px;
  font-size: 14px;
  line-height: 1.4;
}

.ai-message--user .ai-message-content {
  background: #8B4513;
  color: white;
  border-bottom-right-radius: 6px;
}

.ai-message--ai .ai-message-content {
  background: #f5f5f5;
  color: #262626;
  border-bottom-left-radius: 6px;
}

/* 打字動畫 */
.ai-typing {
  display: flex;
  gap: 4px;
  align-items: center;
}

.ai-typing span {
  width: 6px;
  height: 6px;
  background: #8B4513;
  border-radius: 50%;
  animation: typing 1.4s infinite ease-in-out;
}

.ai-typing span:nth-child(2) {
  animation-delay: 0.2s;
}

.ai-typing span:nth-child(3) {
  animation-delay: 0.4s;
}

/* 輸入區域 */
.ai-chat-input {
  padding: 16px;
  border-top: 1px solid #f0f0f0;
}

.voice-input {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
}

.voice-hint {
  margin: 0;
  font-size: 12px;
  color: #666;
}

.text-input {
  width: 100%;
}

/* 動畫 */
.ai-button-enter-active, .ai-button-leave-active {
  transition: all 0.3s ease;
}

.ai-button-enter-from, .ai-button-leave-to {
  opacity: 0;
  transform: translateY(20px) scale(0.8);
}

.ai-chat-enter-active, .ai-chat-leave-active {
  transition: all 0.3s ease;
}

.ai-chat-enter-from, .ai-chat-leave-to {
  opacity: 0;
  transform: translateY(20px) scale(0.9);
}

@keyframes pulse {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.05);
  }
  100% {
    transform: scale(1);
  }
}

@keyframes typing {
  0%, 60%, 100% {
    transform: translateY(0);
  }
  30% {
    transform: translateY(-10px);
  }
}

/* 響應式設計 */
@media (max-width: 480px) {
  .ai-chat-container {
    bottom: 10px;
    right: 10px;
  }
  
  .ai-chat-modal {
    width: calc(100vw - 40px);
    height: calc(100vh - 120px);
  }
}
</style>
```

---

**🏗️ 共享元件庫第一部分開發完成！**

包含：
- ✅ **基礎元件**: CoffeeButton, CoffeeTable, CoffeeForm
- ✅ **業務元件**: ProductSelector商品選擇器  
- ✅ **AI元件**: AIChatButton全域AI助手

**下一步**: 繼續開發共享後端函式庫？
