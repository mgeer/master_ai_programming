# 第六章：Plan — 架构设计

## 6.1 引言

**本章目标**：基于功能规范进行领域建模和技术设计，产出完整的架构实现方案。

如果 `spec.md` 是我们要去的目的地，Plan 则是详细的行程攻略。Constitution 提供了底层规矩（DDD 架构、FastAPI、bcrypt 哈希），Plan 则根据这些规矩为当前 Feature 设计数据结构、API 契约和业务逻辑。

Plan 阶段产出：领域建模、数据模型、API 契约设计、目录结构和安全实现方案。

## 6.2 领域建模

### 核心思路

领域建模从需求提取核心概念，梳理业务流程与规则，为后续代码设计提供业务地图。它回答了对象有哪些属性及关系。

### 实操演示

> **你**：基于 @spec.md 和 @constitution.md，识别核心领域对象。列出属性和关系，使用 Mermaid 类图表示。

> **你**：补充细节：密码字段名为 `hashed_password`，`Todo` 增加 `deleted_at` 字段支持软删除。

> AI 产出示例：
>
> ````mermaid
> classDiagram
>     class User {
>         +UUID id
>         +String email
>         +String hashed_password
>         +DateTime created_at
>         +DateTime updated_at
>     }
> 
>     class Todo {
>         +UUID id
>         +String title
>         +String description
>         +TodoStatus status
>         +UUID user_id
>         +DateTime created_at
>         +DateTime updated_at
>         +DateTime deleted_at
>     }
> 
>     class TodoStatus {
>         <<enumeration>>
>         pending
>         completed
>     }
> 
>     User "1" --> "*" Todo : owns
>     Todo --> TodoStatus : has
> ````

### 审查要点

检查对象是否全部来源于 `spec.md`，主键是否为 UUID，关系是否正确且无过度设计。

## 6.3 业务流程与规则梳理

### 核心思路

提炼核心操作的业务流程，并将所有散落的数据校验和权限约束集中为明确的业务规则表，供后续 AI 编码参考。

### 实操演示

> **你**：梳理 Feature 1 的认证流程和 Todo 操作流程，用 Mermaid 流程图表示。提炼业务规则并分类。

> AI 产出示例：
>
> ````mermaid
> flowchart TD
>     A[已认证用户] --> B[操作 Todo]
>     B --> C{是否是自己的 Todo?}
>     C -- 否 --> D[统一返回 404 Not Found]
>     C -- 是 --> E[执行修改/删除]
> ````

### 审查要点

确保越权访问严格返回 404（对齐 AC-4 等验收标准），数据验证失败严格返回 422 Unprocessable Entity（对齐 AC-2/3 等验证约束）。密码存储方案留到 6.7 安全实现方案中检查。

## 6.4 数据模型设计

### 核心思路

将领域模型翻译为数据库表结构。明确字段类型、唯一约束、外键关联和索引设计。

### 实操演示

> **你**：基于领域模型和 @constitution.md 的编码规范，设计数据库表结构。用 Mermaid ER 图表示。

> AI 产出示例：
>
> ````mermaid
> erDiagram
>     users ||--o{ todos : "owns"
>     users {
>         uuid id PK
>         varchar email UK
>         varchar hashed_password
>         timestamp created_at
>         timestamp updated_at
>     }
>     todos {
>         uuid id PK
>         varchar title
>         text description
>         varchar status
>         uuid user_id FK
>         timestamp created_at
>         timestamp updated_at
>         timestamp deleted_at
>     }
> ````

### 审查要点

确认主键为 UUID，表名为 `users` 和 `todos`，并建立了合适的索引（如 `user_id`）。

## 6.5 API 契约设计

### 核心思路

API 契约定义了前后端的交互标准。必须严格规定路径格式、请求/响应的 JSON 结构及错误码。

### 实操演示

> **你**：基于规范设计 RESTful API 契约。路径前缀 `/api/v1`，使用 kebab-case。严格对应验收标准的 HTTP 状态码。

> AI 产出示例：
>
> ````markdown
> ### POST /api/v1/auth/register
> - 成功（201）：返回 User 信息与 Token。
> - 数据验证失败：返回 422 Unprocessable Entity。
> 
> ### GET /api/v1/todos/{id}
> - 成功（200）：返回 Todo 详情。
> - 越权或不存在：一律返回 404 Not Found。
> ````

### 审查要点

检查所有数据验证错误（缺失字段、超长等）是否均为 422。检查越权是否均为 404。

## 6.6 目录结构设计

### 核心思路

将 DDD 四层架构落地为具体的文件结构，明确各类代码的存放位置，防止 AI 跨层调用。

### 实操演示

> **你**：设计具体的项目目录结构，精确到文件级别，体现 DDD 分层。

> AI 产出示例：
>
> ````text
> src/
> ├── auth/
> │   ├── api/routes.py
> │   ├── application/service.py
> │   ├── domain/entities.py
> │   └── infrastructure/models.py
> ├── todo/
> │   ├── api/
> │   ├── application/
> │   ├── domain/
> │   └── infrastructure/
> └── main.py
> ````

### 审查要点

检查 `domain` 层内是否存在对基础设施的依赖，确保层级结构清晰。

## 6.7 安全实现方案

### 核心思路

将 Constitution 中的安全策略具体化。包括 JWT 的签名流程、bcrypt 密码哈希的执行步骤，以及查询级过滤的授权策略。

### 实操演示

> **你**：设计完整安全方案。明确 JWT 鉴权流程。授权采用查询级过滤，越权必须返回 404。密码必须使用 bcrypt（cost factor 12）。

> **小白锦囊**：
> 觉得“查询级过滤”、“bcrypt”这些词太高级？别担心，你可以直接贴需求，让 AI 帮你想方案：
> *"我是个后端新手。按照规范，用户只能看自己的 Todo，而且密码需要安全保存。请帮我设计一套最安全且好实现的代码层安全方案（比如密码怎么存、如何避免查到别人的数据），并向我通俗解释为什么这么做。"*

> AI 产出示例：
>
> ````markdown
> ### 认证与密码处理
> - 注册时提取密码，执行 bcrypt 哈希（cost factor 12）后存入数据库。
> - 鉴权中间件验证 JWT，失败返回 401。
> 
> ### 授权实现伪代码
> ```python
> def get_todo(todo_id: UUID, current_user_id: UUID):
>     stmt = select(Todo).where(
>         Todo.id == todo_id,
>         Todo.user_id == current_user_id,
>         Todo.deleted_at.is_(None)
>     )
>     todo = db.execute(stmt).scalar_one_or_none()
>     if not todo:
>         raise NotFoundError() # 统一返回 404
>     return todo
> ```
> ````

### 审查要点

确认密码处理步骤严格采用了 bcrypt 哈希算法，且查询级过滤有效防止了越权并天然返回了 404 错误。

## 6.8 整体方案审查

### 核心思路

在结束 Plan 阶段前，进行最后的全盘审查，确保没有任何过度设计和规范偏离。

### 实操演示

> **你**：审查技术方案。对照 @spec.md 检查 AC 编号覆盖情况。对照 @constitution.md 确认技术细节无冲突。

> AI 产出示例：
>
> ````markdown
> - AC-1 ~ AC-8（认证）：契约已覆盖。
> - AC-9 ~ AC-19（CRUD）：包含 404 越权和 422 验证，契约已覆盖。
> - AC-20 ~ AC-25（分页）：包含 422 超限，契约已覆盖。
> - AC-26 ~ AC-29（边界）：数据验证已覆盖。
> - AC-30 ~ AC-32（时间范围）：API 已覆盖。
> - AC-33 ~ AC-35（Token刷新）：认证方案已覆盖。
> ````

### 审查要点

根据审查报告修正偏差，确认最终技术方案。

## 6.9 本章小结

**核心要点回顾**：
- 领域建模与流程图必须全部使用 Mermaid 表达。
- 业务设计严格遵守主键 UUID、密码 bcrypt 哈希。
- 接口与规则全面遵守数据验证 422 和越权访问 404。
- API 与数据库设计完全符合 Constitution 规范。

### 产出物清单
- `.spec/features/001-todo-core/plan.md` 包含图表与实现契约的完整文档。

第 7 章，我们将进入 Tasks & Implement 阶段，执行任务拆解与编码。
