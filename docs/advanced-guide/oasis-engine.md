# OASIS 模拟引擎深度解析

> 深入理解 OASIS 社交模拟的原理和定制

**难度**: ⭐⭐⭐⭐⭐ | **预计时间**: 90 分钟

---

## 学习目标

完成本章节后，你将能够：

### 基础目标（必掌握）

- [ ] 理解 OASIS 模拟引擎的核心架构
- [ ] 掌握 Agent Profile 的结构和生成方式
- [ ] 理解模拟循环的基本流程
- [ ] 掌握 Twitter 和 Reddit 平台的模拟机制

### 进阶目标（建议掌握）

- [ ] 理解 Agent 决策机制的实现原理
- [ ] 能够定制 Profile 生成逻辑
- [ ] 能够调整模拟参数优化效果
- [ ] 理解平台抽象层的设计

### 专家目标（挑战）

- [ ] 能够为 MiroFish 添加新的社交平台支持
- [ ] 能够设计新的 Agent 行为模式
- [ ] 能够进行大规模模拟的性能优化
- [ ] 能够扩展 OASIS 引擎功能

---

## 什么是 OASIS？

OASIS (Open Agent Social Interaction Simulation) 是一个专业的社交媒体模拟引擎，由 CAMEL-AI 团队开发。

### 核心特性

| 特性 | 说明 |
|------|------|
| **真实平台模拟** | Twitter、Reddit 等社交平台 |
| **智能体驱动** | 每个 Agent 有独立人格和记忆 |
| **行动系统** | 丰富的社交动作（发帖、评论、点赞等）|
| **状态管理** | 模拟环境状态的动态变化 |

---

## OASIS 架构

### 分层架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                        │
│                  (MiroFish Integration)                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────┐          │
│  │         Simulation Manager                    │          │
│  │  · 模拟生命周期管理                             │          │
│  │  · Profile 转换                               │          │
│  │  · 配置生成                                   │          │
│  └──────────────────────────────────────────────┘          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                    OASIS Framework Layer                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────┐          │
│  │         Agent Layer                           │          │
│  │  · Agent Definition                           │          │
│  │  · Role & Personality                         │          │
│  │  · Memory System                              │          │
│  └──────────────────────────────────────────────┘          │
│                        │                                    │
│  ┌──────────────────────────────────────────────┐          │
│  │         Action Layer                          │          │
│  │  · Platform Actions                          │          │
│  │  · Action Validator                          │          │
│  │  · Response Generator                        │          │
│  └──────────────────────────────────────────────┘          │
│                        │                                    │
│  ┌──────────────────────────────────────────────┐          │
│  │         Environment Layer                     │          │
│  │  · Platform (Twitter/Reddit)                 │          │
│  │  · State Management                          │          │
│  │  · Event System                              │          │
│  └──────────────────────────────────────────────┘          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## MiroFish 中的集成

### 1. Profile 转换

MiroFish 需要将 Zep 实体转换为 OASIS Agent Profile。

#### Twitter Profile 格式

```python
def convert_to_twitter_profile(entity: Dict, profile_data: Dict) -> Dict:
    """转换为 Twitter Profile"""
    return {
        "user_id": entity["index"],
        "username": generate_username(entity["name"]),
        "name": entity["name"],
        "bio": profile_data.get("bio", ""),
        "followers_count": random.randint(10, 1000),
        "following_count": random.randint(10, 500),
        "verified": False,
        "protected": False
    }
```

#### Reddit Profile 格式

```python
def convert_to_reddit_profile(entity: Dict, profile_data: Dict) -> Dict:
    """转换为 Reddit Profile"""
    return {
        "user_id": entity["index"],
        "username": generate_username(entity["name"]),
        "name": entity["name"],
        "bio": profile_data.get("bio", ""),
        "karma": random.randint(100, 5000),
        "subreddits": profile_data.get("interests", ["general"])
    }
```

### 2. 模拟配置

```python
def generate_oasis_config(
    simulation_id: str,
    profiles: List[Dict],
    parameters: SimulationParameters
) -> Dict:
    """生成 OASIS 配置"""
    return {
        "simulation_id": simulation_id,
        "platform": "twitter",  # or "reddit"
        "max_rounds": parameters.max_rounds,
        "agent_profiles": profiles,
        "action_types": get_action_types("twitter"),
        "environment_params": {
            "min_activity": parameters.min_activity,
            "max_activity": parameters.max_activity,
            "time_step_hours": parameters.time_step
        }
    }
```

### 3. 运行模拟

```python
def run_simulation(config: Dict) -> subprocess.Popen:
    """运行 OASIS 模拟"""
    script_path = get_script_path(config["platform"])
    config_path = save_config(config)

    cmd = [
        "python", script_path,
        "--config", config_path,
        "--simulation-id", config["simulation_id"]
    ]

    process = subprocess.Popen(
        cmd,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE
    )

    return process
```

---

## OASIS Agent 详解

### Agent 结构

```python
@dataclass
class OasisAgent:
    """OASIS 智能体定义"""
    user_id: int
    username: str
    name: str
    bio: str

    # Personality
    personality: Dict[str, Any]

    # Memory
    short_term_memory: List[Dict]
    long_term_memory: List[Dict]

    # Goals
    goals: List[str]

    # State
    current_state: Dict[str, Any]
    action_history: List[Dict]
```

### 决策流程

```
┌─────────────────────────────────────────────────────────────┐
│                    Agent Decision Flow                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 感知环境 (Perceive)                                      │
│     · 读取当前环境状态                                        │
│     · 查看短期记忆中的最近事件                               │
│     · 注意其他 Agent 的动作                                  │
│                                                              │
│  2. 思考 (Think)                                             │
│     · 结合人格和目标                                         │
│     · 考虑当前情境                                           │
│     · 评估可能的行动                                         │
│                                                              │
│  3. 决策 (Decide)                                            │
│     · 选择最佳行动                                           │
│     · 确定行动参数                                           │
│                                                              │
│  4. 行动 (Act)                                               │
│     · 执行选定的行动                                         │
│     · 更新环境状态                                           │
│                                                              │
│  5. 记忆 (Remember)                                          │
│     · 将行动存入短期记忆                                     │
│     · 重要事件存入长期记忆                                   │
│     · 更新内部状态                                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Personality 定义

```python
personality = {
    "openness": 0.8,      # 开放性
    "conscientiousness": 0.6,  # 尽责性
    "extraversion": 0.7,   # 外向性
    "agreeableness": 0.5,  # 宜人性
    "neuroticism": 0.4     # 神经质
}
```

---

## 平台动作

### Twitter 动作

| 动作 | 参数 | 说明 |
|------|------|------|
| CREATE_POST | content | 发布推文 |
| LIKE_POST | post_id | 点赞推文 |
| RETWEET | post_id | 转发推文 |
| QUOTE_POST | post_id, content | 引用转发 |
| REPLY | post_id, content | 回复推文 |
| FOLLOW | user_id | 关注用户 |
| UNFOLLOW | user_id | 取消关注 |
| DO_NOTHING | - | 不执行动作 |

### Reddit 动作

| 动作 | 参数 | 说明 |
|------|------|------|
| CREATE_POST | title, content, subreddit | 发布帖子 |
| CREATE_COMMENT | post_id, content | 发表评论 |
| UPVOTE | post_id | 投票支持 |
| DOWNVOTE | post_id | 投票反对 |
| REPLY_COMMENT | comment_id, content | 回复评论 |
| FOLLOW | user_id | 关注用户 |
| MUTE | user_id | 屏蔽用户 |
| SEARCH_POSTS | query | 搜索帖子 |
| DO_NOTHING | - | 不执行动作 |

---

## 扩展 OASIS

### 添加新平台

```python
# 1. 定义平台动作
WEIBO_ACTIONS = [
    'CREATE_POST',
    'LIKE_POST',
    'REPOST',
    'COMMENT',
    'FOLLOW',
    'DO_NOTHING'
]

# 2. 定义 Profile 格式
def weibo_profile_format(entity: Dict) -> Dict:
    return {
        "user_id": entity["index"],
        "username": generate_username(entity["name"]),
        "name": entity["name"],
        "bio": entity.get("bio", ""),
        "followers_count": random.randint(10, 1000),
        "following_count": random.randint(10, 500)
    }

# 3. 创建运行脚本
# scripts/run_weibo_simulation.py
```

### 自定义动作

```python
class CustomAction(Action):
    def __init__(self, platform: str, action_type: str):
        super().__init__(platform, action_type)

    def validate(self, agent: OasisAgent, environment: Environment) -> bool:
        """验证动作是否可执行"""
        # 自定义验证逻辑
        return True

    def execute(self, agent: OasisAgent, environment: Environment) -> ActionResult:
        """执行动作"""
        # 自定义执行逻辑
        return ActionResult(success=True, data={})
```

---

## 性能优化

### 1. 并行处理

```python
from concurrent.futures import ThreadPoolExecutor

def parallel_agent_decision(agents: List[OasisAgent], environment: Environment):
    """并行处理 Agent 决策"""
    with ThreadPoolExecutor(max_workers=10) as executor:
        futures = [
            executor.submit(agent.decide, environment)
            for agent in agents
        ]

        actions = [f.result() for f in as_completed(futures)]

    return actions
```

### 2. 动作批处理

```python
def batch_actions(actions: List[Action]) -> List[ActionResult]:
    """批量执行动作"""
    results = []

    # 按类型分组
    grouped = group_actions_by_type(actions)

    # 批量执行同类型动作
    for action_type, action_list in grouped.items():
        batch_result = execute_batch(action_type, action_list)
        results.extend(batch_result)

    return results
```

---

## 下一步

- **[Report Agent 原理与定制](./report-agent.md)** - ReACT 模式
- **[IPC 通信机制详解](./ipc-communication.md)** - 进程间通信

---

## 自检清单

### 基础理解

- [ ] 能解释 OASIS 模拟引擎的架构
- [ ] 理解 Agent Profile 的结构
- [ ] 掌握模拟循环的基本流程
- [ ] 理解平台动作的定义

### 进阶掌握

- [ ] 能定制 Profile 生成逻辑
- [ ] 能调整模拟参数
- [ ] 理解 Agent 决策机制
- [ ] 能进行基本的性能优化

### 专家能力

- [ ] 能添加新平台支持
- [ ] 能设计新的 Agent 行为模式
- [ ] 能进行大规模模拟优化
- [ ] 能扩展 OASIS 引擎功能
