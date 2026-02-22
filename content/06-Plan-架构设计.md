# 第六章：Plan — 架构设计

## 6.1 引言

**本章目标**：学会基于功能规范进行领域建模和技术设计，产出完整的实现方案。

上一章我们写好了功能规范（spec.md）——它告诉我们 Feature 1 要做什么。用户能注册登录、能增删改查 Todo、能筛选分页。每个功能点都有明确的验收标准和边界场景。

但 spec.md 没有回答一个关键问题：**怎么做？**

"用户可以创建 Todo"——好的，但数据存在哪里？表结构是什么？API 路径是什么？请求体长什么样？响应格式呢？认证怎么接入？目录怎么组织？

这些问题就是 Plan 阶段要解决的。

打个比方：spec.md 是建筑的"需求书"（三室一厅、朝南、要有阳台），Plan 是建筑的"施工图"（承重墙在哪、水电怎么走、用什么材料）。你不能拿着需求书就开始砌墙，得先画施工图。

**Plan 和 Constitution 是什么关系？**

Constitution 定的是项目级的规矩——DDD 四层架构、FastAPI + PostgreSQL、TDD 开发方法论。Plan 是在这些规矩的约束下，为具体 Feature 做设计。Constitution 说"用 DDD"，Plan 说"Todo 这个功能的领域模型长什么样"。Constitution 管"原则"，Plan 管"方案"。

| 属于 Constitution（原则） | 属于 Plan（方案） |
|---|---|
| 采用 DDD 四层架构 | Todo 模块的领域对象和关系 |
| 使用 JWT 认证 | JWT 的具体签名配置和刷新流程 |
| API 路径用 kebab-case | 具体的 API 路径和请求/响应格式 |
| 数据库表用 snake_case 复数 | 具体的表结构和字段定义 |

Plan 阶段要产出四大块内容：

1. **领域建模**：业务概念是什么、它们之间是什么关系、有哪些业务规则
2. **技术设计**：数据库表、API 契约、目录结构
3. **安全实现方案**：认证流程、授权规则的具体实现
4. **方案审查**：检查是否过度设计、是否有遗漏、是否与 Constitution 一致

接下来，我们按这个顺序一步步推进。

> **本章的产出物**：`.spec/features/001-todo-core/plan.md`

---

## 6.2 领域建模的目的

在进入具体操作之前，先说说为什么要做领域建模。

你可能会想："我一个 Todo 应用，有什么好建模的？直接写表结构不就行了？"

确实，对于简单的 CRUD 应用，直接写表结构也能做。但领域建模的价值不在于"画个图"，而在于**让你在写代码之前就想清楚业务**。

领域建模做三件事：

1. **提取业务概念**：从需求里找出核心的"东西"——用户、待办事项、状态——给它们取名字、定属性、理关系。
2. **梳理业务流程**：一个 Todo 从创建到完成到删除，经历了哪些步骤？每步谁能操作、有什么限制？
3. **提炼业务规则**：哪些操作有前置条件？哪些状态变化是允许的？哪些数据有约束？

这三件事做完，你就有了一张清晰的"业务地图"。后面的数据模型、API 设计、代码结构，都是从这张地图推导出来的。

**领域建模属于 Plan，不属于 Constitution。** 这一点在第 4 章已经强调过。Constitution 管规矩（"用 DDD"），领域建模是设计工作（"这个 Feature 的领域对象是什么"）。不同 Feature 的领域模型不同，但都遵守同一套 Constitution。

---

## 6.3 识别核心领域对象

### 讲

领域建模的第一步：**从需求里找出核心的"东西"。**

看看我们的 spec.md，出现了哪些业务概念？

- **用户**（User）：能注册、登录、拥有 Todo
- **待办事项**（Todo）：有标题、描述、状态，属于某个用户
- **状态**（Status）：pending 和 completed 两种

找出这些"东西"之后，要搞清楚三件事：

1. **每个对象有什么属性？** 比如 User 有邮箱、密码、创建时间。
2. **对象之间是什么关系？** 比如一个 User 拥有多个 Todo（一对多）。
3. **哪些是实体、哪些是值对象？** 在 DDD 里，有唯一标识的是实体（User、Todo），没有唯一标识、仅通过值来区分的是值对象（Status）。

这一步的目标是画出一张**领域关系图**。不需要画得多复杂——Mermaid 类图就够了。

### 做

#### 第一层：一句话启动

```
基于 @spec.md 和 @constitution.md，识别 Feature 1 的核心领域对象。
列出每个对象的属性和对象之间的关系，用 Mermaid 类图表示。
```

AI 会分析 spec.md 里提到的业务概念，产出领域对象和它们的关系。

#### 第二层：对话补充

拿到 AI 的初版后，你可能需要补充几个细节：

```
补充以下几点：
1. User 的密码字段存的是哈希值，不是明文，字段名用 hashed_password
2. Todo 需要 deleted_at 字段来支持软删除
3. Status 用枚举类型，不是字符串
4. 把创建时间和更新时间加到所有实体上
```

#### AI 产出示例

经过对话后，AI 产出的领域对象模型大致如下：

```markdown
## 领域模型

### 核心领域对象

#### User（用户）— 实体

| 属性 | 类型 | 说明 |
|------|------|------|
| id | UUID | 唯一标识 |
| email | String | 邮箱（唯一，存储为小写） |
| hashed_password | String | 密码哈希值 |
| created_at | DateTime | 创建时间（UTC） |
| updated_at | DateTime | 更新时间（UTC） |

#### Todo（待办事项）— 实体

| 属性 | 类型 | 说明 |
|------|------|------|
| id | UUID | 唯一标识 |
| title | String | 标题（1-200 字符） |
| description | String? | 描述（最多 1000 字符，可选） |
| status | TodoStatus | 状态（pending / completed） |
| user_id | UUID | 所属用户 |
| created_at | DateTime | 创建时间（UTC） |
| updated_at | DateTime | 更新时间（UTC） |
| deleted_at | DateTime? | 软删除时间（null 表示未删除） |

#### TodoStatus（待办状态）— 值对象（枚举）

- `pending`：待完成
- `completed`：已完成

### 领域关系图

​```mermaid
classDiagram
    class User {
        +UUID id
        +String email
        +String hashed_password
        +DateTime created_at
        +DateTime updated_at
    }

    class Todo {
        +UUID id
        +String title
        +String? description
        +TodoStatus status
        +UUID user_id
        +DateTime created_at
        +DateTime updated_at
        +DateTime? deleted_at
    }

    class TodoStatus {
        <<enumeration>>
        pending
        completed
    }

    User "1" --> "*" Todo : owns
    Todo --> TodoStatus : has
​```
```

### 审

拿到领域对象模型后，检查这几个点：

- **对象是否从 spec.md 推导而来？** 每个对象都应该能在需求中找到依据。如果 AI 凭空加了一个"Category"或"Tag"对象——Feature 1 没有这个需求，删掉。
- **属性是否完整？** 对照 spec.md 的功能需求逐条检查。比如 spec 说"自动记录创建时间"，那 created_at 必须在模型里。
- **关系是否正确？** User 和 Todo 是一对多，这个没什么争议。但如果 AI 画了一个多对多关系——想想是不是过度设计了。
- **实体 vs 值对象的区分对吗？** User 和 Todo 有 ID，是实体。TodoStatus 只有两个值，没有 ID，是值对象。如果 AI 把 Status 也做成实体（给它加了 ID 和表），那就过度设计了。
- **命名是否符合 Constitution？** Constitution 说类名用 PascalCase、属性用 snake_case。检查一下。

---

## 6.4 业务流程梳理

### 讲

领域对象回答了"有什么东西"。业务流程回答的是：**这些东西怎么运转？**

一个用户使用 Todo 应用的典型流程：注册 → 登录 → 创建 Todo → 查看/修改 Todo → 标记完成 → 删除。每一步都有前置条件（必须登录）、操作逻辑（标题不能为空）、后置效果（记录创建时间）。

把这些流程画出来，有两个好处：

1. **验证需求的完整性**：流程图画不通的地方，说明需求还有漏洞
2. **指导后续的 API 设计**：每个流程节点通常对应一个 API 端点

对于我们的 Feature 1，主要有两个核心流程：
- **认证流程**：注册 → 登录 → Token 刷新
- **Todo 操作流程**：创建 → 查看/修改 → 完成 → 删除

### 做

#### 第一层：一句话启动

```
基于 @spec.md，梳理 Feature 1 的核心业务流程。
用 Mermaid 流程图表示认证流程和 Todo 操作流程。
```

#### 第二层：对话补充

```
补充以下细节：
1. 认证流程要包含 Token 过期后的刷新路径
2. Todo 操作流程要体现权限检查（是否是自己的 Todo）
3. 每个流程节点标注成功和失败两条路径
```

#### AI 产出示例

```markdown
## 业务流程

### 认证流程

​```mermaid
flowchart TD
    A[用户] --> B{已有账号?}
    B -- 否 --> C[注册: 邮箱+密码]
    C --> D{邮箱是否已注册?}
    D -- 是 --> E[返回 409 冲突]
    D -- 否 --> F[创建用户 + 生成 Token]
    F --> G[返回用户信息 + Access Token + Refresh Token]

    B -- 是 --> H[登录: 邮箱+密码]
    H --> I{凭证是否正确?}
    I -- 否 --> J[返回 401 用户名或密码错误]
    I -- 是 --> K[生成 Token]
    K --> G

    G --> L[携带 Access Token 访问 API]
    L --> M{Token 是否有效?}
    M -- 是 --> N[正常访问]
    M -- 否 --> O{有 Refresh Token?}
    O -- 是 --> P[用 Refresh Token 换新 Access Token]
    P --> Q{Refresh Token 有效?}
    Q -- 是 --> L
    Q -- 否 --> R[返回 401 需重新登录]
    O -- 否 --> R
​```

### Todo 操作流程

​```mermaid
flowchart TD
    A[已认证用户] --> B[选择操作]

    B --> C[创建 Todo]
    C --> D{标题有效?}
    D -- 否 --> E[返回 422 验证错误]
    D -- 是 --> F[创建记录 + 状态=pending]
    F --> G[返回 201 + Todo 对象]

    B --> H[查看/修改/删除 Todo]
    H --> I{Todo 存在且未删除?}
    I -- 否 --> J[返回 404]
    I -- 是 --> K{是自己的 Todo?}
    K -- 否 --> J
    K -- 是 --> L[执行操作]

    L --> M[查看详情 → 返回 200]
    L --> N[修改 → 更新字段 + updated_at → 返回 200]
    L --> O[删除 → 设置 deleted_at → 返回 204]

    B --> P[查看列表]
    P --> Q[按 user_id 筛选 + 排除已删除]
    Q --> R[应用筛选条件 + 分页]
    R --> S[返回 200 + 列表 + 分页元信息]
​```
```

### 审

审查业务流程时注意：

- **流程是否覆盖了所有用户故事？** 对照 spec.md 的用户故事逐条检查，确保每个故事在流程图中都有体现。
- **异常路径是否完整？** 只有"成功路径"的流程图是不及格的。每个决策节点都应该有失败分支。
- **权限检查是否到位？** 所有对 Todo 的操作（查看详情、修改、删除）都必须经过"是自己的 Todo？"这个判断。
- **和 spec.md 的验收标准是否一致？** 比如 spec 说"越权访问返回 404"，流程图里也应该是 404 而不是 403。

---

## 6.5 业务规则提炼

### 讲

领域对象定义了"有什么"，业务流程定义了"怎么走"。业务规则定义的是：**走的时候有什么限制？**

业务规则是从需求中提取出来的约束条件。它们散落在 spec.md 的各个角落——功能需求里有一些，验收标准里有一些，边界场景里也有一些。Plan 阶段要把它们集中起来，分门别类。

为什么要集中？因为后面写代码的时候，AI 需要知道所有的规则，而不是翻遍整个 spec.md 去找。把规则提炼出来，就像给 AI 一份"规则速查表"。

### 做

#### 第一层：一句话启动

```
基于 @spec.md，提炼 Feature 1 的所有业务规则。按类别分组，每条规则要具体可执行。
```

#### 第二层：对话补充

```
按以下分类整理：
1. 数据验证规则（字段约束）
2. 权限规则（谁能做什么）
3. 状态流转规则（状态怎么变化）
4. 业务约束（其他限制）
```

#### AI 产出示例

```markdown
## 业务规则

### 数据验证规则

| 编号 | 规则 | 对应需求 |
|------|------|---------|
| V-1 | 邮箱格式必须合法 | FR-1 |
| V-2 | 邮箱不区分大小写，存储时统一转为小写 | FR-1, Clarify |
| V-3 | 邮箱在系统内必须唯一 | FR-1 |
| V-4 | 密码长度 ≥ 8 位 | FR-1 |
| V-5 | Todo 标题长度 1-200 字符，trim 后判断 | FR-3, Clarify |
| V-6 | Todo 描述最多 1000 字符，可选（null 或非空字符串） | FR-3, Clarify |
| V-7 | Todo 状态只能是 pending 或 completed | FR-6 |
| V-8 | 分页 page ≥ 1 | FR-5, Clarify |
| V-9 | 分页 page_size 范围 1-100，默认 20 | FR-5 |
| V-10 | 筛选的结束日期不能早于开始日期 | 边界场景 |

### 权限规则

| 编号 | 规则 | 对应需求 |
|------|------|---------|
| P-1 | 注册和登录接口公开，其余接口需要认证 | NFR-1 |
| P-2 | 用户只能查看、修改、删除自己的 Todo | NFR-1 |
| P-3 | 越权访问返回 404（不暴露资源是否存在） | NFR-1, AC-15/16/17 |

### 状态流转规则

| 编号 | 规则 | 对应需求 |
|------|------|---------|
| S-1 | 新建 Todo 状态强制为 pending | FR-3, Clarify |
| S-2 | pending ↔ completed 双向切换 | FR-6, Clarify |
| S-3 | 软删除后不可再操作（查看/修改/删除均返回 404） | FR-7, AC-14/29 |

### 业务约束

| 编号 | 规则 | 对应需求 |
|------|------|---------|
| B-1 | 登录失败不区分用户不存在和密码错误 | FR-2 |
| B-2 | 注册成功后自动登录，返回 Token | Clarify |
| B-3 | 修改 Todo 采用部分更新策略 | Clarify |
| B-4 | 列表默认按创建时间倒序 | FR-5 |
| B-5 | 所有时间字段使用 UTC | NFR-3 |
| B-6 | API 响应不包含密码等敏感字段 | Constitution 安全策略 |
```

### 审

审查业务规则时注意：

- **每条规则是否有出处？** 规则不是 AI 编的，必须能追溯到 spec.md 或 Clarify 的决策。如果 AI 加了"密码必须包含大小写字母"——spec 里没这个要求，删掉。
- **规则之间是否矛盾？** 比如一条规则说"状态只能 pending → completed"，另一条说"双向切换"——需要统一。
- **是否遗漏？** 对照 spec.md 的验收标准，每个 AC 背后至少对应一条业务规则。
- **是否过度？** 如果 AI 加了"每个用户最多创建 1000 个 Todo"之类的限制——spec 里没提，别加。

---

## 6.6 数据模型设计

### 讲

领域模型是业务视角，数据模型是存储视角。这一步要做的事情是：**把领域对象"翻译"成数据库表。**

翻译过程不是简单的一一对应。你需要考虑：

- **字段类型**：领域模型里的 String 到底是 VARCHAR(200) 还是 TEXT？
- **约束**：哪些字段不能为空？哪些字段需要唯一约束？
- **索引**：哪些查询频繁？需要建什么索引？
- **关系**：外键怎么定义？级联策略是什么？

对于我们的 Todo 应用，数据模型不复杂——两张表（users 和 todos）加一些索引就够了。但即使简单，也要把每个字段的类型和约束想清楚。不然 AI 写代码时就会自己"发挥"——今天用 VARCHAR(255)，明天用 TEXT，后天用 String(100)。

### 做

#### 第一层：一句话启动

```
基于上面的领域模型和 @constitution.md 的技术栈（PostgreSQL + SQLAlchemy），
设计 Feature 1 的数据库表结构。包含字段类型、约束和索引。
```

#### 第二层：对话补充

```
补充以下要求：
1. 主键用 UUID，不用自增 ID
2. 表名遵循 Constitution 的命名约定：snake_case 复数形式
3. 需要为常用查询场景建索引（按 user_id 查 Todo、按状态筛选）
4. 列出外键约束和级联策略
```

#### AI 产出示例

```markdown
## 数据模型

### users 表

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK, DEFAULT uuid_generate_v4() | 主键 |
| email | VARCHAR(255) | NOT NULL, UNIQUE | 邮箱（小写存储） |
| hashed_password | VARCHAR(255) | NOT NULL | 密码哈希 |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | 创建时间 |
| updated_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | 更新时间 |

索引：
- `ix_users_email`：email 上的唯一索引（登录查询）

### todos 表

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| id | UUID | PK, DEFAULT uuid_generate_v4() | 主键 |
| title | VARCHAR(200) | NOT NULL | 标题 |
| description | TEXT | NULLABLE | 描述 |
| status | VARCHAR(20) | NOT NULL, DEFAULT 'pending' | 状态枚举 |
| user_id | UUID | NOT NULL, FK → users.id | 所属用户 |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | 创建时间 |
| updated_at | TIMESTAMP WITH TIME ZONE | NOT NULL, DEFAULT NOW() | 更新时间 |
| deleted_at | TIMESTAMP WITH TIME ZONE | NULLABLE | 软删除时间 |

索引：
- `ix_todos_user_id`：user_id 上的索引（按用户查询）
- `ix_todos_status`：status 上的索引（按状态筛选）
- `ix_todos_created_at`：created_at 上的索引（排序和时间范围筛选）

外键约束：
- `todos.user_id → users.id`，级联策略：ON DELETE CASCADE

### ER 图

​```mermaid
erDiagram
    users ||--o{ todos : "owns"
    users {
        uuid id PK
        varchar email UK
        varchar hashed_password
        timestamp created_at
        timestamp updated_at
    }
    todos {
        uuid id PK
        varchar title
        text description
        varchar status
        uuid user_id FK
        timestamp created_at
        timestamp updated_at
        timestamp deleted_at
    }
​```
```

### 审

审查数据模型时注意：

- **字段类型是否合理？** title 限制 200 字符，所以用 VARCHAR(200) 而不是 TEXT。description 不限制长度上限用 TEXT（Clarify 里虽然说最多 1000 字符，但这个限制在应用层做，数据库层用 TEXT 更灵活）。
- **和领域模型是否一致？** 领域模型有的属性，数据模型里都要有。领域模型里的 TodoStatus 枚举，数据库里用 VARCHAR(20) 存储——这是一种常见做法，避免了数据库枚举类型的迁移麻烦。
- **索引是否覆盖主要查询场景？** 按 user_id 查 Todo（列表查询）、按 status 筛选、按 created_at 排序——这三个是最频繁的查询，对应的索引要建上。
- **软删除字段是否到位？** deleted_at 允许为 null，null 表示未删除。查询时需要加 `WHERE deleted_at IS NULL` 条件——这个逻辑放在应用层还是数据库层，取决于实现细节。
- **命名是否符合 Constitution？** 表名 snake_case 复数（users, todos），字段名 snake_case。

---

## 6.7 API 契约设计

### 讲

API 契约是前后端（或客户端与服务端）之间的"合同"。它规定了：**你发什么请求过来，我返回什么响应给你。**

API 契约设计要覆盖三个方面：

1. **接口列表**：有哪些 API、URL 是什么、用什么 HTTP 方法
2. **请求/响应格式**：每个 API 的请求体和响应体长什么样
3. **错误码定义**：出错时返回什么状态码和错误信息

设计 API 契约时有一个常见错误：**设计得太随意，或者前后不一致**。比如创建返回完整对象，修改只返回 ID；有的接口用 snake_case，有的用 camelCase。这些不一致会让前端开发者崩溃，也会让 AI 在实现时"发挥创意"。

Constitution 已经规定了 API 路径用 kebab-case，所以我们的路径格式是确定的。接下来需要定义每个具体接口。

### 做

#### 第一层：一句话启动

```
基于 @spec.md 和 @constitution.md，设计 Feature 1 的 RESTful API 契约。
包含接口列表、请求/响应格式和错误码定义。
```

#### 第二层：对话补充

```
补充以下要求：
1. API 路径前缀统一为 /api/v1
2. 请求和响应体用 JSON，字段命名用 snake_case
3. 分页响应要包含 pagination 元信息对象
4. 列出每个接口的认证要求（公开 / 需认证）
5. 错误响应统一格式
```

#### AI 产出示例

```markdown
## API 契约

### 接口总览

| 方法 | 路径 | 说明 | 认证 |
|------|------|------|------|
| POST | /api/v1/auth/register | 用户注册 | 公开 |
| POST | /api/v1/auth/login | 用户登录 | 公开 |
| POST | /api/v1/auth/refresh | 刷新 Token | 公开（需 Refresh Token） |
| POST | /api/v1/todos | 创建 Todo | 需认证 |
| GET | /api/v1/todos | 查看 Todo 列表 | 需认证 |
| GET | /api/v1/todos/{id} | 查看 Todo 详情 | 需认证 |
| PATCH | /api/v1/todos/{id} | 修改 Todo | 需认证 |
| DELETE | /api/v1/todos/{id} | 删除 Todo | 需认证 |

### 认证接口

#### POST /api/v1/auth/register

请求体：
​```json
{
  "email": "user@example.com",
  "password": "securepassword"
}
​```

成功响应（201）：
​```json
{
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "created_at": "2024-01-01T00:00:00Z"
  },
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "token_type": "bearer"
}
​```

#### POST /api/v1/auth/login

请求体：
​```json
{
  "email": "user@example.com",
  "password": "securepassword"
}
​```

成功响应（200）：
​```json
{
  "access_token": "eyJ...",
  "refresh_token": "eyJ...",
  "token_type": "bearer"
}
​```

#### POST /api/v1/auth/refresh

请求体：
​```json
{
  "refresh_token": "eyJ..."
}
​```

成功响应（200）：
​```json
{
  "access_token": "eyJ...",
  "token_type": "bearer"
}
​```

### Todo 接口

#### POST /api/v1/todos

请求体：
​```json
{
  "title": "买牛奶",
  "description": "去超市买两瓶鲜牛奶"
}
​```

成功响应（201）：
​```json
{
  "id": "uuid",
  "title": "买牛奶",
  "description": "去超市买两瓶鲜牛奶",
  "status": "pending",
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-01T00:00:00Z"
}
​```

#### GET /api/v1/todos

查询参数：
- `status`（可选）：pending / completed
- `created_after`（可选）：ISO 8601 日期
- `created_before`（可选）：ISO 8601 日期
- `sort_by`（可选）：created_at / updated_at，默认 created_at
- `sort_order`（可选）：asc / desc，默认 desc
- `page`（可选）：页码，默认 1
- `page_size`（可选）：每页条数，默认 20，最大 100

成功响应（200）：
​```json
{
  "items": [
    {
      "id": "uuid",
      "title": "买牛奶",
      "description": "去超市买两瓶鲜牛奶",
      "status": "pending",
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
    }
  ],
  "pagination": {
    "total": 42,
    "page": 1,
    "page_size": 20,
    "total_pages": 3
  }
}
​```

#### GET /api/v1/todos/{id}

成功响应（200）：
​```json
{
  "id": "uuid",
  "title": "买牛奶",
  "description": "去超市买两瓶鲜牛奶",
  "status": "pending",
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-01T00:00:00Z"
}
​```

#### PATCH /api/v1/todos/{id}

请求体（部分更新，所有字段可选）：
​```json
{
  "title": "买鲜牛奶",
  "status": "completed"
}
​```

成功响应（200）：
​```json
{
  "id": "uuid",
  "title": "买鲜牛奶",
  "description": "去超市买两瓶鲜牛奶",
  "status": "completed",
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-01T10:30:00Z"
}
​```

#### DELETE /api/v1/todos/{id}

成功响应（204）：无响应体

### 统一错误响应格式

​```json
{
  "detail": "错误描述信息"
}
​```

### 错误码对照

| 状态码 | 场景 |
|--------|------|
| 401 | 未认证 / Token 无效或过期 / 登录凭证错误 |
| 404 | 资源不存在 / 越权访问（不暴露资源是否存在） |
| 409 | 邮箱已注册 |
| 422 | 请求参数验证失败（格式错误、超出范围等） |
| 500 | 服务端内部错误 |
```

### 审

审查 API 契约时注意：

- **是否覆盖了 spec.md 的所有功能需求？** 每个 FR 都应该有对应的 API 端点。对照着检查一遍。
- **HTTP 方法是否正确？** 创建用 POST、查看用 GET、部分更新用 PATCH（不是 PUT）、删除用 DELETE。Clarify 里确认了"部分更新"，所以用 PATCH。
- **请求/响应格式是否一致？** 所有响应中的 Todo 对象应该有相同的字段结构。不要创建返回 6 个字段，查看返回 5 个。
- **响应中是否泄露了敏感信息？** User 对象不能包含 hashed_password。Todo 对象不需要包含 user_id（因为都是查自己的）。
- **分页格式是否完整？** pagination 要包含 total、page、page_size、total_pages 四个字段——spec.md 的 AC-22 要求了这些。
- **错误码是否符合 spec.md 的验收标准？** 逐条对照 AC，确保每个错误场景的状态码和描述都一致。

---

## 6.8 目录结构设计

### 讲

Constitution 定了 DDD 四层架构和模块化原则，但那只是"原则"。现在要把原则**落地为具体的目录和文件**。

目录结构的作用是：告诉 AI（和未来的你）每段代码应该放在哪里。没有明确的目录结构，AI 写完一个文件后不知道下一个文件该放哪——可能把数据库操作写在 API 层，把路由定义放在 domain 层。

目录结构设计的原则：

1. **和 Constitution 的架构原则一致**：DDD 四层在目录中体现
2. **和领域模型一致**：auth 模块、todo 模块各自独立
3. **足够具体但不过度**：定义到文件级别，但不定义每个类和函数

### 做

#### 第一层：一句话启动

```
基于 @constitution.md 的架构原则和 Feature 1 的领域模型，
设计具体的项目目录结构，精确到文件级别。
```

#### 第二层：对话补充

```
补充以下要求：
1. 每个文件用一句话注释说明职责
2. 共享模块（数据库配置、中间件）放在 shared/ 下
3. 配置文件、入口文件也要列出来
4. 测试目录结构要镜像源码目录
```

#### AI 产出示例

```markdown
## 目录结构

​```
src/
├── main.py                              # 应用入口，FastAPI 实例创建和路由注册
├── config.py                            # 配置管理（环境变量读取）
├── auth/                                # 认证模块
│   ├── api/
│   │   ├── routes.py                    # 认证相关路由（注册、登录、刷新）
│   │   └── schemas.py                   # 请求/响应的 Pydantic 模型
│   ├── application/
│   │   └── service.py                   # 认证用例（注册流程、登录流程）
│   ├── domain/
│   │   ├── entities.py                  # User 实体
│   │   └── repositories.py             # UserRepository 接口定义
│   └── infrastructure/
│       ├── models.py                    # SQLAlchemy User 模型
│       ├── repository.py               # UserRepository 实现
│       └── security.py                 # JWT 生成/验证、密码哈希
├── todo/                                # Todo 模块
│   ├── api/
│   │   ├── routes.py                    # Todo 相关路由（CRUD + 列表）
│   │   └── schemas.py                   # 请求/响应的 Pydantic 模型
│   ├── application/
│   │   └── service.py                   # Todo 用例（创建、修改、删除、查询）
│   ├── domain/
│   │   ├── entities.py                  # Todo 实体、TodoStatus 枚举
│   │   └── repositories.py             # TodoRepository 接口定义
│   └── infrastructure/
│       ├── models.py                    # SQLAlchemy Todo 模型
│       └── repository.py               # TodoRepository 实现
└── shared/                              # 共享模块
    ├── database/
    │   ├── base.py                      # SQLAlchemy Base 声明
    │   └── session.py                   # 数据库会话管理
    └── middleware/
        └── auth.py                      # 认证中间件（Token 验证依赖项）

tests/
├── conftest.py                          # 共享 fixtures（测试数据库、测试客户端）
├── auth/
│   ├── test_api.py                      # 认证 API 集成测试
│   └── test_domain.py                   # User 领域逻辑单元测试
└── todo/
    ├── test_api.py                      # Todo API 集成测试
    └── test_domain.py                   # Todo 领域逻辑单元测试
​```
```

### 审

审查目录结构时注意：

- **是否体现了 DDD 四层？** 每个模块（auth、todo）下都应该有 api/、application/、domain/、infrastructure/ 四个子目录。
- **职责划分是否清晰？** 路由在 api/routes.py，Pydantic 模型在 api/schemas.py，SQLAlchemy 模型在 infrastructure/models.py——不要搞混。
- **依赖方向是否正确？** api → application → domain ← infrastructure。如果 domain/entities.py 里导入了 SQLAlchemy——那就违反了 Constitution。
- **测试目录是否镜像源码？** src/auth/ 对应 tests/auth/，src/todo/ 对应 tests/todo/。
- **有没有缺文件？** 常见遗漏：配置文件（config.py）、数据库迁移配置（alembic.ini、alembic/）、.env.example。不过迁移相关的文件可以在 Implement 阶段再创建。

---

## 6.9 安全实现方案（认证 + 授权）

### 讲

Constitution 规定了"使用 JWT 认证"和"资源级授权"，但具体怎么实现没有定义——那是 Plan 的事。

安全方案要解决两个问题：**认证**（你是谁）和**授权**（你能做什么）。这两者密切相关——认证确定了用户身份后，授权才能基于身份做访问控制。所以放在一起设计更自然。

认证需要回答：JWT 怎么配置？注册/登录/鉴权流程是什么？
授权需要回答：用户之间的数据如何隔离？越权访问怎么处理？

### 做

#### 第一层：一句话启动

```
基于 @constitution.md 的安全策略和 @spec.md 的权限规则，
设计 Feature 1 的完整安全方案，包含 JWT 认证和资源级授权。
```

#### 第二层：对话补充

```
补充以下细节：
1. JWT 配置（算法、有效期、payload 结构）
2. 注册/登录/鉴权/刷新流程（步骤列表）
3. 授权策略说明（查询级过滤 vs 操作前检查）
4. 每个 API 端点的授权规则
5. 授权逻辑伪代码
6. 越权访问要求返回 404
```

#### AI 产出示例

```markdown
## 安全实现方案

### 一、认证方案

#### JWT 配置

| 配置项 | 值 | 说明 |
|--------|-----|------|
| 签名算法 | HS256 | 对称加密，适合单体应用 |
| Access Token 有效期 | 30 分钟 | 短期令牌 |
| Refresh Token 有效期 | 7 天 | 长期令牌，用于换取新 Access Token |
| 密钥 | 环境变量 JWT_SECRET_KEY | 不硬编码 |

#### Token Payload 结构

Access Token：
​```json
{
  "sub": "user-uuid",
  "type": "access",
  "exp": 1704067200
}
​```

Refresh Token：
​```json
{
  "sub": "user-uuid",
  "type": "refresh",
  "exp": 1704672000
}
​```

#### 注册流程

1. 接收邮箱和密码
2. 邮箱转小写 → 格式校验 → 唯一性检查
3. 如果邮箱已存在 → 返回 409
4. 密码长度校验（≥ 8 位）
5. 密码用 bcrypt 哈希（cost factor = 12）
6. 创建 User 记录存入数据库
7. 生成 Access Token + Refresh Token
8. 返回用户信息 + Token

#### 登录流程

1. 接收邮箱和密码
2. 邮箱转小写
3. 根据邮箱查找用户
4. 如果用户不存在 → 返回 401（"用户名或密码错误"）
5. 验证密码（bcrypt 比对）
6. 如果密码错误 → 返回 401（"用户名或密码错误"）
7. 生成 Access Token + Refresh Token
8. 返回 Token

#### 鉴权流程（中间件/依赖项）

1. 从请求头提取 Authorization: Bearer <token>
2. 如果没有 Token → 返回 401
3. 验证 Token 签名和有效期
4. 如果无效或过期 → 返回 401
5. 检查 Token 类型是否为 access
6. 从 payload 提取 user_id
7. 查询数据库确认用户存在
8. 将用户信息注入请求上下文

#### Token 刷新流程

1. 接收 Refresh Token
2. 验证签名和有效期
3. 检查 Token 类型是否为 refresh
4. 如果无效 → 返回 401
5. 从 payload 提取 user_id
6. 生成新的 Access Token
7. 返回新 Access Token（不重新生成 Refresh Token）

### 二、授权方案

#### 设计策略

采用**查询级过滤**策略：在数据查询时直接加入 `user_id` 条件，而不是先查出资源再检查归属。

优势：
- 天然返回 404（查不到就是不存在，不区分"不存在"和"无权限"）
- 性能更好（不需要额外查询判断权限）
- 实现更简单，不容易遗漏权限检查

#### 各端点授权规则

| 端点 | 授权规则 |
|------|---------|
| POST /api/v1/todos | 自动关联当前用户（user_id 从 Token 中获取） |
| GET /api/v1/todos | 只查询当前用户的 Todo（WHERE user_id = :current_user_id AND deleted_at IS NULL） |
| GET /api/v1/todos/{id} | 按 id + user_id 查询（查不到返回 404） |
| PATCH /api/v1/todos/{id} | 按 id + user_id 查询后修改（查不到返回 404） |
| DELETE /api/v1/todos/{id} | 按 id + user_id 查询后软删除（查不到返回 404） |

#### 授权逻辑伪代码

​```python
# 查看 Todo 详情 — 授权通过查询条件实现
def get_todo(todo_id: UUID, current_user_id: UUID):
    todo = db.query(Todo).filter(
        Todo.id == todo_id,
        Todo.user_id == current_user_id,
        Todo.deleted_at.is_(None)
    ).first()

    if not todo:
        raise NotFoundError()  # 统一 404，不区分原因

    return todo
​```
```

### 审

审查安全方案时，分两部分检查：

**认证部分：**
- **是否与 Constitution 的安全策略一致？** HS256、30 分钟有效期、bcrypt cost factor 12——必须和 Constitution 完全一致。
- **Token Payload 是否合理？** 只放 sub、type 和 exp。不要放邮箱、密码等敏感信息。
- **错误处理是否安全？** 登录失败统一返回"用户名或密码错误"（spec.md AC-4/AC-5），不区分原因。
- **鉴权步骤是否完整？** 验证签名 + 检查过期 + 检查类型 + 确认用户存在。
- **密钥管理是否安全？** JWT_SECRET_KEY 从环境变量读取，不硬编码。

**授权部分：**
- **策略是否和 spec.md 一致？** spec 要求越权返回 404，查询级过滤天然满足。如果 AI 写了"返回 403"——改回来。
- **所有端点都覆盖了吗？** 创建时自动关联 user_id，查询/修改/删除时通过 user_id 过滤。
- **列表查询是否排除了软删除？** `WHERE deleted_at IS NULL` 条件必须存在。
- **有没有绕过 user_id 过滤的路径？** 检查批量操作、关联查询等可能的越权途径。

---

## 6.10 整体方案审查

### 讲

到这里，Plan 的所有组成部分都有了：领域模型、业务流程、业务规则、数据模型、API 契约、目录结构、认证方案、授权方案。

但是——这些部分是分步产出的，有可能前后矛盾、有遗漏、或者过度设计。所以最后一步是**让 AI 做一次整体审查**。

整体审查关注三件事：

1. **有没有过度设计？** 对一个 Todo 应用来说，Plan 应该保持简洁。如果 AI 在过程中偷偷加了缓存层、消息队列、事件驱动——这些都是过度设计。
2. **有没有遗漏？** spec.md 的每条功能需求和验收标准，在 Plan 中都应该有对应的设计。
3. **和 Constitution 是否一致？** 技术栈、命名规范、架构原则——Plan 中的选择必须和 Constitution 完全一致。

### 做

#### 第一层：一句话启动

```
审查这份完整的技术方案，检查三个方面：
1. 有没有对 Todo 应用来说过度设计的部分？
2. 对照 @spec.md，有没有遗漏的功能点？
3. 对照 @constitution.md，有没有不一致的地方？
```

#### 第二层：对话补充

```
审查时重点关注：
1. 领域模型 vs 数据模型是否一一对应
2. API 契约是否覆盖了 spec.md 的所有验收标准（逐条对照 AC）
3. 目录结构是否体现了 Constitution 的 DDD 四层原则
4. 安全方案（JWT 配置、密码哈希）是否和 Constitution 的安全策略完全一致
```

#### AI 产出示例

AI 通常会给出这样的审查报告：

> **过度设计检查：**
>
> ✅ 数据模型：两张表，符合 Todo 应用的复杂度
> ✅ API 设计：标准 RESTful，没有多余的端点
> ✅ 目录结构：DDD 四层清晰，没有冗余的分层
> ⚠️ 建议：鉴权流程第 9 步"查询数据库确认用户存在"——每次请求都查数据库可能影响性能。可以考虑在 Token 有效期内信任 Token 中的 user_id，不做额外查询。这取决于对安全性和性能的权衡。
>
> **遗漏检查（对照 spec.md 验收标准）：**
>
> ✅ AC-1 ~ AC-6：认证相关，API 契约已覆盖
> ✅ AC-7 ~ AC-14：Todo CRUD，API 契约已覆盖
> ✅ AC-15 ~ AC-17：授权相关，授权方案已覆盖
> ✅ AC-18 ~ AC-22：筛选分页，API 契约已覆盖
> ✅ AC-23 ~ AC-28：边界场景，数据验证规则已覆盖
> ✅ AC-29：重复删除，授权方案已覆盖（查不到返回 404）
> ✅ AC-30 ~ AC-32：筛选排序分页补充，API 契约已覆盖
> ✅ AC-33 ~ AC-35：Token 刷新，认证方案已覆盖
>
> **Constitution 一致性检查：**
>
> ✅ 架构原则：DDD 四层，目录结构体现
> ✅ 技术栈：FastAPI + PostgreSQL + SQLAlchemy + JWT
> ✅ 命名规范：表名 snake_case 复数，API 路径 kebab-case，类名 PascalCase
> ✅ 安全策略：HS256、Access Token 30min、Refresh Token 7d、bcrypt cost 12
> ✅ 测试策略：测试目录镜像源码，TDD/BDD
> ⚠️ 注意：Constitution 要求 API 路径用 kebab-case（/api/v1/todo-items），
> 但当前设计中用了 /api/v1/todos。两者都是合理的 RESTful 风格。
> 建议保持 /api/v1/todos（更符合资源命名惯例），
> 并在 Constitution 中将 kebab-case 规则调整为"多词路径用 kebab-case"。

#### 你的处理

根据审查结果做决策：

```
审查结果处理：
1. 鉴权步骤：保留数据库查询。虽然有性能开销，但能确保用户被删除后 Token 立刻失效。后续可以加缓存优化。
2. API 路径：保持 /api/v1/todos 不变，这个更符合 REST 惯例。Constitution 的 kebab-case 规则适用于多词路径（如 /api/v1/todo-items），单词路径用复数形式即可。
```

### 审

整体审查本身的审查要点：

- **AI 的审查是否足够严格？** 如果 AI 全打了 ✅ 没有任何问题——大概率是审查不够仔细。追问一下"确定没有遗漏吗？再检查一遍"。
- **⚠️ 的建议是否合理？** AI 提出的建议不一定都要采纳。比如"去掉鉴权时的数据库查询"——性能更好但安全性降低，你得自己权衡。
- **审查结果是否记录了？** 特别是"暂不处理"的项目（比如"后续加缓存"），记录下来方便后面回顾。

---

## 6.11 完整参考提示词（可选）

> 以下是 Plan 阶段的完整版提示词。**它不是必须的**——前面演示的一句话启动 + 多轮对话完全能达到同样效果。放在这里只是给喜欢一次性把事情说清楚的读者参考。

### Plan 完整提示词

```
基于功能规范（@spec.md）和项目宪章（@constitution.md），
为 Feature 1（Todo 核心功能）做完整的技术方案设计。

请按以下结构输出：

1. 领域模型
   - 核心领域对象（实体、值对象），列出属性和类型
   - 对象关系图（Mermaid classDiagram）
   - 业务流程（Mermaid flowchart）：认证流程、Todo 操作流程
   - 业务规则汇总（分类：数据验证、权限、状态流转、业务约束）

2. 数据模型
   - 数据库表结构（字段、类型、约束）
   - 索引设计
   - ER 图（Mermaid erDiagram）

3. API 契约
   - 接口总览表（方法、路径、说明、认证要求）
   - 每个接口的请求/响应格式（JSON 示例）
   - 统一错误响应格式
   - 错误码对照表

4. 目录结构
   - 基于 DDD 四层的完整项目目录（精确到文件级别）
   - 测试目录结构
   - 每个文件的职责说明

5. 认证方案
   - JWT 配置（算法、有效期、密钥管理）
   - Token Payload 结构
   - 注册/登录/鉴权/刷新流程（步骤列表）

6. 授权方案
   - 授权策略说明
   - 每个端点的授权规则
   - 授权逻辑伪代码

约束：
- 技术栈严格遵循 Constitution（FastAPI + PostgreSQL + SQLAlchemy 2.0 + JWT）
- 命名规范严格遵循 Constitution
- 架构严格遵循 DDD 四层
- 不做过度设计——这是一个 Todo 应用，保持简洁
- 主键用 UUID
- 软删除用 deleted_at 字段
- 越权访问返回 404
- 密码哈希用 bcrypt，cost factor 12
```

> 再强调一次：你完全不需要一开始就用这么完整的提示词。从一句话开始，通过几轮对话逐步完善，效果是一样的。完整提示词更适合你已经很熟悉 SDD 流程、想要提高效率的时候用。

---

## 6.12 本章小结

这一章我们完成了 SDD 功能级的第三个阶段——Plan。回顾一下我们做了什么：

**领域建模：**

1. **识别了核心领域对象**：User 实体、Todo 实体、TodoStatus 枚举，用 Mermaid 类图表达关系
2. **梳理了业务流程**：认证流程（注册/登录/刷新）和 Todo 操作流程（CRUD + 权限检查），用 Mermaid 流程图表达
3. **提炼了业务规则**：数据验证、权限、状态流转、业务约束，分四类汇总

**技术设计：**

4. **设计了数据模型**：users 表和 todos 表，包含字段、类型、约束和索引
5. **设计了 API 契约**：8 个端点的完整定义，包含请求/响应格式和错误码
6. **设计了目录结构**：基于 DDD 四层和模块化原则，精确到文件级别

**安全实现方案：**

7. **设计了认证 + 授权方案**：JWT 配置、注册/登录/鉴权/刷新流程、查询级过滤的授权策略

**方案审查：**

8. **整体审查**：检查了过度设计、功能遗漏和 Constitution 一致性

**几个关键提醒：**

- **Plan 是 spec.md 的"翻译"，不是"创新"。** Plan 里不应该出现 spec 里没有的功能。如果你在做 Plan 的时候想到了新需求——回去更新 spec.md，不要在 Plan 里偷偷加。
- **Plan 要和 Constitution 严格一致。** 技术栈、命名规范、架构原则——任何偏离都要有充分的理由并记录。
- **Plan 不需要完美。** 进入 Implement 阶段后如果发现 Plan 有遗漏，可以回来补充。但大的架构决策（数据模型、API 设计）尽量在 Plan 阶段定下来。
- **审查不是走形式。** AI 产出的方案可能有过度设计、遗漏或和 Constitution 冲突的地方。每次审查都要认真对照。

### 产出物清单

| 文件 | 说明 |
|------|------|
| `.spec/features/001-todo-core/plan.md` | 技术方案文档（含领域模型、数据模型、API 契约、目录结构、认证方案、授权方案） |

### 产出物存放

Plan 完成后，项目的 SDD 文档结构如下：

```
.spec/
├── constitution.md                  # 第 4 章产出
└── features/
    └── 001-todo-core/
        ├── spec.md                  # 第 5 章产出
        └── plan.md                  # 本章产出 ← 我们在这里
```

下一章，我们进入 **Tasks & Implement** 阶段——把这份技术方案拆解为可执行的任务列表，然后逐个实现。
