# IPC 通信机制详解

> 深入理解 Flask 后端与 OASIS 模拟进程之间的通信

**难度**: ⭐⭐⭐ | **预计时间**: 30 分钟

---

## 学习目标

完成本章节后，你将能够：

### 基础目标（必掌握）

- [ ] 理解为什么需要进程间通信（IPC）
- [ ] 掌握 MiroFish IPC 架构的设计
- [ ] 理解命令和响应的传递机制
- [ ] 能够排查基本的通信问题

### 进阶目标（建议掌握）

- [ ] 理解 IPC 服务器的实现细节
- [ ] 能够扩展新的命令类型
- [ ] 掌握超时和错误处理机制
- [ ] 能够优化 IPC 通信性能

### 专家目标（挑战）

- [ ] 能够设计更高效的通信机制
- [ ] 能够实现双向通信优化
- [ ] 能够处理大规模并发的 IPC 场景
- [ ] 能够设计替代通信方案（如消息队列）

---

## 为什么需要 IPC？

MiroFish 中，OASIS 模拟作为**独立子进程**运行，原因：

1. **长时间运行**：模拟可能运行数小时，不能阻塞 Flask 服务
2. **资源隔离**：模拟进程的资源使用不影响主服务
3. **独立管理**：可以独立启动、停止、监控模拟进程
4. **错误隔离**：模拟崩溃不影响后端服务

```
┌─────────────────────────────────────────────────────────────┐
│                    MiroFish 后端                             │
│                    (Flask 主进程)                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  API 请求                                                   │
│    │                                                         │
│    ▼                                                         │
│  ┌──────────────┐     ┌──────────────┐                     │
│  │ IPC Client   │────▶│ IPC Commands │                     │
│  └──────────────┘     └──────────────┘                     │
│       │                                                         │
│       │ 文件系统                                               │
│       ▼                                                         │
│  ┌──────────────────────────────────────────────┐          │
│  │         commands/                            │          │
│  │  · interview_a1b2c3.json                     │          │
│  │  · batch_interview_x9y8z7.json               │          │
│  └──────────────────────────────────────────────┘          │
│                                                              │
│  ┌──────────────────────────────────────────────┐          │
│  │         responses/                           │          │
│  │  · interview_a1b2c3.json                     │          │
│  │  · batch_interview_x9y8z7.json               │          │
│  └──────────────────────────────────────────────┘          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                              ▲
                              │ 文件系统
┌─────────────────────────────────────────────────────────────┐
│                    OASIS 模拟进程                             │
│                    (子进程)                                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐     ┌──────────────┐                     │
│  │ IPC Server   │◀────│ IPC Poller   │                     │
│  └──────────────┘     └──────────────┘                     │
│       │                                                         │
│       ▼                                                         │
│  处理命令并返回响应                                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## IPC 设计

### 命令格式

```json
{
  "command_id": "uuid-v4",
  "command_type": "interview",
  "args": {
    "agent_id": 1,
    "prompt": "你最近怎么样？",
    "platform": null
  },
  "timestamp": "2024-01-01T10:00:00Z"
}
```

### 响应格式

```json
{
  "command_id": "uuid-v4",
  "status": "completed",
  "result": {
    "agent_id": 1,
    "agent_name": "张三",
    "response": "我最近还不错..."
  },
  "error": null,
  "timestamp": "2024-01-01T10:00:05Z"
}
```

### 状态枚举

```python
class CommandStatus(str, Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"

class CommandType(str, Enum):
    INTERVIEW = "interview"
    BATCH_INTERVIEW = "batch_interview"
    CLOSE_ENV = "close_env"
```

---

## IPC 客户端 (Flask 端)

### 发送命令

```python
class SimulationIPCClient:
    """IPC 客户端 - Flask 后端使用"""

    def __init__(self, simulation_dir: str):
        self.simulation_dir = simulation_dir
        self.commands_dir = os.path.join(simulation_dir, "ipc_commands")
        self.responses_dir = os.path.join(simulation_dir, "ipc_responses")

        os.makedirs(self.commands_dir, exist_ok=True)
        os.makedirs(self.responses_dir, exist_ok=True)

    def send_command(
        self,
        command_type: CommandType,
        args: Dict[str, Any],
        timeout: float = 60.0
    ) -> IPCResponse:
        """发送命令并等待响应"""

        # 1. 生成命令 ID
        command_id = str(uuid.uuid4())

        # 2. 创建命令对象
        command = IPCCommand(
            command_id=command_id,
            command_type=command_type,
            args=args
        )

        # 3. 写入命令文件
        command_file = os.path.join(self.commands_dir, f"{command_id}.json")
        with open(command_file, 'w', encoding='utf-8') as f:
            json.dump(command.to_dict(), f, ensure_ascii=False)

        # 4. 等待响应
        return self._wait_for_response(command_id, timeout)

    def _wait_for_response(self, command_id: str, timeout: float) -> IPCResponse:
        """等待响应文件"""

        response_file = os.path.join(self.responses_dir, f"{command_id}.json")
        start_time = time.time()

        while time.time() - start_time < timeout:
            if os.path.exists(response_file):
                try:
                    with open(response_file, 'r', encoding='utf-8') as f:
                        response_data = json.load(f)

                    response = IPCResponse.from_dict(response_data)

                    # 清理文件
                    self._cleanup_command(command_id)

                    return response
                except (json.JSONDecodeError, KeyError) as e:
                    logger.warning(f"解析响应失败: {e}")

            time.sleep(0.5)  # 轮询间隔

        raise TimeoutError(f"等待命令响应超时: {command_id}")
```

### 便捷方法

```python
class SimulationIPCClient:
    def send_interview(
        self,
        agent_id: int,
        prompt: str,
        platform: str = None,
        timeout: float = 60.0
    ) -> IPCResponse:
        """发送单个智能体访谈命令"""
        args = {"agent_id": agent_id, "prompt": prompt}

        if platform:
            args["platform"] = platform

        return self.send_command(
            command_type=CommandType.INTERVIEW,
            args=args,
            timeout=timeout
        )

    def send_batch_interview(
        self,
        interviews: List[Dict[str, Any]],
        platform: str = None,
        timeout: float = 120.0
    ) -> IPCResponse:
        """发送批量访谈命令"""
        args = {"interviews": interviews}

        if platform:
            args["platform"] = platform

        return self.send_command(
            command_type=CommandType.BATCH_INTERVIEW,
            args=args,
            timeout=timeout
        )

    def send_close_env(self, timeout: float = 30.0) -> IPCResponse:
        """发送关闭环境命令"""
        return self.send_command(
            command_type=CommandType.CLOSE_ENV,
            args={},
            timeout=timeout
        )
```

---

## IPC 服务器 (模拟脚本端)

### 轮询命令

```python
class SimulationIPCServer:
    """IPC 服务器 - OASIS 模拟脚本使用"""

    def __init__(self, simulation_dir: str):
        self.simulation_dir = simulation_dir
        self.commands_dir = os.path.join(simulation_dir, "ipc_commands")
        self.responses_dir = os.path.join(simulation_dir, "ipc_responses")

        os.makedirs(self.commands_dir, exist_ok=True)
        os.makedirs(self.responses_dir, exist_ok=True)

        self._running = False

    def start(self):
        """标记服务器为运行状态"""
        self._running = True
        self._update_env_status("alive")

    def stop(self):
        """标记服务器为停止状态"""
        self._running = False
        self._update_env_status("stopped")

    def poll_commands(self) -> Optional[IPCCommand]:
        """轮询命令目录，返回下一个待处理命令"""
        if not self._running:
            return None

        if not os.path.exists(self.commands_dir):
            return None

        # 按时间排序获取命令文件
        command_files = []
        for filename in os.listdir(self.commands_dir):
            if filename.endswith('.json'):
                filepath = os.path.join(self.commands_dir, filename)
                command_files.append((filepath, os.path.getmtime(filepath)))

        command_files.sort(key=lambda x: x[1])

        # 返回最早的命令
        for filepath, _ in command_files:
            try:
                with open(filepath, 'r', encoding='utf-8') as f:
                    data = json.load(f)
                return IPCCommand.from_dict(data)
            except (json.JSONDecodeError, KeyError) as e:
                logger.warning(f"读取命令文件失败: {filepath}, {e}")
                continue

        return None

    def send_response(self, response: IPCResponse):
        """发送响应"""
        response_file = os.path.join(self.responses_dir, f"{response.command_id}.json")

        with open(response_file, 'w', encoding='utf-8') as f:
            json.dump(response.to_dict(), f, ensure_ascii=False, indent=2)

        # 删除命令文件
        command_file = os.path.join(self.commands_dir, f"{response.command_id}.json")
        try:
            os.remove(command_file)
        except OSError:
            pass
```

### 处理命令

```python
def handle_command(command: IPCCommand, environment) -> IPCResponse:
    """处理单个命令"""

    command_id = command.command_id

    try:
        if command.command_type == CommandType.INTERVIEW:
            result = handle_interview(command.args, environment)
            return IPCResponse(
                command_id=command_id,
                status=CommandStatus.COMPLETED,
                result=result
            )

        elif command.command_type == CommandType.BATCH_INTERVIEW:
            result = handle_batch_interview(command.args, environment)
            return IPCResponse(
                command_id=command_id,
                status=CommandStatus.COMPLETED,
                result=result
            )

        elif command.command_type == CommandType.CLOSE_ENV:
            # 停止模拟环境
            environment.stop()
            return IPCResponse(
                command_id=command_id,
                status=CommandStatus.COMPLETED,
                result={"message": "Environment closed"}
            )

        else:
            return IPCResponse(
                command_id=command_id,
                status=CommandStatus.FAILED,
                error=f"Unknown command type: {command.command_type}"
            )

    except Exception as e:
        logger.error(f"处理命令失败: {command_id}, {e}")
        return IPCResponse(
            command_id=command_id,
            status=CommandStatus.FAILED,
            error=str(e)
        )
```

### 模拟主循环集成

```python
def simulation_main_loop(config: Dict):
    """模拟主循环"""

    # 初始化 IPC 服务器
    ipc_server = SimulationIPCServer(config["simulation_dir"])
    ipc_server.start()

    # 初始化 OASIS 环境
    environment = OasisEnvironment(config)

    try:
        for round_num in range(1, config["max_rounds"] + 1):
            # 1. 检查 IPC 命令
            command = ipc_server.poll_commands()
            if command:
                response = handle_command(command, environment)
                ipc_server.send_response(response)

            # 2. 执行模拟轮次
            environment.run_round(round_num)

            # 3. 记录动作
            log_actions(environment.get_actions(), round_num)

    finally:
        ipc_server.stop()
```

---

## 高级功能

### 环境状态检查

```python
class SimulationIPCClient:
    def check_env_alive(self) -> bool:
        """检查模拟环境是否存活"""
        status_file = os.path.join(self.simulation_dir, "env_status.json")

        if not os.path.exists(status_file):
            return False

        try:
            with open(status_file, 'r') as f:
                status = json.load(f)
            return status.get("status") == "alive"
        except (json.JSONDecodeError, OSError):
            return False
```

### 超时处理

```python
class SimulationIPCClient:
    def send_command_with_retry(
        self,
        command_type: CommandType,
        args: Dict,
        max_retries: int = 3,
        timeout: float = 60.0
    ) -> IPCResponse:
        """带重试的命令发送"""

        for attempt in range(max_retries):
            try:
                return self.send_command(command_type, args, timeout)
            except TimeoutError:
                if attempt < max_retries - 1:
                    logger.warning(f"命令超时，重试 {attempt + 1}/{max_retries}")
                    continue
                else:
                    raise
```

---

## 最佳实践

### 1. 文件清理

```python
def cleanup_ipc_files(simulation_dir: str):
    """清理 IPC 文件"""
    commands_dir = os.path.join(simulation_dir, "ipc_commands")
    responses_dir = os.path.join(simulation_dir, "ipc_responses")

    for dirname in [commands_dir, responses_dir]:
        if os.path.exists(dirname):
            for filename in os.listdir(dirname):
                filepath = os.path.join(dirname, filename)
                try:
                    os.remove(filepath)
                except OSError:
                    pass
```

### 2. 错误恢复

```python
def handle_ipc_error(error: Exception) -> str:
    """处理 IPC 错误"""
    if isinstance(error, TimeoutError):
        return "模拟进程响应超时，可能仍在运行"
    elif isinstance(error, json.JSONDecodeError):
        return "响应数据格式错误"
    else:
        return f"IPC 通信错误: {str(error)}"
```

### 3. 并发控制

```python
def send_commands_sequentially(
    client: SimulationIPCClient,
    commands: List[Tuple[CommandType, Dict]]
) -> List[IPCResponse]:
    """顺序发送多个命令"""
    responses = []

    for cmd_type, args in commands:
        response = client.send_command(cmd_type, args)
        responses.append(response)

    return responses
```

---

## 下一步

- **[扩展开发指南](./extension-development.md)** - 如何扩展系统

---

## 自检清单

### 基础理解

- [ ] 理解为什么需要 IPC 机制
- [ ] 掌握 MiroFish IPC 架构设计
- [ ] 理解命令和响应的传递机制
- [ ] 能排查基本的通信问题

### 进阶掌握

- [ ] 能实现 IPC 客户端和服务器
- [ ] 能扩展新的命令类型
- [ ] 掌握超时和错误处理
- [ ] 能优化 IPC 通信性能

### 专家能力

- [ ] 能设计更高效的通信机制
- [ ] 能实现双向通信优化
- [ ] 能处理大规模并发场景
- [ ] 能设计替代方案（如消息队列）
