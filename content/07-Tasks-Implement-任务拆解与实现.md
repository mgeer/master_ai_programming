# 第七章：Tasks & Implement — 任务拆解与实现

## 7.1 引言

**本章目标**：学会从 Plan 拆解出可执行的任务列表，并通过 Test → Implement → Review 循环逐个完成。

上一章完成了 Plan。如果直接让 AI 按照方案生成代码，大概率会得到逻辑混杂的实现。本章的核心是：**将大方案拆解为可独立验证的小任务。**

本章分两个阶段：

| 阶段 | 做什么 | 产出 |
|------|--------|------|
| Tasks | 把 Plan 拆解为可执行的任务列表 | `tasks.md` |
| Implement | 对每个任务走 Test → Implement → Review 循环 | 测试代码 + 功能代码 |

打个比方：Plan 是菜谱（用什么食材、什么火候），Tasks 是备菜清单（先洗菜、再切肉、再调酱汁），Implement 是按清单一步步下锅。

> **提示**：本章案例继续使用 Todo 应用的 Feature 1，基于前文产出的 `@spec.md`、`@plan.md` 和 `@constitution.md`。

---

## 7.2 从 Plan 到任务列表

### 核心思路

将 Plan 转化为任务列表的核心思路是：**把方案中的每个设计模块，转化为可独立完成、可独立验证的工作单元，并按依赖关系排好执行顺序。**

任务粒度必须适中。"实现所有 CRUD"太粗，"给模型加 title 字段"太细。"实现 Todo 创建功能（含测试）"则是合适的工作单元。

排序原则：**被依赖的任务先做。**

### 拆解策略

任务拆解有两种常见策略：

- **自顶向下**：从 API 入口出发往下拆（API 路由 → 应用层 → 领域模型 → 数据库）。适合简单 CRUD。
- **自底向上**：从领域模型出发往上搭（领域实体 → 领域服务 → 应用层 → API）。适合复杂业务逻辑（如认证授权）。

| 任务 | 建议策略 | 原因 |
|------|---------|------|
| Todo CRUD | 自顶向下 | 逻辑简单，从 API 入手即可端到端测试 |
| 认证相关 | 自底向上 | 涉及密码哈希、JWT，需优先完成底层机制 |
| 筛选分页 | 自顶向下 | 在已有列表 API 上扩展查询参数 |

### 实操演示

#### 第一层：一句话启动

> **你**：基于 @plan.md，拆解为可执行的任务列表。每个任务要可独立完成和验证。标注任务之间的依赖关系和执行顺序，标注哪些任务可以并行。

#### 第二层：对话补充

> **你**：调整一下：
> 1. 项目基础搭建（数据库连接、配置管理、目录创建）单独作为首个任务。
> 2. 每个 API 端点作为一个独立任务。
> 3. 认证中间件（get_current_user）和测试基础设施单独作为前置任务。
> 4. 所有任务都依赖 Task 1 和 Task 2，认证任务要在 Todo CRUD 之前。

> AI 产出示例：
> 
> **Feature 1：Todo 核心功能 — 任务清单**
> 
> | 阶段 | 任务 | 依赖 | 可并行 |
> |------|------|------|--------|
> | 1 基础 | Task 1 项目搭建 — 目录结构、数据库连接、配置管理、Alembic 初始化 | — | — |
> | 1 基础 | Task 2 测试设施 — conftest.py、测试数据库、fixtures | Task 1 | — |
> | 2 认证 | Task 3 User 模型 — 领域实体、数据库模型、迁移脚本 | Task 1, 2 | 可与 Task 7 并行 |
> | 2 认证 | Task 4 用户注册 — 密码哈希、注册用例、注册 API | Task 3 | — |
> | 2 认证 | Task 5 用户登录 — JWT 工具、登录用例、登录 API | Task 3, 4 | — |
> | 2 认证 | Task 6 认证中间件 — get_current_user 依赖、Token 解析 | Task 5 | — |
> | 3 核心 | Task 7 Todo 模型 — 领域实体、数据库模型、迁移脚本 | Task 1, 2 | 可与 Task 3 并行 |
> | 3 核心 | Task 8 创建 Todo — 创建用例、创建 API | Task 6, 7 | — |
> | 3 核心 | Task 9 查看 Todo — 详情 + 列表（含分页）、授权检查 | Task 8 | 可与 10, 11 并行 |
> | 3 核心 | Task 10 修改 Todo — 部分更新、状态切换、授权检查 | Task 8 | 可与 9, 11 并行 |
> | 3 核心 | Task 11 删除 Todo — 软删除、授权检查 | Task 8 | 可与 9, 10 并行 |
> | 4 增强 | Task 12 筛选与排序 — 按状态/时间筛选、排序 | Task 9 | — |

### 审查要点

拿到任务列表后，检查以下要点：
- **是否覆盖了 Plan 的所有设计？**
- **每个任务是否可独立验证？**
- **有没有遗漏基础设施任务？**
- **依赖关系是否正确？** 例如 Task 8 必须依赖 Task 6。
- **顺序是否合理？** 必须遵循基础设施 → 认证 → 核心功能 → 增强功能的顺序。

---

## 7.3 任务审查

### 核心思路

在动手写代码之前，必须审查任务列表：**验证功能规范（spec）中的每个需求是否都被任务覆盖。**

### 实操演示

> **你**：审查任务列表，对照 @spec.md 的功能需求和验收标准，检查：
> 1. 每个功能需求（FR）是否都有对应的任务？
> 2. 每个验收标准（AC）是否都能通过某个任务的测试覆盖？
> 3. 有没有任务是多余的？

> AI 产出示例：
> 
> **一致性检查结果**
> 
> | 功能需求 | 对应任务 | 状态 |
> |---------|---------|------|
> | FR-1 ~ FR-7 | Task 4, 5, 8-11 | ✅ |
> | NFR-1 安全性 | Task 6 + 各 CRUD 任务 | ✅ |
> | NFR-2 性能 | — | ⚠️ 无专门任务 |
> | AC-1 ~ AC-32 | 各对应任务 | ✅ |
> | AC-33 ~ AC-35 | — | ⚠️ 缺失 |
> 
> **发现的问题**：
> 1. ⚠️ NFR-2 无专门任务。建议在 Task 12 中验证。
> 2. ⚠️ Token 刷新功能遗漏。建议新增 Task 13。

> **你**：根据审查结果：
> 1. 性能要求在 Task 12 验证。
> 2. 新增 Task 13（Token 刷新），依赖 Task 5。更新任务列表。

### 审查要点

最终确认：
- **每个 FR 至少对应一个任务。**
- **每个 AC 至少被某个任务的测试覆盖。**
- **任务总数合理。**

> **提示**：Tasks 阶段完成，最终清单保存至 `.spec/features/001-todo-core/tasks.md`。

---

## 7.4 实现循环概述

每个任务的实现遵循三步循环：**Test → Implement → Review**。

1. **Test（写测试）**：定义完成标准。测试不通过则任务未完成。
2. **Implement（写实现）**：编写代码使所有测试变绿。
3. **Review（审查）**：检查代码质量、架构规范及安全隐患。

---

## 7.5 演示任务 A：创建 Todo

> **前置条件**：假设 Task 1-7 已完成。以下演示标准的 Test → Implement → Review 流程。

### 7.5.1 Test（核心思路→实操演示→审查要点）

#### 核心思路

测试场景源自 `spec.md` 的验收标准。采用 BDD 风格（Given-When-Then）组织测试用例。

#### 实操演示

> **你**：为 Task 8（创建 Todo）编写测试。参考 @spec.md、@plan.md、@constitution.md 和 @tests/conftest.py，覆盖正常创建、字段缺失、超长及未认证等场景。

> AI 产出示例：
> 
> ```python
> # tests/todo/test_create_todo.py
> import pytest
> from httpx import AsyncClient
> 
> @pytest.mark.asyncio
> class TestCreateTodo:
>     """POST /api/v1/todos - 创建待办事项"""
> 
>     async def test_create_todo_success(self, client: AsyncClient, auth_headers: dict):
>         """Given 已认证用户, When 提供有效标题和描述, Then 返回 201 且包含 UUID"""
>         response = await client.post(
>             "/api/v1/todos",
>             json={"title": "买菜", "description": "西红柿"},
>             headers=auth_headers,
>         )
>         assert response.status_code == 201
>         data = response.json()
>         assert data["title"] == "买菜"
>         assert "id" in data
> ```

#### 审查要点

检查测试是否覆盖了正常与异常场景，断言是否包含关键字段（如 UUID 和状态），以及测试间是否独立。

### 7.5.2 Implement（核心思路→实操演示→审查要点）

#### 核心思路

编写代码使测试通过，必须遵循 `constitution.md` 的架构原则与编码规范。

#### 实操演示

> **你**：实现 Task 8（创建 Todo）的功能代码，使 @tests/todo/test_create_todo.py 的所有测试通过。遵循 @plan.md 和 @constitution.md。

> AI 产出示例：

```python
# src/todo/api/schemas.py
from datetime import datetime
from uuid import UUID
from pydantic import BaseModel, Field, field_validator

class TodoCreate(BaseModel):
    title: str = Field(..., min_length=1, max_length=200)
    description: str | None = Field(None, max_length=1000)

    @field_validator("title")
    @classmethod
    def title_must_not_be_blank(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("标题不能为空")
        return v.strip()

class TodoResponse(BaseModel):
    id: UUID
    title: str
    description: str | None
    status: str
    created_at: datetime
    updated_at: datetime

    model_config = {"from_attributes": True}
```

```python
# src/todo/application/todo_service.py
from uuid import UUID
from sqlalchemy.ext.asyncio import AsyncSession
from src.todo.domain.todo import Todo
from src.todo.infrastructure.todo_repository import TodoRepository

class TodoService:
    def __init__(self, session: AsyncSession):
        self.repository = TodoRepository(session)

    async def create_todo(self, title: str, description: str | None, user_id: UUID) -> Todo:
        todo = Todo(
            title=title,
            description=description,
            status="pending",
            user_id=user_id,
        )
        return await self.repository.save(todo)
```

#### 审查要点

检查分层是否正确、类型注解是否完整（尤其注意 `user_id` 必须为 UUID）、`user_id` 的来源是否安全（必须由中间件解析，而非客户端传入）。

### 7.5.3 Review（核心思路→实操演示→审查要点）

#### 核心思路

代码和测试都通过后，通过 Review 环节排查潜在的安全隐患和代码重构空间。

#### 实操演示

> **你**：审查 Task 8（创建 Todo）的代码，包括 schemas、service 和 routes。请将生成的代码与 `@plan.md` 进行反向对比审查（Strict Diff）。严格列出：1. 是否引入了未在 Plan 中声明的新库？2. 是否实现了未在 Plan 中定义的额外逻辑或过度设计（如过度封装、多余的中间件）？若存在，请标红警告并给出精简方案。同时检查安全隐患和边界情况遗漏。

> AI 产出示例：
> 
> **代码审查结果**
> ✅ 分层正确，用户 ID 从 Token 获取，输入校验覆盖全面。
> ⚠️ **建议**：补充一个测试以确保创建成功后的 Todo 绑定的 `user_id` 确实是当前用户的 UUID。

#### 审查要点

采纳合理的审查建议并补充测试，确保代码严谨可靠。

---

## 7.6 演示任务 B：用户注册

### 7.6.1 Test（核心思路→实操演示→审查要点）

#### 核心思路

用户注册涉及核心的认证安全（AC-1 至 AC-4），需要严格遵循 `constitution.md` 的密码 bcrypt 哈希及响应不含密码要求。

#### 实操演示

> **你**：为 Task 4（用户注册）编写测试。覆盖：正常注册、响应不含密码、邮箱重复、格式无效及密码短于 8 位等场景。结合 @constitution.md 与 @spec.md。

> AI 产出示例：

```python
# tests/auth/test_register.py
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
class TestUserRegister:
    async def test_register_response_excludes_password(self, client: AsyncClient):
        """Given 正常注册, When 查看响应, Then 不包含 password 字段"""
        response = await client.post(
            "/api/v1/auth/register",
            json={"email": "safe@test.com", "password": "secure123"},
        )
        data = response.json()
        assert "password" not in data
        assert "password_hash" not in data
```

#### 审查要点

安全测试必须充分，检查边界测试数据是否精准。

### 7.6.2 Implement（核心思路→实操演示→审查要点）

#### 核心思路

注册逻辑偏重底层，采用自底向上策略。需实现密码哈希、注册用例及 API 路由。

#### 实操演示

> **你**：实现 Task 4（用户注册），使相关测试通过。遵循 @constitution.md 和 @plan.md。

> AI 产出示例：

```python
# src/auth/api/schemas.py
from uuid import UUID
from pydantic import BaseModel, EmailStr, Field

class RegisterResponse(BaseModel):
    id: UUID
    email: str
    access_token: str
    token_type: str = "bearer"
```

#### 审查要点

确认密码哈希配置合规，响应模型不含密码字段，且用户 `id` 为 UUID 类型。

### 7.6.3 Review（核心思路→实操演示→审查要点）

#### 核心思路

认证模块对安全性要求极高，需重点排查并发竞态条件等深层缺陷。

#### 实操演示

> **你**：审查 Task 4 的代码，重点关注密码处理、并发注册同一邮箱时的安全性。

> AI 产出示例：
> 
> **安全审查结果**
> ⚠️ **并发竞态条件**：同时注册同一邮箱可能引发数据库 `IntegrityError`。建议在 `UserRepository` 中捕获并转换为 409 Conflict。

#### 审查要点

采纳 AI 建议增加异常捕获机制，提升系统健壮性。

---

## 7.7 当 AI 出错时

开发中 AI 经常出错。以下是应对方法：

- **库名混淆**：如将 `python-jose` 误用为 `PyJWT`。明确指出库名和导入方式并要求修正。
- **过度设计**：引入不必要的事件总线或缓存。引用 Constitution 强制简化结构。
- **命名违规**：使用了驼峰命名而非 `snake_case`。指出违规点要求全面替换。

Review 环节存在的意义正是为了捕获并纠正这些错误。

---

## 7.8 验收对照

### 核心思路

所有任务完成后，必须对照 `spec.md` 的验收标准逐条验证功能。

### 实操演示

> **你**：所有任务已完成。请对照 @spec.md 的验收标准（AC-1 到 AC-35），逐条检查是否有对应的测试覆盖并且通过。

> AI 产出示例：
> 
> **验收标准对照表**
> | 分类 | AC 编号 | 描述 | 对应测试 | 状态 |
> |------|--------|------|---------|------|
> | 注册 | AC-1, AC-2 | 注册成功 / 重复注册 | test_register_* | ✅ |
> | 认证 | AC-7, AC-8 | 无 Token 访问返回 401 | test_auth_* | ✅ |

### 审查要点

确保所有 AC 都有对应测试，测试真实通过，并确认代码测试覆盖率 ≥ 80%。

---

## 7.9 本章小结

**Tasks 阶段**：
1. **任务拆解与排序**：用一个提示词将方案拆分为粒度适中、可独立验证的工作单元，并按依赖关系排好执行顺序。
2. **任务审查**：确保任务完全覆盖所有 FR 和 AC。

**Implement 阶段**：
1. **实现循环**：坚持 Test → Implement → Review 三步走。
2. **验收对照**：对照 spec 逐条验收，完成闭环。

### 产出物清单

| 文件 | 说明 |
|------|------|
| `.spec/features/001-todo-core/tasks.md` | 任务清单（含依赖关系与排序） |
| `src/` | 业务与基础设施层代码 |
| `tests/` | 针对各功能的 BDD 风格测试代码 |
