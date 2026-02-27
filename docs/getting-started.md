# 快速开始

> 15 分钟完成 MiroFish 安装并运行第一个预测任务

**阅读时间**: 约 15 分钟
**适合人群**: 所有新用户
**难度等级**: ⭐

---

## 学习目标

完成本指南后，你将能够：

- [ ] ✅ 完成环境准备（工具安装）
- [ ] ✅ 配置系统（API 密钥设置）
- [ ] ✅ 启动服务（前后端运行）
- [ ] ✅ 运行第一个预测任务
- [ ] ✅ 理解基本故障排查方法

---

## 前置检查清单

在开始之前，请确认你的系统满足以下要求：

### 必需工具

| 工具 | 版本要求 | 检查命令 | 未安装时的操作 |
|------|---------|---------|-----------------|
| **Node.js** | 18.0+ | `node -v` | [nodejs.org](https://nodejs.org/) |
| **Python** | 3.11 - 3.12 | `python --version` | [python.org](https://www.python.org/) |
| **uv** | 最新版 | `uv --version` | 见下方安装指南 |
| **Git** | 任意版本 | `git --version` | [git-scm.com](https://git-scm.com/) |

### 系统要求

- **磁盘空间**: 至少 2GB 可用空间
- **内存**: 建议 4GB 以上
- **网络**: 能访问 GitHub 和 API 服务

---

## 第一部分：工具安装

### 安装 Node.js

```bash
# 检查是否已安装
node -v

# 如果未安装或版本过低，请访问
# https://nodejs.org/ 下载 LTS 版本
```

**推荐版本**: 18.x 或 20.x LTS

### 安装 Python

```bash
# 检查 Python 版本
python --version

# 确保版本在 3.11 到 3.12 之间
# Python 3.13+ 可能存在兼容性问题
```

**重要**: MiroFish 使用 OASIS 模拟引擎，目前仅兼容 Python 3.11-3.12。

### 安装 uv（Python 包管理器）

`uv` 是一个快速的 Python 包管理器，MiroFish 推荐使用它来管理 Python 依赖。

```bash
# macOS / Linux（推荐方式）
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# 或使用 pip 安装
pip install uv

# 验证安装
uv --version
```

**安装验证**：

```bash
$ uv --version
uv 0.1.10 (e9bf8c3 2024-01-15)
```

---

## 第二部分：获取源码

### 克隆仓库

```bash
# 克隆 MiroFish 仓库
git clone https://github.com/666ghj/MiroFish.git

# 进入项目目录
cd MiroFish

# 查看项目结构
ls -la
```

**项目目录结构**：

```
MiroFish/
├── backend/           # Python 后端
├── frontend/          # Vue 3 前端
├── docs/             # 项目文档
├── .env.example       # 环境变量模板
├── docker-compose.yml # Docker 部署配置
└── package.json      # 根目录 npm 脚本
```

---

## 第三部分：配置环境变量

### 创建配置文件

```bash
# 复制环境变量模板
cp .env.example .env
```

### 获取 API 密钥

MiroFish 需要两个核心 API 密钥才能运行：

#### 1. LLM API 密钥

MiroFish 支持任何兼容 OpenAI SDK 格式的 LLM API。

**推荐选项：阿里百炼平台**

```bash
# 1. 访问阿里云百炼平台
# https://bailian.console.aliyun.com/

# 2. 开通服务并创建 API Key
# 控制台 -> API-KEY管理 -> 创建新的API-KEY

# 3. 复制 API Key
```

**其他选项**：

| 提供商 | Base URL | 推荐模型 | 性价比 |
|--------|----------|----------|--------|
| 阿里百炼 | `https://dashscope.aliyuncs.com/compatible-mode/v1` | qwen-plus | ⭐⭐⭐⭐⭐ |
| DeepSeek | `https://api.deepseek.com` | deepseek-chat | ⭐⭐⭐⭐⭐ |
| 智谱 AI | `https://open.bigmodel.cn/api/paas/v4/` | glm-4-flash | ⭐⭐⭐⭐ |

#### 2. Zep Cloud API 密钥

Zep Cloud 提供企业级 GraphRAG 服务。

```bash
# 1. 访问 Zep Cloud
# https://app.getzep.com/

# 2. 注册账号（有免费额度）
# 免费额度通常足够个人使用和小规模测试

# 3. 创建 API Key
# Settings -> API Keys -> Create API Key
```

### 配置 .env 文件

打开 `.env` 文件，填入获取的 API 密钥：

```env
# ===== LLM API 配置 =====
# 支持任何 OpenAI SDK 格式的 LLM API
LLM_API_KEY=your_llm_api_key_here
LLM_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
LLM_MODEL_NAME=qwen-plus

# ===== Zep Cloud 配置 =====
# Zep Cloud 提供企业级 GraphRAG 服务
ZEP_API_KEY=your_zep_api_key_here

# ===== 可选：加速 LLM 配置 =====
# 如果主 LLM 速度较慢，可以配置加速 LLM
# LLM_BOOST_API_KEY=your_boost_api_key
# LLM_BOOST_BASE_URL=your_boost_base_url
# LLM_BOOST_MODEL_NAME=your_boost_model
```

### 配置验证

```bash
# 测试 LLM API 配置
curl -X POST https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions \
  -H "Authorization: Bearer $LLM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen-plus","messages":[{"role":"user","content":"hi"}]}'
```

---

## 第四部分：安装依赖

### 一键安装（推荐）

```bash
# 安装所有依赖（根目录 + 前端 + 后端）
npm run setup:all
```

**执行过程**：

```
┌─────────────────────────────────────────────────────────────┐
│  安装进度                                                      │
├─────────────────────────────────────────────────────────────┤
│                                                                  │
│  □ 安装根目录依赖...                                          │
│  □ 安装前端依赖...                                            │
│    └─> [################------------] 100%                      │
│  □ 安装后端依赖（uv sync）...                                 │
│    └─> [############################] 100%                   │
│  □ 验证安装...                                                  │
│  □ 安装完成！                                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────┘
```

### 分步安装（如遇问题）

```bash
# 仅安装 Node.js 依赖
npm run setup

# 仅安装 Python 依赖
npm run setup:backend
```

### 常见安装问题

**问题 1: uv 安装失败**

```bash
# 尝试使用 pip 安装
pip install uv

# 或使用虚拟环境
python -m venv venv
source venv/bin/activate  # Linux/macOS
# venv\Scripts\activate   # Windows
pip install uv
```

**问题 2: Node.js 依赖安装失败**

```bash
# 清除缓存重试
rm -rf node_modules frontend/node_modules
npm cache clean --force
npm run setup
```

---

## 第五部分：启动服务

### 启动前后端（推荐）

```bash
# 在项目根目录执行
npm run dev
```

**预期输出**：

```
┌─────────────────────────────────────────────────────────────┐
│  [frontend]                                                      │
│    │                                                                 │
│    │  VITE v7.2.4  ready in 250 ms                                  │
│    │                                                                 │
│    │  ➜  Local:   http://localhost:3000/                           │
│    │  ➜  Network: use --host to expose                           │
│    │                                                                 │
│  [backend]                                                      │
│    │                                                                 │
│    │  * Serving Flask app 'mirofish'                               │
│    │  * Debug mode: on                                          │
│    │  * Running on http://127.0.0.1:5001                          │
│    │                                                                 │
└─────────────────────────────────────────────────────────────┘
```

### 服务访问地址

| 服务 | 地址 | 说明 |
|------|------|------|
| 前端 | http://localhost:3000 | 主界面 |
| 后端 API | http://localhost:5001 | API 服务 |
| 健康检查 | http://localhost:5001/health | 状态检查 |

### 单独启动（调试模式）

```bash
# 仅启动后端
cd backend && uv run python run.py
# 或
npm run backend

# 仅启动前端
cd frontend && npm run dev
# 或
npm run frontend
```

### 验证服务状态

```bash
# 1. 检查前端（浏览器访问）
open http://localhost:3000

# 2. 检查后端健康状态
curl http://localhost:5001/health

# 预期响应
# {"status": "ok", "service": "MiroFish Backend"}
```

---

## 第六部分：运行第一个预测任务

### 完整流程演示

让我们用一个实际例子来体验 MiroFish 的完整流程。

#### 准备测试材料

创建一个简单的测试文件 `test_story.txt`：

```text
贾宝玉是《红楼梦》中的男主角，贾府的公子。他性格温和，
不喜欢读书考取功名，偏爱诗词歌赋，对女性有深厚的同情心。

林黛玉是贾府的表小姐，才貌双全，但身体虚弱，多愁善感。
她与贾宝玉情投意合，两人有深厚的感情基础。

薛宝钗是另一个重要人物，她性格稳重，善解人意，深得贾府上下喜爱。

贾府是一个显赫的家族，但逐渐走向衰败。
```

#### Step 1: 图谱构建

1. 打开浏览器访问 http://localhost:3000
2. 点击 **"新建项目"**
3. 输入项目名称：`红楼梦预测`
4. 点击 **"上传文件"**，选择刚创建的 `test_story.txt`
5. 等待本体自动生成（约 10-30 秒）
6. 查看生成的实体类型，确认无误后点击 **"开始构建图谱"**

**预期时间**: 30 秒 - 2 分钟（取决于文本长度）

#### Step 2: 环境搭建

1. 等待图谱构建完成后，查看提取的实体
2. 查看每个实体的详细信息
3. 选择要包含的实体类型（通常选择 `Person`）
4. 设置模拟参数：
   - 模拟轮数：首次使用建议选择 `10` 轮
   - 启用平台：选择 `Twitter` 单平台即可
5. 点击 **"准备模拟环境"**

**预期时间**: 1 - 5 分钟（取决于智能体数量）

#### Step 3: 模拟运行

1. 输入预测需求：
   ```
   请分析贾宝玉和林黛玉的关系发展趋势，
   重点考虑贾府衰败的影响。
   ```
2. 点击 **"开始模拟"**
3. 观察实时进度：
   - 当前轮次
   - 智能体活跃度
   - 动作记录
4. 等待模拟完成

**预期时间**: 5 - 15 分钟（取决于智能体数量和轮数）

#### Step 4: 报告生成

1. 模拟完成后，点击 **"生成报告"**
2. 等待 Report Agent 生成分析报告
3. 查看生成的报告内容
4. 阅读各个章节的分析

**预期时间**: 2 - 5 分钟

#### Step 5: 交互分析

1. 在报告页面，选择一个智能体进行访谈
2. 输入问题，例如：
   ```
   宝玉，你对最近的家族变化有什么感受？
   ```
3. 查看智能体的回复
4. 尝试追问深入了解

---

## 故障排查指南

### 问题诊断决策树

```
┌─────────────────────────────────────────────────────────────┐
│                    遇到问题了？请按此流程诊断                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ 服务能启动吗？  │
                    └─────────┬─────────┘
                              │
              ┌───────────┴───────────────┐
              │                          │
             否                          是
              │                          │
              ▼                          ▼
     ┌─────────────┐             ┌─────────────┐
     │安装依赖问题 │             │API调用失败?  │
     └─────┬───────┘             └─────┬───────┘
           │                          │
           ▼                          ▼
    ┌─────────────────┐         ┌─────────────────┐
    │依赖安装不完整  │         │API Key配置错误  │
    │→重新安装依赖  │         │→检查.env配置   │
    └─────────────────┘         └─────────────────┘
                                       │
                                       ▼
                              ┌─────────────────┐
                              │图谱构建失败?  │
                              └─────────┬─────────┘
                                        │
                              ┌───────────┴───────────┐
                              │                       │
                             是                      否
                              │                       │
                              ▼                       ▼
                       ┌─────────────┐         ┌─────────────┐
                       │Zep配置问题  │         │网络问题     │
                       │→检查API Key│         │→检查网络   │
                       └─────────────┘         └─────────────┘
```

### 常见问题速查表

| 问题 | 可能原因 | 快速解决 |
|------|----------|----------|
| `npm run dev` 失败 | 端口被占用 | 关闭占用进程或换端口 |
| 后端启动失败 | Python 版本不对 | 使用 3.11-3.12 版本 |
| API 调用 401 | API Key 无效 | 检查 `.env` 配置 |
| 图谱构建超时 | Zep 服务慢 | 检查网络，减少文本大小 |
| 模拟运行慢 | 智能体太多 | 减少实体数量或轮数 |
| 报告生成失败 | 配额用完 | 检查 API 账户余额 |

### 详细排查步骤

#### 后端启动失败

```bash
# 1. 检查 Python 版本
python --version
# 输出应该是 Python 3.11.x 或 3.12.x

# 2. 检查虚拟环境
cd backend
uv run python --version

# 3. 手动启动查看错误
cd backend
uv run python run.py

# 4. 检查端口占用
lsof -i :5001  # macOS/Linux
netstat -ano | findstr :5001  # Windows
```

#### API �用失败

```bash
# 1. 验证 API Key
echo $LLM_API_KEY
echo $ZEP_API_KEY

# 2. 测试 LLM API
curl -X POST $LLM_BASE_URL/chat/completions \
  -H "Authorization: Bearer $LLM_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"'$LLM_MODEL_NAME'","messages":[{"role":"user","content":"hi"}]}'

# 3. 测试 Zep API
curl -H "Authorization: Api-Key $ZEP_API_KEY" \
  https://api.getzep.com/v1/graph
```

#### 模拟运行异常

```bash
# 1. 检查模拟进程
ps aux | grep python

# 2. 查看模拟日志
cat backend/uploads/simulations/{sim_id}/actions.jsonl

# 3. 检查 IPC 目录
ls -la backend/uploads/simulations/{sim_id}/ipc_*

# 4. 检查环境状态
cat backend/uploads/simulations/{sim_id}/env_status.json
```

---

## Docker 快速部署（可选）

如果你熟悉 Docker，可以使用以下方式快速部署：

```bash
# 1. 配置环境变量
cp .env.example .env
# 编辑 .env 填入 API 密钥

# 2. 启动容器
docker compose up -d

# 3. 查看日志
docker compose logs -f

# 4. 访问服务
open http://localhost:3000
```

**Docker 部署架构**：

```
┌─────────────────────────────────────────────────────────────┐
│  Docker Compose                                               │
│  ┌───────────────────────────────────────────────────────┐   │
│  │  mirofish (Container)                                    │   │
│  │                                                           │   │
│  │  ┌─────────────────┐    ┌───────────────────────┐  │   │
│  │  │  Nginx (前端)    │    │  Flask (后端)        │  │   │
│  │  │  :3000          │    │  :5001              │  │   │
│  │  └─────────────────┘    └───────────────────────┘  │   │
│  │                                                           │   │
│  │  ┌─────────────────────────────────────────────────────┐  │   │
│  │  │  uploads/ (持久化数据)                              │  │   │
│  │  └─────────────────────────────────────────────────────┘  │   │
│  └───────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 恭喜！🎉

你已经成功完成了 MiroFish 的安装和第一个预测任务！

### 成就检查

- [ ] ✅ 系统成功启动
- [ ] ✅ 环境变量配置正确
- [ ] ✅ 图谱构建完成
- [ ] ✅ 模拟运行成功
- [ ] ✅ 报告生成完毕
- [ ] ✅ 智能体访谈成功

---

## 下一步学习

根据你的目标选择合适的进阶路径：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        下一步学习路径                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  我想掌握完整流程                                                      │
│  └──▶ [用户指南 - Step 1](./user-guide/step1-graph-build.md)         │
│      [用户指南 - Step 2](./user-guide/step2-env-setup.md)           │
│      [用户指南 - Step 3](./user-guide/step3-simulation.md)          │
│                                                                      │
│  我想了解技术细节                                                      │
│  └──▶ [架构设计](./architecture.md)                                       │
│      [开发者指南](./developer-guide/)                                 │
│                                                                      │
│  我想进行二次开发                                                      │
│  └──▶ [开发环境搭建](./developer-guide/development-setup.md)           │
│      [API 接口文档](./developer-guide/api-reference.md)                  │
│                                                                      │
│  我想部署到生产环境                                                    │
│  └──▶ [部署与故障排查](./deployment-troubleshooting.md)                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 自检清单

### 基础技能

- [ ] 能独立完成 MiroFish 的安装和配置
- [ ] 能正确配置 LLM 和 Zep Cloud API
- [ ] 能成功启动前后端服务
- [ ] 能完成一个简单的预测任务
- [ ] 能解决基本的安装和配置问题

### 进阶技能

- [ ] 能使用 Docker 快速部署系统
- [ ] 能诊断和解决常见启动问题
- [ ] 能根据需求调整模拟参数
- [ ] 能理解各组件的职责和交互方式

### 学习验证

- [ ] 能向他人讲解 MiroFish 的核心价值
- [ ] 能帮助新用户完成安装和配置
- [ ] 能识别不同问题的正确排查路径

---

## 获取帮助

如果遇到问题：

1. **查看文档**：先查阅相关文档的故障排查部分
2. **搜索 Issue**：在 GitHub Issues 中搜索类似问题
3. **提 Issue**：如果没有找到解决方案，创建新 Issue
4. **加入社区**：通过 QQ 群或邮件联系我们

**资源链接**：

- 📖 [完整文档](./)
- 🐛 [问题反馈](https://github.com/666ghj/MiroFish/issues)
- 💬 [QQ交流群](../static/image/QQ群.png)
- 📧 联系我们：mirofish@shanda.com
