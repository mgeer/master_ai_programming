# 第六章：Plan — 架构设计

## 6.1 引言

**本章目标**：基于功能规范进行领域建模和技术设计，产出完整的架构实现方案。

如果 `spec.md` 是我们要去的目的地，Plan 则是详细的行程攻略。Constitution 提供了底层规矩，Plan 则根据这些规矩为当前 Feature 设计具体的数据结构、API 契约和业务逻辑。

> **注意**：项目的目录结构已在 Constitution 中确定（DDD 四层架构），属于项目级规范，不需要在每个 Feature 的 Plan 中重新设计。

## 6.2 Plan 的核心产出

一份完整的 `plan.md` 应包含以下内容：

| 产出 | 作用 | 关键审查点 |
|------|------|-----------|
| **领域模型** | 从需求提取核心实体、属性和关系 | 对象是否全部来源于 `spec.md`，关系是否正确且无过度设计 |
| **数据模型** | 将领域模型翻译为数据库表结构 | 字段类型、唯一约束、外键和索引是否合理 |
| **业务流程与规则** | 梳理核心操作流程，集中数据校验和权限约束 | 越权和验证的行为是否对齐验收标准 |
| **API 契约** | 定义路径、请求/响应结构及错误码 | 路径格式、状态码是否严格对应 Constitution 和验收标准 |
| **安全实现方案** | 将 Constitution 的安全策略具体化到本 Feature | 认证流程、授权策略、密码处理是否符合 Constitution |

这些产出不需要分 5 次独立对话——用一个提示词让 AI 一次性产出完整草稿，再通过对话逐项深化即可。

## 6.3 产出完整方案

### 实操演示

#### 第一层：一句话启动

> **你**：基于 @spec.md 和 @constitution.md，产出完整的技术方案（`plan.md`）。要求包含：领域模型（Mermaid 类图）、数据模型（Mermaid ER 图）、业务流程（Mermaid 流程图）、RESTful API 契约、安全实现方案（认证流程 + 授权策略）。

#### 第二层：对话补充

AI 的初版通常能产出 80% 的内容，但细节需要通过对话打磨：

> **你**：补充几个细节：密码字段名为 `hashed_password`，`Todo` 增加 `deleted_at` 字段支持软删除。安全方案中，授权请采用查询级过滤，越权统一返回 404。

> **小白锦囊**：
> 觉得"查询级过滤"这些词太高级？别担心，你可以直接贴需求，让 AI 帮你想方案：
> *"我是个后端新手。按照规范，用户只能看自己的 Todo，而且密码需要安全保存。请帮我设计一套最安全且好实现的代码层安全方案（比如密码怎么存、如何避免查到别人的数据），并向我通俗解释为什么这么做。"*

> AI 产出示例（完整 `plan.md`）：
>
> ````markdown
> # Feature 1：Todo 核心功能 — 技术方案
>
> ## 1. 领域模型
>
> ```mermaid
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
> ```
>
> ## 2. 数据模型
>
> ```mermaid
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
> ```
>
> ## 3. 业务流程
>
> ```mermaid
> flowchart TD
>     A[已认证用户] --> B[操作 Todo]
>     B --> C{是否是自己的 Todo?}
>     C -- 否 --> D[统一返回 404 Not Found]
>     C -- 是 --> E[执行修改/删除]
> ```
>
> ## 4. API 契约
>
> ### POST /api/v1/auth/register
> - 成功（201）：返回 User 信息与 Token。
> - 数据验证失败：返回 422 Unprocessable Entity。
>
> ### GET /api/v1/todos/{id}
> - 成功（200）：返回 Todo 详情。
> - 越权或不存在：一律返回 404 Not Found。
>
> ## 5. 安全实现方案
>
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
>         raise NotFoundError()  # 统一返回 404
>     return todo
> ```
> ````

### 审查要点

拿到 AI 产出的完整方案后，逐项检查：

- **领域模型**：对象是否全部来源于 `spec.md`，关系是否正确且无过度设计。
- **数据模型**：字段类型是否合理，是否建立了必要的索引（如 `user_id`）。
- **业务流程与规则**：越权访问是否严格返回 404，数据验证失败是否严格返回 422，对齐验收标准。
- **API 契约**：路径格式是否符合 Constitution 规范，所有错误码是否对应。
- **安全方案**：密码处理是否符合 Constitution 的安全策略，查询级过滤是否有效防止越权。

## 6.4 整体方案审查

### 核心思路

在结束 Plan 阶段前，进行最后的全盘审查，确保方案完整覆盖所有验收标准，且没有过度设计和规范偏离。

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

## 6.5 本章小结

**核心要点回顾**：
- Plan 将 Specify 的"做什么"翻译为"怎么做"，所有设计决策必须严格遵循 Constitution 规范。
- 用一个提示词一次性产出完整方案（领域模型、数据模型、API 契约、安全方案），再通过对话逐项深化。
- 使用可视化图表（如 Mermaid）表达领域模型和业务流程，降低沟通歧义。
- Plan 完成后须整体审查，确保所有验收标准（AC）都有对应的技术方案覆盖。

### 产出物清单
- `.spec/features/001-todo-core/plan.md` 包含图表与实现契约的完整文档。

第 7 章，我们将进入 Tasks & Implement 阶段，执行任务拆解与编码。
