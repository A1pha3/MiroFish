# 部署与故障排查

> MiroFish 的生产环境部署和常见问题解决

---

## 学习目标

完成本章节后，你将能够：

### 基础目标（必掌握）

- [ ] 使用 Docker Compose 快速部署 MiroFish
- [ ] 配置基本的 Nginx 反向代理
- [ ] 解决常见的部署问题
- [ ] 理解基本的生产环境配置

### 进阶目标（建议掌握）

- [ ] 配置 HTTPS 和 SSL 证书
- [ ] 设置日志轮转和监控
- [ ] 进行性能调优
- [ ] 配置自动备份策略

### 专家目标（挑战）

- [ ] 设计高可用部署架构
- [ ] 实现负载均衡
- [ ] 设计灾难恢复方案
- [ ] 优化生产环境性能

---

## Docker 部署

### 使用 Docker Compose

```bash
# 1. 配置环境变量
cp .env.example .env
vim .env  # 填入必需的 API 密钥

# 2. 启动容器
docker compose up -d

# 3. 查看日志
docker compose logs -f

# 4. 停止容器
docker compose down
```

### 配置说明

```yaml
# docker-compose.yml
services:
  mirofish:
    image: ghcr.io/666ghj/mirofish:latest
    container_name: mirofish
    env_file:
      - .env
    ports:
      - "3000:3000"   # 前端
      - "5001:5001"   # 后端
    volumes:
      - ./backend/uploads:/app/backend/uploads  # 持久化数据
    restart: unless-stopped
```

### 使用镜像加速

```yaml
services:
  mirofish:
    # 使用国内镜像加速
    image: ghcr.nju.edu.cn/666ghj/mirofish:latest
    # ...
```

---

## 生产环境配置

### Nginx 反向代理

```nginx
# /etc/nginx/sites-available/mirofish

server {
    listen 80;
    server_name your-domain.com;

    # 前端
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # 后端 API
    location /api/ {
        proxy_pass http://localhost:5001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;

        # 超时配置
        proxy_read_timeout 300s;
        proxy_connect_timeout 300s;
    }

    # 文件上传大小限制
    client_max_body_size 50M;
}
```

### Systemd 服务

```ini
# /etc/systemd/system/mirofish.service

[Unit]
Description=MiroFish Backend
After=network.target

[Service]
Type=simple
User=mirofish
WorkingDirectory=/opt/mirofish/backend
Environment="PATH=/opt/mirofish/backend/.venv/bin"
ExecStart=/opt/mirofish/backend/.venv/bin/python run.py
Restart=always

[Install]
WantedBy=multi-user.target
```

启动服务：

```bash
sudo systemctl enable mirofish
sudo systemctl start mirofish
sudo systemctl status mirofish
```

---

## 常见问题排查

### 问题 1: 后端启动失败

**症状**：访问 /health 返回 502

**排查步骤**：

```bash
# 1. 检查进程
ps aux | grep python

# 2. 检查端口
lsof -i :5001

# 3. 检查日志
tail -f backend/logs/app.log

# 4. 手动启动
cd backend
uv run python run.py
```

**常见原因**：

| 原因 | 解决方法 |
|------|----------|
| 端口被占用 | 修改 FLASK_PORT 或关闭占用进程 |
| 虚拟环境未激活 | 使用 `uv run python` |
| 依赖缺失 | 运行 `uv sync` |
| 配置错误 | 检查 .env 文件 |

### 问题 2: 图谱构建失败

**症状**：图谱构建一直处于 processing 状态

**排查步骤**：

```bash
# 1. 检查 Zep API Key
curl -H "Authorization: Api-Key $ZEP_API_KEY" \
  https://api.getzep.com/v1/graph

# 2. 检查任务状态
# 查看 backend/logs 中的任务日志

# 3. 检查网络连接
ping api.getzep.com
```

**常见原因**：

| 原因 | 解决方法 |
|------|----------|
| API Key 无效 | 更新 ZEP_API_KEY |
| 网络问题 | 检查网络连接 |
| 文本格式问题 | 确保文件编码正确 |
| 配额不足 | 检查 Zep 账户配额 |

### 问题 3: 模拟运行缓慢

**症状**：模拟每轮耗时过长

**优化方法**：

```bash
# 1. 减少智能体数量
# 只选择主要实体

# 2. 减少模拟轮数
# 设置为 10-20 轮测试

# 3. 使用更快的 LLM
# 切换到响应更快的模型

# 4. 增加并行度
# 在 Step 2 中增加 parallel_profile_count
```

### 问题 4: 报告生成质量差

**症状**：报告内容空洞或不准确

**解决方法**：

1. **优化预测需求**：
```
# 好的需求
"请分析贾宝玉和林黛玉的关系走向，
重点关注贾府衰败事件对两人关系的影响"

# 不好的需求
"生成一份报告"
```

2. **增加模拟轮数**：更多互动数据

3. **调整温度参数**：降低温度（0.3-0.5）

### 问题 5: IPC 通信超时

**症状**：访谈智能体时超时

**排查步骤**：

```bash
# 1. 检查模拟进程状态
ps aux | grep simulation

# 2. 检查 IPC 文件
ls -la backend/uploads/simulations/{sim_id}/ipc_*

# 3. 检查环境状态文件
cat backend/uploads/simulations/{sim_id}/env_status.json
```

**解决方法**：

| 原因 | 解决方法 |
|------|----------|
| 模拟进程已停止 | 重新启动模拟 |
| IPC 目录不存在 | 检查权限配置 |
| 命令文件残留 | 清理 ipc_commands 目录 |

---

## 性能优化

### 后端优化

```python
# 1. 启用缓存
from functools import lru_cache

@lru_cache(maxsize=100)
def get_entity(graph_id: str, uuid: str):
    ...

# 2. 批量操作
def batch_process(items, batch_size=10):
    for i in range(0, len(items), batch_size):
        batch = items[i:i+batch_size]
        process_batch(batch)

# 3. 异步处理
import asyncio

async def async_build_graph(text):
    tasks = [add_chunk(chunk) for chunk in chunks]
    await asyncio.gather(*tasks)
```

### 前端优化

```javascript
// 1. 虚拟滚动
import { useVirtualList } from '@vueuse/core'

const { list, containerProps, wrapperProps } = useVirtualList(
  largeList,
  { itemHeight: 50 }
)

// 2. 防抖搜索
import { debounce } from 'lodash-es'

const debouncedSearch = debounce(performSearch, 300)

// 3. 懒加载路由
const routes = [
  {
    path: '/simulation/:id',
    component: () => import('./views/SimulationRunView.vue')
  }
]
```

---

## 监控与日志

### 日志位置

```
backend/logs/
├── app.log           # 应用日志
├── error.log         # 错误日志
├── simulation.log    # 模拟日志
└── agent.log         # Agent 日志
```

### 日志查询

```bash
# 查看最近的错误
tail -f backend/logs/error.log

# 搜索特定错误
grep "ERROR" backend/logs/app.log

# 统计错误类型
grep "ERROR" backend/logs/app.log | awk '{print $5}' | sort | uniq -c
```

### 健康检查

```bash
# 后端健康检查
curl http://localhost:5001/health

# 预期响应
{"status": "ok", "service": "MiroFish Backend"}
```

---

## 备份与恢复

### 数据备份

```bash
# 1. 备份上传数据
tar -czf backups/uploads-$(date +%Y%m%d).tar.gz backend/uploads/

# 2. 备份数据库（如果使用）
pg_dump mirofish > backups/mirofish-$(date +%Y%m%d).sql

# 3. 自动备份脚本
#!/bin/bash
# backup.sh
DATE=$(date +%Y%m%d)
tar -czf /backup/mirofish-$DATE.tar.gz /opt/mirofish/backend/uploads
find /backup -name "mirofish-*.tar.gz" -mtime +7 -delete
```

### 数据恢复

```bash
# 1. 停止服务
docker compose down

# 2. 恢复数据
tar -xzf backups/uploads-20240101.tar.gz -C /

# 3. 重启服务
docker compose up -d
```

---

## 安全配置

### 环境变量安全

```bash
# 不要将 .env 提交到版本控制
echo ".env" >> .gitignore

# 设置文件权限
chmod 600 .env
```

### CORS 配置

```python
# backend/app/__init__.py

from flask_cors import CORS

# 生产环境配置
CORS(app, resources={
    r"/api/*": {
        "origins": ["https://your-domain.com"],
        "methods": ["GET", "POST", "PUT", "DELETE"],
        "allow_headers": ["Content-Type", "Authorization"]
    }
})
```

---

## 下一步

- **[开发者指南](../developer-guide/)** - 深入开发
- **[高级指南](../advanced-guide/)** - 扩展开发

---

## 自检清单

### 基础技能

- [ ] 能使用 Docker Compose 部署 MiroFish
- [ ] 能配置基本的 Nginx 反向代理
- [ ] 能解决常见的部署问题
- [ ] 能进行基本的数据备份

### 进阶技能

- [ ] 能配置 HTTPS 和 SSL 证书
- [ ] 能设置日志轮转和监控
- [ ] 能进行性能优化
- [ ] 能配置自动备份策略

### 专家能力

- [ ] 能设计高可用部署架构
- [ ] 能实现负载均衡
- [ ] 能设计灾难恢复方案
- [ ] 能进行系统级性能调优
