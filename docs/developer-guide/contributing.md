# 贡献指南

> 如何为 MiroFish 项目做出贡献

**难度**: ⭐ | **预计时间**: 10 分钟

---

## 学习目标

完成本章节后，你将能够：

### 基础目标（必掌握）

- [ ] 理解 MiroFish 项目的贡献流程
- [ ] 掌握 Git 工作流和分支管理
- [ ] 能够创建和提交 Pull Request
- [ ] 遵循项目的代码规范

### 进阶目标（建议掌握）

- [ ] 理解 Conventional Commits 规范
- [ ] 能够编写高质量的测试
- [ ] 能够参与代码审查
- [ ] 能够处理复杂的合并冲突

### 专家目标（挑战）

- [ ] 能够指导其他贡献者
- [ ] 能够设计和实现重大功能
- [ ] 能够参与架构决策
- [ ] 能够维护项目的长期健康发展

---

## 贡献方式

我们欢迎以下类型的贡献：

| 类型 | 说明 | 示例 |
|------|------|------|
| **Bug 修复** | 修复已知问题 | 修复模拟状态更新问题 |
| **新功能** | 添加新功能 | 支持新的模拟平台 |
| **文档改进** | 完善文档 | 修正 API 文档错误 |
| **性能优化** | 提升性能 | 优化 Profile 生成速度 |
| **测试覆盖** | 增加测试 | 添加核心服务单元测试 |
| **代码重构** | 改善代码质量 | 重构 API 模块 |

---

## 开发流程

### 1. Fork 和克隆

```bash
# Fork 项目到你的 GitHub 账号

# 克隆你的 fork
git clone https://github.com/your-username/MiroFish.git
cd MiroFish

# 添加上游仓库
git remote add upstream https://github.com/666ghj/MiroFish.git
```

### 2. 创建功能分支

```bash
# 同步上游最新代码
git fetch upstream
git checkout main
git merge upstream/main

# 创建功能分支
git checkout -b feature-your-feature
```

### 3. 进行开发

```bash
# 安装依赖
npm run setup:all

# 启动开发服务
npm run dev

# 进行开发和测试...
```

### 4. 提交代码

```bash
# 添加变更
git add .

# 提交（使用规范的提交信息）
git commit -m "feat: 添加批量访谈功能"

# 推送到你的 fork
git push origin feature-your-feature
```

### 5. 创建 Pull Request

1. 访问 GitHub 项目页面
2. 点击 "New Pull Request"
3. 选择你的功能分支
4. 填写 PR 模板
5. 等待代码审查

---

## 提交信息规范

我们使用 [Conventional Commits](https://www.conventionalcommits.org/) 规范：

### 格式

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Type（类型）

| Type | 说明 |
|------|------|
| `feat` | 新功能 |
| `fix` | Bug 修复 |
| `docs` | 文档更新 |
| `style` | 代码格式（不影响功能） |
| `refactor` | 重构 |
| `perf` | 性能优化 |
| `test` | 测试相关 |
| `chore` | 构建/工具相关 |

### 示例

```bash
# 新功能
git commit -m "feat(simulation): 添加暂停/继续功能"

# Bug 修复
git commit -m "fix(graph): 修复实体读取时的编码问题"

# 文档更新
git commit -m "docs(api): 更新 API 接口文档"

# 重构
git commit -m "refactor(profile): 优化 Profile 生成逻辑"
```

---

## 代码规范

### Python（后端）

- 遵循 PEP 8
- 使用类型注解
- 编写文档字符串

```python
def generate_profile(entity: Dict[str, Any]) -> OasisAgentProfile:
    """从实体生成智能体 Profile

    Args:
        entity: 实体信息字典

    Returns:
        生成的智能体 Profile

    Raises:
        ValueError: 实体信息不完整
    """
    pass
```

### JavaScript（前端）

- 使用 Vue 3 Composition API
- 遵循 Vue 风格指南
- 使用 ESLint 检查

```javascript
// 推荐：使用 <script setup>
const props = defineProps({
  data: Object
})

// 推荐：使用组合式 API
import { ref, computed } from 'vue'
```

---

## 测试要求

### 编写测试

新增功能需要编写测试：

```python
# tests/test_simulation_manager.py
def test_create_simulation():
    manager = SimulationManager()
    state = manager.create_simulation(
        project_id="proj_1",
        graph_id="graph_1"
    )

    assert state.status == SimulationStatus.CREATED
    assert state.simulation_id is not None
```

### 运行测试

```bash
# 运行所有测试
cd backend && uv run pytest

# 运行特定测试
uv run pytest tests/test_simulation.py

# 查看覆盖率
uv run pytest --cov=app
```

---

## Pull Request 检查清单

提交 PR 前，请确认：

- [ ] 代码符合项目规范
- [ ] 添加了必要的测试
- [ ] 所有测试通过
- [ ] 更新了相关文档
- [ ] 提交信息符合规范
- [ ] PR 描述清晰完整

### PR 模板

```markdown
## 变更类型
- [ ] Bug 修复
- [ ] 新功能
- [ ] 破坏性变更
- [ ] 文档更新

## 变更说明
<!-- 描述你的变更 -->

## 测试
<!-- 说明如何测试你的变更 -->

## 截图（如适用）
<!-- 添加截图展示变更效果 -->

## 检查清单
- [ ] 代码符合规范
- [ ] 测试通过
- [ ] 文档已更新
```

---

## 代码审查

### 审查流程

1. 自动检查：CI 运行测试和 Lint
2. 人工审查：维护者审查代码
3. 反馈修改：根据反馈修改代码
4. 合并：审查通过后合并

### 审查要点

- 代码质量和可读性
- 测试覆盖是否充分
- 文档是否完整
- 是否引入破坏性变更

---

## 获取帮助

如果你在贡献过程中遇到问题：

1. **查看文档**：先查阅相关文档
2. **搜索 Issue**：看是否有类似问题
3. **提 Issue**：创建新的 Issue 描述问题
4. **社区讨论**：加入社区讨论

---

## 许可证

贡献的代码将采用项目的 [AGPL-3.0](../LICENSE) 许可证。

---

## 致谢

感谢所有为 MiroFish 做出贡献的开发者！

---

## 下一步

- 开始贡献：查看 [Good First Issue](https://github.com/666ghj/MiroFish/labels/good%20first%20issue)
- 了解架构：[核心服务详解](./core-services.md)

---

## 自检清单

### 基础技能

- [ ] 能完成 Fork 和克隆操作
- [ ] 能创建功能分支
- [ ] 能编写规范的提交信息
- [ ] 能创建 Pull Request

### 进阶技能

- [ ] 能编写高质量的测试
- [ ] 能进行代码审查
- [ ] 能处理合并冲突
- [ ] 能参与技术讨论

### 专家能力

- [ ] 能指导新贡献者
- [ ] 能设计功能架构
- [ ] 能维护项目规范
- [ ] 能推动社区发展
