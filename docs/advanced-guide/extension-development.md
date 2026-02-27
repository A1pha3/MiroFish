# 扩展开发指南

> 学习如何扩展和定制 MiroFish 系统

**难度**: ⭐⭐⭐⭐ | **预计时间**: 75 分钟

---

## 学习目标

完成本章节后，你将能够：

### 基础目标（必掌握）

- [ ] 理解 MiroFish 的主要扩展点
- [ ] 掌握添加新 API 端点的方法
- [ ] 能够定制报告模板
- [ ] 理解扩展开发的基本流程

### 进阶目标（建议掌握）

- [ ] 能够为 Report Agent 开发新工具
- [ ] 能够集成外部数据源
- [ ] 能够定制 Profile 生成逻辑
- [ ] 掌握扩展的测试方法

### 专家目标（挑战）

- [ ] 能够为 MiroFish 添加新平台支持
- [ ] 能够设计新的系统架构扩展
- [ ] 能够维护向后兼容性
- [ ] 能够贡献核心扩展功能

---

## 扩展点概览

MiroFish 提供了多个扩展点：

| 扩展点 | 难度 | 说明 |
|--------|------|------|
| **新平台支持** | ⭐⭐⭐⭐⭐ | 添加新的社交平台模拟 |
| **新 API 端点** | ⭐⭐ | 添加自定义 API |
| **新工具** | ⭐⭐⭐ | 为 Report Agent 添加工具 |
| **新报告模板** | ⭐⭐ | 定制报告格式 |
| **新数据源** | ⭐⭐⭐⭐ | 集成外部数据源 |

---

## 扩展 1: 添加新平台支持

### 平台架构

```
新平台 (如微博) 集成
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  1. Profile 格式定义                                        │
│     · convert_to_weibo_profile()                            │
│     · save_weibo_profiles()                                 │
├─────────────────────────────────────────────────────────────┤
│  2. 动作定义                                                  │
│     · WEIBO_ACTIONS                                          │
│     · validate_weibo_action()                               │
├─────────────────────────────────────────────────────────────┤
│  3. 运行脚本                                                  │
│     · run_weibo_simulation.py                               │
│     · IPC 服务器集成                                          │
├─────────────────────────────────────────────────────────────┤
│  4. 后端集成                                                  │
│     · Config 更新                                            │
│     · SimulationRunner 支持                                  │
└─────────────────────────────────────────────────────────────┘
```

### 步骤 1: 定义 Profile 格式

```python
# backend/app/services/platforms/weibo_profile.py

def convert_to_weibo_profile(
    entity: Dict,
    profile_data: Dict,
    index: int
) -> Dict:
    """转换为微博 Profile 格式"""
    return {
        "user_id": index,
        "uid": str(uuid.uuid4()),
        "screen_name": generate_username(entity["name"]),
        "name": entity["name"],
        "description": profile_data.get("bio", ""),
        "followers_count": random.randint(10, 1000),
        "friends_count": random.randint(10, 500),
        "statuses_count": random.randint(0, 1000),
        "verified": False,
        "gender": random.choice(["m", "f"]),
        "location": profile_data.get("location", "北京")
    }

def generate_username(name: str) -> str:
    """生成微博用户名"""
    # 转换为拼音或简单处理
    return f"{name}_official"

def save_weibo_profiles(profiles: List[Dict], output_path: str):
    """保存微博 Profile 文件"""
    with open(output_path, 'w', encoding='utf-8') as f:
        json.dump(profiles, f, ensure_ascii=False, indent=2)
```

### 步骤 2: 定义平台动作

```python
# backend/app/config.py - 添加

class Config:
    # ... 现有配置

    # 微博平台动作
    OASIS_WEIBO_ACTIONS = [
        'CREATE_POST',           # 发布微博
        'REPOST',                # 转发
        'COMMENT',               # 评论
        'LIKE_POST',             # 点赞
        'FOLLOW',                # 关注
        'UNFOLLOW',              # 取消关注
        'DO_NOTHING'             # 不执行动作
    ]
```

### 步骤 3: 创建运行脚本

```python
# backend/scripts/run_weibo_simulation.py

import argparse
import json
from camel_oasis import OasisEnvironment
from app.services.simulation_ipc import SimulationIPCServer, IPCCommand, IPCResponse, CommandType

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--config", required=True, help="配置文件路径")
    parser.add_argument("--simulation-id", required=True, help="模拟 ID")
    args = parser.parse_args()

    # 加载配置
    with open(args.config, 'r') as f:
        config = json.load(f)

    # 初始化 IPC 服务器
    ipc_server = SimulationIPCServer(config["simulation_dir"])
    ipc_server.start()

    # 初始化 OASIS 环境
    environment = OasisEnvironment(
        platform="weibo",
        agents=config["agent_profiles"],
        max_rounds=config["max_rounds"]
    )

    try:
        # 主循环
        for round_num in range(1, config["max_rounds"] + 1):
            # 检查 IPC 命令
            command = ipc_server.poll_commands()
            if command:
                handle_command(command, environment, ipc_server)

            # 运行一轮
            environment.run_round(round_num)

    finally:
        ipc_server.stop()

def handle_command(command: IPCCommand, environment, ipc_server):
    """处理 IPC 命令"""
    # 处理 interview, batch_interview, close_env
    ...

if __name__ == "__main__":
    main()
```

### 步骤 4: 后端集成

```python
# backend/app/services/simulation_runner.py

class SimulationRunner:
    def get_script_path(self, platform: str) -> str:
        """获取平台对应的脚本路径"""
        scripts_dir = os.path.abspath(os.path.join(
            os.path.dirname(__file__), '../../scripts'
        ))

        script_map = {
            "twitter": os.path.join(scripts_dir, "run_twitter_simulation.py"),
            "reddit": os.path.join(scripts_dir, "run_reddit_simulation.py"),
            "weibo": os.path.join(scripts_dir, "run_weibo_simulation.py")  # 新增
        }

        return script_map.get(platform)
```

---

## 扩展 2: 添加新 API 端点

### 创建蓝图

```python
# backend/app/api/custom.py

from flask import Blueprint, request, jsonify
from ..services.custom_service import CustomService

custom_bp = Blueprint('custom', __name__)

@custom_bp.route('/analytics', methods=['POST'])
def generate_analytics():
    """生成自定义分析"""
    data = request.get_json()

    service = CustomService()
    result = service.generate_analytics(data)

    return jsonify({
        "success": True,
        "data": result
    })
```

### 注册蓝图

```python
# backend/app/__init__.py

def create_app(config_class=Config):
    app = Flask(__name__)
    app.config.from_object(config_class)

    # 注册蓝图
    from .api import graph_bp, simulation_bp, report_bp
    from .api import custom_bp  # 新增

    app.register_blueprint(graph_bp, url_prefix='/api/graph')
    app.register_blueprint(simulation_bp, url_prefix='/api/simulation')
    app.register_blueprint(report_bp, url_prefix='/api/report')
    app.register_blueprint(custom_bp, url_prefix='/api/custom')  # 新增

    return app
```

---

## 扩展 3: 添加 Report Agent 工具

### 定义工具

```python
# backend/app/services/custom_tools.py

from typing import Dict, Any

class CustomAnalyticsTool:
    """自定义分析工具"""

    def __init__(self, simulation_id: str, graph_id: str):
        self.simulation_id = simulation_id
        self.graph_id = graph_id

    def analyze_trends(self) -> Dict[str, Any]:
        """分析趋势"""
        # 1. 获取模拟数据
        actions = self._get_simulation_actions()

        # 2. 分析趋势
        trends = self._calculate_trends(actions)

        # 3. 生成洞察
        insights = self._generate_insights(trends)

        return {
            "trends": trends,
            "insights": insights
        }

    def _get_simulation_actions(self) -> List[Dict]:
        """获取模拟动作"""
        from ..simulation_manager import SimulationManager

        manager = SimulationManager()
        sim_dir = manager._get_simulation_dir(self.simulation_id)
        actions_file = os.path.join(sim_dir, "actions.jsonl")

        actions = []
        with open(actions_file, 'r') as f:
            for line in f:
                actions.append(json.loads(line))

        return actions
```

### 注册到 Report Agent

```python
# backend/app/services/report_agent.py

class ReportAgent:
    def __init__(self, ...):
        # ... 现有初始化

        # 注册自定义工具
        from .custom_tools import CustomAnalyticsTool
        self.custom_tools = CustomAnalyticsTool(
            simulation_id=self.simulation_id,
            graph_id=self.graph_id
        )

    def _get_tools(self) -> Dict[str, Callable]:
        tools = {
            # 现有工具
            "insight_forge": self.zep_tools.insight_forge,
            "panorama_search": self.zep_tools.panorama_search,
            "quick_search": self.zep_tools.quick_search,
            "entity_lookup": self.zep_tools.entity_lookup,
            "interview_agent": self._interview_agent,

            # 新增工具
            "analyze_trends": self.custom_tools.analyze_trends
        }
        return tools
```

---

## 扩展 4: 添加报告模板

### 定义模板

```python
# backend/app/services/report_templates.py

class FinancialReportTemplate:
    """金融报告模板"""

    def generate_outline(self, requirement: str, context: Dict) -> Dict:
        """生成报告大纲"""
        return {
            "title": "金融市场分析报告",
            "sections": [
                {
                    "title": "市场概况",
                    "description": "当前市场状态概述",
                    "key_points": ["指数走势", "成交量", "市场情绪"]
                },
                {
                    "title": "价格走势分析",
                    "description": "详细的价格分析",
                    "key_points": ["技术分析", "支撑位/阻力位", "趋势线"]
                },
                {
                    "title": "风险评估",
                    "description": "投资风险分析",
                    "key_points": ["波动率", "下行风险", "系统性风险"]
                },
                {
                    "title": "投资建议",
                    "description": "基于分析的建议",
                    "key_points": ["配置建议", "入场时机", "止损策略"]
                }
            ]
        }

class SocialReportTemplate:
    """社交网络报告模板"""

    def generate_outline(self, requirement: str, context: Dict) -> Dict:
        """生成报告大纲"""
        return {
            "title": "社交网络分析报告",
            "sections": [
                {
                    "title": "网络结构",
                    "description": "社交网络结构分析",
                    "key_points": ["中心节点", "社群划分", "桥接节点"]
                },
                {
                    "title": "信息传播",
                    "description": "信息传播路径分析",
                    "key_points": ["传播速度", "影响范围", "关键传播者"]
                },
                {
                    "title": "舆情演化",
                    "description": "舆情随时间的变化",
                    "key_points": ["情绪变化", "观点分化", "群体极化"]
                }
            ]
        }
```

### 使用模板

```python
# backend/app/services/report_agent.py

class ReportAgent:
    def _plan_outline(self, requirement: str) -> ReportOutline:
        """规划报告大纲"""

        # 根据需求选择模板
        if self._is_financial_report(requirement):
            template = FinancialReportTemplate()
        elif self._is_social_report(requirement):
            template = SocialReportTemplate()
        else:
            template = DefaultReportTemplate()

        outline_dict = template.generate_outline(requirement, self.context)
        return ReportOutline.from_dict(outline_dict)
```

---

## 扩展 5: 集成外部数据源

### 数据源适配器

```python
# backend/app/services/data_sources/base.py

from abc import ABC, abstractmethod

class DataSourceAdapter(ABC):
    """数据源适配器基类"""

    @abstractmethod
    def fetch_data(self, query: str) -> List[Dict]:
        """获取数据"""
        pass

    @abstractmethod
    def normalize(self, raw_data: Any) -> List[Dict]:
        """标准化数据格式"""
        pass

# backend/app/services/data_sources/news_api.py

class NewsAPIAdapter(DataSourceAdapter):
    """新闻 API 适配器"""

    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://newsapi.org/v2"

    def fetch_data(self, query: str) -> List[Dict]:
        """获取新闻数据"""
        import requests

        response = requests.get(
            f"{self.base_url}/everything",
            params={
                "q": query,
                "apiKey": self.api_key,
                "language": "zh"
            }
        )

        return response.json().get("articles", [])

    def normalize(self, raw_data: List[Dict]) -> List[Dict]:
        """标准化为实体数据"""
        normalized = []

        for article in raw_data:
            normalized.append({
                "name": article.get("source", {}).get("name"),
                "type": "NewsSource",
                "description": article.get("description"),
                "url": article.get("url"),
                "published_at": article.get("publishedAt")
            })

        return normalized
```

### 集成到系统

```python
# backend/app/services/text_processor.py

class TextProcessor:
    def __init__(self):
        # ... 现有初始化

        # 初始化数据源
        from .data_sources.news_api import NewsAPIAdapter
        self.news_adapter = NewsAPIAdapter(
            api_key=os.environ.get("NEWS_API_KEY")
        )

    def fetch_external_context(self, query: str) -> str:
        """获取外部上下文信息"""
        news_data = self.news_adapter.fetch_data(query)
        normalized = self.news_adapter.normalize(news_data)

        # 转换为文本
        context_text = "\n\n".join([
            f"{item['name']}: {item['description']}"
            for item in normalized
        ])

        return context_text
```

---

## 测试扩展

### 单元测试

```python
# tests/test_custom_tools.py

import pytest
from app.services.custom_tools import CustomAnalyticsTool

def test_analyze_trends():
    """测试趋势分析"""
    tool = CustomAnalyticsTool(
        simulation_id="test_sim",
        graph_id="test_graph"
    )

    result = tool.analyze_trends()

    assert "trends" in result
    assert "insights" in result
    assert isinstance(result["trends"], dict)
```

### 集成测试

```python
# tests/integration/test_custom_api.py

def test_custom_analytics_endpoint(client):
    """测试自定义分析 API"""
    response = client.post('/api/custom/analytics', json={
        "simulation_id": "test_sim",
        "analysis_type": "trend"
    })

    assert response.status_code == 200
    data = response.get_json()
    assert data["success"] is True
```

---

## 部署扩展

### Docker 部署

```dockerfile
# 扩展基础镜像
FROM ghcr.io/666ghj/mirofish:latest

# 添加自定义代码
COPY backend/app/services/custom_tools.py /app/backend/app/services/
COPY backend/app/api/custom.py /app/backend/app/api/

# 安装额外依赖
RUN pip install newsapi-python

# 更新启动脚本
COPY entrypoint.sh /app/
RUN chmod +x /app/entrypoint.sh

ENTRYPOINT ["/app/entrypoint.sh"]
```

---

## 最佳实践

### 1. 向后兼容

```python
# 使用版本化 API
@custom_bp.route('/v2/analytics', methods=['POST'])
def analytics_v2():
    """新版本 API"""
    ...

@custom_bp.route('/v1/analytics', methods=['POST'])
def analytics_v1():
    """旧版本 API（保持兼容）"""
    ...
```

### 2. 错误处理

```python
def custom_function():
    try:
        # 扩展功能
        result = perform_extended_operation()
        return result
    except SpecificError as e:
        logger.error(f"扩展功能失败: {e}")
        # 返回降级结果
        return fallback_result()
```

### 3. 配置管理

```python
# 添加新的环境变量
class Config:
    # ... 现有配置

    # 扩展配置
    CUSTOM_API_KEY = os.environ.get('CUSTOM_API_KEY')
    CUSTOM_FEATURE_ENABLED = os.environ.get('CUSTOM_FEATURE_ENABLED', 'false').lower() == 'true'
```

---

## 下一步

你已经了解了 MiroFish 的主要扩展方式。开始你的扩展开发吧！

- 查看 GitHub Issues 寻找扩展需求
- 提交你的扩展代码
- 与社区分享你的经验

---

## 自检清单

### 基础技能

- [ ] 能添加新的 API 端点
- [ ] 能定制报告模板
- [ ] 理解扩展开发的基本流程
- [ ] 能编写扩展测试

### 进阶技能

- [ ] 能为 Report Agent 开发新工具
- [ ] 能集成外部数据源
- [ ] 能定制 Profile 生成
- [ ] 能维护向后兼容性

### 专家能力

- [ ] 能添加新平台支持
- [ ] 能设计系统架构扩展
- [ ] 能贡献核心扩展功能
- [ ] 能指导他人进行扩展开发
