# 后端架构详解

> 深入理解 MiroFish 后端的设计思想与实现细节

**难度**: ⭐⭐⭐ | **预计时间**: 60 分钟

---

## 学习目标

完成本章节学习后，你将能够：

### 基础目标（必掌握）

- [ ] 理解 MiroFish 后端的四层架构设计
- [ ] 掌握 Flask 应用工厂模式的实现方式
- [ ] 理解蓝图（Blueprint）的组织方式和路由设计
- [ ] 掌握服务层（Service Layer）的职责划分
- [ ] 理解异步任务处理的基本机制

### 进阶目标（建议掌握）

- [ ] 理解进程间通信（IPC）的设计与实现
- [ ] 掌握状态机在模拟生命周期中的应用
- [ ] 理解 ReACT 模式在 Report Agent 中的实现
- [ ] 能够设计合理的错误处理和重试机制

### 专家目标（挑战）

- [ ] 能够评估和优化系统性能瓶颈
- [ ] 能够设计新的服务扩展点
- [ ] 理解并改进并发处理策略
- [ ] 能够进行架构级别的优化

---

## 为什么这样设计？

### 设计原则

MiroFish 后端架构遵循以下核心原则：

| 原则 | 说明 | 实现方式 |
|------|------|----------|
| **关注点分离** | 每一层只负责特定的职责 | API/Service/Utility/External 四层架构 |
| **可测试性** | 便于单元测试和集成测试 | 依赖注入、应用工厂模式 |
| **可扩展性** | 易于添加新功能 | 蓝图机制、服务抽象 |
| **容错性** | 优雅处理错误和重试 | 重试装饰器、异常捕获 |
| **可观测性** | 便于监控和调试 | 结构化日志、任务追踪 |

### 架构演进

**v1.0 - 简单 Flask 应用**
```
问题：所有代码混在一起，难以维护
```

**v2.0 - 引入蓝图**
```
改进：按功能域划分 API 模块
遗留：业务逻辑仍在路由处理函数中
```

**v3.0 - 服务层抽象**（当前版本）
```
改进：引入 Service Layer，分离业务逻辑
收益：代码复用、易于测试、清晰职责
```

---

## 架构概览

### 四层架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        API Layer (表示层)                               │
│                        职责：路由、请求验证、响应格式化                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐     │
│  │   graph_bp       │  │  simulation_bp   │  │   report_bp      │     │
│  │  /api/graph/*    │  │ /api/simulation/*│  │  /api/report/*   │     │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘     │
│  Flask Blueprints → RESTful API Endpoints                              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      Service Layer (业务逻辑层)                         │
│                        职责：核心业务逻辑、工作流编排                    │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐     │
│  │  GraphBuilder    │  │ SimulationManager│  │   ReportAgent    │     │
│  │  OntologyGenerator│ │ ProfileGenerator │  │  ZepToolsService │     │
│  │  ConfigGenerator │  │ SimulationRunner │  │                  │     │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘     │
│  Stateless Services → Business Logic                                   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       Utility Layer (工具层)                            │
│                        职责：通用工具、基础设施                          │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐     │
│  │   LLMClient      │  │   FileParser     │  │  StructuredLogger│     │
│  │  RetryDecorator  │  │  TextProcessor   │  │   Validators     │     │
│  │  TaskManager     │  │  EntityReader    │  │                  │     │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘     │
│  Reusable Utilities → Cross-cutting Concerns                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    External Services (外部服务层)                       │
│                        职责：与外部系统交互                              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐     │
│  │   Zep Cloud      │  │     OASIS        │  │   LLM API        │     │
│  │   GraphRAG       │  │  Social Sim      │  │  OpenAI/Compatible│     │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘     │
│  External APIs → Third-party Services                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 数据流图

```
用户请求
    │
    ▼
┌─────────┐
│ API 层  │ ← 验证请求参数、解析输入
└────┬────┘
     │
     ▼
┌─────────┐
│服务层   │ ← 执行业务逻辑、调用工具层
└────┬────┘
     │
     ▼
┌─────────┐
│工具层   │ ← 封装外部 API 调用、数据处理
└────┬────┘
     │
     ▼
┌──────────────────┐
│ 外部服务（Zep/LLM)│
└──────────────────┘
     │
     ▼
   响应数据 ← 原路返回
```

---

## 核心模块详解

### 1. 应用工厂模式 (`app/__init__.py`)

#### 为什么使用应用工厂？

> **设计决策**：应用工厂模式 vs 直接实例化 Flask

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 直接实例化 | 简单直接 | 难以测试、配置不灵活 | 小型项目 |
| 应用工厂 | 易于测试、多实例、灵活配置 | 稍复杂 | 生产级项目 ✅ |

#### 完整实现

```python
# app/__init__.py
from flask import Flask
from flask_cors import CORS
from .config import Config
import atexit

def create_app(config_class=Config):
    """应用工厂函数

    设计要点：
    1. 延迟初始化 - 便于测试
    2. 配置注入 - 支持多环境
    3. 蓝图注册 - 模块化组织
    4. 生命周期管理 - 清理资源

    Args:
        config_class: 配置类，默认为 Config

    Returns:
        Flask: 配置好的应用实例
    """
    app = Flask(__name__)

    # 1. 加载配置
    app.config.from_object(config_class)

    # 2. 配置 JSON 编码（支持中文）
    app.json.ensure_ascii = False
    app.json.sort_keys = False  # 保持字段顺序

    # 3. 配置 CORS
    CORS(app, resources={
        r"/api/*": {
            "origins": "*",
            "methods": ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
            "allow_headers": ["Content-Type"]
        }
    })

    # 4. 注册蓝图
    _register_blueprints(app)

    # 5. 注册错误处理器
    _register_error_handlers(app)

    # 6. 注册生命周期钩子
    _register_lifecycle_hooks(app)

    return app


def _register_blueprints(app):
    """注册所有蓝图"""
    from .api.graph import graph_bp
    from .api.simulation import simulation_bp
    from .api.report import report_bp

    app.register_blueprint(graph_bp, url_prefix='/api/graph')
    app.register_blueprint(simulation_bp, url_prefix='/api/simulation')
    app.register_blueprint(report_bp, url_prefix='/api/report')


def _register_error_handlers(app):
    """注册全局错误处理器"""
    from flask import jsonify

    @app.errorhandler(400)
    def bad_request(error):
        return jsonify({"error": "Bad Request", "message": str(error)}), 400

    @app.errorhandler(404)
    def not_found(error):
        return jsonify({"error": "Not Found", "message": str(error)}), 404

    @app.errorhandler(500)
    def internal_error(error):
        return jsonify({"error": "Internal Server Error", "message": str(error)}), 500


def _register_lifecycle_hooks(app):
    """注册生命周期钩子"""
    from app.services.simulation.runner import SimulationRunner

    # 服务器关闭时清理
    atexit.register(SimulationRunner.cleanup_all)

    @app.teardown_appcontext
    def shutdown_session(exception=None):
        # 清理请求级别的资源
        pass
```

#### 测试友好设计

```python
# tests/conftest.py
import pytest
from app import create_app
from app.config import TestConfig

@pytest.fixture
def app():
    """创建测试应用实例"""
    app = create_app(TestConfig)
    yield app
    # 清理代码


@pytest.fixture
def client(app):
    """创建测试客户端"""
    return app.test_client()
```

---

### 2. 配置管理 (`app/config.py`)

#### 环境变量优先级

```
命令行参数 > 环境变量 > 配置文件 > 默认值
```

#### 完整配置类

```python
# app/config.py
import os
from pathlib import Path

class Config:
    """基础配置类

    配置加载顺序：
    1. 环境变量
    2. .env 文件
    3. 默认值
    """

    # ==================== Flask 配置 ====================
    SECRET_KEY = os.environ.get('SECRET_KEY', 'dev-secret-key')
    DEBUG = os.environ.get('DEBUG', 'False').lower() == 'true'
    TESTING = False

    # ==================== LLM 配置 ====================
    LLM_API_KEY = os.environ.get('LLM_API_KEY')
    LLM_BASE_URL = os.environ.get('LLM_BASE_URL', 'https://api.openai.com/v1')
    LLM_MODEL_NAME = os.environ.get('LLM_MODEL_NAME', 'gpt-4o-mini')

    # LLM 超时和重试
    LLM_TIMEOUT = int(os.environ.get('LLM_TIMEOUT', '60'))
    LLM_MAX_RETRIES = int(os.environ.get('LLM_MAX_RETRIES', '3'))

    # ==================== Zep Cloud 配置 ====================
    ZEP_API_KEY = os.environ.get('ZEP_API_KEY')

    # ==================== 文件上传配置 ====================
    MAX_CONTENT_LENGTH = 50 * 1024 * 1024  # 50MB
    UPLOAD_FOLDER = Path(os.environ.get(
        'UPLOAD_FOLDER',
        os.path.join(os.getcwd(), 'uploads')
    ))
    ALLOWED_EXTENSIONS = {'pdf', 'md', 'txt', 'markdown'}

    # ==================== OASIS 配置 ====================
    OASIS_DEFAULT_MAX_ROUNDS = int(os.environ.get(
        'OASIS_DEFAULT_MAX_ROUNDS', '40'
    ))
    OASIS_DEFAULT_PLATFORM = os.environ.get(
        'OASIS_DEFAULT_PLATFORM', 'twitter'
    )

    # ==================== 报告生成配置 ====================
    REPORT_MAX_TOOL_CALLS = int(os.environ.get(
        'REPORT_MAX_TOOL_CALLS', '20'
    ))
    REPORT_MAX_AGENTS = int(os.environ.get(
        'REPORT_MAX_AGENTS', '10'
    ))

    # ==================== 日志配置 ====================
    LOG_LEVEL = os.environ.get('LOG_LEVEL', 'INFO')
    LOG_FORMAT = '%(asctime)s - %(name)s - %(levelname)s - %(message)s'

    # ==================== IPC 配置 ====================
    IPC_COMMANDS_DIR = 'commands'
    IPC_RESPONSES_DIR = 'responses'
    IPC_TIMEOUT = int(os.environ.get('IPC_TIMEOUT', '30'))


class DevelopmentConfig(Config):
    """开发环境配置"""
    DEBUG = True
    LOG_LEVEL = 'DEBUG'


class ProductionConfig(Config):
    """生产环境配置"""
    DEBUG = False
    TESTING = False

    @classmethod
    def init_app(cls, app):
        # 生产环境额外配置
        import logging
        from logging.handlers import RotatingFileHandler

        if not os.path.exists('logs'):
            os.mkdir('logs')

        file_handler = RotatingFileHandler(
            'logs/mirofish.log',
            maxBytes=10240000,
            backupCount=10
        )
        file_handler.setFormatter(logging.Formatter(
            '%(asctime)s %(levelname)s: %(message)s'
        ))
        file_handler.setLevel(logging.INFO)
        app.logger.addHandler(file_handler)


class TestConfig(Config):
    """测试环境配置"""
    TESTING = True
    LLM_TIMEOUT = 5  # 测试时缩短超时
    UPLOAD_FOLDER = '/tmp/mirofish-test-uploads'


# 配置字典
config = {
    'development': DevelopmentConfig,
    'production': ProductionConfig,
    'testing': TestConfig,
    'default': DevelopmentConfig
}
```

---

### 3. 蓝图架构

#### 蓝图组织原则

```
按功能域划分 → 每个功能域一个蓝图
每个蓝图独立 → 便于维护和测试
统一响应格式 → API 一致性
```

#### 图谱蓝图示例

```python
# app/api/graph.py
from flask import Blueprint, request, jsonify
from app.services.graph.builder import GraphBuilder
from app.services.graph.ontology import OntologyGenerator
from app.utils.logger import get_logger
from app.utils.decorators import validate_request

graph_bp = Blueprint('graph', __name__)
logger = get_logger(__name__)


@graph_bp.route('/generate-ontology', methods=['POST'])
@validate_request(required_fields=['text'])
def generate_ontology():
    """生成本体定义

    Request:
        {
            "text": str,              # 输入文本
            "user_requirement": str   # 用户需求（可选）
        }

    Response:
        {
            "success": true,
            "data": {
                "entity_types": [...],
                "relation_types": [...]
            }
        }
    """
    try:
        data = request.get_json()
        text = data['text']
        user_requirement = data.get('user_requirement', '社会模拟')

        logger.info("生成本体请求", extra={
            "text_length": len(text),
            "user_requirement": user_requirement
        })

        generator = OntologyGenerator()
        ontology = generator.generate(text, user_requirement)

        return jsonify({
            "success": True,
            "data": ontology.to_dict()
        })

    except Exception as e:
        logger.error("本体生成失败", extra={"error": str(e)})
        return jsonify({
            "success": False,
            "error": str(e)
        }), 500


@graph_bp.route('/build', methods=['POST'])
@validate_request(required_fields=['text', 'ontology'])
def build_graph():
    """构建知识图谱（异步）

    Request:
        {
            "text": str,
            "ontology": dict,
            "session_id": str
        }

    Response:
        {
            "success": true,
            "data": {
                "task_id": str,     # 任务 ID
                "status": "pending"
            }
        }
    """
    try:
        data = request.get_json()
        text = data['text']
        ontology = data['ontology']
        session_id = data.get('session_id')

        builder = GraphBuilder()
        task_id = builder.build_graph_async(
            text=text,
            ontology=ontology,
            session_id=session_id
        )

        return jsonify({
            "success": True,
            "data": {
                "task_id": task_id,
                "status": "pending"
            }
        })

    except Exception as e:
        logger.error("图谱构建失败", extra={"error": str(e)})
        return jsonify({
            "success": False,
            "error": str(e)
        }), 500


@graph_bp.route('/task/<task_id>', methods=['GET'])
def get_task_status(task_id):
    """获取任务状态"""
    try:
        from app.utils.task_manager import get_task_status

        status = get_task_status(task_id)

        return jsonify({
            "success": True,
            "data": status
        })

    except Exception as e:
        return jsonify({
            "success": False,
            "error": str(e)
        }), 500


@graph_bp.route('/entities/<graph_id>', methods=['GET'])
def get_entities(graph_id):
    """获取图谱实体

    Query Params:
        limit: 数量限制
        offset: 偏移量
        type: 实体类型过滤
    """
    try:
        limit = request.args.get('limit', 100, type=int)
        offset = request.args.get('offset', 0, type=int)
        entity_type = request.args.get('type')

        from app.services.graph.entity_reader import EntityReader
        reader = EntityReader()

        entities = reader.get_entities(
            graph_id=graph_id,
            limit=limit,
            offset=offset,
            entity_type=entity_type
        )

        return jsonify({
            "success": True,
            "data": entities
        })

    except Exception as e:
        return jsonify({
            "success": False,
            "error": str(e)
        }), 500
```

#### 请求验证装饰器

```python
# app/utils/decorators.py
from functools import wraps
from flask import request, jsonify

def validate_request(required_fields=None, optional_fields=None):
    """请求验证装饰器

    Args:
        required_fields: 必需字段列表
        optional_fields: 可选字段列表（用于类型检查）
    """
    def decorator(f):
        @wraps(f)
        def wrapped(*args, **kwargs):
            # 验证 Content-Type
            if request.method in ['POST', 'PUT', 'PATCH']:
                if not request.is_json:
                    return jsonify({
                        "success": False,
                        "error": "Content-Type must be application/json"
                    }), 400

            # 验证必需字段
            if required_fields:
                data = request.get_json() or {}
                missing = [f for f in required_fields if f not in data]
                if missing:
                    return jsonify({
                        "success": False,
                        "error": f"Missing required fields: {', '.join(missing)}"
                    }), 400

            return f(*args, **kwargs)
        return wrapped
    return decorator
```

---

### 4. 异步任务处理

#### 为什么需要异步处理？

GraphRAG 图谱构建可能需要几分钟，不能阻塞 HTTP 请求。

#### 任务管理器实现

```python
# app/utils/task_manager.py
import threading
import time
from dataclasses import dataclass, asdict
from enum import Enum
from typing import Optional, Callable
from app.utils.logger import get_logger

logger = get_logger(__name__)


class TaskStatus(Enum):
    """任务状态枚举"""
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"


@dataclass
class Task:
    """任务数据结构"""
    task_id: str
    task_type: str
    status: TaskStatus
    progress: float  # 0.0 - 1.0
    result: Optional[dict]
    error: Optional[str]
    created_at: float
    updated_at: float


class TaskManager:
    """任务管理器

    职责：
    1. 创建和追踪任务
    2. 线程安全的任务状态更新
    3. 提供任务查询接口
    """

    _instance = None
    _lock = threading.Lock()

    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
                    cls._instance._tasks = {}
        return cls._instance

    def create_task(self, task_type: str) -> str:
        """创建新任务

        Args:
            task_type: 任务类型标识

        Returns:
            str: 任务 ID
        """
        import uuid
        task_id = f"{task_type}_{uuid.uuid4().hex[:8]}"

        task = Task(
            task_id=task_id,
            task_type=task_type,
            status=TaskStatus.PENDING,
            progress=0.0,
            result=None,
            error=None,
            created_at=time.time(),
            updated_at=time.time()
        )

        with self._lock:
            self._tasks[task_id] = task

        logger.info("任务创建", extra={"task_id": task_id, "task_type": task_type})
        return task_id

    def update_task(self, task_id: str, **updates):
        """更新任务状态

        Args:
            task_id: 任务 ID
            **updates: 要更新的字段
        """
        with self._lock:
            if task_id in self._tasks:
                task = self._tasks[task_id]
                for key, value in updates.items():
                    if hasattr(task, key):
                        setattr(task, key, value)
                task.updated_at = time.time()

    def get_task(self, task_id: str) -> Optional[Task]:
        """获取任务信息"""
        with self._lock:
            return self._tasks.get(task_id)

    def run_task_async(
        self,
        task_id: str,
        worker_func: Callable,
        args: tuple = (),
        kwargs: dict = None
    ):
        """异步执行任务

        Args:
            task_id: 任务 ID
            worker_func: 工作函数
            args: 位置参数
            kwargs: 关键字参数
        """
        kwargs = kwargs or {}

        def worker_wrapper():
            try:
                self.update_task(task_id, status=TaskStatus.RUNNING)
                logger.info("任务开始执行", extra={"task_id": task_id})

                result = worker_func(*args, **kwargs)

                self.update_task(
                    task_id,
                    status=TaskStatus.COMPLETED,
                    progress=1.0,
                    result=result
                )
                logger.info("任务完成", extra={"task_id": task_id})

            except Exception as e:
                self.update_task(
                    task_id,
                    status=TaskStatus.FAILED,
                    error=str(e)
                )
                logger.error("任务失败", extra={"task_id": task_id, "error": str(e)})

        thread = threading.Thread(target=worker_wrapper, daemon=True)
        thread.start()


# 全局实例
task_manager = TaskManager()


def get_task_status(task_id: str) -> Optional[dict]:
    """获取任务状态（API 友好）"""
    task = task_manager.get_task(task_id)
    if task:
        return {
            "task_id": task.task_id,
            "task_type": task.task_type,
            "status": task.status.value,
            "progress": task.progress,
            "result": task.result,
            "error": task.error,
            "created_at": task.created_at,
            "updated_at": task.updated_at
        }
    return None
```

#### 使用示例

```python
# app/services/graph/builder.py
from app.utils.task_manager import task_manager

class GraphBuilder:
    def build_graph_async(self, text: str, ontology: dict, session_id: str = None) -> str:
        """异步构建图谱

        Returns:
            str: 任务 ID
        """
        # 1. 创建任务
        task_id = task_manager.create_task("graph_build")

        # 2. 异步执行
        task_manager.run_task_async(
            task_id=task_id,
            worker_func=self._build_graph_worker,
            kwargs={
                "text": text,
                "ontology": ontology,
                "session_id": session_id,
                "task_id": task_id
            }
        )

        return task_id

    def _build_graph_worker(self, text: str, ontology: dict, task_id: str, **kwargs):
        """实际的工作函数（在后台线程执行）"""
        from app.utils.task_manager import task_manager

        try:
            # 步骤 1: 连接到 Zep
            task_manager.update_task(task_id, progress=0.1)
            session = self._create_zep_session()

            # 步骤 2: 添加文档
            task_manager.update_task(task_id, progress=0.3)
            self._add_documents(session, text)

            # 步骤 3: 构建图谱
            task_manager.update_task(task_id, progress=0.7)
            graph_id = self._extract_graph(session)

            # 完成
            task_manager.update_task(task_id, progress=1.0)
            return {"graph_id": graph_id}

        except Exception as e:
            raise
```

---

### 5. 状态机设计

#### SimulationStatus 状态机

```
                    ┌─────────────┐
                    │   CREATED   │  初始状态
                    └──────┬──────┘
                           │ prepare()
                           ▼
                    ┌─────────────┐
              ┌────→│  PREPARING  │  准备中（生成 Profile、配置）
              │     └──────┬──────┘
              │            │ complete / fail
              │            ▼
              │     ┌─────────────┐
              │     │    READY    │  就绪（可以启动）
              │     └──────┬──────┘
              │            │ start()
              │            ▼
              │     ┌─────────────┐
              │     │   RUNNING   │  运行中
              │     └──────┬──────┘
              │            │ complete / pause / error
              │     ┌──────┴──────┐
              │     │             │
              │     ▼             ▼
              │  ┌─────────┐  ┌─────────┐
              │  │COMPLETED│  │ PAUSED  │
              │  └─────────┘  └────┬────┘
              │                     │ stop()
              │                     ▼
              │                  ┌─────────┐
              └─────────────────→│ STOPPED │
                                └─────────┘
                                     │
                              所有状态都可能失败
                                     ▼
                                ┌─────────┐
                                │  FAILED  │
                                └─────────┘
```

#### 状态转换验证

```python
# app/services/simulation/state_machine.py
from enum import Enum
from typing import Dict, Set

class SimulationStatus(Enum):
    """模拟状态枚举"""
    CREATED = "created"
    PREPARING = "preparing"
    READY = "ready"
    RUNNING = "running"
    PAUSED = "paused"
    STOPPED = "stopped"
    COMPLETED = "completed"
    FAILED = "failed"


class StateMachine:
    """状态机

    确保状态转换的合法性
    """

    # 定义合法的状态转换
    _VALID_TRANSITIONS: Dict[SimulationStatus, Set[SimulationStatus]] = {
        SimulationStatus.CREATED: {SimulationStatus.PREPARING, SimulationStatus.FAILED},
        SimulationStatus.PREPARING: {SimulationStatus.READY, SimulationStatus.FAILED},
        SimulationStatus.READY: {SimulationStatus.RUNNING, SimulationStatus.FAILED},
        SimulationStatus.RUNNING: {
            SimulationStatus.COMPLETED,
            SimulationStatus.PAUSED,
            SimulationStatus.FAILED
        },
        SimulationStatus.PAUSED: {SimulationStatus.RUNNING, SimulationStatus.STOPPED},
        SimulationStatus.STOPPED: set(),
        SimulationStatus.COMPLETED: set(),
        SimulationStatus.FAILED: set(),
    }

    @classmethod
    def can_transition(
        cls,
        from_status: SimulationStatus,
        to_status: SimulationStatus
    ) -> bool:
        """检查状态转换是否合法

        Args:
            from_status: 当前状态
            to_status: 目标状态

        Returns:
            bool: 转换是否合法
        """
        valid_targets = cls._VALID_TRANSITIONS.get(from_status, set())
        return to_status in valid_targets

    @classmethod
    def transition(
        cls,
        from_status: SimulationStatus,
        to_status: SimulationStatus
    ) -> SimulationStatus:
        """执行状态转换

        Args:
            from_status: 当前状态
            to_status: 目标状态

        Returns:
            SimulationStatus: 新状态

        Raises:
            ValueError: 非法的状态转换
        """
        if not cls.can_transition(from_status, to_status):
            raise ValueError(
                f"非法的状态转换: {from_status.value} → {to_status.value}"
            )
        return to_status


# 使用示例
class SimulationManager:
    def prepare_simulation(self, sim_id: str):
        """准备模拟"""
        state = self._load_state(sim_id)

        # 检查状态
        if state.status != SimulationStatus.CREATED:
            raise ValueError(
                f"只能准备 CREATED 状态的模拟，当前状态: {state.status.value}"
            )

        # 状态转换
        state.status = StateMachine.transition(
            state.status,
            SimulationStatus.PREPARING
        )

        try:
            # 执行准备工作
            self._do_prepare(state)

            # 转换到 READY
            state.status = StateMachine.transition(
                state.status,
                SimulationStatus.READY
            )
        except Exception as e:
            # 转换到 FAILED
            state.status = StateMachine.transition(
                state.status,
                SimulationStatus.FAILED
            )
            raise
```

---

## 并发处理模式

### 1. 线程池并行处理

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

class ProfileGenerator:
    """智能体 Profile 生成器"""

    def generate_profiles(
        self,
        entities: list,
        parallel_count: int = 3
    ) -> list:
        """并行生成 Profile

        Args:
            entities: 实体列表
            parallel_count: 并发数

        Returns:
            list: Profile 列表
        """
        results = []

        with ThreadPoolExecutor(max_workers=parallel_count) as executor:
            # 提交所有任务
            future_to_entity = {
                executor.submit(self._generate_profile, entity): entity
                for entity in entities
            }

            # 收集结果
            for future in as_completed(future_to_entity):
                entity = future_to_entity[future]
                try:
                    profile = future.result()
                    results.append(profile)
                except Exception as e:
                    logger.error(
                        "Profile 生成失败",
                        extra={"entity_name": entity.name, "error": str(e)}
                    )
                    # 使用降级策略
                    results.append(self._generate_fallback_profile(entity))

        return results
```

### 2. 异步 I/O 优化

```python
import asyncio
import aiohttp

class AsyncLLMClient:
    """异步 LLM 客户端"""

    async def chat_async(self, messages: list) -> str:
        """异步聊天"""
        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.base_url}/chat/completions",
                headers={
                    "Authorization": f"Bearer {self.api_key}",
                    "Content-Type": "application/json"
                },
                json={
                    "model": self.model,
                    "messages": messages
                }
            ) as response:
                result = await response.json()
                return result["choices"][0]["message"]["content"]

    async def batch_chat(self, messages_list: list) -> list:
        """批量并发请求"""
        tasks = [self.chat_async(msgs) for msgs in messages_list]
        return await asyncio.gather(*tasks)
```

---

## 错误处理策略

### 重试装饰器

```python
# app/utils/decorators.py
import time
import functools
from typing import Callable, Type
from app.utils.logger import get_logger

logger = get_logger(__name__)


def retry_with_backoff(
    max_retries: int = 3,
    initial_delay: float = 1.0,
    backoff_factor: float = 2.0,
    exceptions: tuple = (Exception,)
):
    """指数退避重试装饰器

    Args:
        max_retries: 最大重试次数
        initial_delay: 初始延迟（秒）
        backoff_factor: 退避因子
        exceptions: 需要重试的异常类型
    """
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            delay = initial_delay
            last_exception = None

            for attempt in range(max_retries + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    last_exception = e

                    if attempt < max_retries:
                        logger.warning(
                            f"{func.__name__} 失败，{delay}秒后重试 ({attempt + 1}/{max_retries})",
                            extra={"error": str(e)}
                        )
                        time.sleep(delay)
                        delay *= backoff_factor
                    else:
                        logger.error(
                            f"{func.__name__} 达到最大重试次数",
                            extra={"error": str(e)}
                        )

            raise last_exception

        return wrapper
    return decorator


# 使用示例
@retry_with_backoff(max_retries=3, initial_delay=1.0)
def call_zep_api(prompt: str) -> dict:
    """调用 Zep API（带重试）"""
    # 可能失败的 API 调用
    ...
```

---

## 数据持久化

### 文件系统存储

MiroFish 使用文件系统存储项目数据：

```
data/
├── projects/
│   ├── {project_id}/
│   │   ├── project.json          # 项目元数据
│   │   ├── graph/
│   │   │   ├── ontology.json     # 本体定义
│   │   │   └── entities.json     # 实体列表
│   │   ├── simulations/
│   │   │   ├── {sim_id}/
│   │   │   │   ├── simulation.json
│   │   │   │   ├── profiles.json
│   │   │   │   ├── config.yaml
│   │   │   │   └── runs/
│   │   │   │       └── {run_id}/
│   │   │   │           ├── actions.json
│   │   │   │           └── interview.json
│   │   │   └── ...
│   │   └── reports/
│   │       └── {report_id}/
│   │           ├── report.json
│   │           └── chat_history.json
```

### 存储服务抽象

```python
# app/services/storage.py
from abc import ABC, abstractmethod
from pathlib import Path
import json
import shutil

class StorageBackend(ABC):
    """存储后端抽象接口"""

    @abstractmethod
    def save(self, path: str, data: dict):
        """保存数据"""
        pass

    @abstractmethod
    def load(self, path: str) -> dict:
        """加载数据"""
        pass

    @abstractmethod
    def exists(self, path: str) -> bool:
        """检查路径是否存在"""
        pass


class FileSystemStorage(StorageBackend):
    """文件系统存储实现"""

    def __init__(self, base_path: str):
        self.base_path = Path(base_path)
        self.base_path.mkdir(parents=True, exist_ok=True)

    def _resolve_path(self, path: str) -> Path:
        """解析完整路径"""
        return self.base_path / path

    def save(self, path: str, data: dict):
        """保存 JSON 数据"""
        file_path = self._resolve_path(path)
        file_path.parent.mkdir(parents=True, exist_ok=True)

        with open(file_path, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)

    def load(self, path: str) -> dict:
        """加载 JSON 数据"""
        file_path = self._resolve_path(path)

        with open(file_path, 'r', encoding='utf-8') as f:
            return json.load(f)

    def exists(self, path: str) -> bool:
        """检查文件是否存在"""
        return self._resolve_path(path).exists()

    def delete(self, path: str):
        """删除文件或目录"""
        file_path = self._resolve_path(path)
        if file_path.is_dir():
            shutil.rmtree(file_path)
        else:
            file_path.unlink()


# 全局存储实例
storage = FileSystemStorage('data')
```

---

## 日志系统

### 结构化日志

```python
# app/utils/logger.py
import logging
import sys
from typing import Any

class StructuredFormatter(logging.Formatter):
    """结构化日志格式化器"""

    def format(self, record: logging.LogRecord) -> str:
        # 基础信息
        log_data = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
        }

        # 额外上下文
        if hasattr(record, 'extra'):
            log_data.update(record.extra)

        # 异常信息
        if record.exc_info:
            log_data["exception"] = self.formatException(record.exc_info)

        return json.dumps(log_data, ensure_ascii=False)


def get_logger(name: str) -> logging.Logger:
    """获取结构化日志记录器

    Args:
        name: 日志记录器名称

    Returns:
        logging.Logger: 配置好的日志记录器
    """
    logger = logging.getLogger(name)

    if not logger.handlers:
        # 控制台处理器
        console_handler = logging.StreamHandler(sys.stdout)
        console_handler.setFormatter(StructuredFormatter())

        logger.addHandler(console_handler)
        logger.setLevel(logging.INFO)

    return logger


# 使用示例
logger = get_logger('mirofish.graph')

logger.info("图谱构建开始", extra={
    "project_id": project_id,
    "text_length": len(text),
    "entity_types": len(ontology.entity_types)
})
```

---

## 性能优化建议

### 1. 数据库连接池

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    'postgresql://...',
    poolclass=QueuePool,
    pool_size=10,
    max_overflow=20,
    pool_pre_ping=True  # 连接健康检查
)
```

### 2. 缓存策略

```python
from functools import lru_cache
import hashlib

class CacheOntology:
    """本体缓存装饰器"""

    def __init__(self, maxsize: int = 128):
        self.maxsize = maxsize

    def __call__(self, func):
        @lru_cache(maxsize=self.maxsize)
        def cached(text_hash: str, requirement: str):
            return func(text, requirement)

        def wrapper(text: str, requirement: str):
            text_hash = hashlib.md5(text.encode()).hexdigest()
            return cached(text_hash, requirement)

        return wrapper


@CacheOntology(maxsize=100)
def generate_ontology(text: str, requirement: str) -> dict:
    """带缓存的本体生成"""
    ...
```

---

## 学习检查清单

### 基础理解

- [ ] 能解释四层架构的职责划分
- [ ] 能说出应用工厂模式的好处
- [ ] 能理解蓝图的作用
- [ ] 能实现一个简单的 API 端点

### 进阶掌握

- [ ] 能设计状态机处理复杂业务流程
- [ ] 能实现异步任务处理
- [ ] 能设计合理的重试机制
- [ ] 能使用结构化日志

### 专家能力

- [ ] 能识别性能瓶颈
- [ ] 能设计缓存策略
- [ ] 能进行架构优化
- [ ] 能设计新的服务扩展点

---

## 下一步学习

- **[核心服务详解](./core-services.md)** - 深入各服务的具体实现
- **[API 接口文档](./api-reference.md)** - 完整的 API 参考
- **[前端架构详解](./frontend-architecture.md)** - 了解前端实现
