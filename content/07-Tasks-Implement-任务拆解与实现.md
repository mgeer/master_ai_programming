# 第七章：Tasks & Implement — 任务拆解与实现

## 7.1 引言

**本章目标**：学会从 Plan 拆解出可执行的任务列表，并通过 Test → Implement → Review 循环逐个完成。

上一章我们完成了 Plan，拿到了一份完整的技术方案（`plan.md`）——领域模型、数据库表结构、API 契约、认证授权方案，全都设计好了。

现在的问题是：**这么多东西，从哪里开始写代码？**

如果你把整个 `plan.md` 扔给 AI 说"照着这个写"，会怎样？AI 大概率会一口气吐出几千行代码——路由、模型、服务、测试全混在一起。看起来能跑，但结构乱、Bug 多、后面改起来头疼。

这就是本章要解决的问题：**怎么把一个大方案拆成一个个小任务，然后一个一个地、有节奏地完成。**

本章分两个阶段：

| 阶段 | 做什么 | 产出 |
|------|--------|------|
| Tasks | 把 Plan 拆解为可执行的任务列表 | `tasks.md` |
| Implement | 对每个任务走 Test → Implement → Review 循环 | 测试代码 + 功能代码 |

打个比方：Plan 是菜谱（用什么食材、什么火候），Tasks 是备菜清单（先洗菜、再切肉、再调酱汁），Implement 是按清单一步步下锅。

> **本章的案例**：继续用 Todo 应用的 Feature 1（CRUD + 认证授权 + 筛选分页），基于前面产出的 `@spec.md`、`@plan.md` 和 `@constitution.md`。

---

## 7.2 从 Plan 到任务列表

### 讲

Plan 里有领域模型、数据模型、API 设计、认证流程、授权规则……内容很多，但它们不是"任务"。Plan 告诉你"系统长什么样"，Tasks 告诉你"先做什么、后做什么"。

把 Plan 变成任务列表，核心思路是：**把方案中的每个设计模块，转化为一个可独立完成、可独立验证的工作单元。**

什么叫"可独立验证"？就是做完一个任务后，你能跑测试确认它是对的，不依赖后面还没做的任务。比如"实现用户注册 API"是一个可独立验证的任务——写完后可以跑测试确认注册功能正常。但"写一半数据库模型"不是——你没法验证一半模型。

任务粒度的拿捏很重要：
- **太粗**："实现所有 CRUD"——这不是一个任务，是五个任务
- **太细**："给 Todo 模型加 title 字段"——太碎了，没有独立价值
- **刚好**："实现 Todo 创建功能（含测试）"——可独立完成、可独立验证

### 做

#### 第一层：一句话启动

```
基于 @plan.md，拆解为可执行的任务列表。每个任务要可独立完成和验证。
```

AI 会给出一份初版的任务列表。通常这个初版会有这些特点：
- 大方向是对的（会按模块拆分）
- 粒度可能不均匀（有的太粗，有的太细）
- 依赖关系可能没理清

#### 第二层：对话补充

看完初版后，通过 1-2 轮追问调整：

```
调整一下任务拆分：
1. 项目基础搭建（数据库连接、配置管理、目录创建）单独作为第一个任务
2. 每个 API 端点作为一个独立任务，不要把所有 CRUD 合成一个
3. 每个任务标注预计的文件变动范围
4. 每个任务的描述里要引用 plan.md 的对应章节
```

```
再补充：
1. 认证中间件（get_current_user）单独作为一个任务，因为后面所有受保护的 API 都依赖它
2. 把测试基础设施（conftest.py、测试数据库配置）也作为一个前置任务
```

#### AI 产出示例

经过对话调整后，AI 产出的任务列表大致如下：

> **Feature 1：Todo 核心功能 — 任务清单**
>
> **基础设施**
> - Task 1：项目基础搭建 — 目录结构、数据库连接、配置管理、Alembic 初始化
> - Task 2：测试基础设施 — conftest.py、测试数据库、fixtures
>
> **认证模块**
> - Task 3：User 模型 — 领域实体、数据库模型、迁移脚本
> - Task 4：用户注册 — 密码哈希、注册用例、注册 API
> - Task 5：用户登录 — JWT 工具、登录用例、登录 API
> - Task 6：认证中间件 — get_current_user 依赖、Token 解析
>
> **Todo 模块**
> - Task 7：Todo 模型 — 领域实体、数据库模型、迁移脚本
> - Task 8：创建 Todo — 创建用例、创建 API
> - Task 9：查看 Todo — 详情 + 列表（含分页）、授权检查
> - Task 10：修改 Todo — 部分更新、状态切换、授权检查
> - Task 11：删除 Todo — 软删除、授权检查
> - Task 12：筛选与排序 — 按状态/时间筛选、排序
>
> 每个任务都标注了验证条件，比如 Task 8 的验证条件是"认证用户能创建 Todo，未认证返回 401"。

### 审

拿到任务列表后，检查这几个点：

- **是否覆盖了 Plan 的所有设计？** 把 `plan.md` 的章节过一遍，每个设计点都应该落在某个任务里。
- **每个任务是否可独立验证？** 如果一个任务做完后没法测试它是否正确——粒度不对，要么太粗要么依赖了后面的任务。
- **有没有遗漏基础设施任务？** 项目搭建、测试配置、数据库迁移——这些"不是功能但必须先做"的事容易被忘掉。
- **有没有把不相关的事情塞进同一个任务？** 比如"实现注册 + 登录 + 认证中间件"——这三件事应该拆开。

---

## 7.3 拆解策略

任务拆解有两种常见策略：

**自顶向下**：从 API 入口出发往下拆——API 路由 → 应用层 → 领域模型 → 数据库。适合简单 CRUD，做完一个 API 就能端到端测试。

**自底向上**：从领域模型出发往上搭——领域实体 → 领域服务 → 应用层 → API。适合复杂业务逻辑（如认证授权），先把底层搞清楚再往上搭。

**怎么选？看复杂度。**

| 任务 | 建议策略 | 原因 |
|------|---------|------|
| Todo CRUD | 自顶向下 | 逻辑简单，从 API 入手就行 |
| 认证相关 | 自底向上 | 涉及密码哈希、JWT、中间件，底层先搞定 |
| 筛选分页 | 自顶向下 | 在已有列表 API 上添加查询参数 |

大多数任务用自顶向下就够了。只有业务逻辑确实复杂的，才需要自底向上。

---

## 7.4 任务排序与依赖

### 讲

任务列表有了，接下来的问题是：**先做哪个？**

随便挑一个开始做？不行。任务之间有依赖关系——你不可能在数据库连接都没配好的时候去写 API，也不可能在没有认证中间件的情况下测试"受保护端点返回 401"。

任务排序的核心原则：**被依赖的任务先做。**

简单说就是画依赖图，然后做拓扑排序。不过对于大多数项目，你不需要真画图——看一眼任务列表，凭直觉就能排出来。

### 做

#### 第一层：一句话启动

```
给上面的任务列表排序，标注依赖关系，哪些任务可以并行。
```

#### 第二层：对话补充

```
排序时注意：
1. 所有任务都依赖 Task 1（项目搭建）和 Task 2（测试基础设施）
2. 认证相关的任务要在 Todo CRUD 之前，因为 CRUD 需要认证
3. 标注哪些任务在依赖满足后可以并行执行
```

#### AI 产出示例

> **任务依赖与排序**
>
> | 阶段 | 任务 | 依赖 | 可并行 |
> |------|------|------|--------|
> | 1 基础 | Task 1 项目搭建 | — | — |
> | 1 基础 | Task 2 测试设施 | Task 1 | — |
> | 2 认证 | Task 3 User 模型 | Task 1, 2 | 可与 Task 7 并行 |
> | 2 认证 | Task 4 用户注册 | Task 3 | — |
> | 2 认证 | Task 5 用户登录 | Task 3, 4 | — |
> | 2 认证 | Task 6 认证中间件 | Task 5 | — |
> | 3 核心 | Task 7 Todo 模型 | Task 1, 2 | 可与 Task 3 并行 |
> | 3 核心 | Task 8 创建 Todo | Task 6, 7 | — |
> | 3 核心 | Task 9 查看 Todo | Task 8 | 可与 10, 11 并行 |
> | 3 核心 | Task 10 修改 Todo | Task 8 | 可与 9, 11 并行 |
> | 3 核心 | Task 11 删除 Todo | Task 8 | 可与 9, 10 并行 |
> | 4 增强 | Task 12 筛选排序 | Task 9 | — |

### 审

审查排序时注意：

- **依赖关系是否正确？** 特别注意隐含的依赖。比如 Task 8（创建 Todo）依赖 Task 6（认证中间件）——因为创建 Todo 需要认证用户。这个依赖容易被忽略。
- **顺序是否合理？** 基础设施 → 认证 → Todo CRUD → 增强功能。如果 AI 把筛选排在了创建前面——不对，你总得先有数据才能筛选。
- **并行标注是否准确？** 标注为并行的任务之间真的没有依赖吗？比如 Task 9（查看 Todo）和 Task 10（修改 Todo）如果都需要先创建 Todo 的测试数据，它们可以并行，但各自的测试中都要先调用创建逻辑。

---

## 7.5 任务审查

### 讲

任务拆好了、排好了，在动手写代码之前，还有最后一步：**审查任务列表本身。**

为什么？因为任务列表是 Plan 到代码之间的桥梁。如果这个桥梁有漏洞——某个 spec 需求没有对应的任务——那这个需求就会被彻底遗忘。等你做完所有任务、自以为大功告成的时候，验收时才发现漏了东西。

任务审查本质上是一致性检查：**spec 的每个功能需求，是否都被某个任务覆盖了？**

### 做

#### 第一层：一句话启动

```
审查任务列表，对照 @spec.md 的功能需求和验收标准，检查：
1. 每个功能需求（FR）是否都有对应的任务？
2. 每个验收标准（AC）是否都能通过某个任务的测试覆盖？
3. 有没有任务是多余的（spec 里没有对应需求）？
```

#### AI 产出示例

> **一致性检查结果**
>
> | 功能需求 | 对应任务 | 状态 |
> |---------|---------|------|
> | FR-1 ~ FR-7（注册/登录/CRUD） | Task 4, 5, 8-11 | ✅ |
> | NFR-1 安全性（认证授权） | Task 6 + 各 CRUD 任务 | ✅ |
> | NFR-2 性能 | — | ⚠️ 无专门任务 |
> | AC-1 ~ AC-32（验收标准） | 各对应任务 | ✅ |
> | AC-33 ~ AC-35（Token 刷新） | — | ⚠️ 缺失 |
>
> **发现的问题**：
> 1. ⚠️ NFR-2（性能）没有专门任务。建议在 Task 12 中附带验证大数据量分页性能。
> 2. ⚠️ Token 刷新功能遗漏。建议新增 Task 13。

#### 你的处理

```
根据审查结果：
1. 性能要求——同意你的建议，在 Task 12 里附带验证，不单独建任务
2. Token 刷新——新增 Task 13，依赖 Task 5（登录），排在阶段 2 末尾

更新任务列表。
```

### 审

审查完成后，做最终确认：

- **每个 FR 至少对应一个任务？** 没有对应任务的 FR 等于不会被实现。
- **每个 AC 至少被某个任务的测试覆盖？** 这一点决定了最终验收能否通过。
- **新增的任务有没有正确排序和标注依赖？** Task 13 加进去后，依赖关系要同步更新。
- **任务总数合理吗？** Feature 1 有 12-13 个任务是合理的。如果超过 20 个，说明拆得太细；如果少于 5 个，说明拆得太粗。

> **至此，Tasks 阶段完成。** 最终的任务列表保存到 `.spec/features/001-todo-core/tasks.md`。

---

## 7.6 实现循环概述

任务列表准备好了，现在进入 Implement 阶段——动手写代码。

每个任务的实现遵循三步循环：**Test → Implement → Review**。

```
┌──────────┐     ┌─────────────┐     ┌──────────┐
│   Test   │ ──→ │  Implement  │ ──→ │  Review  │
│  写测试  │     │   写实现    │     │   审查   │
└──────────┘     └─────────────┘     └──────────┘
```

为什么是这个顺序？

**Test 先行**——先定义"什么算完成"。你告诉 AI"写 Todo 创建的测试"，AI 会根据 spec 的验收标准写出测试用例。这些测试就是这个任务的完成标准。没通过？任务就没做完。

**Implement 其次**——让测试通过。有了测试用例，实现的目标就很明确：写代码让所有测试变绿。AI 不会多做也不会少做。

**Review 最后**——检查代码质量。测试通过只说明"功能对了"，不能说明"代码好"。Review 阶段检查代码是否符合 Constitution 的规范、有没有安全隐患、有没有重构空间。

接下来，我们选两个代表性任务做**完整演示**：

- **任务 A（Task 8）：创建 Todo**——基础 CRUD，逻辑简单，适合展示标准的 Test → Implement → Review 流程
- **任务 B（Task 4）：用户注册**——认证相关，逻辑稍复杂，重点展示如何引用 Plan 中的安全设计

其余任务按同样模式推进，不再逐个展示。

---

## 7.7 演示任务 A：创建 Todo

> **前置条件**：假设 Task 1-7 已经完成——项目搭建好了、数据库能连、User 和 Todo 模型都有了、注册登录和认证中间件都可用。
>
> Task 1-7 的实现模式与下文演示的 Task 8 完全相同——都是 Test → Implement → Review 循环。建议你先通读本节了解完整模式，然后从 Task 1 开始按顺序实践。如果你希望直接从 Task 8 开始跟练，可以用以下提示词让 AI 先完成基础代码：
>
> `按照 @tasks.md 中 Task 1-7 的描述，逐个实现。每个 Task 遵循测试 → 实现 → 审查的模式。`

### 7.7.1 Test（讲→做→审）

#### 讲

Test 阶段做的事情：**为这个任务设计测试场景，然后写测试代码。**

测试场景从哪来？从 spec 的验收标准来。打开 `spec.md`，找到和"创建 Todo"相关的验收标准：

- AC-9：创建 Todo 提供标题，返回完整 Todo 对象，状态码 201
- AC-10：创建 Todo 不提供标题，返回验证错误，状态码 422
- AC-11：创建 Todo 标题超过 200 字符，返回验证错误，状态码 422
- AC-26：创建 Todo 标题为纯空格，返回 422
- AC-7：不携带 Token 访问受保护接口，返回 401

另外，spec 的 FR-3 规定描述最多 1000 字符，虽然没有单独的 AC 编号，也要作为测试场景覆盖。

这六条验收标准加上描述超长的边界场景，就是我们的测试场景。

Constitution 定了用 BDD 风格设计测试（Given-When-Then），所以测试用例应该这样组织：
- Given：一个已认证的用户
- When：发送创建 Todo 的请求
- Then：返回预期的结果

#### 做

```
为 Task 8（创建 Todo）编写测试。

参考：
- 验收标准：@spec.md 的 AC-7, AC-9, AC-10, AC-11, AC-26 + FR-3（描述长度）
- API 契约：@plan.md 的 POST /api/v1/todos
- 测试规范：@constitution.md 的开发方法论（BDD 风格、pytest）
- 测试基础设施：@tests/conftest.py

测试应覆盖：
1. 正常创建（提供标题和描述）
2. 只提供标题（描述可选）
3. 缺少标题
4. 标题超长（>200 字符）
5. 标题为纯空格
6. 描述超长（>1000 字符）
7. 未认证访问
```

AI 产出示例：

```python
# tests/todo/test_create_todo.py
import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
class TestCreateTodo:
    """POST /api/v1/todos - 创建待办事项"""

    async def test_create_todo_with_title_and_description(
        self, client: AsyncClient, auth_headers: dict
    ):
        """Given 已认证用户, When 提供标题和描述创建 Todo, Then 返回 201 和完整对象"""
        response = await client.post(
            "/api/v1/todos",
            json={"title": "买菜", "description": "西红柿、鸡蛋"},
            headers=auth_headers,
        )
        assert response.status_code == 201
        data = response.json()
        assert data["title"] == "买菜"
        assert data["description"] == "西红柿、鸡蛋"
        assert data["status"] == "pending"
        assert "id" in data
        assert "created_at" in data

    async def test_create_todo_title_only(
        self, client: AsyncClient, auth_headers: dict
    ):
        """Given 已认证用户, When 只提供标题, Then 返回 201，描述为 null"""
        response = await client.post(
            "/api/v1/todos",
            json={"title": "跑步"},
            headers=auth_headers,
        )
        assert response.status_code == 201
        assert response.json()["description"] is None

    async def test_create_todo_without_title(
        self, client: AsyncClient, auth_headers: dict
    ):
        """Given 已认证用户, When 不提供标题, Then 返回 422"""
        response = await client.post(
            "/api/v1/todos",
            json={"description": "缺少标题"},
            headers=auth_headers,
        )
        assert response.status_code == 422

    async def test_create_todo_title_too_long(
        self, client: AsyncClient, auth_headers: dict
    ):
        """Given 已认证用户, When 标题超过 200 字符, Then 返回 422"""
        response = await client.post(
            "/api/v1/todos",
            json={"title": "a" * 201},
            headers=auth_headers,
        )
        assert response.status_code == 422

    async def test_create_todo_title_whitespace_only(
        self, client: AsyncClient, auth_headers: dict
    ):
        """Given 已认证用户, When 标题为纯空格, Then 返回 422"""
        response = await client.post(
            "/api/v1/todos",
            json={"title": "   "},
            headers=auth_headers,
        )
        assert response.status_code == 422

    async def test_create_todo_description_too_long(
        self, client: AsyncClient, auth_headers: dict
    ):
        """Given 已认证用户, When 描述超过 1000 字符, Then 返回 422"""
        response = await client.post(
            "/api/v1/todos",
            json={"title": "测试", "description": "a" * 1001},
            headers=auth_headers,
        )
        assert response.status_code == 422

    async def test_create_todo_unauthenticated(self, client: AsyncClient):
        """Given 未认证用户, When 创建 Todo, Then 返回 401"""
        response = await client.post(
            "/api/v1/todos",
            json={"title": "无权限"},
        )
        assert response.status_code == 401
```

#### 审

审查测试代码时注意：

- **是否覆盖了所有相关的验收标准？** AC-7、AC-9、AC-10、AC-11、AC-26 ——逐条对照，每条至少有一个测试用例。描述超长虽然没有独立 AC 编号，也要有测试。
- **测试场景是否包含正常和异常？** 不能全是 happy path。上面的七个测试覆盖了 2 个正常场景和 5 个异常场景——合理。
- **断言是否充分？** 不能只断言状态码。创建成功后要检查返回体的各个字段：title、description、status、id、created_at。
- **测试之间是否独立？** 每个测试用例不应依赖其他测试的结果。如果 AI 写的测试有顺序依赖——需要改。
- **是否用了 BDD 风格的 docstring？** Constitution 要求 Given-When-Then，检查一下每个测试是否有清晰的 docstring 描述。

### 7.7.2 Implement（讲→做→审）

#### 讲

测试写好了，现在跑一下——全部红。这是预期的，因为功能代码还没写。

Implement 阶段的目标很简单：**写代码让所有测试变绿。**

这个过程中，AI 需要参考：
- `plan.md` 的 API 契约（请求/响应格式、路径）
- `plan.md` 的目录结构（代码放在哪个文件）
- `constitution.md` 的编码规范（命名、类型注解、代码风格）
- 刚写好的测试代码（这就是"完成标准"）

#### 做

```
实现 Task 8（创建 Todo）的功能代码，使 @tests/todo/test_create_todo.py 的所有测试通过。

参考：
- API 契约：@plan.md 的 POST /api/v1/todos
- 目录结构：@plan.md 的项目目录结构
- 编码规范：@constitution.md

需要创建/修改的文件：
1. todo/api/schemas.py — 请求和响应的 Pydantic 模型
2. todo/application/todo_service.py — 创建 Todo 用例
3. todo/api/routes.py — 路由定义
4. main.py — 注册路由
```

AI 产出示例：

```python
# src/todo/api/schemas.py
from datetime import datetime
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

```python
# src/todo/api/routes.py
from fastapi import APIRouter, Depends, status
from sqlalchemy.ext.asyncio import AsyncSession

from src.shared.database import get_session
from src.shared.dependencies import get_current_user
from src.todo.api.schemas import TodoCreate, TodoResponse
from src.todo.application.todo_service import TodoService

router = APIRouter(prefix="/api/v1/todos", tags=["todos"])


@router.post("", status_code=status.HTTP_201_CREATED, response_model=TodoResponse)
async def create_todo(
    body: TodoCreate,
    current_user=Depends(get_current_user),
    session: AsyncSession = Depends(get_session),
) -> TodoResponse:
    """创建待办事项。"""
    service = TodoService(session)
    todo = await service.create_todo(
        title=body.title,
        description=body.description,
        user_id=current_user.id,
    )
    return todo
```

#### 审

实现完成后，先跑测试——确认全部通过。然后审查代码质量：

- **分层是否正确？** API 层只做请求解析和响应封装（`routes.py`），应用层做业务编排（`todo_service.py`），数据校验在 schema 层（`schemas.py`）。如果 AI 在 routes 里写了数据库操作——违反了 Constitution 的架构原则。
- **命名是否符合规范？** 文件名 `snake_case`、类名 `PascalCase`、函数名 `snake_case`——对照 Constitution 的编码规范。
- **类型注解完整吗？** Constitution 要求所有函数必须有类型注解。检查每个函数的参数和返回值。
- **user_id 的来源安全吗？** 必须从 `get_current_user` 依赖获取，不能从请求体中传入。看 `routes.py` 的 `create_todo` 函数——`user_id=current_user.id`，从 Token 解析出来的用户获取，正确。
- **输入校验够不够？** 标题长度 1-200、不能纯空格、描述最多 1000 字符——对照 spec 的要求。

### 7.7.3 Review（讲→做→审）

#### 讲

测试通过了，代码也检查过了。Review 阶段做最后一轮检查：**让 AI 以代码审查者的身份审查刚写的代码。**

为什么需要这一步？因为前面你已经"身在其中"——写测试、写实现、看代码。Review 是换个角度看：有没有安全隐患？有没有边界情况没处理？有没有重构机会？

#### 做

```
审查 Task 8（创建 Todo）的代码，包括：
- @src/todo/api/schemas.py
- @src/todo/application/todo_service.py
- @src/todo/api/routes.py
- @tests/todo/test_create_todo.py

审查要点：
1. 是否符合 @constitution.md 的编码规范和架构原则？
2. 有没有安全隐患？
3. 有没有边界情况遗漏？
4. 有没有重构机会？
```

AI 产出示例：

> **代码审查结果**
>
> **✅ 合格项**
> - 分层正确：API 层不含业务逻辑
> - 类型注解完整
> - 用户 ID 从 Token 获取，不接受客户端传入
> - 输入校验覆盖标题和描述的长度限制
>
> **⚠️ 建议改进**
>
> 1. **描述字段的处理**：空字符串和 `null` 的区别需要确认。当前实现中，客户端传 `""` 和不传 `description` 的行为可能不同。建议加一个 validator：空字符串也转为 `None`。
>
> 2. **错误消息国际化**：`field_validator` 里的错误消息是中文 `"标题不能为空"`。如果 API 面向国际用户，建议改为英文。但对于本项目（中文 Todo 应用），中文错误消息是合理的。
>
> 3. **测试补充建议**：可以增加一个测试——创建成功后的 Todo 状态一定是 `pending`，且 `user_id` 是当前用户的 ID。虽然 AC-9 已经覆盖了部分，但显式验证 `user_id` 更安全。

#### 审

审查 AI 的审查结果——这里你是"审查审查者"：

- **建议是否合理？** AI 说的"空字符串转 None"确实有道理——spec 里澄清过"描述填空字符串也视为 null"。采纳。
- **建议是否值得做？** "错误消息国际化"——对于 Todo 案例不需要。拒绝。
- **建议是否安全相关？** 如果有安全相关的建议，优先处理。这次没有安全问题。
- **补充的测试值得加吗？** 验证 `user_id` 是当前用户——确实有价值，加上。

根据 Review 结果做相应修改，然后重新跑测试确保全部通过。

---

## 7.8 演示任务 B：用户注册

> 任务 B 选取 Task 4（用户注册），重点展示如何引用 Plan 中的安全设计来指导实现。

### 7.8.1 Test（讲→做→审）

#### 讲

用户注册涉及认证安全，spec 里有大量相关的验收标准：

- AC-1：合法邮箱和密码注册成功，返回 201，响应包含 access token
- AC-2：已注册邮箱再次注册，返回 409
- AC-3：无效邮箱格式注册，返回 422
- AC-4：密码少于 8 位，返回 422

另外，Plan 里的安全设计给了我们额外的约束：

- `plan.md` 的注册流程：校验 → 查重 → 哈希 → 入库 → 返回
- `constitution.md` 的安全策略：密码用 bcrypt 哈希，响应不含密码字段
- `spec.md` 的 Clarify 决策：邮箱不区分大小写，注册后自动登录

这些约束都要体现在测试里。

#### 做

```
为 Task 4（用户注册）编写测试。

参考：
- 验收标准：@spec.md 的 AC-1, AC-2, AC-3, AC-4
- 注册流程：@plan.md 的认证流程设计
- 安全策略：@constitution.md（密码 bcrypt 哈希、响应不含密码）
- Clarify 决策：邮箱不区分大小写、注册后自动返回 token

测试应覆盖：
1. 正常注册（返回用户信息 + token）
2. 响应中不包含密码
3. 邮箱已注册
4. 邮箱格式无效
5. 密码少于 8 位
6. 邮箱大小写不敏感（大写邮箱注册，能用小写登录）
```

AI 产出示例：

```python
# tests/auth/test_register.py
import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
class TestUserRegister:
    """POST /api/v1/auth/register - 用户注册"""

    async def test_register_success(self, client: AsyncClient):
        """Given 合法邮箱和密码, When 注册, Then 返回 201 和用户信息及 token"""
        response = await client.post(
            "/api/v1/auth/register",
            json={"email": "new@test.com", "password": "secure123"},
        )
        assert response.status_code == 201
        data = response.json()
        assert data["email"] == "new@test.com"
        assert "id" in data
        assert "access_token" in data
        assert data["token_type"] == "bearer"

    async def test_register_response_excludes_password(self, client: AsyncClient):
        """Given 正常注册, When 查看响应, Then 不包含 password 字段"""
        response = await client.post(
            "/api/v1/auth/register",
            json={"email": "safe@test.com", "password": "secure123"},
        )
        data = response.json()
        assert "password" not in data
        assert "password_hash" not in data

    async def test_register_duplicate_email(self, client: AsyncClient):
        """Given 邮箱已注册, When 再次注册, Then 返回 409"""
        await client.post(
            "/api/v1/auth/register",
            json={"email": "dup@test.com", "password": "secure123"},
        )
        response = await client.post(
            "/api/v1/auth/register",
            json={"email": "dup@test.com", "password": "another123"},
        )
        assert response.status_code == 409

    async def test_register_invalid_email(self, client: AsyncClient):
        """Given 无效邮箱格式, When 注册, Then 返回 422"""
        response = await client.post(
            "/api/v1/auth/register",
            json={"email": "not-an-email", "password": "secure123"},
        )
        assert response.status_code == 422

    async def test_register_short_password(self, client: AsyncClient):
        """Given 密码少于 8 位, When 注册, Then 返回 422"""
        response = await client.post(
            "/api/v1/auth/register",
            json={"email": "short@test.com", "password": "1234567"},
        )
        assert response.status_code == 422

    async def test_register_email_case_insensitive(self, client: AsyncClient):
        """Given 大写邮箱注册, When 用小写邮箱重复注册, Then 返回 409"""
        await client.post(
            "/api/v1/auth/register",
            json={"email": "UPPER@test.com", "password": "secure123"},
        )
        response = await client.post(
            "/api/v1/auth/register",
            json={"email": "upper@test.com", "password": "secure123"},
        )
        assert response.status_code == 409
```

#### 审

审查测试时注意：

- **安全相关的测试是否充分？** 特别关注"响应不含密码"和"邮箱大小写"这两个测试——它们直接对应 Constitution 和 Clarify 中的安全决策。
- **测试数据是否合理？** 密码"secure123"满足 ≥8 位的要求，"1234567"是 7 位——刚好卡在边界上。这种边界值测试很重要。
- **重复邮箱测试的方式对吗？** 需要先注册成功一次，再注册第二次——确保是"注册后再重复注册"的场景。
- **有没有遗漏的场景？** 比如邮箱为空、密码为空——这些也是验证错误场景。如果 AC 里没列，可以考虑补充。

### 7.8.2 Implement（讲→做→审）

#### 讲

用户注册的实现比 Todo CRUD 复杂一些，因为涉及安全相关的基础设施：密码哈希、Token 生成。这正是自底向上策略的适用场景——先搞定底层工具，再搭上层逻辑。

实现时要紧盯 Plan 的注册流程时序图：校验 → 查重 → 哈希 → 入库 → 生成 Token → 返回。每一步都要在代码里有对应。

#### 做

```
实现 Task 4（用户注册），使 @tests/auth/test_register.py 的所有测试通过。

参考：
- 注册流程：@plan.md 的注册时序图
- 安全策略：@constitution.md（bcrypt cost=12、JWT HS256）
- 目录结构：@plan.md

需要创建/修改的文件：
1. auth/api/schemas.py — 注册请求/响应模型
2. auth/infrastructure/password_hasher.py — 密码哈希
3. auth/infrastructure/jwt_handler.py — JWT 生成（如果 Task 5 还没做，先实现生成部分）
4. auth/application/auth_service.py — 注册用例
5. auth/api/routes.py — 注册路由
```

AI 产出示例：

```python
# src/auth/api/schemas.py
from pydantic import BaseModel, EmailStr, Field


class RegisterRequest(BaseModel):
    email: EmailStr
    password: str = Field(..., min_length=8)


class RegisterResponse(BaseModel):
    id: UUID
    email: str
    access_token: str
    token_type: str = "bearer"
```

```python
# src/auth/infrastructure/password_hasher.py
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto", bcrypt__rounds=12)


def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)
```

```python
# src/auth/application/auth_service.py
from sqlalchemy.ext.asyncio import AsyncSession

from src.auth.domain.user import User
from src.auth.infrastructure.jwt_handler import create_access_token
from src.auth.infrastructure.password_hasher import hash_password
from src.auth.infrastructure.user_repository import UserRepository
from src.shared.exceptions import ConflictError


class AuthService:
    def __init__(self, session: AsyncSession):
        self.repository = UserRepository(session)

    async def register(self, email: str, password: str) -> dict:
        email = email.lower()

        existing = await self.repository.find_by_email(email)
        if existing:
            raise ConflictError("该邮箱已被注册")

        password_hash = hash_password(password)
        user = User(email=email, password_hash=password_hash)
        user = await self.repository.save(user)

        access_token = create_access_token(user_id=user.id)

        return {
            "id": user.id,
            "email": user.email,
            "access_token": access_token,
            "token_type": "bearer",
        }
```

```python
# src/auth/api/routes.py
from fastapi import APIRouter, Depends, status
from sqlalchemy.ext.asyncio import AsyncSession

from src.auth.api.schemas import RegisterRequest, RegisterResponse
from src.auth.application.auth_service import AuthService
from src.shared.database import get_session

router = APIRouter(prefix="/api/v1/auth", tags=["auth"])


@router.post(
    "/register", status_code=status.HTTP_201_CREATED, response_model=RegisterResponse
)
async def register(
    body: RegisterRequest,
    session: AsyncSession = Depends(get_session),
) -> RegisterResponse:
    """用户注册。"""
    service = AuthService(session)
    result = await service.register(email=body.email, password=body.password)
    return result
```

#### 审

实现完成后，跑测试确认通过。然后审查代码：

- **邮箱是否转小写了？** 在 `auth_service.py` 的 `register` 方法里，`email = email.lower()` ——正确。这是 Clarify 阶段决定的"邮箱不区分大小写"。
- **密码哈希是否符合要求？** `bcrypt__rounds=12` ——和 Constitution 的"cost factor 12"一致。
- **响应里有没有密码？** `RegisterResponse` 只有 `id`、`email`、`access_token`、`token_type` ——没有密码字段，安全。
- **重复邮箱的处理？** 先查询再判断，有则抛 `ConflictError`（返回 409）——正确。
- **分层是否正确？** 路由层只做请求解析，服务层做业务编排，密码哈希在基础设施层——符合 DDD 四层架构。

### 7.8.3 Review（讲→做→审）

#### 讲

认证相关的代码要格外仔细审查，因为安全漏洞的后果比功能 Bug 严重得多。

#### 做

```
审查 Task 4（用户注册）的代码，重点关注安全性。

审查要点：
1. 密码处理：是否正确使用 bcrypt？明文密码是否在任何地方泄露？
2. 邮箱处理：大小写统一是否在所有入口都生效？
3. 错误信息：是否暴露了不该暴露的信息？
4. 并发安全：两个请求同时注册同一个邮箱会怎样？
```

AI 产出示例：

> **安全审查结果**
>
> **✅ 通过**
> - 密码用 bcrypt 哈希，cost factor 12，符合 Constitution
> - 响应不包含密码或密码哈希
> - 邮箱统一转小写后再查重和存储
> - 重复邮箱返回通用错误信息，不暴露已注册用户列表
>
> **⚠️ 需要注意**
>
> 1. **并发注册竞态条件**：两个请求几乎同时注册同一个邮箱，`find_by_email` 都返回 None，两个都尝试 INSERT，可能导致数据库唯一约束异常。建议：在 `user_repository.save()` 中捕获 `IntegrityError`，转为 `ConflictError`。
>
> 2. **密码长度校验的位置**：当前在 Pydantic schema（API 层）做的 `min_length=8`。可以考虑在领域层也加一个校验——双重保障。但对于当前规模，API 层校验已经足够。
>
> 3. **日志安全**：如果后续加日志，确保不记录 `password` 参数。当前代码没有日志，暂时没问题，但值得提前意识到。

#### 审

审查 AI 的安全审查建议：

- **并发竞态条件** ——这是个真实问题。两个请求同时注册同一邮箱，查重都通过了，但 INSERT 时数据库的唯一约束会报错。应该捕获这个异常，返回 409 而不是 500。采纳，加上异常处理。
- **领域层双重校验** ——对于 Todo 应用不需要。如果是银行系统或者医疗系统，可以考虑。拒绝。
- **日志安全** ——好建议，记住就行，当前不需要改代码。

根据 Review 结果修改 `user_repository.py`，捕获 `IntegrityError`，然后重新跑测试。

---

## 7.9 其余任务

Task 8（创建 Todo）和 Task 4（用户注册）我们做了完整演示。其余任务按同样的 Test → Implement → Review 模式推进。这里概述每个任务的要点和完成状态：

### 基础设施

| 任务 | 要点 | 状态 |
|------|------|------|
| Task 1：项目基础搭建 | 目录结构、数据库连接、Alembic 初始化 | ✅ 完成 |
| Task 2：测试基础设施 | conftest.py、测试数据库、fixtures | ✅ 完成 |

### 认证模块

| 任务 | 要点 | 状态 |
|------|------|------|
| Task 3：User 模型 | SQLAlchemy 模型、迁移脚本 | ✅ 完成 |
| Task 4：用户注册 | bcrypt 哈希、邮箱查重、自动登录返回 Token（本章已演示） | ✅ 完成 |
| Task 5：用户登录 | 密码验证、JWT 生成、统一错误信息 | ✅ 完成 |
| Task 6：认证中间件 | get_current_user 依赖、Token 解析 | ✅ 完成 |
| Task 13：Token 刷新 | Refresh Token 生成与验证 | ✅ 完成 |

**Task 5（用户登录）要点**：测试要覆盖"正确凭证返回 Token"、"错误密码返回 401"、"不存在的邮箱返回 401"三种场景。关键：错误时统一返回"用户名或密码错误"，不区分是哪种错误。

**Task 6（认证中间件）要点**：这是所有受保护 API 的前置依赖。测试要覆盖"无 Token"、"无效 Token"、"过期 Token"、"有效 Token"四种场景。实现时用 FastAPI 的 `Depends` 机制。

### Todo 模块

| 任务 | 要点 | 状态 |
|------|------|------|
| Task 7：Todo 模型 | SQLAlchemy 模型、迁移脚本、软删除字段 | ✅ 完成 |
| Task 8：创建 Todo | 标题校验、状态默认 pending、关联当前用户（本章已演示） | ✅ 完成 |
| Task 9：查看 Todo | 详情 + 列表、授权检查（只能看自己的）、分页 | ✅ 完成 |
| Task 10：修改 Todo | 部分更新、状态双向切换、授权检查 | ✅ 完成 |
| Task 11：删除 Todo | 软删除、删除后返回 404、授权检查 | ✅ 完成 |
| Task 12：筛选与排序 | 按状态/时间筛选、排序、大数据量验证 | ✅ 完成 |

**Task 9 ~ 12 要点速览**：

- **Task 9（查看）**：详情 + 列表两个子功能。授权是重点——用户 B 查看用户 A 的 Todo 返回 404。分页要测默认值、自定义值、超页码。
- **Task 10（修改）**：部分更新语义（只更新传入的字段）。状态双向切换都要测。
- **Task 11（删除）**：软删除后查询返回 404，但数据库记录还在。重复删除返回 404。
- **Task 12（筛选）**：在列表 API 基础上增强。组合筛选是 AND 关系。

---

## 7.10 当 AI 出错时

前面的演示中，AI 的产出都很顺利。但真实项目中，AI 经常出错。这里列几个典型场景和应对方法，让你有心理准备。

### 场景 1：库名混淆

你在 Constitution 中写了"使用 python-jose 做 JWT"，但 AI 生成的代码用了：

```python
import jwt  # 这是 PyJWT 库的写法

token = jwt.encode(payload, secret, algorithm="HS256")
```

而 `python-jose` 的正确用法是：

```python
from jose import jwt  # python-jose 的写法

token = jwt.encode(payload, secret, algorithm="HS256")
```

两个库的 API 很像，但不是同一个东西。AI 经常搞混——因为训练数据中两者都大量出现。

**诊断方法**：看 import 语句 → 对照 `requirements.txt` 的依赖列表。

**修复方法**：

```
我们使用的是 python-jose 库（from jose import jwt），不是 PyJWT。
请修正所有 JWT 相关的 import 和调用方式。
```

### 场景 2：过度设计

你让 AI 实现"创建 Todo"，它给你加了缓存层、事件发布、观察者模式：

```python
class TodoService:
    def __init__(self, repo, cache, event_bus):  # 一个 Todo 应用要什么事件总线？
        ...
```

**诊断方法**：看 AI 产出的类和方法数量。如果一个简单的 CRUD 操作涉及了 5 个以上的类——过度设计。

**修复方法**：

```
这个实现过度设计了。对一个 Todo 应用来说不需要缓存层和事件总线。
请简化为直接的 Repository → Service → Route 调用链。
参考 @constitution.md 的架构原则。
```

### 场景 3：和 Constitution 不一致

Constitution 规定"命名用 snake_case"，但 AI 生成了 camelCase 的字段名：

```python
class TodoResponse(BaseModel):
    createdAt: datetime  # 应该是 created_at
    updatedAt: datetime  # 应该是 updated_at
```

**诊断方法**：审查代码时对照 Constitution 的命名规范逐项检查。

**修复方法**：

```
字段命名不符合 @constitution.md 的规范，应使用 snake_case。
请将 createdAt 改为 created_at，updatedAt 改为 updated_at。
```

### 关键心态

AI 出错不是意外，是常态。**"审"环节存在的意义就是抓住这些错误。** 每个 Task 实现完毕后的 Review 不是走过场——认真对照 Constitution、Plan 和测试结果，才能保证代码质量。

如果某类错误反复出现（比如 AI 总是用错库），可以在 Constitution 或规则文件中加一条明确提醒，减少后续出错概率。

---

## 7.11 验收对照

### 讲

所有任务都完成了。但"任务完成"不等于"Feature 完成"。

怎么确认 Feature 1 真的做完了？**对照 spec.md 的验收标准逐条检查。**

这就像考试交卷前的最后一遍核对——题目都答了吗？有没有漏掉的？有没有答错的？

验收对照不是走形式。它是 Specify 阶段埋下的"验证锚点"——当初你花时间写的每一条验收标准，现在到了发挥作用的时候。

### 做

```
所有任务已完成。请对照 @spec.md 的验收标准（AC-1 到 AC-35），逐条检查：
1. 是否有对应的测试覆盖
2. 测试是否通过
3. 有没有遗漏的验收标准

格式：每条标注 ✅ 通过 / ❌ 未通过 / ⚠️ 缺失
```

AI 产出示例：

> **验收标准对照表**（节选，完整表格 35 条）
>
> | 分类 | AC 编号 | 描述 | 对应测试 | 状态 |
> |------|--------|------|---------|------|
> | 注册 | AC-1, AC-2 | 注册成功 / 重复注册 | test_register_success, test_register_duplicate_email | ✅ |
> | 注册校验 | AC-3, AC-4 | 邮箱格式 / 密码长度 | test_register_invalid_email, test_register_short_password | ✅ |
> | 登录 | AC-5, AC-6 | 登录成功 / 失败 | test_login_* | ✅ |
> | 认证 | AC-7, AC-8 | 无 Token / 过期 Token 返回 401 | test_create_todo_unauthenticated | ✅ |
> | 创建 | AC-9 ~ 11 | 创建 Todo 正常 / 异常 | test_create_todo_* | ✅ |
> | 查改删 | AC-12 ~ 19 | 查看 / 修改 / 删除 + 授权（跨用户返回 404） | test_get/update/delete_todo_* | ✅ |
> | 筛选分页 | AC-20 ~ 25 | 状态筛选 / 分页 / 空列表 / 排序 | test_filter/pagination_* | ✅ |
> | 边界 | AC-26 ~ 29 | 纯空格标题 / 分页边界 / 重复删除 | test_create/delete_todo_* | ✅ |
> | 筛选排序 | AC-30 ~ 32 | 时间范围 / 自定义排序 | test_filter/sort_* | ✅ |
> | 刷新 | AC-33 ~ 35 | Refresh Token 相关 | test_refresh_token_* | ✅ |
>
> **总结**：35 条验收标准全部通过 ✅

### 审

验收对照是最后一道关卡。审查时注意：

- **所有 AC 都有对应测试吗？** 如果有 AC 没有对应测试，说明 Tasks 阶段就遗漏了。现在还能补，比验收时才发现好。
- **测试真的通过了吗？** 不要只看 AI 打的 ✅。实际跑一遍 `pytest`，用测试结果来确认。
- **有没有 AC 被映射到了错误的测试？** 比如 AC-7（未携带 Token 返回 401）对应的是 `test_create_todo_unauthenticated`——但这只测了创建接口。其他受保护接口（查看、修改、删除）也需要测试 401 场景吗？严格来说需要，但在 Todo 应用中，只要认证中间件正确工作，一个接口测了就够了。
- **整体覆盖率如何？** Constitution 要求 ≥ 80%。跑一下覆盖率报告，确认达标。

```
pytest --cov=src --cov-report=term-missing
```

如果覆盖率不够，找到未覆盖的代码行，补充测试。

---

## 7.12 本章小结

这一章我们走完了 SDD 的最后两个阶段——Tasks 和 Implement。回顾一下整个过程：

**Tasks 阶段**（把方案变成计划）：

1. **从 Plan 到任务列表**：把 `plan.md` 的设计模块拆解为可独立完成、可独立验证的任务
2. **拆解策略**：简单 CRUD 用自顶向下，复杂业务逻辑用自底向上
3. **任务排序与依赖**：基础设施 → 认证 → Todo CRUD → 增强功能
4. **任务审查**：对照 spec 逐条检查，确保没有遗漏

**Implement 阶段**（把计划变成代码）：

5. **实现循环**：每个任务走 Test → Implement → Review 三步
6. **演示任务 A（创建 Todo）**：标准 CRUD 的完整演示
7. **演示任务 B（用户注册）**：认证相关的完整演示，重点引用 Plan 安全设计
8. **其余任务概述**：按同样模式推进
9. **验收对照**：对照 spec 的 35 条验收标准逐条确认

**关键心得**：

- **任务粒度要适中**。太粗了不好管理、不好测试；太细了效率低、上下文频繁切换。一个任务对应一个可独立验证的功能点——这是比较好的粒度。
- **Test 先行不是教条**。先写测试的好处是：让 AI 明确知道"什么算完成"。如果你让 AI 先写代码再写测试，它容易"凑测试来通过代码"，而不是"写代码来通过测试"。测试先行保证了测试的独立性。
- **Review 不是可选的**。测试通过只说明功能对了，不能说明代码好。安全隐患、架构违规、代码异味——这些都需要 Review 来发现。特别是认证相关的代码，Review 是必须的。
- **验收对照是闭环**。从 Specify 阶段写下验收标准，到 Implement 阶段逐条验证——这个闭环保证了"做了什么"和"要求做什么"一致。没有闭环，SDD 就只是"写了一堆文档但没人检查"。

### 产出物清单

| 文件 | 说明 |
|------|------|
| `.spec/features/001-todo-core/tasks.md` | 任务清单（13 个任务，含依赖关系和排序） |
| `tests/auth/test_register.py` | 用户注册测试 |
| `tests/auth/test_login.py` | 用户登录测试 |
| `tests/auth/test_middleware.py` | 认证中间件测试 |
| `tests/todo/test_create_todo.py` | 创建 Todo 测试 |
| `tests/todo/test_get_todo.py` | 查看 Todo 测试 |
| `tests/todo/test_update_todo.py` | 修改 Todo 测试 |
| `tests/todo/test_delete_todo.py` | 删除 Todo 测试 |
| `tests/todo/test_filter_sort.py` | 筛选排序测试 |
| `src/auth/` | 认证模块代码（注册、登录、JWT、密码哈希） |
| `src/todo/` | Todo 模块代码（CRUD、筛选、分页） |
| `src/shared/` | 共享模块（数据库、认证依赖、异常） |

### 产出物存放

```
.spec/
├── constitution.md                  # 第 4 章产出
└── features/
    └── 001-todo-core/
        ├── spec.md                  # 第 5 章产出
        ├── plan.md                  # 第 6 章产出
        └── tasks.md                 # 本章产出 ← 我们在这里
src/                                 # 本章产出：功能代码
tests/                               # 本章产出：测试代码
```

至此，Feature 1 的 SDD 全流程走完了：Constitution → Specify → Clarify → Plan → Tasks → Implement。

下一章，我们用一个新的 Feature（Todo 标签功能）做一次快速迭代，体验 Constitution 已有情况下的真实工作节奏。
