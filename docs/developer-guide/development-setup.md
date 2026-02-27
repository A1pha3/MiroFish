# 开发环境搭建

> 配置 MiroFish 的本地开发环境

**难度**: ⭐ | **预计时间**: 15 分钟

---

## 学习目标

完成本章节后，你将能够：

### 基础目标（必掌握）

- [ ] 配置本地开发所需的所有工具和依赖
- [ ] 成功启动前后端开发服务
- [ ] 理解开发环境的基本目录结构
- [ ] 验证开发环境配置正确性

### 进阶目标（建议掌握）

- [ ] 理解各工具在开发中的作用
- [ ] 配置 VS Code 等开发工具
- [ ] 掌握开发工作流的基本操作
- [ ] 能够独立排查环境配置问题

### 专家目标（挑战）

- [ ] 能够为新的开发者准备环境配置指南
- [ ] 理解开发环境与生产环境的差异
- [ ] 能够优化开发环境配置

---

## 前置要求

| 工具 | 版本 | 检查命令 | 下载地址 |
|------|------|----------|----------|
| **Node.js** | 18+ | `node -v` | [nodejs.org](https://nodejs.org/) |
| **Python** | 3.11-3.12 | `python --version` | [python.org](https://www.python.org/) |
| **uv** | 最新 | `uv --version` | [astral.sh/uv](https://docs.astral.sh/uv/) |
| **Git** | 最新 | `git --version` | [git-scm.com](https://git-scm.com/) |

### 推荐工具

| 工具 | 用途 |
|------|------|
| VS Code | 代码编辑器 |
| Vue DevTools | Vue 调试 |
| Postman | API 测试 |

---

## 步骤 1：克隆仓库

```bash
# 克隆仓库
git clone https://github.com/666ghj/MiroFish.git
cd MiroFish

# 创建开发分支（可选）
git checkout -b dev-your-name
```

---

## 步骤 2：配置环境变量

```bash
# 复制示例配置
cp .env.example .env

# 编辑配置文件
vim .env  # 或使用其他编辑器
```

### 必需配置

```env
# LLM API 配置
LLM_API_KEY=your_api_key_here
LLM_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
LLM_MODEL_NAME=qwen-plus

# Zep Cloud 配置
ZEP_API_KEY=your_zep_api_key_here
```

### 开发环境配置（可选）

```env
# Flask 配置
FLASK_DEBUG=True
FLASK_HOST=127.0.0.1
FLASK_PORT=5001
```

---

## 步骤 3：安装后端依赖

### 使用 uv（推荐）

```bash
cd backend

# 同步依赖（自动创建虚拟环境）
uv sync

# 激活虚拟环境
source .venv/bin/activate  # Linux/macOS
# 或
.venv\Scripts\activate     # Windows
```

### 验证安装

```bash
# 检查 Python 版本
python --version

# 检查已安装的包
uv pip list
```

---

## 步骤 4：安装前端依赖

```bash
cd frontend

# 安装依赖
npm install

# 验证安装
npm list --depth=0
```

---

## 步骤 5：启动开发服务

### 方式 1：同时启动前后端（推荐）

```bash
# 在项目根目录
npm run dev
```

### 方式 2：分别启动

**启动后端**：

```bash
cd backend
uv run python run.py
```

后端将运行在 http://127.0.0.1:5001

**启动前端**：

```bash
cd frontend
npm run dev
```

前端将运行在 http://localhost:3000

---

## 步骤 6：验证开发环境

### 检查后端

```bash
# 访问健康检查
curl http://127.0.0.1:5001/health

# 预期响应
{"status": "ok", "service": "MiroFish Backend"}
```

### 检查前端

打开浏览器访问 http://localhost:3000，应该看到主页面。

---

## 开发工具配置

### VS Code 推荐扩展

```json
{
  "recommendations": [
    "vue.volar",
    "vue.vscode-typescript-vue-plugin",
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "ms-python.python",
    "ms-python.vscode-pylance"
  ]
}
```

### VS Code 工作区配置

创建 `.vscode/settings.json`：

```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/backend/.venv/bin/python",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  }
}
```

---

## 测试环境

### 后端测试

```bash
cd backend

# 运行所有测试
uv run pytest

# 运行特定测试文件
uv run pytest tests/test_graph.py

# 查看覆盖率
uv run pytest --cov=app
```

### 前端测试

```bash
cd frontend

# 运行测试（如果配置了）
npm run test
```

---

## 常见问题

### Q: uv 安装失败？

**A**: 尝试其他安装方式：

```bash
# 使用 pip
pip install uv

# 使用脚本
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### Q: 前端启动失败，端口被占用？

**A**:

```bash
# 查找占用进程
lsof -i :3000

# 或使用其他端口
npm run dev -- --port 3001
```

### Q: 后端导入错误？

**A**:

```bash
# 确保虚拟环境已激活
source .venv/bin/activate

# 重新安装依赖
uv sync --reinstall
```

---

## 开发工作流

### 日常开发

```bash
# 1. 更新代码
git pull origin main

# 2. 拉取最新依赖（如有变化）
cd backend && uv sync
cd frontend && npm install

# 3. 启动开发服务
npm run dev

# 4. 创建功能分支
git checkout -b feature-your-feature

# 5. 开发和测试...

# 6. 提交代码
git add .
git commit -m "feat: your feature"
git push origin feature-your-feature
```

### 调试技巧

**后端调试**：

```python
# 在代码中添加断点
import pdb; pdb.set_trace()

# 或使用 logging
from app.utils.logger import get_logger
logger = get_logger('your_module')
logger.debug("Debug info")
```

**前端调试**：

```javascript
// 使用 console.log
console.log('Debug info:', data);

// 或使用 Vue DevTools
// 在组件中添加
// <script>
// export default {
//   name: 'DebugComponent'
// }
// </script>
```

---

## 下一步

开发环境配置完成后，你可以：

- **[后端架构详解](./backend-architecture.md)** - 深入后端实现
- **[前端架构详解](./frontend-architecture.md)** - 深入前端实现
- **[API 接口文档](./api-reference.md)** - 了解 API 接口

---

## 自检清单

### 基础技能

- [ ] 能独立完成开发环境搭建
- [ ] 能成功启动前后端服务
- [ ] 能验证环境配置正确
- [ ] 能解决常见的配置问题

### 进阶技能

- [ ] 能配置 VS Code 等开发工具
- [ ] 理解各工具的作用和配置
- [ ] 能进行基本的开发调试
- [ ] 能帮助他人配置开发环境

### 专家能力

- [ ] 能编写环境配置指南
- [ ] 能诊断复杂的环境问题
- [ ] 能优化开发工作流程
- [ ] 能搭建自动化开发环境
