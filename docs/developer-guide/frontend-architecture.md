# 前端架构详解

> 深入理解 MiroFish 前端的设计思想与 Vue 3 最佳实践

**难度**: ⭐⭐ | **预计时间**: 45 分钟

---

## 学习目标

完成本章节学习后，你将能够：

### 基础目标（必掌握）

- [ ] 理解 MiroFish 前端的分层架构设计
- [ ] 掌握 Vue 3 Composition API 的使用方式
- [ ] 理解组件化设计的核心原则
- [ ] 掌握路由配置和页面导航
- [ ] 理解 API 客户端的封装方式

### 进阶目标（建议掌握）

- [ ] 掌握状态管理的设计模式
- [ ] 理解 D3.js 图谱可视化的实现
- [ ] 能够设计可复用的组件
- [ ] 掌握错误处理和边界情况

### 专家目标（挑战）

- [ ] 能够进行前端性能优化
- [ ] 能够设计复杂的可视化组件
- [ ] 能够进行组件库的架构设计
- [ ] 能够处理大规模状态管理

---

## 为什么这样设计？

### Vue 3 选择理由

| 框架 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **Vue 3** | 渐进式、易学习、性能优秀 | 生态相对较小 | 中小型 SPA ✅ |
| React | 生态强大、灵活 | 学习曲线陡峭 | 大型应用 |
| Angular | 完整框架、企业级 | 过于重量级 | 大型企业应用 |

MiroFish 选择 Vue 3 的原因：
1. **渐进式架构** - 可以逐步采用高级特性
2. **Composition API** - 更好的代码组织和复用
3. **性能优秀** - 编译时优化、响应式系统
4. **开发体验** - 单文件组件、热模块替换

### 架构设计原则

```
┌─────────────────────────────────────────────────────────────┐
│                    前端架构设计原则                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 组件化 (Component-Based)                                │
│     └─> UI 拆分为独立、可复用的组件                          │
│                                                              │
│  2. 单向数据流 (Unidirectional Data Flow)                   │
│     └─> Props down, Events up                               │
│                                                              │
│  3. 关注点分离 (Separation of Concerns)                     │
│     └─> 视图 / 逻辑 / 样式分离                              │
│                                                              │
│  4. 渐进式增强 (Progressive Enhancement)                     │
│     └─> 从简单开始，逐步添加高级特性                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 架构概览

### 技术栈分层

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Presentation Layer (视图层)                       │
│                        职责：UI 渲染、用户交互                          │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐     │
│  │  Step Components │  │  Shared UI       │  │  Visualization   │     │
│  │  Step1-5.vue     │  │  Buttons, Cards  │  │  D3.js Graph     │     │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘     │
│  Vue 3 SFC + Vite + TailwindCSS                                     │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       State Layer (状态层)                              │
│                        职责：状态管理、数据流控制                        │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐     │
│  │  Local State     │  │  Shared State    │  │  Server State    │     │
│  │  ref, reactive   │  │  provide/inject  │  │  API Client      │     │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘     │
│  Vue Reactivity API                                                │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       Routing Layer (路由层)                            │
│                        职责：页面导航、权限控制                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    Vue Router 4                                  │  │
│  │  /, /main, /simulation/:id, /report/:id, /interaction/:id      │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│  Lazy Loading + Route Guards                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        API Layer (接口层)                               │
│                        职责：HTTP 通信、数据转换                         │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐     │
│  │  Axios Client    │  │  API Modules     │  │  Interceptors    │     │
│  │  Base Config     │  │  graph/sim/report│  │  Error Handling  │     │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘     │
│  Axios + Request/Response Interceptors                               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      Backend API (后端 API)                             │
│                        Flask REST API                                  │
│  /api/graph/*, /api/simulation/*, /api/report/*                        │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 项目结构详解

### 完整目录结构

```
frontend/
├── public/                     # 静态资源（不经过 Vite 处理）
│   └── logo.svg
│
├── src/
│   ├── api/                    # API 层 - 所有后端通信
│   │   ├── index.js           # Axios 客户端配置
│   │   ├── graph.js           # 图谱相关 API
│   │   ├── simulation.js      # 模拟相关 API
│   │   └── report.js          # 报告相关 API
│   │
│   ├── assets/                 # 资源文件
│   │   └── logo/
│   │
│   ├── components/             # 可复用组件
│   │   ├── shared/            # 通用 UI 组件
│   │   │   ├── Button.vue
│   │   │   ├── Card.vue
│   │   │   ├── Modal.vue
│   │   │   └── Loading.vue
│   │   │
│   │   ├── workflow/          # 工作流步骤组件
│   │   │   ├── Step1GraphBuild.vue
│   │   │   ├── Step2EnvSetup.vue
│   │   │   ├── Step3Simulation.vue
│   │   │   ├── Step4Report.vue
│   │   │   └── Step5Interaction.vue
│   │   │
│   │   └── visualization/     # 可视化组件
│   │       ├── GraphPanel.vue     # D3.js 图谱
│   │       ├── ActionTimeline.vue # 动作时间线
│   │       └── EntityNetwork.vue  # 实体网络
│   │
│   ├── composables/           # 组合式函数（可复用逻辑）
│   │   ├── useTask.js        # 任务状态管理
│   │   ├── useUpload.js      # 文件上传
│   │   └── useWebSocket.js   # WebSocket 连接
│   │
│   ├── router/                # 路由配置
│   │   └── index.js
│   │
│   ├── stores/                # 全局状态（如需 Pinia）
│   │   └── project.js
│   │
│   ├── utils/                 # 工具函数
│   │   ├── validators.js     # 验证函数
│   │   ├── formatters.js     # 格式化函数
│   │   └── constants.js      # 常量定义
│   │
│   ├── views/                 # 页面视图
│   │   ├── Home.vue          # 首页
│   │   ├── MainView.vue      # 主工作流视图
│   │   ├── SimulationRunView.vue
│   │   ├── ReportView.vue
│   │   └── InteractionView.vue
│   │
│   ├── App.vue                # 根组件
│   └── main.js                # 应用入口
│
├── index.html                 # HTML 模板
├── vite.config.js            # Vite 配置
├── tailwind.config.js        # TailwindCSS 配置
└── package.json
```

---

## 核心概念详解

### 1. Vue 3 Composition API

#### 为什么使用 Composition API？

| 方式 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| Options API | 简单直观、熟悉 | 代码复用困难、逻辑分散 | 简单组件 |
| **Composition API** | **逻辑复用、更好组织** | **需要学习曲线** | **复杂组件 ✅** |

#### Setup 语法完整示例

```vue
<!-- components/workflow/Step1GraphBuild.vue -->
<template>
  <div class="step-container">
    <!-- 标题区 -->
    <StepTitle
      :title="title"
      :description="description"
      :step-number="1"
    />

    <!-- 文件上传区 -->
    <div class="upload-section">
      <FileUploadZone
        :files="uploadedFiles"
        :is-uploading="isUploading"
        :is-processing="isProcessing"
        @upload="handleFileUpload"
        @remove="handleFileRemove"
      />

      <!-- 进度条 -->
      <ProgressBar
        v-if="showProgress"
        :progress="uploadProgress"
        :status="uploadStatus"
      />
    </div>

    <!-- 本体预览 -->
    <OntologyPreview
      v-if="ontology"
      :entity-types="ontology.entityTypes"
      :relation-types="ontology.relationTypes"
    />

    <!-- 操作按钮 -->
    <StepActions
      :can-proceed="canProceedToNext"
      :is-loading="isBuildingGraph"
      @next="handleNextStep"
    />
  </div>
</template>

<script setup>
/**
 * Step 1: 图谱构建
 *
 * 职责：
 * 1. 处理文件上传
 * 2. 生成本体定义
 * 3. 构建 GraphRAG 图谱
 * 4. 提供实体列表
 */

import { ref, computed, onMounted, watch } from 'vue'
import { useRouter } from 'vue-router'
import { useProjectStore } from '@/stores/project'
import graphApi from '@/api/graph'
import StepTitle from '@/components/shared/StepTitle.vue'
import FileUploadZone from '@/components/shared/FileUploadZone.vue'
import ProgressBar from '@/components/shared/ProgressBar.vue'
import OntologyPreview from '@/components/shared/OntologyPreview.vue'
import StepActions from '@/components/shared/StepActions.vue'

// ==================== 常量定义 ====================
const title = "图谱构建"
const description = "上传文档，生成本体定义，构建知识图谱"

// ==================== 依赖注入 ====================
const router = useRouter()
const projectStore = useProjectStore()

// ==================== 响应式状态 ====================
// 文件上传状态
const uploadedFiles = ref([])
const isUploading = ref(false)
const uploadProgress = ref(0)
const uploadStatus = ref('uploading')  // uploading, processing, completed, error

// 图谱构建状态
const ontology = ref(null)
const graphId = ref(null)
const isBuildingGraph = ref(false)
const isProcessing = ref(false)

// 错误状态
const error = ref(null)

// ==================== 计算属性 ====================
const showProgress = computed(() => {
  return isUploading.value || isBuildingGraph.value
})

const canProceedToNext = computed(() => {
  return graphId.value !== null && !isBuildingGraph.value
})

// ==================== 方法 ====================

/**
 * 处理文件上传
 */
async function handleFileUpload(files) {
  isUploading.value = true
  uploadProgress.value = 0
  uploadStatus.value = 'uploading'
  error.value = null

  try {
    // 1. 上传文件
    uploadProgress.value = 20
    const uploaded = await uploadFiles(files)

    uploadedFiles.value = uploaded
    uploadProgress.value = 50

    // 2. 提取文本
    uploadStatus.value = 'processing'
    const text = await extractText(uploaded)

    uploadProgress.value = 70

    // 3. 生成本体
    await generateOntology(text)

    uploadProgress.value = 100
    uploadStatus.value = 'completed'

  } catch (err) {
    error.value = err.message
    uploadStatus.value = 'error'
    console.error('文件上传失败:', err)
  } finally {
    isUploading.value = false
  }
}

/**
 * 上传文件到服务器
 */
async function uploadFiles(files) {
  const formData = new FormData()

  for (const file of files) {
    formData.append('files', file)
  }

  const response = await fetch('/api/upload', {
    method: 'POST',
    body: formData
  })

  if (!response.ok) {
    throw new Error('文件上传失败')
  }

  return await response.json()
}

/**
 * 提取文件文本内容
 */
async function extractText(files) {
  const texts = files.map(f => f.content).join('\n\n')
  return texts
}

/**
 * 生成本体定义
 */
async function generateOntology(text) {
  const response = await graphApi.generateOntology(
    text,
    "社会模拟"
  )

  if (response.success) {
    ontology.value = response.data
  } else {
    throw new Error(response.error)
  }
}

/**
 * 移除文件
 */
function handleFileRemove(file) {
  const index = uploadedFiles.value.findIndex(f => f.id === file.id)
  if (index !== -1) {
    uploadedFiles.value.splice(index, 1)
  }

  // 清空相关状态
  if (uploadedFiles.value.length === 0) {
    ontology.value = null
    graphId.value = null
  }
}

/**
 * 进入下一步
 */
async function handleNextStep() {
  // 构建图谱
  if (!graphId.value && ontology.value) {
    await buildGraph()
  }

  // 保存到 store
  projectStore.setGraphId(graphId.value)
  projectStore.setOntology(ontology.value)

  // 导航到下一步
  router.push({ name: 'MainView', query: { step: 2 } })
}

/**
 * 构建 GraphRAG 图谱
 */
async function buildGraph() {
  isBuildingGraph.value = true

  try {
    const text = uploadedFiles.value
      .map(f => f.content)
      .join('\n\n')

    const response = await graphApi.buildGraph({
      text: text,
      ontology: ontology.value,
      session_id: projectStore.currentSessionId
    })

    if (response.success) {
      // 轮询任务状态
      const taskId = response.data.task_id
      graphId.value = await pollTaskCompletion(taskId)
    }
  } catch (err) {
    error.value = err.message
    throw err
  } finally {
    isBuildingGraph.value = false
  }
}

/**
 * 轮询任务完成状态
 */
async function pollTaskCompletion(taskId) {
  const maxAttempts = 300  // 最多 5 分钟
  const interval = 1000    // 每秒检查一次

  for (let i = 0; i < maxAttempts; i++) {
    const status = await graphApi.getTaskStatus(taskId)

    if (status.data.status === 'completed') {
      return status.data.result.graph_id
    }

    if (status.data.status === 'failed') {
      throw new Error(status.data.error || '图谱构建失败')
    }

    await new Promise(resolve => setTimeout(resolve, interval))
  }

  throw new Error('图谱构建超时')
}

// ==================== 生命周期钩子 ====================
onMounted(() => {
  // 恢复之前的状态
  if (projectStore.currentGraphId) {
    graphId.value = projectStore.currentGraphId
  }

  if (projectStore.currentOntology) {
    ontology.value = projectStore.currentOntology
  }
})

// ==================== 监听器 ====================
watch(graphId, (newId) => {
  if (newId) {
    console.log('图谱 ID 更新:', newId)
  }
})
</script>

<style scoped>
.step-container {
  @apply max-w-4xl mx-auto p-6;
}

.upload-section {
  @apply mt-8;
}
</style>
```

---

### 2. Composables 设计模式

#### 什么是 Composable？

Composable 是一个利用 Composition API 封装和复用**有状态逻辑**的函数。

#### useTask - 任务状态管理

```javascript
// composables/useTask.js
import { ref, computed } from 'vue'

/**
 * 任务状态管理 Composable
 *
 * 用途：管理异步任务的统一模式
 *
 * @param {Function} taskFn - 要执行的任务函数
 * @returns {Object} 任务状态和控制方法
 */
export function useTask(taskFn) {
  // 状态
  const data = ref(null)
  const error = ref(null)
  const status = ref('idle')  // idle, loading, success, error

  // 计算属性
  const isLoading = computed(() => status.value === 'loading')
  const isSuccess = computed(() => status.value === 'success')
  const isError = computed(() => status.value === 'error')

  /**
   * 执行任务
   */
  async function execute(...args) {
    status.value = 'loading'
    error.value = null

    try {
      const result = await taskFn(...args)
      data.value = result
      status.value = 'success'
      return result
    } catch (err) {
      error.value = err
      status.value = 'error'
      throw err
    }
  }

  /**
   * 重置状态
   */
  function reset() {
    data.value = null
    error.value = null
    status.value = 'idle'
  }

  return {
    // 状态
    data,
    error,
    status,
    // 计算属性
    isLoading,
    isSuccess,
    isError,
    // 方法
    execute,
    reset
  }
}

// ==================== 使用示例 ====================
// 在组件中使用
/*
import { useTask } from '@/composables/useTask'
import graphApi from '@/api/graph'

const ontologyTask = useTask(graphApi.generateOntology)

// 执行
await ontologyTask.execute(text, "社会模拟")

// 状态判断
if (ontologyTask.isSuccess) {
  console.log(ontologyTask.data)
}
*/
```

#### useUpload - 文件上传

```javascript
// composables/useUpload.js
import { ref } from 'vue'

export function useUpload(options = {}) {
  const {
    maxFiles = 10,
    maxFileSize = 50 * 1024 * 1024,  // 50MB
    allowedTypes = ['pdf', 'md', 'txt', 'markdown']
  } = options

  const files = ref([])
  const isUploading = ref(false)
  const progress = ref(0)
  const error = ref(null)

  /**
   * 验证文件
   */
  function validateFile(file) {
    // 检查类型
    const ext = file.name.split('.').pop().toLowerCase()
    if (!allowedTypes.includes(ext)) {
      throw new Error(`不支持的文件类型: ${ext}`)
    }

    // 检查大小
    if (file.size > maxFileSize) {
      throw new Error(`文件过大: ${file.name}`)
    }

    return true
  }

  /**
   * 添加文件
   */
  function addFiles(newFiles) {
    if (files.value.length + newFiles.length > maxFiles) {
      throw new Error(`最多只能上传 ${maxFiles} 个文件`)
    }

    for (const file of newFiles) {
      validateFile(file)

      files.value.push({
        id: Date.now() + Math.random(),
        name: file.name,
        size: file.size,
        type: file.type,
        raw: file,
        content: null
      })
    }
  }

  /**
   * 移除文件
   */
  function removeFile(fileId) {
    const index = files.value.findIndex(f => f.id === fileId)
    if (index !== -1) {
      files.value.splice(index, 1)
    }
  }

  /**
   * 上传文件
   */
  async function upload(endpoint = '/api/upload') {
    if (files.value.length === 0) {
      throw new Error('没有文件可上传')
    }

    isUploading.value = true
    progress.value = 0
    error.value = null

    try {
      const formData = new FormData()

      for (const file of files.value) {
        formData.append('files', file.raw)
      }

      const response = await fetch(endpoint, {
        method: 'POST',
        body: formData
      })

      if (!response.ok) {
        throw new Error('上传失败')
      }

      const result = await response.json()

      // 更新文件内容
      for (let i = 0; i < files.value.length; i++) {
        if (result.files[i]) {
          files.value[i].content = result.files[i].content
        }
      }

      progress.value = 100
      return result

    } catch (err) {
      error.value = err.message
      throw err
    } finally {
      isUploading.value = false
    }
  }

  /**
   * 清空文件
   */
  function clear() {
    files.value = []
    progress.value = 0
    error.value = null
  }

  return {
    files,
    isUploading,
    progress,
    error,
    addFiles,
    removeFile,
    upload,
    clear
  }
}
```

---

### 3. 组件通信模式

#### Props Down, Events Up

```vue
<!-- 父组件 -->
<template>
  <div>
    <ChildComponent
      :user-info="userData"
      :is-editable="canEdit"
      @update="handleUserUpdate"
      @delete="handleUserDelete"
    />
  </div>
</template>

<script setup>
import { ref } from 'vue'
import ChildComponent from './ChildComponent.vue'

const userData = ref({
  name: 'Alice',
  age: 30
})

const canEdit = ref(true)

function handleUserUpdate(updatedUser) {
  userData.value = updatedUser
}

function handleUserDelete(userId) {
  console.log('删除用户:', userId)
}
</script>
```

```vue
<!-- 子组件 -->
<template>
  <div>
    <h2>{{ userInfo.name }}</h2>
    <p>年龄: {{ userInfo.age }}</p>

    <button
      v-if="isEditable"
      @click="handleEdit"
    >
      编辑
    </button>

    <button
      @click="handleDelete"
    >
      删除
    </button>
  </div>
</template>

<script setup>
// Props 声明（带类型）
const props = defineProps({
  userInfo: {
    type: Object,
    required: true,
    validator: (value) => {
      return value.name && typeof value.age === 'number'
    }
  },
  isEditable: {
    type: Boolean,
    default: false
  }
})

// Emits 声明
const emit = defineEmits({
  // 带验证的 emit
  update: (payload) => {
    if (!payload.name) {
      console.warn('更新事件缺少 name')
      return false
    }
    return true
  },
  delete: null  // 无验证
})

function handleEdit() {
  const updated = { ...props.userInfo, age: props.userInfo.age + 1 }
  emit('update', updated)
}

function handleDelete() {
  emit('delete', props.userInfo.id)
}
</script>
```

#### Provide / Inject (跨层级通信)

```javascript
// 祖先组件
<script setup>
import { provide, ref, readonly } from 'vue'

// 响应式状态
const theme = ref('light')
const user = ref(null)

// 提供只读状态
provide('theme', readonly(theme))
provide('user', readonly(user))

// 提供修改方法
provide('setTheme', (newTheme) => {
  theme.value = newTheme
})

provide('setUser', (newUser) => {
  user.value = newUser
})
</script>
```

```javascript
// 后代组件
<script setup>
import { inject } from 'vue'

// 注入状态
const theme = inject('theme')
const user = inject('user')

// 注入方法
const setTheme = inject('setTheme')

function toggleTheme() {
  setTheme(theme.value === 'light' ? 'dark' : 'light')
}
</script>
```

---

### 4. 路由设计

#### 路由配置

```javascript
// router/index.js
import { createRouter, createWebHistory } from 'vue-router'

// 路由懒加载
const Home = () => import('@/views/Home.vue')
const MainView = () => import('@/views/MainView.vue')
const SimulationRunView = () => import('@/views/SimulationRunView.vue')
const ReportView = () => import('@/views/ReportView.vue')
const InteractionView = () => import('@/views/InteractionView.vue')

const routes = [
  {
    path: '/',
    name: 'Home',
    component: Home,
    meta: {
      title: 'MiroFish - 群体智能预测引擎'
    }
  },
  {
    path: '/main',
    name: 'MainView',
    component: MainView,
    meta: {
      requiresAuth: false,
      title: '工作流'
    },
    children: [
      {
        path: '',
        redirect: { name: 'MainView', query: { step: 1 } }
      }
    ]
  },
  {
    path: '/simulation/:id',
    name: 'SimulationRun',
    component: SimulationRunView,
    meta: {
      title: '模拟运行'
    },
    props: true  // 将路由参数作为 props 传递
  },
  {
    path: '/report/:id',
    name: 'ReportView',
    component: ReportView,
    meta: {
      title: '分析报告'
    },
    props: true
  },
  {
    path: '/interaction/:id',
    name: 'InteractionView',
    component: InteractionView,
    meta: {
      title: '交互分析'
    },
    props: true
  },
  {
    // 404 页面
    path: '/:pathMatch(.*)*',
    name: 'NotFound',
    component: () => import('@/views/NotFound.vue')
  }
]

const router = createRouter({
  history: createWebHistory(),
  routes,
  scrollBehavior(to, from, savedPosition) {
    // 滚动行为
    if (savedPosition) {
      return savedPosition
    }
    return { top: 0 }
  }
})

// 全局前置守卫
router.beforeEach((to, from, next) => {
  // 设置页面标题
  document.title = to.meta.title || 'MiroFish'

  // 权限检查
  if (to.meta.requiresAuth && !isAuthenticated()) {
    next({ name: 'Login', query: { redirect: to.fullPath } })
  } else {
    next()
  }
})

// 全局后置钩子
router.afterEach((to, from) => {
  // 页面访问统计
  trackPageView(to.path)
})

export default router

function isAuthenticated() {
  return localStorage.getItem('token') !== null
}

function trackPageView(path) {
  // 实现页面追踪
  console.log('Page view:', path)
}
```

#### 组件内导航

```vue
<script setup>
import { useRouter, useRoute } from 'vue-router'

const router = useRouter()
const route = useRoute()

// 编程式导航
function goToNextStep() {
  router.push({ name: 'MainView', query: { step: 2 } })
}

function goBack() {
  router.back()
}

// 获取路由参数
const simulationId = route.params.id
const step = route.query.step || 1
</script>
```

---

### 5. D3.js 图谱可视化

#### GraphPanel 组件实现

```vue
<!-- components/visualization/GraphPanel.vue -->
<template>
  <div class="graph-panel" ref="containerRef">
    <!-- 加载状态 -->
    <div v-if="isLoading" class="loading-overlay">
      <LoadingSpinner />
    </div>

    <!-- 空状态 -->
    <div v-else-if="!hasData" class="empty-state">
      <p>暂无图谱数据</p>
    </div>

    <!-- 图谱控制 -->
    <div v-if="hasData" class="graph-controls">
      <button @click="handleZoomIn" title="放大">+</button>
      <button @click="handleZoomOut" title="缩小">-</button>
      <button @click="handleResetZoom" title="重置">⟲</button>
      <button @click="handleExport" title="导出">⬇</button>
    </div>

    <!-- 节点信息提示 -->
    <div
      v-if="hoveredNode"
      class="node-tooltip"
      :style="{ left: tooltipX + 'px', top: tooltipY + 'px' }"
    >
      <h3>{{ hoveredNode.name }}</h3>
      <p>{{ hoveredNode.type }}</p>
    </div>
  </div>
</template>

<script setup>
/**
 * GraphPanel - D3.js 力导向图谱可视化
 *
 * 职责：
 * 1. 渲染节点和边
 * 2. 处理缩放和拖拽
 * 3. 响应式更新
 */

import { ref, computed, onMounted, watch, onUnmounted } from 'vue'
import * as d3 from 'd3'

// ==================== Props ====================
const props = defineProps({
  nodes: {
    type: Array,
    default: () => []
  },
  links: {
    type: Array,
    default: () => []
  },
  width: {
    type: Number,
    default: 800
  },
  height: {
    type: Number,
    default: 600
  },
  nodeSize: {
    type: Number,
    default: 8
  }
})

// ==================== Emits ====================
const emit = defineEmits(['nodeClick', 'nodeHover'])

// ==================== 状态 ====================
const containerRef = ref(null)
const isLoading = ref(false)
const hoveredNode = ref(null)
const tooltipX = ref(0)
const tooltipY = ref(0)

// D3 对象引用
let svg = null
let simulation = null
let zoom = null
let g = null

// ==================== 计算属性 ====================
const hasData = computed(() => {
  return props.nodes.length > 0
})

// ==================== 方法 ====================

/**
 * 初始化 SVG
 */
function initSvg() {
  if (!containerRef.value) return

  // 清空容器
  d3.select(containerRef.value).selectAll('*').remove()

  // 创建 SVG
  svg = d3.select(containerRef.value)
    .append('svg')
    .attr('width', '100%')
    .attr('height', '100%')
    .attr('viewBox', [0, 0, props.width, props.height])

  // 创建缩放行为
  zoom = d3.zoom()
    .scaleExtent([0.1, 4])
    .on('zoom', (event) => {
      g.attr('transform', event.transform)
    })

  svg.call(zoom)

  // 创建主组
  g = svg.append('g')
}

/**
 * 渲染图谱
 */
function renderGraph() {
  if (!g || props.nodes.length === 0) return

  // 创建力导向模拟
  simulation = d3.forceSimulation(props.nodes)
    .force('link', d3.forceLink(props.links)
      .id(d => d.id)
      .distance(100)
    )
    .force('charge', d3.forceManyBody()
      .strength(-300)
    )
    .force('center', d3.forceCenter(
      props.width / 2,
      props.height / 2
    )
    )
    .force('collision', d3.forceCollide()
      .radius(props.nodeSize + 2)
    )

  // 绘制连线
  const link = g.append('g')
    .attr('class', 'links')
    .selectAll('line')
    .data(props.links)
    .join('line')
    .attr('stroke', '#999')
    .attr('stroke-opacity', 0.6)
    .attr('stroke-width', d => Math.sqrt(d.value || 1))

  // 绘制节点
  const node = g.append('g')
    .attr('class', 'nodes')
    .selectAll('circle')
    .data(props.nodes)
    .join('circle')
    .attr('r', props.nodeSize)
    .attr('fill', d => getNodeColor(d.type))
    .attr('stroke', '#fff')
    .attr('stroke-width', 1.5)
    .style('cursor', 'pointer')

  // 节点交互
  node
    .on('mouseover', handleNodeMouseOver)
    .on('mouseout', handleNodeMouseOut)
    .on('click', handleNodeClick)

  // 拖拽行为
  node.call(d3.drag()
    .on('start', dragStarted)
    .on('drag', dragged)
    .on('end', dragEnded)
  )

  // 更新位置
  simulation.on('tick', () => {
    link
      .attr('x1', d => d.source.x)
      .attr('y1', d => d.source.y)
      .attr('x2', d => d.target.x)
      .attr('y2', d => d.target.y)

    node
      .attr('cx', d => d.x)
      .attr('cy', d => d.y)
  })
}

/**
 * 根据类型获取节点颜色
 */
function getNodeColor(type) {
  const colors = {
    'Person': '#ff6b6b',
    'Organization': '#4ecdc4',
    'Location': '#ffe66d',
    'Event': '#95e1d3',
    'default': '#a8a8a8'
  }
  return colors[type] || colors.default
}

/**
 * 节点悬停处理
 */
function handleNodeMouseOver(event, d) {
  hoveredNode.value = d

  // 高亮连接的边
  d3.selectAll('.links line')
    .attr('stroke-opacity', l =>
      (l.source === d || l.target === d) ? 1 : 0.1
    )

  // 高亮相关节点
  d3.selectAll('.nodes circle')
    .attr('opacity', n =>
      (n === d || isConnected(d, n)) ? 1 : 0.3
    )

  // 更新提示位置
  const [x, y] = d3.pointer(event, containerRef.value)
  tooltipX.value = x + 10
  tooltipY.value = y + 10

  emit('nodeHover', d)
}

/**
 * 节点移出处理
 */
function handleNodeMouseOut() {
  hoveredNode.value = null

  // 恢复所有边的透明度
  d3.selectAll('.links line')
    .attr('stroke-opacity', 0.6)

  // 恢复所有节点的透明度
  d3.selectAll('.nodes circle')
    .attr('opacity', 1)
}

/**
 * 节点点击处理
 */
function handleNodeClick(event, d) {
  emit('nodeClick', d)
}

/**
 * 检查两节点是否相连
 */
function isConnected(a, b) {
  return props.links.some(l =>
    (l.source === a && l.target === b) ||
    (l.source === b && l.target === a)
  )
}

/**
 * 拖拽开始
 */
function dragStarted(event, d) {
  if (!event.active) simulation.alphaTarget(0.3).restart()
  d.fx = d.x
  d.fy = d.y
}

/**
 * 拖拽中
 */
function dragged(event, d) {
  d.fx = event.x
  d.fy = event.y
}

/**
 * 拖拽结束
 */
function dragEnded(event, d) {
  if (!event.active) simulation.alphaTarget(0)
  d.fx = null
  d.fy = null
}

/**
 * 放大
 */
function handleZoomIn() {
  svg.transition().call(
    zoom.scaleBy, 1.3
  )
}

/**
 * 缩小
 */
function handleZoomOut() {
  svg.transition().call(
    zoom.scaleBy, 0.7
  )
}

/**
 * 重置缩放
 */
function handleResetZoom() {
  svg.transition().call(
    zoom.transform,
    d3.zoomIdentity
  )
}

/**
 * 导出图谱
 */
function handleExport() {
  const svgData = new XMLSerializer().serializeToString(
    containerRef.value.querySelector('svg')
  )

  const blob = new Blob([svgData], {
    type: 'image/svg+xml;charset=utf-8'
  })

  const url = URL.createObjectURL(blob)
  const link = document.createElement('a')
  link.href = url
  link.download = 'graph.svg'
  link.click()
  URL.revokeObjectURL(url)
}

// ==================== 生命周期 ====================
onMounted(() => {
  initSvg()
  renderGraph()
})

onUnmounted(() => {
  // 清理模拟
  if (simulation) {
    simulation.stop()
  }
})

// ==================== 监听器 ====================
watch(
  () => [props.nodes, props.links],
  () => {
    // 重新渲染
    if (g) {
      g.selectAll('*').remove()
      renderGraph()
    }
  },
  { deep: true }
)
</script>

<style scoped>
.graph-panel {
  @apply relative w-full h-full bg-gray-50 rounded-lg overflow-hidden;
}

.loading-overlay,
.empty-state {
  @apply absolute inset-0 flex items-center justify-center;
}

.graph-controls {
  @apply absolute top-4 right-4 flex gap-2;
}

.graph-controls button {
  @apply w-8 h-8 flex items-center justify-center
         bg-white rounded shadow hover:bg-gray-100;
}

.node-tooltip {
  @apply absolute bg-white p-3 rounded shadow-lg
         pointer-events-none z-50 min-w-48;
}

.node-tooltip h3 {
  @apply font-semibold text-gray-800;
}

.node-tooltip p {
  @apply text-sm text-gray-600;
}
</style>
```

---

### 6. API 客户端设计

#### Axios 配置

```javascript
// api/index.js
import axios from 'axios'

/**
 * 创建 Axios 实例
 */
const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || 'http://localhost:5001',
  timeout: 60000,
  headers: {
    'Content-Type': 'application/json'
  }
})

/**
 * 请求拦截器
 */
apiClient.interceptors.request.use(
  (config) => {
    // 添加认证 token
    const token = localStorage.getItem('token')
    if (token) {
      config.headers.Authorization = `Bearer ${token}`
    }

    // 添加请求 ID（用于追踪）
    config.headers['X-Request-ID'] = generateRequestId()

    // 日志
    if (import.meta.env.DEV) {
      console.log(`[API Request] ${config.method.toUpperCase()} ${config.url}`, config.data)
    }

    return config
  },
  (error) => {
    console.error('[API Request Error]', error)
    return Promise.reject(error)
  }
)

/**
 * 响应拦截器
 */
apiClient.interceptors.response.use(
  (response) => {
    // 成功响应
    if (import.meta.env.DEV) {
      console.log(`[API Response] ${response.config.url}`, response.data)
    }

    return response.data
  },
  (error) => {
    // 错误处理
    const { response, request, message } = error

    if (response) {
      // 服务器响应了错误状态码
      handleServerError(response)
    } else if (request) {
      // 请求发送了但没有收到响应
      handleNetworkError(error)
    } else {
      // 请求配置出错
      handleRequestError(error)
    }

    return Promise.reject(error)
  }
)

/**
 * 处理服务器错误
 */
function handleServerError(response) {
  const { status, data } = response

  switch (status) {
    case 401:
      // 未授权 - 清除 token 并跳转登录
      localStorage.removeItem('token')
      window.location.href = '/login'
      break

    case 403:
      // 禁止访问
      console.error('没有权限执行此操作')
      break

    case 404:
      // 资源不存在
      console.error('请求的资源不存在')
      break

    case 500:
      // 服务器错误
      console.error('服务器内部错误:', data.error)
      break

    default:
      console.error(`请求失败: ${status}`)
  }
}

/**
 * 处理网络错误
 */
function handleNetworkError(error) {
  console.error('网络错误，请检查网络连接')

  // 可以显示全局错误提示
  showErrorToast('网络连接失败，请稍后重试')
}

/**
 * 处理请求错误
 */
function handleRequestError(error) {
  console.error('请求配置错误:', error.message)
}

/**
 * 生成请求 ID
 */
function generateRequestId() {
  return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`
}

/**
 * 显示错误提示
 */
function showErrorToast(message) {
  // 实现或集成 toast 组件
  // toast.error(message)
  console.error(message)
}

export default apiClient
```

#### API 模块示例

```javascript
// api/graph.js
import api from './index'

export default {
  /**
   * 生成本体定义
   */
  generateOntology(text, userRequirement = '社会模拟') {
    return api.post('/api/graph/generate-ontology', {
      text,
      user_requirement: userRequirement
    })
  },

  /**
   * 构建知识图谱
   */
  buildGraph(params) {
    return api.post('/api/graph/build', params)
  },

  /**
   * 获取任务状态
   */
  getTaskStatus(taskId) {
    return api.get(`/api/graph/task/${taskId}`)
  },

  /**
   * 获取实体列表
   */
  getEntities(graphId, options = {}) {
    const params = new URLSearchParams()

    if (options.limit) params.append('limit', options.limit)
    if (options.offset) params.append('offset', options.offset)
    if (options.type) params.append('type', options.type)

    return api.get(`/api/graph/entities/${graphId}?${params}`)
  },

  /**
   * 搜索实体
   */
  searchEntities(graphId, query) {
    return api.get(`/api/graph/entities/${graphId}/search?q=${query}`)
  }
}
```

---

## 性能优化

### 1. 路由懒加载

```javascript
// ✅ 好 - 懒加载
const Home = () => import('@/views/Home.vue')

// ❌ 不好 - 直接导入
import Home from '@/views/Home.vue'
```

### 2. 计算属性缓存

```vue
<script setup>
import { computed } from 'vue'

const items = ref([...])

// ✅ 好 - 使用计算属性（有缓存）
const filteredItems = computed(() => {
  return items.value.filter(item => item.active)
})

// ❌ 不好 - 使用方法（每次调用都重新计算）
function filteredItems() {
  return items.value.filter(item => item.active)
}
</script>
```

### 3. 虚拟滚动

```javascript
import { useVirtualList } from '@vueuse/core'

const list = ref([...])  // 大数组

const {
  list: virtualList,
  containerProps,
  wrapperProps
} = useVirtualList(list, { itemHeight: 50 })
```

---

## 学习检查清单

### 基础理解

- [ ] 能解释 Vue 3 的响应式原理
- [ ] 能使用 Composition API 编写组件
- [ ] 能实现父子组件通信
- [ ] 能配置路由和导航

### 进阶掌握

- [ ] 能设计 Composables 复用逻辑
- [ ] 能实现复杂的数据可视化
- [ ] 能处理异步状态管理
- [ ] 能进行错误处理

### 专家能力

- [ ] 能进行性能优化
- [ ] 能设计组件库架构
- [ ] 能处理大规模状态管理
- [ ] 能优化包体积

---

## 下一步学习

- **[API 接口文档](./api-reference.md)** - 完整的 API 参考
- **[后端架构详解](./backend-architecture.md)** - 了解后端实现
- **[核心服务详解](./core-services.md)** - 深入核心服务
