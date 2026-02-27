# Report Agent 原理与定制

> 深入理解 Report Agent 的 ReACT 模式和定制方法

**难度**: ⭐⭐⭐⭐ | **预计时间**: 60 分钟

---

## 学习目标

完成本章节后，你将能够：

### 基础目标（必掌握）

- [ ] 理解 ReACT (Reasoning + Acting) 模式的核心原理
- [ ] 掌握 Report Agent 的工作流程
- [ ] 理解工具调用的机制
- [ ] 掌握基本的报告生成流程

### 进阶目标（建议掌握）

- [ ] 能够设计和实现新的检索工具
- [ ] 理解 Prompt 模板的优化方法
- [ ] 能够调试和优化 Report Agent 行为
- [ ] 掌握反思机制的作用

### 专家目标（挑战）

- [ ] 能够设计复杂的多步工具链
- [ ] 能够实现自定义的 ReACT 循环策略
- [ ] 能够优化 Report Agent 的性能
- [ ] 能够集成新的数据源到工具系统中

---

## Report Agent 概述

Report Agent 是 MiroFish 的报告生成核心，使用 ReACT (Reasoning + Acting) 模式与 Zep 图谱深度交互，生成高质量的预测报告。

### 核心能力

| 能力 | 说明 |
|------|------|
| **自主规划** | 根据需求自动规划报告大纲 |
| **工具使用** | 灵活调用多种 Zep 检索工具 |
| **自我反思** | 通过多轮反思优化报告质量 |
| **对话交互** | 支持生成后的追问和澄清 |

---

## ReACT 模式

### 原理

ReACT = Reasoning (推理) + Acting (行动)

```
┌─────────────────────────────────────────────────────────────┐
│                    ReACT 循环                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  初始查询 (Query)                                            │
│      │                                                       │
│      ▼                                                       │
│  ┌──────────────┐                                           │
│  │ Thought      │  "我需要了解 X 信息"                       │
│  └──────────────┘                                           │
│      │                                                       │
│      ▼                                                       │
│  ┌──────────────┐                                           │
│  │ Action       │  调用 InsightForge 工具                    │
│  └──────────────┘                                           │
│      │                                                       │
│      ▼                                                       │
│  ┌──────────────┐                                           │
│  │ Observation  │  获取检索结果                              │
│  └──────────────┘                                           │
│      │                                                       │
│      └───→ 循环直到可以给出最终答案 ───→ Final Answer       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### MiroFish 中的实现

```python
class ReportAgent:
    def generate_report(
        self,
        simulation_id: str,
        report_requirement: str
    ) -> str:
        """生成报告的主流程"""

        # 阶段 1: 规划大纲
        outline = self._plan_outline(report_requirement)

        # 阶段 2: 生成各章节
        sections = []
        for section in outline.sections:
            content = self._generate_section(section)
            sections.append(content)

        # 阶段 3: 反思优化
        refined = self._reflect_and_refine(sections)

        # 阶段 4: 组装报告
        report = self._assemble_report(refined)

        return report

    def _react_loop(self, query: str, context: Dict) -> str:
        """ReACT 思考循环"""
        thoughts = []

        for iteration in range(self.max_iterations):
            # 1. 思考
            thought = self._think(query, thoughts, context)
            thoughts.append({"type": "thought", "content": thought})

            # 2. 判断是否可以回答
            if self._should_answer(thoughts):
                return self._final_answer(thoughts)

            # 3. 决策行动
            action = self._decide_action(thought, context)

            # 4. 执行行动
            observation = self._execute_action(action, context)
            thoughts.append({
                "type": "action",
                "action": action,
                "observation": observation
            })

        return self._final_answer(thoughts)
```

---

## 可用工具

### 工具列表

| 工具 | 功能 | 复杂度 | 使用场景 |
|------|------|--------|----------|
| **InsightForge** | 深度洞察检索 | 高 | 复杂问题分析 |
| **PanoramaSearch** | 广度全景搜索 | 中 | 获取全局信息 |
| **QuickSearch** | 快速简单搜索 | 低 | 简单事实查询 |
| **EntityLookup** | 实体详细信息 | 低 | 查看特定实体 |
| **InterviewAgent** | 智能体访谈 | 中 | 获取智能体观点 |

### 工具实现

```python
class ZepToolsService:
    def insight_forge(
        self,
        query: str,
        simulation_requirement: str,
        graph_id: str
    ) -> InsightForgeResult:
        """深度洞察检索 - 最强大的工具"""

        # 1. 生成子问题
        sub_queries = self._generate_sub_queries(query)

        # 2. 多维度并行检索
        semantic_facts = self._semantic_search(sub_queries, graph_id)
        entity_insights = self._entity_insights(sub_queries, graph_id)
        relationship_chains = self._relationship_analysis(graph_id)

        # 3. 综合分析
        synthesis = self._synthesize(
            semantic_facts,
            entity_insights,
            relationship_chains
        )

        return InsightForgeResult(
            query=query,
            sub_queries=sub_queries,
            synthesis=synthesis
        )

    def interview_agent(
        self,
        agent_id: int,
        prompt: str,
        platform: str
    ) -> InterviewResult:
        """访谈智能体"""

        # 通过 IPC 访谈
        ipc_client = SimulationIPCClient(self.simulation_dir)

        response = ipc_client.send_interview(
            agent_id=agent_id,
            prompt=prompt,
            platform=platform
        )

        return InterviewResult(
            agent_id=agent_id,
            question=prompt,
            answer=response.result["response"],
            platform=response.result.get("platform", platform)
        )
```

---

## 报告生成流程

### 1. 大纲规划

```python
def _plan_outline(self, requirement: str) -> ReportOutline:
    """规划报告大纲"""

    # 获取上下文
    context = self._gather_context()

    # 生成大纲 Prompt
    prompt = f"""
基于以下预测需求，规划一份详细的分析报告大纲：

预测需求：{requirement}

模拟信息：
- 实体数量：{context['entities_count']}
- 模拟轮数：{context['total_rounds']}
- 总动作数：{context['total_actions']}

请生成报告大纲，包含：
1. 执行摘要
2. 分析背景
3. 详细分析（3-5个章节）
4. 预测结论
5. 建议和启示
"""

    response = self.llm_client.chat_json(prompt)

    return ReportOutline(
        title=response["title"],
        sections=[
            Section(
                title=s["title"],
                description=s["description"],
                key_points=s.get("key_points", [])
            )
            for s in response["sections"]
        ]
    )
```

### 2. 章节生成

```python
def _generate_section(
    self,
    section: Section,
    context: Dict
) -> SectionContent:
    """生成单个章节"""

    content_parts = []

    # 使用 ReACT 循环生成内容
    for iteration in range(self.max_reflection_rounds + 1):
        # 1. 收集信息
        info = self._collect_section_info(section, context)

        # 2. 生成内容
        prompt = self._build_section_prompt(section, info, iteration)

        content = self.llm_client.chat(
            prompt,
            temperature=self.temperature
        )

        # 3. 如果需要反思
        if iteration < self.max_reflection_rounds:
            critique = self._critique_content(content, section)
            if critique["needs_improvement"]:
                # 根据反馈改进
                continue

        content_parts.append(content)
        break

    return SectionContent(
        title=section.title,
        content="\n\n".join(content_parts)
    )
```

### 3. 反思优化

```python
def _reflect_and_refine(self, sections: List[SectionContent]) -> List[SectionContent]:
    """反思和优化报告"""

    refined = []

    for section in sections:
        # 1. 自我批评
        critique_prompt = f"""
请批评以下章节，指出需要改进的地方：

{section.content}

请从以下角度批评：
1. 内容完整性
2. 逻辑连贯性
3. 数据支撑
4. 结论可信度
"""

        critique = self.llm_client.chat(critique_prompt)

        # 2. 根据批评改进
        if self._needs_improvement(critique):
            refinement_prompt = f"""
基于以下批评意见，改进章节内容：

原始内容：
{section.content}

批评意见：
{critique}

请生成改进后的版本。
"""

            refined_content = self.llm_client.chat(refinement_prompt)
            refined.append(SectionContent(
                title=section.title,
                content=refined_content
            ))
        else:
            refined.append(section)

    return refined
```

---

## 定制 Report Agent

### 添加新工具

```python
class CustomZepTools(ZepToolsService):
    """扩展的工具服务"""

    def custom_analytics(self, query: str) -> Dict:
        """自定义分析工具"""

        # 1. 收集数据
        data = self._collect_data(query)

        # 2. 执行分析
        analysis = self._analyze_data(data)

        # 3. 生成洞察
        insights = self._generate_insights(analysis)

        return {
            "data": data,
            "analysis": analysis,
            "insights": insights
        }

# 注册到 Report Agent
agent = ReportAgent()
agent.register_tool("custom_analytics", custom_tools.custom_analytics)
```

### 自定义报告模板

```python
class CustomReportGenerator(ReportAgent):
    """自定义报告生成器"""

    def _plan_outline(self, requirement: str) -> ReportOutline:
        """自定义大纲规划"""

        # 使用不同的模板
        if self._is_financial_report(requirement):
            return self._financial_outline(requirement)
        elif self._is_social_report(requirement):
            return self._social_outline(requirement)
        else:
            return super()._plan_outline(requirement)

    def _financial_outline(self, requirement: str) -> ReportOutline:
        """金融报告模板"""
        return ReportOutline(
            title="金融市场分析报告",
            sections=[
                Section("市场概况", "..."),
                Section("价格走势", "..."),
                Section("风险评估", "..."),
                Section("投资建议", "...")
            ]
        )
```

### 调整 ReACT 参数

```python
# 创建 Report Agent 时传入配置
agent = ReportAgent(
    max_tool_calls=10,          # 最大工具调用次数
    max_reflection_rounds=3,     # 最大反思轮数
    temperature=0.7,            # 生成温度
    tools=[...],                 # 可用工具
    system_prompt=custom_prompt  # 自定义系统提示
)
```

---

## 最佳实践

### 1. Prompt 优化

```python
# 好的 Prompt
good_prompt = f"""
你是一位资深的{domain}分析专家。

任务：{task}

上下文信息：
{context}

要求：
1. 基于模拟数据进行分析
2. 引用具体的事实和数据
3. 给出明确的结论
4. 标注不确定性

请生成分析报告。
"""

# 不好的 Prompt
bad_prompt = f"生成一份关于{task}的报告。"
```

### 2. 工具选择策略

```python
def _select_tool(self, query: str) -> str:
    """根据查询选择合适的工具"""

    # 简单事实查询
    if self._is_simple_query(query):
        return "quick_search"

    # 复杂关系分析
    if self._needs_relation_analysis(query):
        return "insight_forge"

    # 实体信息
    if self._is_entity_query(query):
        return "entity_lookup"

    # 默认使用深度检索
    return "insight_forge"
```

### 3. 错误处理

```python
def _execute_action_safe(self, action: str, context: Dict) -> str:
    """安全执行动作"""

    try:
        return self._execute_action(action, context)
    except Exception as e:
        logger.error(f"工具执行失败: {action}, {e}")

        # 返回错误信息让 LLM 知道
        return f"工具执行失败: {str(e)}。请尝试其他工具或方法。"
```

---

## 下一步

- **[IPC 通信机制详解](./ipc-communication.md)** - 进程间通信
- **[扩展开发指南](./extension-development.md)** - 如何扩展系统

---

## 自检清单

### 基础理解

- [ ] 能解释 ReACT 模式的核心原理
- [ ] 理解 Report Agent 的工作流程
- [ ] 掌握工具调用的机制
- [ ] 理解报告生成的基本流程

### 进阶掌握

- [ ] 能设计和实现新的检索工具
- [ ] 能优化 Prompt 模板
- [ ] 能调试和优化 Agent 行为
- [ ] 掌握反思机制的使用

### 专家能力

- [ ] 能设计复杂的多步工具链
- [ ] 能实现自定义 ReACT 策略
- [ ] 能优化 Report Agent 性能
- [ ] 能集成新的数据源
