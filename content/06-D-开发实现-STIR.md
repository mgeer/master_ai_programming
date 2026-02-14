# 第六章：D - Develop（开发实现 - STIR 四步法）

## 6.1 引言

**本章目标**：学会用 STIR 四步法进行测试驱动开发。

上一章完成了 BASER 架构设计，产出了业务架构、技术架构、安全方案、部署配置和开发规范五份文档。现在终于可以写代码了。

但"写代码"不是打开编辑器就让 AI 开干。如果你直接说"帮我实现整个 Todo API"，AI 会一次性输出几百行代码——没有测试、没有审查、质量不可控。

**STIR 四步法的核心思路是：把大任务拆成小任务，每个小任务都走"测试 → 实现 → 审查"的循环。**

就像盖房子——不是一次性浇筑整栋楼，而是一层一层往上盖，每一层都验收合格再盖下一层。

---

## 6.2 STIR 四步法概览

```mermaid
graph LR
    S["S - Split<br/>任务拆解"] --> T["T - Test<br/>行为测试先行"]
    T --> I["I - Implement<br/>实现"]
    I --> R["R - Review<br/>代码审查"]
    R -->|"下一个任务"| T
```

**迭代模式**：先 S 拆解出所有任务，然后对**每个任务**依次走 T → I → R 循环。

| 步骤 | 做什么 | 产出 |
|------|--------|------|
| S | 把功能拆解成可独立开发的小任务 | 任务清单 + 开发顺序 |
| T | 为当前任务写测试用例和测试代码 | 测试代码（此时测试会失败） |
| I | 实现功能代码，让测试通过 | 功能代码 |
| R | 审查代码质量，重构优化 | 高质量的可测试代码 |

---

## 6.3 S - Split（任务拆解）

### 拆解依据

任务拆解不是凭感觉，而是**基于架构设计的产出物**：

- **B（业务架构）** 的领域对象 → 告诉你"有哪些功能模块"
- **A（技术架构）** 的模块划分和接口清单 → 告诉你"每个模块要实现什么"

### 两种拆解策略

| 策略 | 适用场景 | 开发顺序 |
|------|---------|---------|
| **自顶向下** | 简单 CRUD | API 路由 → Service 业务逻辑 → 数据库操作 |
| **自底向上** | 复杂业务逻辑 | 领域模型 → 领域服务 → 应用服务 → API |

**如何判断用哪种？** 简单标准：如果业务逻辑就是"接收数据、存数据库、返回结果"，用自顶向下；如果业务逻辑有复杂的状态流转、计算规则、多步骤流程，用自底向上。

比如电商的订单模块，价格计算涉及优惠券、满减、会员折扣等复杂规则，就适合自底向上——先实现价格计算的领域逻辑，再包装成订单服务和 API。

Todo API 的 CRUD 操作属于简单场景，用自顶向下。

### Todo API 示例

**提示词：**

```
基于 @docs/tech-architecture.md，帮我将开发工作拆解为独立的小任务，
确定开发顺序和依赖关系。基础设施（数据库连接、项目配置）作为第一个任务。
```

**AI 产出的任务清单：**

| 序号 | 任务 | 依赖 | 说明 |
|------|------|------|------|
| 1 | 项目基础设施 | 无 | 数据库连接、配置管理、项目结构搭建 |
| 2 | 用户注册 | 任务 1 | POST /api/auth/register |
| 3 | 用户登录 | 任务 2 | POST /api/auth/login（JWT 生成） |
| 4 | 认证中间件 | 任务 3 | JWT Token 验证的依赖注入 |
| 5 | 创建待办事项 | 任务 4 | POST /api/todos |
| 6 | 查询待办事项列表 | 任务 5 | GET /api/todos（分页、筛选） |
| 7 | 查询单个待办事项 | 任务 5 | GET /api/todos/{id} |
| 8 | 修改待办事项 | 任务 5 | PUT /api/todos/{id} |
| 9 | 删除待办事项 | 任务 5 | DELETE /api/todos/{id}（软删除） |
| 10 | 标记完成/取消完成 | 任务 5 | PATCH /api/todos/{id}/complete |

**开发顺序**：1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 10

拆解完成后，每个任务都足够小（一个接口或一个页面），可以在一次对话中完成 T → I → R 循环。

> **前端开发同理**：如果项目包含前端（如 Vue），拆解和开发流程完全一致——按页面/组件拆解任务，每个任务走 T→I→R 循环（T 阶段使用 Vitest + Vue Test Utils 编写组件测试）。本章以后端 API 为例演示，读者可自行用相同的 STIR 流程完成前端部分。

将任务清单保存为 `docs/tasks.md`，后续开发过程中可以用它跟踪进度。

> **任务清单不需要一次拆分到位。** 先拆出当前能看清的任务，开始 TIR 循环。开发过程中发现新需求或遗漏的功能，单独创建一个任务文件（如 `docs/task-batch-delete.md`），描述清楚上下文和要求，然后对它走 TIR。`tasks.md` 是初始计划，不是唯一的任务来源。

### 启动项目与基础任务（任务 1-4）

任务拆解完成后，先搭建项目基础设施（任务 1），再完成用户认证（任务 2-4）作为前置条件。完整的 T→I→R 循环在下面几节以任务 5 为例展开。

#### 任务 1：项目基础设施

这是整个项目的"地基"——创建项目骨架、配置数据库连接、搭建测试基础设施。这个任务比较特殊，主要是搭建环境，后续任务才开始标准的 T → I → R 循环。

**提示词：**

```
基于 @docs/tech-architecture.md，帮我初始化项目：
项目骨架、配置管理、数据库连接、应用入口、测试基础设施。
测试使用 SQLite 测试数据库，和生产环境隔离，每个测试前后自动建表和清表。
```

AI 会根据技术架构文档自动判断需要哪些文件、用什么目录结构。你不需要列出具体的文件清单——**告诉 AI 目标，让它决定怎么组织**。只提 AI 可能猜错的关键约束（比如测试用 SQLite 隔离）。

**关键产出：tests/conftest.py**

后续所有任务的测试都依赖这个文件，它是最重要的基础设施。让 AI 生成时，确保它包含以下三个核心 fixture：

1. **测试数据库隔离**（`setup_database`）：用 SQLite 替代 PostgreSQL，每个测试自动建表和清表，互不干扰
2. **异步测试客户端**（`client`）：提供 httpx 的 `AsyncClient`，可以像真实客户端一样向 FastAPI 发请求
3. **认证 Token**（`auth_token`）：自动注册并登录测试用户，返回 JWT Token——后续测试直接用它访问需要认证的接口

> 注意：`auth_token` fixture 依赖任务 2-3（注册和登录接口）。在完成任务 3 之后，需要认证的测试才能跑通。具体的 conftest.py 代码不需要你自己写——把以上三点作为 Constraint 告诉 AI，它会生成完整的测试基础设施。

#### 任务 2-4：用户认证

任务 2（用户注册）、3（用户登录）、4（认证中间件）是核心业务开发的前置条件。每个任务独立走 T→I→R 循环——先写测试、再实现、最后审查——和下面任务 5 的详细演示完全一致。看完任务 5 的演示后，你会知道怎么写这三个任务的提示词，格式一样，只是接口和测试场景不同。

**完成任务 1-4 后，项目已经具备**：
- 可运行的项目骨架和配置（任务 1）
- 用户注册和登录接口（任务 2-3）
- JWT 认证中间件（任务 4）
- 完整的测试基础设施（conftest.py）

现在可以开始实现核心业务功能了。下面以**任务 5（创建待办事项）**为例，完整演示一次 STIR 的 T → I → R 循环。

---

## 6.4 T - Test（行为测试先行）

### 为什么先写测试？

很多人觉得"先写测试"是浪费时间。但在 AI 编程中，测试先行有三个独特的优势：

1. **测试是最精确的需求描述**——比自然语言更明确，AI 看到测试就知道你要什么
2. **测试是验收标准的代码化**——写完代码跑一下测试就知道对不对，不用人工逐项检查
3. **AI 写测试比人写测试快得多**——让 AI 做它擅长的事

### 测什么？测行为，不测实现

测试先行不是要你给每个函数都写测试。AI 编程中，**重点测试系统的外部行为**——用户调一个接口，返回的结果对不对。

为什么？因为 AI 最常犯的错不在单个函数内部，而在**模块之间的衔接**——接口理解偏差、数据格式不一致、业务规则遗漏。这些错误只有在模块协作时才暴露，所以测试要对准这个层次。

具体来说：

- **重点写：API / 集成测试**——验证系统行为和业务规则。简单系统测"能不能用"（增删改查跑通），业务复杂的系统还要测"规则对不对"（如"订单超过 1000 元需审批"、"普通用户不能删除他人数据"）。本章的 Todo API 示例就是这个层次
- **选择性写：复杂业务逻辑的单元测试**——当业务规则组合太多、通过 API 难以穷举覆盖时（如复杂定价引擎、多状态流转），才需要单独写单元测试细测
- **不用专门写：简单内部函数的单元测试**——API 测试已经间接覆盖，单独测反而增加维护成本

还有一个实用好处：API 测试验证的是**行为**，不绑定内部实现细节。AI 编程中重构很频繁，API 测试不会因为内部重构而失效，维护成本低得多。

> **一句话记住**：测试写在系统边界上——验证"系统做了什么"，而不是"代码怎么写的"。

### 测试即文档

好的测试代码本身就是一份**活的需求文档**。任何人读你的测试，都应该能理解这个功能"做了什么"和"不允许什么"。

所以对测试代码的**可读性要求甚至高于功能代码**：

| 要求 | 说明 | 示例 |
|------|------|------|
| **函数命名要讲故事** | 名字本身就能看出测试意图 | `test_create_todo_with_past_due_date_should_fail` |
| **docstring 描述预期** | 一句话说明"在什么条件下，预期什么结果" | `"""截止日期为过去的日期时，应返回 422"""` |
| **测试数据要有业务含义** | 不要用 "aaa"、"test123"，用真实的业务数据 | `title="完成项目报告"` 而不是 `title="test"` |
| **断言要具体** | 不只是检查状态码，还要验证返回值的关键字段 | `assert data["priority"] == "high"` |
| **一个测试只验一件事** | 不要在一个函数里塞多个场景 | 正常创建和空标题是两个测试函数 |

**反面示例：**

```python
# ❌ 看不出测试意图，数据无业务含义
async def test_1(client, token):
    r = await client.post("/api/todos", json={"title": "aaa"}, headers={"Authorization": f"Bearer {token}"})
    assert r.status_code == 201
```

**正面示例：**

```python
# ✅ 命名清晰、数据有含义、断言具体
async def test_create_todo_with_only_title_should_use_default_priority(
    client: AsyncClient, auth_token: str
):
    """只提交 title 时，priority 应默认为 medium"""
    response = await client.post(
        "/api/todos",
        json={"title": "整理会议纪要"},
        headers={"Authorization": f"Bearer {auth_token}"},
    )
    assert response.status_code == 201
    assert response.json()["priority"] == "medium"
```

一个月后你回来看这段测试，不需要看功能代码，就能知道"只传 title 时 priority 默认是 medium"。**这就是测试即文档的力量。**

### 测试用例矩阵

为每个任务设计测试用例时，覆盖三类场景：

| 场景类型 | 说明 | 示例 |
|---------|------|------|
| **正常场景** | 正确的输入，预期的输出 | 提交完整数据，成功创建 Todo |
| **边界条件** | 输入处于临界值 | title 长度恰好 100 个字符 |
| **异常场景** | 错误的输入或异常情况 | title 为空、未登录、操作他人数据 |

### Todo API 示例：为"创建待办事项"写测试

**提示词：**

```
为 POST /api/todos（创建待办事项）编写 pytest 测试。
需求规则见 @docs/requirements.md，开发规范见规则文件。
先列出测试用例矩阵（正常、边界、异常），再编写测试代码。
测试即文档：命名要能看出测试意图，数据要有业务含义。
```

**AI 产出示例：**

**测试用例矩阵：**

| 类型 | 测试用例 | 预期结果 |
|------|---------|---------|
| 正常 | 提交完整数据（title + description + priority + due_date） | 201，返回完整 Todo |
| 正常 | 只提交 title（其他用默认值） | 201，priority 为 medium |
| 边界 | title 长度恰好 1 个字符 | 201 |
| 边界 | title 长度恰好 100 个字符 | 201 |
| 异常 | title 为空字符串 | 422 |
| 异常 | title 超过 100 个字符 | 422 |
| 异常 | priority 为无效值（如 "urgent"） | 422 |
| 异常 | due_date 是过去的日期 | 422 |
| 异常 | 不带 Token 请求 | 401 |

**测试代码（关键部分）：**

```python
import pytest
from datetime import date, timedelta
from httpx import AsyncClient


@pytest.mark.asyncio
async def test_create_todo_with_full_data(client: AsyncClient, auth_token: str):
    """提交完整数据，成功创建待办事项"""
    response = await client.post(
        "/api/todos",
        json={
            "title": "完成项目报告",
            "description": "包含 Q1 数据分析",
            "priority": "high",
            "due_date": str(date.today() + timedelta(days=7)),
        },
        headers={"Authorization": f"Bearer {auth_token}"},
    )
    assert response.status_code == 201
    data = response.json()
    assert data["title"] == "完成项目报告"
    assert data["priority"] == "high"
    assert data["status"] == "pending"
    assert "id" in data


@pytest.mark.asyncio
async def test_create_todo_with_only_title(client: AsyncClient, auth_token: str):
    """只提交 title，其他字段使用默认值"""
    response = await client.post(
        "/api/todos",
        json={"title": "简单任务"},
        headers={"Authorization": f"Bearer {auth_token}"},
    )
    assert response.status_code == 201
    data = response.json()
    assert data["priority"] == "medium"
    assert data["description"] is None


@pytest.mark.asyncio
async def test_create_todo_with_empty_title(client: AsyncClient, auth_token: str):
    """title 为空，应返回 422"""
    response = await client.post(
        "/api/todos",
        json={"title": ""},
        headers={"Authorization": f"Bearer {auth_token}"},
    )
    assert response.status_code == 422


@pytest.mark.asyncio
async def test_create_todo_without_auth(client: AsyncClient):
    """不带 Token，应返回 401"""
    response = await client.post(
        "/api/todos",
        json={"title": "未认证请求"},
    )
    assert response.status_code == 401
```

> **关于测试中的 422 状态码**：你可能注意到测试用例矩阵中，校验失败的预期状态码是 422 而非需求文档中的 400（第五章规则文件也约定了"业务错误使用 400"）。这是因为 FastAPI 的 Pydantic 校验失败默认返回 422。这种"规范与框架默认行为"的冲突是**有意保留**的——在第七章（E 阶段），你会看到整体评估如何发现并处理这类问题。

此时运行测试，**所有测试都会失败**——因为还没有实现功能代码。这是正常的，接下来进入 I 阶段。

---

## 6.5 I - Implement（实现）

### 关键：把上下文给够

让 AI 实现功能代码时，最重要的是**把上下文给够**。需要给 AI 的信息：

1. **测试代码**（T 阶段产出）—— AI 看到测试就知道你要什么
2. **架构设计文档**（O 阶段产出）—— 目录结构、分层规范
3. **开发规范**（规则文件）—— 编码风格、错误处理规范

在 AI IDE 中，你可以通过文件引用功能直接引用这些文件，不需要手动粘贴。

### Todo API 示例：实现"创建待办事项"

**提示词：**

```
实现"创建待办事项"功能，让 @tests/test_create_todo.py 的所有测试通过。
参考 @docs/tech-architecture.md 的目录结构和规则文件的开发规范。
接口层（routers）只负责路由，业务逻辑放应用层（application）。
```

> 注：以上提示词使用 AI IDE 的文件引用语法（如 Cursor 的 `@`），可以直接引用项目中的文件。如果你的工具不支持此语法，手动粘贴相关文件内容即可。AI 会根据测试代码和架构文档自动判断需要创建哪些文件，不需要你逐一列出。

**AI 产出示例（关键部分）：**

> 以下展示 AI 生成的三个文件。你不需要逐行理解每一行代码——重点看 AI 如何按照架构设计的四层结构（interfaces / application / domain / infrastructure）组织代码。

**app/interfaces/schemas/todo.py：**

```python
from datetime import date
from enum import Enum
from typing import Optional

from pydantic import BaseModel, Field, field_validator


class Priority(str, Enum):
    high = "high"
    medium = "medium"
    low = "low"


class CreateTodoRequest(BaseModel):
    """创建待办事项的请求模型"""
    title: str = Field(..., min_length=1, max_length=100)
    description: Optional[str] = Field(None, max_length=500)
    priority: Priority = Field(default=Priority.medium)
    due_date: Optional[date] = None

    @field_validator("due_date")
    @classmethod
    def due_date_must_be_future(cls, v: Optional[date]) -> Optional[date]:
        if v is not None and v <= date.today():
            raise ValueError("截止日期必须是未来的日期")
        return v


class TodoResponse(BaseModel):
    """待办事项的响应模型"""
    id: int
    title: str
    description: Optional[str]
    priority: str
    status: str
    due_date: Optional[date]
    created_at: str
```

**app/application/todo_service.py：**

```python
from sqlalchemy.orm import Session

from app.domain.todo import Todo
from app.interfaces.schemas.todo import CreateTodoRequest


def create_todo(db: Session, request: CreateTodoRequest, user_id: int) -> Todo:
    """创建待办事项"""
    todo = Todo(
        title=request.title,
        description=request.description,
        priority=request.priority.value,
        due_date=request.due_date,
        user_id=user_id,
    )
    db.add(todo)
    db.commit()
    db.refresh(todo)
    return todo
```

**app/interfaces/routers/todos.py：**

```python
from fastapi import APIRouter, Depends, status
from sqlalchemy.orm import Session

from app.infrastructure.database import get_db
from app.infrastructure.auth import get_current_user
from app.interfaces.schemas.todo import CreateTodoRequest, TodoResponse
from app.application.todo_service import create_todo

router = APIRouter(prefix="/api/todos", tags=["todos"])


@router.post("", status_code=status.HTTP_201_CREATED, response_model=TodoResponse)
async def create(
    request: CreateTodoRequest,
    db: Session = Depends(get_db),
    current_user_id: int = Depends(get_current_user),
) -> TodoResponse:
    """创建待办事项"""
    todo = create_todo(db, request, current_user_id)
    return TodoResponse.model_validate(todo, from_attributes=True)
```

> **关于同步与异步**：你可能注意到测试用了 `AsyncClient`（异步），而 Service 层用了同步的 `Session`。这在 FastAPI 中完全正常——第五章选择 FastAPI 是因为它**支持**异步，但这不意味着所有层都必须用异步。FastAPI 的优势在于它同时兼容同步和异步代码，会自动将同步代码放到线程池执行。对于大多数项目，同步 SQLAlchemy 更简单直观。如果未来有高并发需求，可以切换到 SQLAlchemy 的异步模式。

实现完成后，运行测试：

```bash
pytest tests/test_create_todo.py -v
```

**所有测试通过** → 进入 R 阶段。

如果有测试失败，把错误信息反馈给 AI：

```
以下测试失败了，请分析原因并修复代码：

[粘贴 pytest 的错误输出]
```

**关键技巧**：把完整的错误输出（包括 traceback 和断言失败信息）给 AI，而不是只说"测试没过"。AI 需要看到具体的错误信息才能精准定位问题。通常 1-2 轮反馈就能修复。

---

## 6.6 R - Review（代码审查）

### 时机和定位

- **时机**：每个任务的 T → I 完成后立即进行
- **AI 角色**：Code Reviewer
- **目标**：检查代码质量，发现可以改进的地方

注意和 E 阶段（整体评估）的区别：
- **R 是模块级审查**：只看当前任务的代码（如"创建待办事项"）
- **E 是全局评估**：所有任务完成后，从功能、性能、安全等维度整体评估

### Todo API 示例：审查"创建待办事项"

**提示词：**

```
作为 Code Reviewer，审查"创建待办事项"功能的代码：
@app/interfaces/schemas/todo.py @app/application/todo_service.py @app/interfaces/routers/todos.py

审查维度：规范合规、分层合理性、错误处理、代码坏味道。
```

**AI 审查意见示例：**

> **问题 1：应用层缺少异常处理**
> `create_todo` 函数中 `db.commit()` 可能抛出数据库异常（如唯一约束冲突），但没有 try-except。建议捕获 `SQLAlchemyError` 并抛出明确的业务异常。
>
> **问题 2：TodoResponse 的 created_at 类型不够精确**
> `created_at: str` 应该改为 `created_at: datetime`，让 Pydantic 自动序列化，避免手动格式转换。
>
> **问题 3：建议补充**
> 应用层的 `create_todo` 函数建议增加日志记录（符合规则文件中"关键业务操作记录 INFO 日志"的要求）。

**根据审查意见修改后，重新运行测试确保仍然通过：**

```bash
pytest tests/test_create_todo.py -v
```

测试通过 → 这个任务完成 → 进入下一个任务的 T → I → R 循环。

---

## 6.7 完整循环演示

以下是"创建待办事项"功能完整的 STIR 循环：

```mermaid
graph TD
    S["当前任务：POST /api/todos"] -.-> T1["T: 设计测试用例矩阵（9 个用例）"]
    T1 --> T2["T: 生成 pytest 测试代码"]
    T2 --> T3["运行测试 → 全部失败（正常）"]
    T3 --> I1["I: 实现 schemas + application + routers"]
    I1 --> I2["运行测试 → 全部通过"]
    I2 --> R1["R: Code Reviewer 审查代码"]
    R1 --> R2["根据审查意见修改"]
    R2 --> R3["运行测试 → 仍然通过"]
    R3 --> DONE["✅ 任务完成，进入下一个任务"]
```

**整个过程中你做了什么？**

1. **决策**：确定测试用例要覆盖哪些场景
2. **判断**：审查意见中哪些要采纳，哪些不需要
3. **验收**：确认测试通过、代码质量达标

**AI 做了什么？**

1. **生成**：测试代码、功能代码
2. **审查**：发现代码问题
3. **修改**：根据你的指令修改代码

**人做决策，AI 做执行——STIR 的每一步都是这个模式。**

### 任务切换的上下文管理

完成一个任务的 T→I→R 循环后，进入下一个任务时，建议**开一个新对话**。原因是：上一个任务的对话中积累了大量测试代码、审查意见和修改过程，这些信息对新任务是噪音，会干扰 AI 的判断。

新对话的开头，通过 `@` 引用为 AI 提供必要上下文即可：

```
我在继续开发 Todo API，当前进度见 @docs/tasks.md。
接下来实现任务 6：GET /api/todos（查询待办事项列表）。
相关文件：@docs/requirements.md @docs/tech-architecture.md
已实现的代码：@app/application/todo_service.py @app/interfaces/routers/todos.py
```

不需要把上一个任务的对话内容带过来——**代码在文件里，上下文在文档里**，这就是第五章强调文档产出的价值。

---

## 6.8 常见误区

### 误区 1：不拆解，直接让 AI 实现全部

```
❌ "帮我实现整个 Todo API 的所有接口"
```

AI 会一次性输出几百行代码，没有测试、没有审查、质量不可控。一旦有问题，很难定位是哪个接口出了错。

**拆解成小任务，每个任务单独实现，问题容易定位，质量可以控制。**

### 误区 2：跳过测试，直接实现

"测试以后再补吧。" ——然后就永远不补了。

在 AI 编程中跳过测试更危险：AI 生成的代码你不一定完全理解，没有测试就没有安全网。如果 AI 引入了一个隐藏 bug，你可能几周后才发现。

**测试先行不是浪费时间，而是最高效的质量保障。** AI 写测试很快，没有理由跳过。

### 误区 3：Review 只走形式

```
❌ "审查一下这段代码"
```

然后 AI 说"代码整体不错"，你就跳过了。

**有效的 Review 需要明确审查维度**（参考第二章的角色扮演技巧），并且要**认真对待 AI 的审查意见**——它提出的问题可能正是你忽略的。

---

## 6.9 练习题与本章小结

### 练习

**前置条件**：请先按照本章内容，完成任务 1-5 的开发。任务 1 可参考"启动项目与基础任务"中的提示词，任务 2-4 按照任务 5 演示的 TIR 流程完成（接口和测试场景不同，提示词格式一致），任务 5 的完整 STIR 循环已在本章详细演示。

在此基础上，选择**任务 6（查询待办事项列表）**，独立完成一次 STIR 循环：

1. **T**：为 GET /api/todos 设计测试用例矩阵，涵盖分页、筛选、权限校验，生成测试代码
2. **I**：让 AI 实现功能代码，确保测试通过
3. **R**：让 AI 审查代码，根据意见修改，确认测试仍然通过

### 本章小结

**核心要点回顾：**

- **S（Split）**：大任务拆成小任务，基于架构设计的产出物来拆解
- **T（Test）**：先写测试——测试是最精确的需求描述，也是质量的安全网
- **I（Implement）**：把测试代码和架构文档一起给 AI，上下文越充分输出越好
- **R（Review）**：每个任务完成后立即审查，指定具体维度，认真对待审查意见
- **迭代模式**：S 拆解一次，T → I → R 循环 N 次，直到所有任务完成

**下一章预告：**

所有任务都开发完成后，进入 CODE 的最后一步——**E - Evaluate（整体评估）**，从功能、质量、性能、安全四个维度做全局检查。
