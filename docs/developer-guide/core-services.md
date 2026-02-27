# 核心服务详解

> 深入分析 MiroFish 后端核心服务的实现原理和最佳实践

**难度**: ⭐⭐⭐⭐ | **预计时间**: 75 分钟 | **前置知识**: Python 高级编程、异步编程

---

## 学习目标

完成本指南学习后，你将能够：

### 基础目标（必掌握）

- [ ] 理解每个核心服务的职责和接口
- [ ] 掌握服务之间的协作关系
- [ ] 能够使用和扩展核心服务
- [ ] 理解关键的设计决策

### 进阶目标（建议掌握）

- [ ] 深入理解服务的实现细节
- [ ] 能够优化服务性能
- [ ] 能够调试服务问题
- [ ] 能够进行服务级别的定制

### 专家目标（挑战）

- [ ] 能够设计新的服务
- [ ] 能够重构现有服务
- [ ] 能够进行架构级别的优化

---

## 服务概览

MiroFish 后端采用服务分层架构，核心服务按职责划分：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        MiroFish 核心服务架构                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  图谱层 (Graph Layer)                                        │  │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │  │
│  │  │ OntologyGenerator│  │  GraphBuilder    │  │ EntityReader │  │  │
│  │  │                  │  │                  │  │              │  │  │
│  │  │ 生成本体定义      │  │ 构建 GraphRAG    │  │ 读取实体数据  │  │  │
│  │  └──────────────────┘  └──────────────────┘  └──────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  模拟层 (Simulation Layer)                                 │  │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │  │
│  │  │SimulationManager │  │ ProfileGenerator│  │ConfigGenerator│  │  │
│  │  │                  │  │                  │  │              │  │  │
│  │  │管理模拟生命周期  │  │生成智能体人设   │  │生成模拟配置  │  │  │
│  │  └──────────────────┘  └──────────────────┘  └──────────────┘  │  │
│  │                                                              │  │
│  │  ┌──────────────────────────────────────────────────────────┐  │  │
│  │  │            SimulationRunner - 子进程管理和监控           │  │  │
│  │  │            SimulationIPC - 进程间通信                   │  │  │
│  │  └──────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  报告层 (Report Layer)                                      │  │
│  │  ┌──────────────────────────────────────────────────────────┐  │  │
│  │  │                    ReportAgent - ReACT 模式              │  │  │
│  │  └──────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                          │                                          │
│                          ▼                                          │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  工具层 (Utility Layer)                                       │  │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │  │
│  │  │ ZepToolsService │  │ LLMClient       │  │ TextProcessor│  │  │
│  │  │                  │  │                  │  │              │  │  │
│  │  │ Zep API 封装     │  │ LLM API 调用     │  │ 文本处理工具  │  │  │
│  │  └──────────────────┘  └──────────────────┘  └──────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. OntologyGenerator - 本体生成服务

### 服务职责

从文本内容和用户需求生成适合的实体类型和关系类型定义。

### 核心实现

```python
# backend/app/services/ontology_generator.py

from typing import Dict, Any, List
from ..utils.llm_client import LLMClient
from .base_service import BaseService

class OntologyGenerator(BaseService):
    """本体生成服务"""

    def __init__(self, llm_client: LLMClient = None):
        super().__init__()
        self.llm_client = llm_client or LLMClient()
        self.system_prompt = self._load_system_prompt()

    def generate(
        self,
        text: str,
        user_requirement: str = "社会模拟"
    ) -> Ontology:
        """
        生成本体定义

        Args:
            text: 输入文本样本
            user_requirement: 用户需求描述

        Returns:
            Ontology: 包含 entity_types 和 edge_types 的本体对象
        """
        # 1. 构建提示词
        prompt = self._build_prompt(text, user_requirement)

        # 2. 调用 LLM
        messages = [
            {"role": "system", "content": self.system_prompt},
            {"role": "user", "content": prompt}
        ]

        # 3. 使用 JSON mode 确保输出格式
        response = self.llm_client.chat_json(
            messages=messages,
            temperature=0.3,  # 低温度保证稳定性
            response_schema=self._get_json_schema()
        )

        # 4. 解析和验证响应
        ontology = self._parse_response(response)

        # 5. 后处理：确保兜底类型存在
        ontology = self._ensure_fallback_types(ontology)

        return ontology

    def _build_prompt(self, text: str, user_requirement: str) -> str:
        """构建 LLM 提示词"""
        return f"""
基于以下文本，生成适合"{user_requirement}"的本体定义。

文本内容：
{text[:2000]}...

请生成：
1. 8-12 个实体类型，必须包括 Person 和 Organization
2. 8-12 个关系类型
3. 每个类型提供清晰的描述
4. 每个类型提供 2-3 个示例

要求：
- 实体类型必须是可发声的主体（智能体可以扮演）
- 关系类型应该是描述性的动词短语
- 避免过于抽象的概念

按照以下 JSON 格式输出：
{json.dumps(self._get_json_schema(), indent=2)}
"""
```

### Prompt Engineering 策略

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Ontology Prompt 设计策略                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  策略 1: 明确数量要求                                            │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ ❌ 不好的提示: "生成一些实体类型"                          │  │
│  │ ✅ 好的提示: "生成 8-12 个实体类型，必须包括 Person..."   │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  策略 2: 提供具体示例                                              │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ 在提示词中包含期望的输出格式示例                             │  │
│  │ 帮助 LLM 理解预期的结构                                     │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  策略 3: 使用 Schema 约束                                            │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ 利用 LLM 的 JSON mode 功能                                  │  │
│  │ 定义严格的输出 schema                                       │  │
│  │ 确保输出可解析                                             │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  策略 4: 兜底类型保证                                              │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ 如果 LLM 没有返回必需类型，自动添加                         │  │
│  │ 确保 Person、Organization 等核心类型始终存在                 │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 输出 Schema

```python
def _get_json_schema(self) -> Dict[str, Any]:
    """获取 JSON Schema"""
    return {
        "type": "object",
        "properties": {
            "entity_types": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "name": {"type": "string"},
                        "description": {"type": "string"},
                        "examples": {"type": "array"}
                    },
                    "required": ["name", "description"]
                }
            },
            "edge_types": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "name": {"type": "string"},
                        "description": {"type": "string"},
                        "source_targets": {
                            "type": "array",
                            "items": {
                                "type": "object",
                                "properties": {
                                    "source": {"type": "string"},
                                    "target": {"type": "string"}
                                }
                            }
                        }
                    },
                    "required": ["name", "description"]
                }
            }
        },
        "required": ["entity_types", "edge_types"]
    }
```

---

## 2. GraphBuilder - 图谱构建服务

### 服务职责

调用 Zep Cloud API 构建 GraphRAG 知识图谱。

### 架构设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                    GraphBuilder 工作流程                              │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────┐
│  1. 任务创建阶段                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ user → build_graph_async()                                  │  │
│  │      │                                                       │  │
│  │      ▼                                                       │  │
│  │  TaskManager.create_task("graph_build")                       │  │
│  │      └─> 返回 task_id                                         │  │
│  │                                                              │  │
│  │  Thread(target=_build_graph_worker)                        │  │
│  │      └─> 后台执行开始                                        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ 2. 图谱构建阶段 (后台线程)                                   │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │ a) 创建 Zep 图谱会话                                    │  │  │
│  │  │    session = client.graph.create_graph(...)               │  │  │
│  │  │                                                         │  │  │
│  │  │ b) 文本分块处理                                        │  │  │
│  │  │    chunks = split_text(text, chunk_size, overlap)        │  │  │
│  │  │                                                         │  │  │
│  │  │ c) 批量添加到图谱                                        │  │  │
│  │  │    for batch in chunks:                                 │  │  │
│  │  │        session.add_episodes([{"content": c}...])      │  │  │
│  │  │                                                         │  │  │
│  │  │ d) 等待 Zep 处理完成                                      │  │  │
│  │  │    while session.get_status() != "ready":               │  │  │
│  │  │        time.sleep(5)                                      │  │  │
│  │  │                                                         │  │  │
│  │  │ e) 更新任务状态                                          │  │  │
│  │  │    task_manager.complete_task(task_id, result)         │  │  │
│  └─────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│   3. 状态查询阶段                                                  │
│  │  user → check_task_status(task_id)                           │  │
│  │      └─> TaskManager.get_task(task_id)                     │  │  │
│  │      └─> 返回 {status, result}                               │  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 核心代码实现

```python
# backend/app/services/graph_builder.py

import threading
import time
from typing import Dict, Any, Optional
from zep import Zep
from ..models.ontology import Ontology
from .task_manager import TaskManager

class GraphBuilder:
    """GraphRAG 图谱构建服务"""

    def __init__(self, api_key: str = None):
        self.client = Zep(api_key=api_key)
        self.task_manager = TaskManager()

    def build_graph_async(
        self,
        text: str,
        ontology: Ontology,
        graph_name: str = "MiroFish Graph",
        chunk_size: int = 500,
        chunk_overlap: int = 50,
        batch_size: int = 10
    ) -> str:
        """
        异步构建图谱

        Args:
            text: 输入文本
            ontology: 本体定义
            graph_name: 图谱名称
            chunk_size: 文本块大小
            chunk_overlap: 重叠大小
            batch_size: 批量大小

        Returns:
            str: 任务 ID，用于查询构建状态
        """
        # 1. 创建任务
        task_id = self.task_manager.create_task(
            task_type="graph_build",
            metadata={
                "graph_name": graph_name,
                "chunk_size": chunk_size
            }
        )

        # 2. 启动后台线程
        thread = threading.Thread(
            target=self._build_graph_worker,
            args=(task_id, text, ontology, chunk_size, chunk_overlap, batch_size),
            daemon=True
        )
        thread.start()

        return task_id

    def _build_graph_worker(
        self,
        task_id: str,
        text: str,
        ontology: Ontology,
        chunk_size: int,
        chunk_overlap: int,
        batch_size: int
    ):
        """图谱构建工作线程（在后台执行）"""
        try:
            # 更新任务状态为处理中
            self.task_manager.update_task_status(
                task_id, "processing", "正在初始化..."
            )

            # 步骤 1: 创建图谱会话
            session = self.client.graph.create_graph(
                name=f"{graph_name} ({task_id[:8]})",
                description="Built by MiroFish",
                config={
                    "embedder": "huggingface/small"
                }
            )

            # 添加本体到图谱
            self._add_ontology_to_session(session, ontology)

            # 步骤 2: 分块处理文本
            self.task_manager.update_task_status(
                task_id, "processing", "正在处理文本..."
            )

            chunks = self._split_text(text, chunk_size, chunk_overlap)

            # 步骤 3: 批量添加到图谱
            self.task_manager.update_task_status(
                task_id, "processing",
                f"正在添加 {len(chunks)} 个文本块..."
            )

            for i in range(0, len(chunks), batch_size):
                batch = chunks[i:i + batch_size]

                # 添加批次
                episodes = [{"content": chunk} for chunk in batch]
                session.add_episodes(episodes)

                # 更新进度
                progress = int((i + batch_size) / len(chunks) * 100)
                self.task_manager.update_task_status(
                    task_id, "processing",
                    f"处理进度: {progress}%"
                )

                # 等待 Zep 处理
                self._wait_for_ready(session)

            # 步骤 4: 获取统计信息
            node_count = session.node_count
            edge_count = session.edge_count

            # 完成任务
            self.task_manager.complete_task(task_id, {
                "graph_id": session.graph_id,
                "node_count": node_count,
                "edge_count": edge_count,
                "status": "ready"
            })

        except Exception as e:
            # 失败任务
            self.task_manager.fail_task(
                task_id,
                f"图谱构建失败: {str(e)}"
            )

    def _wait_for_ready(self, session, timeout: int = 300):
        """等待图谱进入 ready 状态"""
        start_time = time.time()
        while time.time() - start_time < timeout:
            status = session.get_status()
            if status in ["ready", "completed"]:
                return True
            time.sleep(5)
        raise TimeoutError("图谱构建超时")
```

### 性能优化策略

```python
# 批量处理优化
def _optimize_batch_processing(chunks, batch_size):
    """
    优化批量处理策略

    Args:
        chunks: 文本块列表
        batch_size: 批量大小

    Returns:
        优化后的批次生成器
    """
    # 小块合并策略
    if len(chunks) < batch_size:
        batch_size = len(chunks)

    # 按大小分组
    groups = []
    current_group = []
    current_size = 0

    for chunk in chunks:
        chunk_size = len(chunk)
        if current_size + chunk_size > batch_size * 1000:  # 字符数限制
            if current_group:
                groups.append(current_group)
            current_group = [chunk]
            current_size = chunk_size
        else:
            current_group.append(chunk)
            current_size += chunk_size

    if current_group:
        groups.append(current_group)

    return groups
```

---

## 3. SimulationManager - 模拟管理服务

### 状态机设计

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SimulationManager 状态机                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  CREATED ──────────────────────────────────────────────────────►  │
│    │                                                                │
│    │ (调用 prepare_simulation())                                     │
│    ▼                                                                │
│  PREPARING ─────────────────────────────────────────────────────► │
│    │                                                                │
│    │ (准备完成)                                                     │
│    ▼                                                                │
│  READY ────────────────────────────────────────────────────────────► │
│    │                                                                │
│    │ (调用 start_simulation())                                      │
│    ▼                                                                │
│  RUNNING ────────────────────────────────────────────────────────► │
│    │           │                                                    │
│    │           ├─► (完成所有轮数) COMPLETED                     │
│    │           │                                                    │
│    │           ├─► (调用 stop()) STOPPED                            │
│    │           │                                                    │
│    │           └─► (发生错误) FAILED                              │
│    │                                                                │
└─────────────────────────────────────────────────────────────────────┘
```

### 状态枚举

```python
from enum import Enum

class SimulationStatus(str, Enum):
    """模拟状态枚举"""
    CREATED = "created"           # 已创建
    PREPARING = "preparing"       # 准备中
    READY = "ready"               # 准备就绪
    RUNNING = "running"           # 运行中
    PAUSED = "paused"             # 已暂停
    STOPPED = "stopped"           # 已停止
    COMPLETED = "completed"       # 已完成
    FAILED = "failed"             # 已失败

    def can_transition_to(self, target: 'SimulationStatus') -> bool:
        """检查是否可以转换到目标状态"""
        valid_transitions = {
            SimulationStatus.CREATED: [
                SimulationStatus.PREPARING,
                SimulationStatus.FAILED
            ],
            SimulationStatus.PREPARING: [
                SimulationStatus.READY,
                SimulationStatus.FAILED
            ],
            SimulationStatus.READY: [
                SimulationStatus.RUNNING,
                SimulationStatus.FAILED
            ],
            SimulationStatus.RUNNING: [
                SimulationStatus.PAUSED,
                SimulationStatus.COMPLETED,
                SimulationStatus.STOPPED,
                SimulationStatus.FAILED
            ],
            SimulationStatus.PAUSED: [
                SimulationStatus.RUNNING,
                SimulationStatus.STOPPED
            ],
            SimulationStatus.STOPPED: [
                SimulationStatus.READY
            ]
        }
        return target in valid_transitions.get(self, [])
```

### 核心方法实现

```python
# backend/app/services/simulation_manager.py

class SimulationManager:
    """模拟生命周期管理"""

    def prepare_simulation(
        self,
        simulation_id: str,
        simulation_requirement: str,
        document_text: str,
        defined_entity_types: List[str],
        use_llm_for_profiles: bool = True,
        parallel_profile_count: int = 3
    ) -> SimulationState:
        """
        准备模拟环境

        工作流程:
        1. 读取图谱实体
        2. 生成智能体 Profile
        3. 生成模拟配置
        4. 保存配置文件
        """
        state = self._load_simulation_state(simulation_id)

        try:
            # 状态转换检查
            if not state.status.can_transition_to(SimulationStatus.PREPARING):
                raise InvalidStateTransitionError(
                    f"Cannot prepare simulation in {state.status} state"
                )

            state.status = SimulationStatus.PREPARING
            self._save_simulation_state(state)

            # 阶段 1: 读取实体
            logger.info(f"[{simulation_id}] Reading entities from graph...")
            filtered_entities = self.entity_reader.filter_defined_entities(
                graph_id=state.graph_id,
                entity_types=defined_entity_types,
                enrich_with_edges=True
            )
            state.entities_count = len(filtered_entities.entities)
            state.entities = filtered_entities.entities

            # 阶段 2: 生成 Profile
            logger.info(f"[{simulation_id}] Generating {state.entities_count} profiles...")
            profiles = self.profile_generator.generate_profiles_from_entities(
                entities=filtered_entities.entities,
                document_text=document_text,
                use_llm=use_llm_for_profiles,
                parallel_count=parallel_profile_count,
                progress_callback=lambda step, msg: self._update_progress(
                    simulation_id, "profile_gen", step, msg
                )
            )
            state.profiles_count = len(profiles)

            # 保存 Profile 文件
            profile_path = self._get_profile_path(simulation_id)
            self.profile_generator.save_profiles(profiles, profile_path)

            # 阶段 3: 生成配置
            logger.info(f"[{simulation_id}] Generating simulation config...")
            config = self.config_generator.generate_config(
                simulation_id=simulation_id,
                simulation_requirement=simulation_requirement,
                document_text=document_text,
                entities=state.entities,
                profiles=profiles
            )

            # 保存配置文件
            config_path = self._get_config_path(simulation_id)
            self.config_generator.save_config(config, config_path)

            # 状态转换到 READY
            state.status = SimulationStatus.READY
            self._save_simulation_state(state)

            return state

        except Exception as e:
            logger.error(f"[{simulation_id}] Preparation failed: {str(e)}")
            state.status = SimulationStatus.FAILED
            state.error = str(e)
            self._save_simulation_state(state)
            raise

    def start_simulation(self, simulation_id: str) -> bool:
        """
        启动模拟

        Returns:
            bool: 是否成功启动
        """
        state = self._load_simulation_state(simulation_id)

        # 状态检查
        if not state.status.can_transition_to(SimulationStatus.RUNNING):
            raise InvalidStateTransitionError(
                f"Cannot start simulation in {state.status} state"
            )

        # 获取脚本路径
        script_path = self.simulation_runner.get_script_path(state.platform_type)

        # 启动子进程
        return self.simulation_runner.start_simulation(
            simulation_id=simulation_id,
            script_path=script_path,
            config_path=self._get_config_path(simulation_id)
        )
```

---

## 4. ReportAgent - 报告生成服务

### ReACT 模式实现

```python
# backend/app/services/report_agent.py

from typing import Dict, Any, Callable, List
from .zep_tools_service import ZepToolsService

class ReportAgent:
    """ReACT 模式报告生成 Agent"""

    def __init__(
        self,
        llm_client: LLMClient,
        zep_tools: ZepToolsService,
        simulation_id: str,
        graph_id: str
    ):
        self.llm_client = llm_client
        self.zep_tools = zep_tools
        self.simulation_id = simulation_id
        self.graph_id = graph_id

        # 可用工具
        self.tools: Dict[str, Callable] = self._get_tools()

        # ReACT 参数
        self.max_iterations = 5
        self.max_reflection_rounds = 2

    def generate_report(
        self,
        report_requirement: str,
        max_tool_calls: int = 5,
        temperature: float = 0.5
    ) -> str:
        """
        使用 ReACT 模式生成报告

        流程:
        1. 规划报告大纲
        2. ReACT 循环 (Thought → Action → Observation)
        3. 生成最终报告
        4. 反思和优化
        """
        # 阶段 1: 规划大纲
        outline = self._plan_outline(report_requirement)

        # 阶段 2: ReACT 循环
        context = {
            "outline": outline,
            "iterations": 0,
            "tool_calls": [],
            "observations": []
        }

        for iteration in range(self.max_iterations):
            # 检查是否已有足够信息
            if self._can_answer_with_current_info(context):
                break

            # Thought: 推理
            thought = self._generate_thought(context, report_requirement)
            context["observations"].append({
                "type": "thought",
                "content": thought
            })

            # Action: 选择并执行工具
            action, action_input = self._decide_action(context, thought)
            context["tool_calls"].append(action)

            # Observation: 执行并观察结果
            observation = self._execute_action(action, action_input)
            context["observations"].append({
                "type": "observation",
                "action": action,
                "result": observation
            })

            context["iterations"] += 1

        # 阶段 3: 生成最终报告
        report = self._generate_final_report(context, report_requirement)

        # 阶段 4: 反思优化
        if self.max_reflection_rounds > 0:
            report = self._reflect_and_optimize(report, context)

        return report

    def _decide_action(
        self,
        context: Dict[str, Any],
        thought: str
    ) -> tuple[str, Dict[str, Any]]:
        """
        根据当前思考和上下文决定下一个行动

        Returns:
            (action_name, action_input)
        """
        # 分析思考内容
        if "实体信息" in thought:
            return "entity_lookup", {"entity_name": self._extract_entity_name(thought)}

        elif "关系搜索" in thought or "全局" in thought:
            return "panorama_search", {"query": self._extract_search_query(thought)}

        elif "深度分析" in thought or "洞察" in thought:
            return "insight_forge", {"query": self._extract_analytical_query(thought)}

        elif "访谈" in thought or "观点" in thought:
            return "interview_agent", {"agent_name": self._extract_agent_name(thought)}

        else:
            # 默认快速搜索
            return "quick_search", {"query": self._extract_search_query(thought)}

    def _execute_action(self, action: str, action_input: Dict) -> str:
        """执行工具并返回观察结果"""
        tool_func = self.tools.get(action)
        if not tool_func:
            return f"工具 {action} 不可用"

        try:
            result = tool_func(**action_input)
            return self._format_observation(action, result)
        except Exception as e:
            return f"执行 {action} 时出错: {str(e)}"
```

### 工具系统

```python
def _get_tools(self) -> Dict[str, Callable]:
    """获取可用工具集"""
    return {
        "insight_forge": self.zep_tools.insight_forge,
        "panorama_search": self.zep_tools.panorama_search,
        "quick_search": self.zep_tools.quick_search,
        "entity_lookup": self.zep_tools.entity_lookup,
        "interview_agent": self._interview_agent
    }

def _interview_agent(self, agent_name: str, question: str) -> str:
    """访谈智能体工具"""
    # 通过 SimulationIPC 发送访谈命令
    result = self.simulation_ipc.send_command(
        CommandType.INTERVIEW,
        {"agent_name": agent_name, "prompt": question}
    )
    return result.get("response", "访谈失败")
```

---

## 5. ZepToolsService - Zep 工具集

### 工具对比

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Zep 工具对比矩阵                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  工具名称         │ 速度    │ 深度    │ 成本    │ 使用场景              │
│  ├─────────────────────────────────────────────────────────────┤  │
│  │ QuickSearch      │ ⭐⭐⭐⭐  │ ⭐      │ ⭐      │ 简单事实查询          │
│  │ PanoramaSearch  │ ⭐⭐⭐    │ ⭐⭐⭐  │ ⭐⭐    │ 全局关系搜索          │
│  │ InsightForge     │ ⭐⭐      │ ⭐⭐⭐⭐ │ ⭐⭐⭐⭐ │ 复杂问题分析          │
│  │ EntityLookup    │ ⭐⭐⭐⭐  │ ⭐⭐⭐  │ ⭐⭐     │ 特定实体详情          │
│  │ InterviewAgent  │ ⭐⭐      │ ⭐⭐⭐⭐ │ ⭐⭐⭐⭐ │ 智能体观点获取        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 使用建议

```
┌─────────────────────────────────────────────────────────────────────┐
│                    工具选择决策树                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  你想查询什么？                                                   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 简单事实（如"贾宝玉在第几轮做了什么？"）                    │   │
│  └─▶ 使用 QuickSearch - 快速、低成本                             │   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 全局关系（如"所有与贾宝玉相关的互动"）                      │   │
│  └─▶ 使用 PanoramaSearch - 全面、覆盖面广                          │   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 复杂分析（如"贾府衰败对宝黛关系的深层影响"）                │   │
│  └─▶ 使用 InsightForge - 深度、有洞察                             │   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 特定实体（如"林黛玉的完整人设详情"）                        │   │
│  └─▶ 使用 EntityLookup - 精确、完整                                │   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 智能体观点（如"宝玉对黛玉最近的帖子怎么看？"）              │   │
│  └─▶ 使用 InterviewAgent - 第一人称、实时                            │   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 服务间协作

### 典型工作流：完整的预测流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    MiroFish 完整预测流程 - 服务协作              │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────┐
│  Step 1: 图谱构建                                                 │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ 用户 → API → GraphBuilder.build_graph_async()           │ │
│  │                     │                                   │ │
│  │                     ▼                                   │ │
│  │              OntologyGenerator.generate()               │ │
│  │                     │                                   │ │
│  │                     ▼                                   │ │
│  │              Zep Cloud API                              │ │
│  │                     │                                   │ │
│  │                     ▼                                   │ │
│  │              返回 graph_id + task_id                   │ │
│  └─────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────┐
│  Step 2: 环境搭建                                                 │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ 用户 → API → SimulationManager.prepare_simulation()    │ │
│  │                     │                                   │ │
│  │                     ├─→ EntityReader.read_entities()        │ │
│  │                     │                                   │ │
│  │                     ├─→ OasisProfileGenerator.generate()    │ │
│ │                     │                                   │ │
│  │                     └─→ SimulationConfigGenerator.generate()│ │
│  │                     │                                   │ │
│  │                     ▼                                   │ │
│  │              保存 simulation_state + profiles             │ │
│  └─────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────┐
│  Step 3: 模拟运行                                                 │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ 用户 → API → SimulationManager.start_simulation()        │ │
│  │                     │                                   │ │
│  │                     ▼                                   │ │
│  │              SimulationRunner.start_simulation()           │ │
│  │                     │                                   │ │
│ │  ┌─────────────────────────────────────────────────────┐ │ │
│  │  │ subprocess.Popen([python, run_xxx_sim.py])       │ │ │
│  │  │                                                     │ │ │
│  │  │ ┌───────────────────────────────────────────────┐ │ │ │
│  │  │ │ OASIS Environment                             │ │ │ │
│  │  │ │   - 每轮决策                                     │ │ │ │
│  │  │ │   - 生成动作                                     │ │ │ │
│  │  │ │   - 更新 Zep 记忆                                 │ │ │ │
│  │  │ │                                                     │ │ │ │
│  │  │ └───────────────────────────────────────────────┘ │ │ │
│  │  │                                                     │ │ │
│  │  │ ┌───────────────────────────────────────────────┐ │ │ │
│  │  │ │ SimulationIPCServer - 接收访谈请求           │ │ │ │
│  │  │ └───────────────────────────────────────────────┘ │ │ │
│  │  └─────────────────────────────────────────────────────┘ │ │
│  │                     │                                   │ │
│  │                     ▼                                   │ │
│  │              actions.jsonl (动作记录)                     │ │
│  └─────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────┐
│  Step 4: 报告生成                                                 │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │ 用户 → API → ReportAgent.generate_report()              │ │
│  │                     │                                   │ │
│  │                     ▼                                   │ │
│  │              ReACT 循环:                                 │ │
│  │  │  Thought → Action → Observation → ...               │ │
│  │  │                                                     │ │
│  │  │  ├─→ InsightForge (分析关系变化)                 │ │ │
│  │  │  ├─→ EntityLookup (查看实体详情)                   │ │ │
│  │  │  ├─→ InterviewAgent (获取智能体观点)               │ │ │
│  │  │  └─> 生成结构化报告                                │ │ │
│  └─────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

---

## 学习检查清单

### 服务理解

- [ ] 能够说出每个核心服务的职责
- [ ] 理解服务之间的依赖关系
- [ ] 理解模拟状态的转换流程
- [ ] 理解 ReACT 循环的工作原理

### 实践能力

- [ ] 能够使用服务提供的接口
- [ ] 能够处理服务抛出的异常
- [ ] 能够扩展服务功能
- [ ] 能够调试服务问题

---

## 进阶主题

### 性能优化

#### 1. 批量处理优化

```python
# 优化前: 逐个添加
for entity in entities:
    session.add_node(entity)

# 优化后: 批量添加
session.add_nodes(entities)
```

#### 2. 并发处理

```python
from concurrent.futures import ThreadPoolExecutor

def generate_profiles_parallel(entities, parallel_count=3):
    """并行生成 Profile"""
    with ThreadPoolExecutor(max_workers=parallel_count) as executor:
        futures = {
            executor.submit(generate_profile, entity)
            for entity in entities
        }
        results = [f.result() for f in futures]
    return results
```

### 错误处理

#### 统一错误处理

```python
# 自定义异常
class GraphBuildError(Exception):
    """图谱构建错误"""
    pass

class SimulationError(Exception):
    """模拟运行错误"""
    pass

# 使用示例
try:
    graph_id = graph_builder.build_graph_async(...)
except GraphBuildError as e:
    logger.error(f"图谱构建失败: {e}")
    # 处理逻辑
except Exception as e:
    logger.error(f"未知错误: {e}")
    # 降级处理
```

---

## 相关文档

- **[后端架构详解](./backend-architecture.md)** - 架构层面的深入分析
- **[API 参考文档](./api-reference.md)** - 完整的 API 接口说明
- **[高级指南 - Zep 集成](../advanced-guide/zep-integration.md)** - Zep Cloud 深入解析
