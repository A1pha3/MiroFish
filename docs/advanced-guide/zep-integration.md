# Zep Cloud 集成详解

> 深入理解 MiroFish 中的 GraphRAG 实现

**难度**: ⭐⭐⭐⭐ | **预计时间**: 60 分钟

---

## 学习目标

完成本章节后，你将能够：

### 基础目标（必掌握）

- [ ] 理解 GraphRAG 的核心原理和优势
- [ ] 掌握 Zep Cloud API 的基本使用方法
- [ ] 理解 MiroFish 中图谱构建的完整流程
- [ ] 掌握基本的检索操作

### 进阶目标（建议掌握）

- [ ] 理解不同检索策略的适用场景
- [ ] 能够优化图谱质量和检索效果
- [ ] 掌握 Zep Cloud 的高级 API 功能
- [ ] 能够诊断和解决图谱相关问题

### 专家目标（挑战）

- [ ] 能够设计自定义的检索策略
- [ ] 能够进行 GraphRAG 系统的性能调优
- [ ] 能够评估和比较不同的 GraphRAG 实现
- [ ] 能够设计领域特定的图谱 schema

---

## 什么是 Zep Cloud？

Zep Cloud 是一个企业级的 GraphRAG（Graph Retrieval-Augmented Generation）服务，提供：

- **知识图谱构建**：自动从文本提取实体和关系
- **语义检索**：基于向量的相似度搜索
- **时序记忆**：记录信息的时间维度
- **实体洞察**：深入分析实体相关信息

---

## GraphRAG 原理

### 传统 RAG vs GraphRAG

```
┌─────────────────────────────────────────────────────────────┐
│                    传统 RAG                                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Query                                                       │
│    │                                                         │
│    ▼                                                         │
│  ┌──────────────┐                                           │
│  │ 向量检索      │ ────→ 相关文档块                         │
│  └──────────────┘                                           │
│    │                                                         │
│    ▼                                                         │
│  ┌──────────────┐                                           │
│  │ LLM 生成答案  │                                           │
│  └──────────────┘                                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    GraphRAG                                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Query                                                       │
│    │                                                         │
│    ▼                                                         │
│  ┌──────────────┐     ┌──────────────┐                     │
│  │ 实体识别      │ ───→│ 关系遍历      │                     │
│  └──────────────┘     └──────────────┘                     │
│    │                       │                                │
│    └───────────┬───────────┘                                │
│                ▼                                            │
│  ┌──────────────────────────────┐                          │
│  │ 多路径检索                    │                          │
│  │ · 实体详细信息                │                          │
│  │ · 相关关系链                  │                          │
│  │ · 语义上下文                  │                          │
│  └──────────────────────────────┘                          │
│                │                                            │
│                ▼                                            │
│  ┌──────────────┐                                           │
│  │ LLM 生成答案  │                                           │
│  └──────────────┘                                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Zep GraphRAG 特点

| 特性 | 说明 | 优势 |
|------|------|------|
| **实体中心** | 以实体为核心组织信息 | 更符合人类认知 |
| **关系遍历** | 沿关系链探索 | 发现隐含联系 |
| **时序记忆** | 记录信息的时间有效性 | 支持时序查询 |
| **多跳检索** | 跨多个实体的检索 | 获取全局视角 |

---

## MiroFish 中的使用

### 1. 图谱构建

```python
from zep_cloud.client import Zep
from zep_cloud import EpisodeData

class GraphBuilder:
    def __init__(self, api_key: str):
        self.client = Zep(api_key=api_key)

    def build_graph(self, text: str, ontology: dict) -> str:
        # 创建图谱会话
        session = self.client.graph.create_graph(
            name="My Graph",
            entity_types=ontology["entity_types"],
            relation_types=ontology["edge_types"]
        )

        # 分块添加文本
        chunks = split_text_into_chunks(text, 500, 50)
        for chunk in chunks:
            session.add_episodes([
                EpisodeData(content=chunk)
            ])

        return session.graph_id
```

### 2. 实体检索

```python
def get_entities(self, graph_id: str, entity_type: str = None):
    """获取图谱中的实体"""
    filters = {}
    if entity_type:
        filters["label"] = entity_type

    entities = self.client.graph.get_entities(
        graph_id=graph_id,
        filters=filters
    )

    return [
        {
            "uuid": e.uuid,
            "name": e.name,
            "labels": e.labels,
            "summary": e.summary
        }
        for e in entities
    ]
```

### 3. 语义搜索

```python
def search(self, graph_id: str, query: str, search_type: str = "semantic"):
    """语义搜索"""
    results = self.client.graph.search(
        graph_id=graph_id,
        query=query,
        search_type=search_type
    )

    return {
        "query": query,
        "results": [
            {
                "content": r.result["content"],
                "score": r.score,
                "metadata": r.result["metadata"]
            }
            for r in results.results
        ]
    }
```

### 4. 实体洞察

```python
def get_entity_insight(self, graph_id: str, entity_uuid: str):
    """获取实体详细洞察"""
    # 1. 获取实体信息
    entity = self.client.graph.get_entity(
        graph_id=graph_id,
        uuid=entity_uuid
    )

    # 2. 获取相关关系
    edges = self.client.graph.get_entity_edges(
        graph_id=graph_id,
        uuid=entity_uuid
    )

    # 3. 获取时序事实
    facts = self.client.graph.get_facts(
        graph_id=graph_id,
        entity_uuid=entity_uuid
    )

    return {
        "entity": entity,
        "relationships": edges,
        "facts": facts
    }
```

---

## 高级用法

### 1. InsightForge 棱镜检索

InsightForge 是最强大的检索工具，结合多种检索方式：

```python
class InsightForge:
    def search(self, query: str, simulation_requirement: str):
        # 1. 生成子问题
        sub_queries = self._generate_sub_queries(
            query,
            simulation_requirement
        )

        # 2. 多维度并行检索
        with ThreadPoolExecutor(max_workers=3) as executor:
            semantic_future = executor.submit(
                self._semantic_search, sub_queries
            )
            entity_future = executor.submit(
                self._entity_search, sub_queries
            )
            relation_future = executor.submit(
                self._relation_search, sub_queries
            )

            semantic_facts = semantic_future.result()
            entity_insights = entity_future.result()
            relation_chains = relation_future.result()

        # 3. 综合分析
        synthesis = self._synthesize_insights(
            semantic_facts,
            entity_insights,
            relation_chains
        )

        return InsightForgeResult(
            query=query,
            sub_queries=sub_queries,
            semantic_facts=semantic_facts,
            entity_insights=entity_insights,
            relation_chains=relation_chains,
            synthesis=synthesis
        )
```

### 2. 关系链遍历

```python
def traverse_relationships(
    self,
    graph_id: str,
    start_entity_uuid: str,
    max_depth: int = 3
):
    """遍历关系链"""
    visited = set()
    chains = []

    def dfs(current_uuid, depth, path):
        if depth > max_depth or current_uuid in visited:
            return

        visited.add(current_uuid)
        path.append(current_uuid)

        # 获取相邻实体
        edges = self.client.graph.get_entity_edges(
            graph_id=graph_id,
            uuid=current_uuid
        )

        for edge in edges:
            next_uuid = edge.target_uuid if edge.source_uuid == current_uuid else edge.source_uuid

            # 记录关系链
            if depth > 0:
                chains.append({
                    "path": path.copy(),
                    "edge": edge
                })

            dfs(next_uuid, depth + 1, path)

        path.pop()

    dfs(start_entity_uuid, 0, [])
    return chains
```

### 3. 时序查询

```python
def get_temporal_facts(
    self,
    graph_id: str,
    entity_uuid: str,
    valid_after: str = None,
    valid_before: str = None
):
    """获取时序事实"""
    filters = {"entity_uuid": entity_uuid}

    if valid_after:
        filters["valid_after"] = valid_after
    if valid_before:
        filters["valid_before"] = valid_before

    facts = self.client.graph.get_facts(
        graph_id=graph_id,
        filters=filters
    )

    # 按时间排序
    facts.sort(key=lambda f: f.created_at)

    return [
        {
            "fact": f.fact,
            "valid_at": f.valid_at,
            "invalid_at": f.invalid_at,
            "created_at": f.created_at
        }
        for f in facts
    ]
```

---

## 性能优化

### 1. 批量操作

```python
# 不推荐：逐个添加
for chunk in chunks:
    session.add_episodes([EpisodeData(content=chunk)])

# 推荐：批量添加
batch_size = 10
for i in range(0, len(chunks), batch_size):
    batch = chunks[i:i+batch_size]
    session.add_episodes([
        EpisodeData(content=chunk)
        for chunk in batch
    ])
```

### 2. 缓存策略

```python
from functools import lru_cache

class CachedZepClient:
    def __init__(self, api_key: str):
        self.client = Zep(api_key=api_key)

    @lru_cache(maxsize=1000)
    def get_entity(self, graph_id: str, uuid: str):
        return self.client.graph.get_entity(
            graph_id=graph_id,
            uuid=uuid
        )

    def clear_cache(self):
        self.get_entity.cache_clear()
```

### 3. 异步调用

```python
import asyncio
from zep_cloud.aio import Zep as AsyncZep

class AsyncGraphBuilder:
    def __init__(self, api_key: str):
        self.client = AsyncZep(api_key=api_key)

    async def build_graph_async(self, texts: list):
        tasks = [
            self._add_episodes(text)
            for text in texts
        ]
        await asyncio.gather(*tasks)

    async def _add_episodes(self, text: str):
        # 异步添加
        pass
```

---

## 最佳实践

### 1. 本体设计

- **实体数量**：控制在 10-20 个类型
- **关系方向**：明确源实体和目标实体
- **属性设计**：只添加必要的属性

### 2. 文本预处理

- **分块大小**：500-1000 字符
- **重叠大小**：50-100 字符
- **编码处理**：确保 UTF-8 编码

### 3. 查询优化

- **具体化查询**：避免过于宽泛的查询
- **使用过滤器**：缩小检索范围
- **限制结果数**：控制返回结果数量

---

## 下一步

- **[OASIS 模拟引擎深度解析](./oasis-engine.md)** - 了解模拟引擎

---

## 自检清单

### 基础理解

- [ ] 能解释 GraphRAG 相比传统 RAG 的优势
- [ ] 能使用 Zep Cloud API 进行基本操作
- [ ] 能理解图谱构建的完整流程
- [ ] 能进行简单的实体检索

### 进阶掌握

- [ ] 能选择合适的检索策略
- [ ] 能优化图谱质量
- [ ] 能使用 InsightForge 进行复杂检索
- [ ] 能诊断图谱相关问题

### 专家能力

- [ ] 能设计自定义检索策略
- [ ] 能进行性能调优
- [ ] 能设计领域特定的 schema
- [ ] 能评估不同的 GraphRAG 实现
