# API 接口文档

> MiroFish 后端 API 完整参考与最佳实践

**难度**: ⭐⭐ | **预计时间**: 45 分钟

---

## 学习目标

完成本章节学习后，你将能够：

### 基础目标（必掌握）

- [ ] 理解 MiroFish API 的整体设计
- [ ] 掌握图谱 API 的使用方法
- [ ] 掌握模拟 API 的使用方法
- [ ] 掌握报告 API 的使用方法
- [ ] 处理 API 错误和异常

### 进阶目标（建议掌握）

- [ ] 优化 API 调用性能
- [ ] 实现 API 调用重试机制
- [ ] 设计合理的 API 客户端封装
- [ ] 进行 API 调用的监控和调试

### 专家目标（挑战）

- [ ] 设计 API 版本控制策略
- [ ] 实现 API 限流和缓存
- [ ] 设计可扩展的 API 架构

---

## API 设计原则

### RESTful 设计

MiroFish 遵循 REST 架构风格：

| 原则 | 实现方式 | 示例 |
|------|----------|------|
| **资源导向** | URL 表示资源 | `/api/graph/entities/:graph_id` |
| **统一接口** | 标准 HTTP 方法 | GET/POST/PUT/DELETE |
| **无状态** | 每个请求独立 | 包含所有必要参数 |
| **分层系统** | API → Service → External | 清晰的层次结构 |

### 请求/响应契约

```
┌─────────────────────────────────────────────────────────────────────┐
│                      API 通信契约                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                              │
│  客户端                              服务端                      │
│  ┌─────────────────┐                ┌─────────────────┐            │
│  │                 │                │                 │            │
│  │  1. 构造请求     │────────────────>│  2. 验证请求     │            │
│  │     - 方法       │    HTTP         │     - 参数       │            │
│  │     - URL        │    Request      │     - 权限       │            │
│  │     - Headers    │                │     - 格式       │            │
│  │     - Body       │                │                 │            │
│  │                 │                │                 │            │
│  └─────────────────┘                └────────┬────────┘            │
│                                             │                      │
│                                             │ 3. 处理业务           │
│                                             │                      │
│  ┌─────────────────┐                ┌────────┴────────┘            │
│  │                 │                │                 │            │
│  │  5. 处理响应     │<────────────────│  4. 构造响应     │            │
│  │     - 解析数据   │    HTTP         │     - 状态码     │            │
│  │     - 错误处理   │    Response     │     - 数据       │            │
│  │     - 更新UI     │                │     - 错误信息   │            │
│  │                 │                │                 │            │
│  └─────────────────┘                └─────────────────┘            │
│                                                              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 通用规范

### Base URL

| 环境 | URL | 说明 |
|------|-----|------|
| **开发** | `http://localhost:5001` | 本地开发 |
| **生产** | `https://api.mirofish.example.com` | 生产环境（需配置） |

### 请求格式

**Headers**:

```http
Content-Type: application/json
Accept: application/json
Authorization: Bearer <token>  # 如需要
```

**Body**:

```json
{
  "param1": "value1",
  "param2": "value2"
}
```

### 响应格式

**成功响应**:

```json
{
  "success": true,
  "data": {
    // 业务数据
  }
}
```

**错误响应**:

```json
{
  "success": false,
  "error": "错误描述",
  "error_code": "ERROR_CODE",
  "details": {}
}
```

### HTTP 状态码

| 状态码 | 说明 | 使用场景 |
|--------|------|----------|
| **200** | OK | 请求成功 |
| **201** | Created | 资源创建成功 |
| **400** | Bad Request | 请求参数错误 |
| **401** | Unauthorized | 未授权 |
| **404** | Not Found | 资源不存在 |
| **500** | Internal Server Error | 服务器内部错误 |
| **503** | Service Unavailable | 服务不可用 |

---

## 图谱 API (`/api/graph`)

图谱 API 负责知识图谱的构建和管理。

### 1. 生成本体

生成适合特定用途的实体类型和关系类型定义。

**端点**: `POST /api/graph/generate-ontology`

**请求参数**:

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `text` | string | ✅ | 输入文本（用于分析） |
| `user_requirement` | string | ❌ | 用户需求描述，默认"社会模拟" |

**请求示例**:

```json
{
  "text": "贾宝玉是贾政与王夫人之子，衔玉而生...",
  "user_requirement": "红楼梦人物关系分析"
}
```

**响应示例**:

```json
{
  "success": true,
  "data": {
    "entity_types": [
      {
        "name": "Person",
        "description": "具有独立人格的人物",
        "attributes": ["age", "gender", "social_status"],
        "examples": ["贾宝玉", "林黛玉", "薛宝钗"]
      },
      {
        "name": "Organization",
        "description": "组织、家族、机构",
        "attributes": ["founded_date", "location", "size"],
        "examples": ["贾府", "荣国府", "朝廷"]
      },
      {
        "name": "Location",
        "description": "地理位置、场所",
        "attributes": ["type", "owner"],
        "examples": ["大观园", "荣国府", "北京"]
      }
    ],
    "relationship_types": [
      {
        "name": "家属关系",
        "description": "父母与子女、兄弟姐妹等",
        "direction": "bidirectional"
      },
      {
        "name": "情感关系",
        "description": "喜欢、爱慕、讨厌等情感",
        "direction": "directed"
      },
      {
        "name": "社会关系",
        "description": "主仆、朋友、同事等",
        "direction": "bidirectional"
      }
    ]
  }
}
```

**使用场景**:
- 上传新文档前，了解系统将识别哪些实体类型
- 根据分析需求调整本体定义
- 验证系统是否正确理解文本内容

### 2. 构建图谱

异步构建 GraphRAG 知识图谱。

**端点**: `POST /api/graph/build`

**请求参数**:

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `text` | string | ✅ | 要构建图谱的文本内容 |
| `ontology` | object | ✅ | 本体定义（来自生成本体 API） |
| `graph_name` | string | ❌ | 图谱名称 |
| `chunk_size` | int | ❌ | 文本分块大小，默认 500 |
| `chunk_overlap` | int | ❌ | 分块重叠大小，默认 50 |
| `session_id` | string | ❌ | 会话 ID，用于关联数据 |

**请求示例**:

```json
{
  "text": "红楼梦全文...",
  "ontology": {
    "entity_types": [...],
    "relationship_types": [...]
  },
  "graph_name": "红楼梦人物关系",
  "chunk_size": 500,
  "chunk_overlap": 50,
  "session_id": "session_abc123"
}
```

**响应示例**:

```json
{
  "success": true,
  "data": {
    "task_id": "graph_build_abc123",
    "status": "pending",
    "estimated_time_seconds": 120
  }
}
```

**异步处理流程**:

```
客户端 POST /api/graph/build
         │
         ▼
┌─────────────────┐
│  立即响应        │  → task_id: "graph_build_abc123"
└─────────────────┘
         │
         ▼ (后台处理开始)
┌─────────────────────────────────────┐
│  1. 文本分块                        │
│  2. 调用 Zep API 上传文档            │
│  3. 提取实体和关系                  │
│  4. 构建图结构                     │
│  5. 检测社群                       │
└─────────────────────────────────────┘
         │
         ▼ (处理完成)
    task.status = "completed"
```

### 3. 查询任务状态

查询图谱构建任务的状态和进度。

**端点**: `GET /api/graph/task/:task_id`

**路径参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| `task_id` | string | 任务 ID |

**响应示例**:

```json
{
  "success": true,
  "data": {
    "task_id": "graph_build_abc123",
    "status": "completed",
    "progress": 1.0,
    "result": {
      "graph_id": "graph_xyz789",
      "node_count": 127,
      "edge_count": 389,
      "community_count": 5
    },
    "error": null
  }
}
```

**状态值说明**:

| 状态 | 说明 |
|------|------|
| `pending` | 等待处理 |
| `processing` | 处理中 |
| `completed` | 已完成 |
| `failed` | 处理失败 |

### 4. 获取图谱实体

获取图谱中的实体列表，支持筛选和分页。

**端点**: `GET /api/graph/entities/:graph_id`

**路径参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| `graph_id` | string | 图谱 ID |

**查询参数**:

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `entity_types` | string | ❌ | 实体类型过滤，逗号分隔 |
| `limit` | int | ❌ | 返回数量限制，默认 100 |
| `offset` | int | ❌ | 偏移量，默认 0 |
| `search` | string | ❌ | 搜索关键词 |

**请求示例**:

```http
GET /api/graph/entities/graph_xyz789?entity_types=Person,Organization&limit=50&offset=0
```

**响应示例**:

```json
{
  "success": true,
  "data": {
    "entities": [
      {
        "uuid": "uuid-001",
        "name": "贾宝玉",
        "labels": ["Person"],
        "summary": "贾政与王夫人之子，衔玉而生，性情温和...",
        "attributes": {
          "age": 17,
          "gender": "male",
          "social_status": "贵族"
        },
        "relationship_count": 25
      },
      {
        "uuid": "uuid-002",
        "name": "林黛玉",
        "labels": ["Person"],
        "summary": "贾府表小姐，多愁善感，才华横溢...",
        "attributes": {
          "age": 16,
          "gender": "female",
          "social_status": "贵族"
        },
        "relationship_count": 18
      }
    ],
    "total_count": 127,
    "filtered_count": 45
  }
}
```

### 5. 搜索实体

在图谱中搜索匹配的实体。

**端点**: `GET /api/graph/search/:graph_id`

**查询参数**:

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `q` | string | ✅ | 搜索关键词 |
| `type` | string | ❌ | 实体类型过滤 |

**请求示例**:

```http
GET /api/graph/search/graph_xyz789?q=贾&type=Person
```

**响应示例**:

```json
{
  "success": true,
  "data": {
    "results": [
      {
        "uuid": "uuid-001",
        "name": "贾宝玉",
        "labels": ["Person"],
        "match_score": 0.95,
        "matched_fields": ["name"]
      },
      {
        "uuid": "uuid-045",
        "name": "贾政",
        "labels": ["Person"],
        "match_score": 0.90,
        "matched_fields": ["name"]
      }
    ]
  }
}
```

---

## 模拟 API (`/api/simulation`)

模拟 API 负责社交模拟的完整生命周期管理。

### 1. 创建模拟

创建一个新的社交模拟实例。

**端点**: `POST /api/simulation/create`

**请求参数**:

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `project_id` | string | ✅ | 项目 ID |
| `graph_id` | string | ✅ | 图谱 ID |
| `enable_twitter` | bool | ❌ | 启用 Twitter 平台，默认 true |
| `enable_reddit` | bool | ❌ | 启用 Reddit 平台，默认 false |

**请求示例**:

```json
{
  "project_id": "project_abc123",
  "graph_id": "graph_xyz789",
  "enable_twitter": true,
  "enable_reddit": false
}
```

**响应示例**:

```json
{
  "success": true,
  "data": {
    "simulation_id": "sim_def456",
    "status": "created",
    "created_at": "2024-01-15T10:30:00Z"
  }
}
```

### 2. 准备模拟环境

为模拟生成智能体 Profile 和配置。

**端点**: `POST /api/simulation/prepare`

**请求参数**:

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `simulation_id` | string | ✅ | 模拟 ID |
| `simulation_requirement` | string | ❌ | 预测需求描述 |
| `document_text` | string | ❌ | 原始文档文本 |
| `defined_entity_types` | array | ❌ | 选择的实体类型 |
| `use_llm_for_profiles` | bool | ❌ | 使用 LLM 生成 Profile，默认 true |
| `parallel_profile_count` | int | ❌ | 并行生成数，默认 3 |

**请求示例**:

```json
{
  "simulation_id": "sim_def456",
  "simulation_requirement": "预测贾宝玉和林黛玉的关系走向",
  "document_text": "红楼梦前80回...",
  "defined_entity_types": ["Person"],
  "use_llm_for_profiles": true,
  "parallel_profile_count": 3
}
```

**响应示例**:

```json
{
  "success": true,
  "data": {
    "simulation_id": "sim_def456",
    "status": "ready",
    "entities_count": 45,
    "profiles_count": 45,
    "config": {
      "max_rounds": 40,
      "platforms": ["twitter"]
    }
  }
}
```

### 3. 启动模拟

启动一个已准备好的模拟。

**端点**: `POST /api/simulation/start`

**请求参数**:

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `simulation_id` | string | ✅ | 模拟 ID |

**响应示例**:

```json
{
  "success": true,
  "data": {
    "simulation_id": "sim_def456",
    "status": "running",
    "started_at": "2024-01-15T11:00:00Z"
  }
}
```

### 4. 获取模拟状态

获取模拟的实时状态和进度。

**端点**: `GET /api/simulation/status/:simulation_id`

**路径参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| `simulation_id` | string | 模拟 ID |

**响应示例**:

```json
{
  "success": true,
  "data": {
    "simulation_id": "sim_def456",
    "status": "running",
    "current_round": 15,
    "total_rounds": 40,
    "progress": 0.375,
    "platforms": {
      "twitter": {
        "status": "running",
        "actions_count": 456
      },
      "reddit": {
        "status": "not_started",
        "actions_count": 0
      }
    },
    "total_actions": 456,
    "estimated_remaining_seconds": 600
  }
}
```

### 5. 获取动作记录

获取模拟中的动作记录，支持按轮次筛选。

**端点**: `GET /api/simulation/actions/:simulation_id`

**查询参数**:

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `round` | int | ❌ | 轮次筛选 |
| `platform` | string | ❌ | 平台筛选 |
| `agent_id` | int | ❌ | 智能体筛选 |
| `limit` | int | ❌ | 返回数量，默认 50 |
| `offset` | int | ❌ | 偏移量，默认 0 |

**请求示例**:

```http
GET /api/simulation/actions/sim_def456?round=15&platform=twitter&limit=20
```

**响应示例**:

```json
{
  "success": true,
  "data": {
    "actions": [
      {
        "id": "action_001",
        "round_num": 15,
        "timestamp": "2024-01-15T11:05:00Z",
        "platform": "twitter",
        "agent_id": 1,
        "agent_name": "贾宝玉",
        "action_type": "CREATE_POST",
        "content": "今日读诗，颇有感慨...",
        "metadata": {
          "likes": 2,
          "retweets": 0
        }
      },
      {
        "id": "action_002",
        "round_num": 15,
        "timestamp": "2024-01-15T11:05:30Z",
        "platform": "twitter",
        "agent_id": 2,
        "agent_name": "林黛玉",
        "action_type": "LIKE_POST",
        "content": null,
        "metadata": {
          "target_action_id": "action_001"
        }
      }
    ],
    "total_count": 456,
    "round_count": 23
  }
}
```

### 6. 智能体访谈

与模拟中的智能体进行实时对话。

**端点**: `POST /api/simulation/interview`

**请求参数**:

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `simulation_id` | string | ✅ | 模拟 ID |
| `agent_id` | int | ✅ | 智能体 ID |
| `prompt` | string | ✅ | 问题/提示 |
| `platform` | string | ❌ | 指定平台，null 表示自动 |

**请求示例**:

```json
{
  "simulation_id": "sim_def456",
  "agent_id": 1,
  "prompt": "你对林黛玉最近怎么看？",
  "platform": null
}
```

**响应示例**:

```json
{
  "success": true,
  "data": {
    "agent_id": 1,
    "agent_name": "贾宝玉",
    "response": "林妹妹最近的帖子透着一股忧伤...",
    "platform": "twitter",
    "timestamp": "2024-01-15T11:10:00Z"
  }
}
```

### 7. 暂停/继续模拟

控制模拟的执行。

**端点**: `POST /api/simulation/pause` 或 `POST /api/simulation/resume`

**请求参数**:

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `simulation_id` | string | ✅ | 模拟 ID |

### 8. 停止模拟

停止正在运行的模拟。

**端点**: `POST /api/simulation/stop`

**请求参数**:

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `simulation_id` | string | ✅ | 模拟 ID |

---

## 报告 API (`/api/report`)

报告 API 负责预测报告的生成和交互。

### 1. 生成报告

使用 Report Agent 生成预测报告。

**端点**: `POST /api/report/generate`

**请求参数**:

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `simulation_id` | string | ✅ | 模拟 ID |
| `report_requirement` | string | ✅ | 预测需求描述 |
| `max_tool_calls` | int | ❌ | 最大工具调用次数，默认 5 |
| `max_reflection_rounds` | int | ❌ | 反思轮数，默认 2 |
| `temperature` | float | ❌ | LLM 温度，默认 0.5 |

**请求示例**:

```json
{
  "simulation_id": "sim_def456",
  "report_requirement": "分析贾宝玉和林黛玉的关系走向，预测贾府衰败对两人关系的影响，给出三种可能的结局",
  "max_tool_calls": 7,
  "max_reflection_rounds": 2,
  "temperature": 0.5
}
```

**响应示例**:

```json
{
  "success": true,
  "data": {
    "report_id": "report_ghi789",
    "status": "generating",
    "created_at": "2024-01-15T12:00:00Z"
  }
}
```

### 2. 获取报告状态

查询报告生成状态。

**端点**: `GET /api/report/status/:report_id`

**响应示例**:

```json
{
  "success": true,
  "data": {
    "report_id": "report_ghi789",
    "status": "completed",
    "title": "红楼梦人物关系预测报告",
    "content": "# 执行摘要\n\n...",
    "metadata": {
      "tool_calls_count": 5,
      "reflection_rounds": 2,
      "generated_at": "2024-01-15T12:05:00Z"
    }
  }
}
```

### 3. 与 Report Agent 对话

与已生成的报告进行深度对话。

**端点**: `POST /api/report/chat`

**请求参数**:

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `report_id` | string | ✅ | 报告 ID |
| `message` | string | ✅ | 用户消息 |

**请求示例**:

```json
{
  "report_id": "report_ghi789",
  "message": "报告中提到关系会恶化，具体是哪些事件导致的？"
}
```

**响应示例**:

```json
{
  "success": true,
  "data": {
    "response": "根据模拟记录分析，关系恶化主要由以下关键事件导致...",
    "tool_calls": [
      {
        "tool": "PanoramaSearch",
        "query": "贾宝玉 林黛玉 互动记录"
      }
    ]
  }
}
```

---

## 错误处理

### 错误码清单

| 错误码 | HTTP 状态 | 说明 | 解决方案 |
|--------|----------|------|----------|
| `INVALID_REQUEST` | 400 | 请求参数无效 | 检查请求参数格式和内容 |
| `MISSING_PARAMETER` | 400 | 缺少必需参数 | 补充必需参数 |
| `INVALID_ONTOLOGY` | 400 | 本体格式错误 | 检查 ontology 结构 |
| `GRAPH_NOT_FOUND` | 404 | 图谱不存在 | 检查 graph_id 是否正确 |
| `SIMULATION_NOT_FOUND` | 404 | 模拟不存在 | 检查 simulation_id 是否正确 |
| `REPORT_NOT_FOUND` | 404 | 报告不存在 | 检查 report_id 是否正确 |
| `LLM_API_ERROR` | 500 | LLM API 调用失败 | 检查 API Key 和网络 |
| `ZEP_API_ERROR` | 500 | Zep API 调用失败 | 检查 Zep API Key |
| `SIMULATION_FAILED` | 500 | 模拟运行失败 | 查看模拟日志 |
| `INTERNAL_ERROR` | 500 | 服务器内部错误 | 联系管理员 |

### 错误处理示例

**Python**:

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

class MiroFishAPI:
    def __init__(self, base_url: str, api_key: str = None):
        self.base_url = base_url.rstrip('/')
        self.api_key = api_key
        self.session = self._create_session()

    def _create_session(self):
        """创建带重试的会话"""
        session = requests.Session()

        # 配置重试策略
        retry_strategy = Retry(
            total=3,
            backoff_factor=1,
            status_forcelist=[429, 500, 502, 503, 504]
        )
        adapter = HTTPAdapter(max_retries=retry_strategy)
        session.mount("http://", adapter)
        session.mount("https://", adapter)

        return session

    def _request(self, method: str, endpoint: str, **kwargs):
        """统一的请求处理"""
        url = f"{self.base_url}{endpoint}"

        # 添加认证头
        headers = kwargs.pop('headers', {})
        if self.api_key:
            headers['Authorization'] = f'Bearer {self.api_key}'

        try:
            response = self.session.request(
                method,
                url,
                headers=headers,
                timeout=30,
                **kwargs
            )
            response.raise_for_status()
            return response.json()

        except requests.exceptions.HTTPError as e:
            error_data = e.response.json()
            raise APIException(error_data.get('error'), error_data)

        except requests.exceptions.Timeout:
            raise APIException("请求超时，请稍后重试")

        except requests.exceptions.ConnectionError:
            raise APIException("网络连接失败，请检查网络")

    def generate_ontology(self, text: str, user_requirement: str = "社会模拟"):
        """生成本体"""
        return self._request(
            'POST',
            '/api/graph/generate-ontology',
            json={
                'text': text,
                'user_requirement': user_requirement
            }
        )


class APIException(Exception):
    """API 异常"""
    pass
```

**JavaScript**:

```javascript
import axios from 'axios';

class MiroFishAPI {
  constructor(baseUrl, apiKey = null) {
    this.baseURL = baseUrl.replace(/\/$/, '');
    this.apiKey = apiKey;
    this.client = axios.create({
      baseURL: this.baseURL,
      timeout: 30000,
      headers: {
        'Content-Type': 'application/json'
      }
    });

    this._setupInterceptors();
  }

  _setupInterceptors() {
    // 请求拦截器
    this.client.interceptors.request.use(
      config => {
        if (this.apiKey) {
          config.headers.Authorization = `Bearer ${this.apiKey}`;
        }
        return config;
      },
      error => Promise.reject(error)
    );

    // 响应拦截器
    this.client.interceptors.response.use(
      response => response.data,
      error => {
        if (error.response) {
          const { status, data } = error.response;
          throw new APIError(
            data.error || '请求失败',
            status,
            data.error_code
          );
        }
        if (error.request) {
          throw new APIError('网络连接失败', null, 'NETWORK_ERROR');
        }
        throw new APIError(error.message, null, 'UNKNOWN_ERROR');
      }
    );
  }

  async generateOntology(text, userRequirement = '社会模拟') {
    return await this.client.post('/api/graph/generate-ontology', {
      text,
      user_requirement: userRequirement
    });
  }
}

class APIError extends Error {
  constructor(message, status, code) {
    super(message);
    this.name = 'APIError';
    this.status = status;
    this.code = code;
  }
}
```

---

## 最佳实践

### 1. 轮询策略

对于异步任务，使用合理的轮询间隔：

```python
import time

def poll_task(task_id: str, interval: int = 2, max_attempts: int = 150):
    """轮询任务状态"""
    for i in range(max_attempts):
        result = api.get_task_status(task_id)

        if result['data']['status'] == 'completed':
            return result['data']['result']

        if result['data']['status'] == 'failed':
            raise Exception(result['data']['error'])

        # 指数退避
        sleep_time = interval * (1.1 ** min(i, 10))
        time.sleep(sleep_time)

    raise TimeoutError("任务超时")
```

### 2. 批量操作

批量访谈智能体时，使用并发：

```python
from concurrent.futures import ThreadPoolExecutor

def batch_interview(simulation_id: str, agent_ids: list, question: str):
    """批量访谈"""
    def interview_one(agent_id):
        return api.interview_agent(simulation_id, agent_id, question)

    with ThreadPoolExecutor(max_workers=5) as executor:
        results = list(executor.map(interview_one, agent_ids))

    return results
```

### 3. 缓存策略

缓存不经常变化的数据：

```python
from functools import lru_cache
import hashlib

class CachedAPI:
    def __init__(self, api):
        self.api = api
        self._cache = {}

    def get_entities(self, graph_id, use_cache=True):
        cache_key = f"entities:{graph_id}"

        if use_cache and cache_key in self._cache:
            return self._cache[cache_key]

        result = self.api.get_entities(graph_id)
        self._cache[cache_key] = result
        return result
```

---

## 学习检查清单

### 基础理解

- [ ] 能解释 RESTful API 设计原则
- [ ] 能使用 Postman 或 curl 调用 API
- [ ] 能处理 JSON 请求和响应
- [ ] 能识别和处理常见错误

### 进阶掌握

- [ ] 能实现 API 客户端封装
- [ ] 能设计合理的轮询策略
- [ ] 能处理异步操作
- [ ] 能优化 API 调用性能

### 专家能力

- [ ] 能设计 API 版本控制
- [ ] 能实现限流和缓存
- [ ] 能设计可扩展的架构
- [ ] 能监控和调试 API 问题

---

## 下一步

- **[核心服务详解](./core-services.md)** - 深入了解服务实现
- **[后端架构详解](./backend-architecture.md)** - 了解架构设计
