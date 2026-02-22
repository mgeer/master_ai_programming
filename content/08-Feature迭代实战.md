# 第八章：Feature 迭代实战

## 8.1 引言

**本章目标**：通过第二个 Feature 演示 SDD 的迭代循环，展示 Constitution 已有情况下的真实工作节奏。

前几章我们完整走了一遍 SDD 全流程：从 Constitution 建章立制，到 Specify 定义需求，到 Plan 设计架构，到 Tasks 拆解任务，再到 Implement 编码实现。每个阶段都详细讲解了概念、提示词和审查要点。

但真实项目不会只有一个 Feature。

**Feature 做完一个，马上要做下一个——这才是日常工作的常态。** 本章要演示的就是这个"下一个"怎么做。你会发现，当 Constitution 已经存在、SDD 流程已经跑通一遍之后，第二个 Feature 的推进速度会快得多。

我们的任务：**为 Todo 应用增加标签（Tag）功能。**

本章不会重复讲解概念——你已经在前几章学会了。这里只给关键提示词、产出摘要，以及与 Feature 1 的差异。篇幅会紧凑很多，这也正是迭代的真实节奏：熟练之后，每个 Feature 的 SDD 流程会越来越快。

---

## 8.2 迭代的起点

做 Feature 1 时，我们从零开始：先写 Constitution，再一步步走 Specify → Clarify → Plan → Tasks → Implement。

做 Feature 2 时，起点完全不同：

| 维度 | Feature 1 | Feature 2 |
|------|-----------|-----------|
| Constitution | 从零创建 | 已存在，直接复用 |
| 规则文件 | 从零生成 | 已存在，AI 自动遵守 |
| 领域模型 | 从零设计 | 在已有模型上扩展 |
| 项目结构 | 从零搭建 | 已有结构，增量添加 |
| 起始阶段 | Constitution | Specify |

**关键区别：Constitution 不需要重新创建。** 除非你发现 Feature 1 做完后 Constitution 有需要修正的地方，否则直接从 Specify 开始。你省掉了整个第 4 章的工作量——SDD 的"项目级一次性"设计在这里体现出了价值。

新 Feature 的文档放在独立目录下：

```
.spec/
├── constitution.md                  # 不动
└── features/
    ├── 001-todo-core/               # Feature 1（已完成）
    │   ├── spec.md
    │   ├── plan.md
    │   └── tasks.md
    └── 002-todo-tags/               # Feature 2（本章）
        ├── spec.md
        ├── plan.md
        └── tasks.md
```

好，直接开始。

---

## 8.3 Specify：定义标签功能需求

### 提示词

```
我要为 Todo 应用增加标签（Tag）功能。基于 @constitution.md 和 @spec.md（Feature 1），
帮我为这个新功能编写功能规范。

核心需求：
- 用户可以创建、管理标签
- 给 Todo 打标签（多对多关系）
- 按标签筛选 Todo
```

注意和 Feature 1 的区别：**提示词里引用了已有文档。** `@constitution.md` 确保 AI 遵守项目规矩，`@spec.md` 让 AI 了解已有功能，避免重复或冲突。这是迭代 Feature 最重要的习惯——不引用已有文档，AI 可能会重新定义一套用户模型。

### 产出摘要

AI 生成的 `spec.md` 核心内容：

```markdown
# Feature 2：Todo 标签功能

## 用户故事
- 作为用户，我希望创建自定义标签，以便对 Todo 进行分类管理
- 作为用户，我希望给 Todo 打上一个或多个标签，以便快速归类
- 作为用户，我希望按标签筛选 Todo 列表，以便聚焦特定类别的任务
- 作为用户，我希望管理（重命名、删除）已有标签，以便保持标签体系整洁

## 功能需求

### 标签管理
- 创建标签：名称 + 颜色（可选）
- 标签名称在同一用户下唯一
- 重命名标签、删除标签（同时解除与所有 Todo 的关联）

### Todo 与标签的关联
- 创建 Todo 时可同时指定标签
- 为已有 Todo 添加/移除标签
- 一个 Todo 可以有多个标签

### 按标签筛选
- Todo 列表接口支持按标签 ID 筛选
- 支持多标签筛选（AND 逻辑）

## 非功能需求
- 每个用户最多 50 个标签，每个 Todo 最多 10 个标签
- 标签相关接口响应时间 < 200ms

## 验收标准
- [ ] 可以创建、重命名、删除标签
- [ ] 标签名称同一用户下唯一
- [ ] 可以给 Todo 添加/移除标签
- [ ] Todo 详情包含标签列表
- [ ] 按标签筛选 Todo 正常工作
- [ ] 删除标签时自动解除关联
- [ ] 标签数量限制生效
```

**保存到**：`.spec/features/002-todo-tags/spec.md`

---

## 8.4 Clarify：快速澄清

### 提示词

```
审查这份标签功能规范（@spec.md），找出不够明确的地方，逐条向我提问。
```

### AI 提出的关键问题

1. **标签颜色**：固定选项还是自定义 hex 值？
2. **多标签筛选逻辑**：AND（同时包含所有标签）还是 OR（包含任一标签）？
3. **删除标签的行为**：只解除关联，还是有其他影响？
4. **标签排序**：默认排序方式？

### 你的回答

```
1. 颜色用预定义的 8 种颜色，用枚举值（red、blue、green 等），不支持自定义 hex
2. 默认 AND 逻辑，后续可以扩展 OR
3. 删除标签只解除关联，不影响 Todo 本身
4. 按创建时间倒序
```

AI 根据回答更新 spec.md，Clarify 完成。

### 与 Feature 1 的差异

Feature 1 我们演示了结构化澄清、自由补充、验收清单审查三种方式。Feature 2 需求相对简单，**一轮快速问答就够了**。

这就是迭代中的真实节奏：**小需求的 Clarify 可以很快，但不能跳过。** 不 Clarify 直接做 Plan，你可能到实现阶段才发现"颜色到底是枚举还是自定义"——那时候改成本就高了。

---

## 8.5 Plan：扩展架构设计

### 提示词

```
基于 @constitution.md、@spec.md（Feature 2）和 @plan.md（Feature 1），
为标签功能做技术方案设计。重点关注：领域模型扩展、新增 API、数据模型变更。
```

这里引用了三个文档：Constitution 确保遵守规矩，Feature 2 的 spec 是设计依据，Feature 1 的 plan 是现有架构的基础——**新设计必须在已有架构上扩展，而不是另起炉灶。**

### 产出摘要

#### 领域模型扩展

新增两个对象：

- **Tag**：id、name（用户内唯一）、color（枚举：red/blue/green/yellow/purple/orange/pink/gray）、user_id、created_at
- **TodoTag**（关联关系）：todo_id、tag_id

对象关系：User 1:N Tag，Todo N:N Tag（通过 TodoTag 关联）。对比 Feature 1 只有 User 和 Todo，Feature 2 新增了 Tag 实体和一条多对多关系——**模型是增量扩展的，不是重新设计。**

#### 新增 API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /api/v1/tags | 创建标签 |
| GET | /api/v1/tags | 获取当前用户所有标签 |
| PUT | /api/v1/tags/{id} | 重命名 / 修改颜色 |
| DELETE | /api/v1/tags/{id} | 删除标签 |
| POST | /api/v1/todos/{id}/tags | 为 Todo 添加标签 |
| DELETE | /api/v1/todos/{id}/tags/{tag_id} | 移除标签 |

已有 API 变更：`GET /api/v1/todos` 新增 `tag_ids` 查询参数；`GET /api/v1/todos/{id}` 响应新增 `tags` 字段；`POST /api/v1/todos` 请求新增可选 `tag_ids` 字段。

#### 数据模型变更

新增 `tags` 表（id、name、color、user_id、created_at，唯一约束 `(user_id, name)`）和 `todo_tags` 关联表（todo_id、tag_id，联合主键，外键均设 `ON DELETE CASCADE`）。

**保存到**：`.spec/features/002-todo-tags/plan.md`

### 与 Feature 1 的差异

Feature 1 的 Plan 我们详细讲了领域建模方法、数据模型映射原则、API 设计规范。Feature 2 不需要重复——直接在已有基础上扩展。**Plan 的思维模式从"从零设计"变成了"增量设计"**：标签筛选不是新建接口，而是扩展已有的 `GET /api/v1/todos` 增加查询参数。

---

## 8.6 Tasks：增量任务拆解

### 提示词

```
基于 @plan.md（Feature 2），拆解为可执行的任务列表。
注意：这是增量开发，项目已有 User 和 Todo 的完整实现。
```

### 产出摘要

```markdown
# Feature 2 任务列表

## Task 1：Tag 领域模型与数据库
- 创建 Tag 实体和 TagColor 枚举（领域层）
- 创建 tags 表和 todo_tags 关联表（基础设施层）
- 编写 Alembic 迁移脚本 + 单元测试

## Task 2：标签 CRUD API
- 实现 TagRepository、TagService、标签路由
- 编写集成测试｜依赖：Task 1

## Task 3：Todo-标签关联
- 扩展 TodoRepository 支持标签关联操作
- 实现添加/移除标签的 API
- 修改 Todo 详情响应包含标签列表
- 编写集成测试｜依赖：Task 1, Task 2

## Task 4：按标签筛选 Todo
- 扩展 TodoRepository 查询方法支持 tag_ids 过滤
- 修改 GET /api/v1/todos 增加 tag_ids 参数
- 编写集成测试｜依赖：Task 3

## Task 5：业务规则与边界处理
- 标签名称唯一性校验
- 标签数量上限（用户 50 个，Todo 10 个）
- 删除标签级联解除关联
- 编写边界场景测试｜依赖：Task 2, Task 3
```

**保存到**：`.spec/features/002-todo-tags/tasks.md`

Feature 1 的任务列表包含基础设施搭建（数据库连接、项目骨架等），Feature 2 不需要这些——**基础设施已经就位，任务聚焦在业务功能本身。** 5 个任务 vs Feature 1 的十几个，精简不少。

---

## 8.7 Implement：快速实现演示

选 Task 1（Tag 领域模型与数据库）做一个快速演示。

### 提示词

```
按 @tasks.md 的 Task 1，实现 Tag 领域模型与数据库。
遵守 @constitution.md 的架构原则和编码规范。先写测试，再写实现。
```

### AI 产出概览

AI 按照 Constitution 的 DDD 四层架构，生成了以下文件：

- **领域层**（`src/todo/domain/tag.py`）：`Tag` dataclass + `TagColor` 枚举（8 种颜色），纯业务对象，不依赖任何框架
- **基础设施层**（`src/todo/infrastructure/tag_model.py`）：`TagModel` 和 `todo_tags` 关联表，包含 `(user_id, name)` 唯一约束
- **迁移脚本**：Alembic 迁移，创建 `tags` 表和 `todo_tags` 关联表
- **测试**（`tests/todo/test_tag_domain.py`）：验证 Tag 创建和 TagColor 枚举，BDD 风格

这里不展开完整的 Test → Implement → Review 循环——第 7 章已经完整演示过了。剩余 Task 2-5 按同样模式推进即可。

---

## 8.8 验收对照

所有任务完成后，对照 spec.md 的验收标准逐条检查。

### 提示词

```
对照 @spec.md（Feature 2）的验收标准，逐条检查当前实现是否满足。
已满足的打勾，未满足的说明原因。
```

### 验收结果

```markdown
- [x] 可以创建、重命名、删除标签
- [x] 标签名称同一用户下唯一
- [x] 可以给 Todo 添加/移除标签
- [x] Todo 详情包含标签列表
- [x] 按标签筛选 Todo 正常工作
- [x] 删除标签时自动解除关联
- [x] 标签数量限制生效
```

全部通过，Feature 2 完成。**每个 Feature 结束时都要做验收对照，没有例外。** "跑起来没报错"不等于"需求全部满足"。

---

## 8.9 迭代中的决策与回退

在 Feature 2 的开发过程中，有一些关键的"决策点"值得单独讲讲——因为真实项目中，迭代不总是一路顺畅的。

### 决策点：标签实现方案选择

Plan 阶段，标签功能的实现方式有多种选择。假设你（或 AI）一开始给出了两个方案：

**方案 A：Todo 表加 tags 字段（JSON 数组）**

```json
// todos 表新增字段
"tags": ["工作", "紧急"]
```

优势：简单，不用加新表。劣势：标签没有独立实体，无法管理（重命名、设颜色），筛选性能差。

**方案 B：独立标签表 + 关联表（我们最终选择的方案）**

优势：标签是独立实体，支持 CRUD、颜色、唯一性约束，筛选可以走索引。劣势：多了两张表，查询需要 JOIN。

怎么选？回到 spec.md 的需求：标签要支持创建、重命名、删除、颜色——这些都需要标签是独立实体。**方案 A 一看就不满足需求，方案 B 是唯一选择。**

这就是"方案对比"技巧在 Plan 阶段的应用：**列出选项，对照需求，选有充分理由的那个。** 不确定的时候可以让 AI 帮你做对比分析：

```
标签功能有两种实现方式：JSON 数组存在 Todo 表 vs 独立标签表+关联表。
请对比分析两者的优劣，结合 @spec.md 的需求推荐一种。
```

### 回退点：发现需要修改 Plan

Feature 2 的 Plan 完成后，在 Tasks 拆解时你可能会发现：`GET /api/v1/todos` 现在需要新增 `tag_ids` 查询参数做标签筛选，但 Feature 1 的 plan.md 里没有预留这个扩展点。

这就是第 3 章说过的**阶段回退**：Tasks 阶段发现 Plan 需要调整。

正确做法：

1. **先回到 Plan，更新文档**：在 Feature 2 的 plan.md 中明确记录"已有 API 变更"（我们已经做了——8.5 节的"已有 API 变更"部分）
2. **评估对 Feature 1 的影响**：新增查询参数是向后兼容的（原有请求不受影响），所以不需要修改 Feature 1 的 plan.md
3. **继续推进 Tasks**

如果新需求不向后兼容（比如要改已有 API 的响应格式），就需要：更新 Feature 1 的 plan.md → 相关代码也要改 → 可能需要创建一个专门的重构任务。

**回退不可怕，可怕的是不回退——硬着头皮往下走，最后 API 设计自相矛盾。**

---

## 8.10 迭代中的实际节奏

做完两个 Feature，可以总结一下 SDD 在迭代中的实际节奏。

### 哪些步骤可以简化？

| 步骤 | Feature 1 | Feature 2（小需求） | 说明 |
|------|-----------|-------------------|------|
| Constitution | 完整创建 | 跳过 | 项目级，只做一次 |
| Specify | 详细定义 | 简洁定义 | 需求明确时可以更简短 |
| Clarify | 多轮澄清 | 一轮快速问答 | 小需求的模糊点少 |
| Plan | 从零设计 | 增量扩展 | 在已有架构上添加 |
| Tasks | 完整拆解 | 精简拆解 | 没有基础设施搭建任务 |
| Implement | 逐任务演示 | 快速推进 | 模式已熟练 |
| 验收 | 逐条检查 | 逐条检查 | **不能简化** |

### 哪些步骤不能跳过？

**三个阶段绝不能跳过：Specify、Plan、验收。**

- **Specify 不能跳过**：不写规范直接做 Plan，你和 AI 对"要做什么"的理解可能完全不同。哪怕需求很简单，一份简短的规范也比口头描述可靠。
- **Plan 不能跳过**：不做 Plan 直接写代码，AI 会按自己的理解设计数据模型和 API——可能和你的项目架构格格不入。
- **验收不能跳过**：不做验收，你怎么知道功能真的做完了？

**可以简化但不建议跳过的：Clarify 和 Tasks。** Clarify 可以一轮搞定，Tasks 可以只列 5 条，但产出物一定要有。

### 规范文档的增量演进

随着 Feature 不断迭代，`.spec/` 目录会越来越充实：

```
.spec/
├── constitution.md              # 稳定不变（偶尔修订）
└── features/
    ├── 001-todo-core/           # 已完成
    ├── 002-todo-tags/           # 已完成
    ├── 003-todo-reminders/      # 下一个 Feature
    └── ...
```

每个 Feature 的 Plan 都是在前一个的基础上扩展的。领域模型从 User + Todo，到 User + Todo + Tag，未来再加 Reminder、Category……**每次只新增，不推翻。**

当然，有时候确实需要重构已有设计。这时候：先更新相关的 plan.md 记录变更，再创建专门的重构 Feature（如 `006-refactor-todo-model/`），按 SDD 流程走一遍。**不要偷偷改代码不更新文档**——规范文档是"单一真相源"。

---

## 8.11 本章小结

这一章我们用标签功能（Feature 2）演示了 SDD 的迭代循环。核心要点：

**1. 迭代从 Specify 开始，不从 Constitution 开始。** Constitution 做一次就够了，后续每个 Feature 直接进入 Specify → Clarify → Plan → Tasks → Implement → 验收。

**2. 提示词中引用已有文档是关键习惯。** 每个阶段都引用 `@constitution.md` 和前序产出物，确保 AI 在已有上下文中工作，而不是从零开始。

**3. 小需求可以快，但流程不能乱。** 每个阶段都比 Feature 1 快，但没有跳过任何阶段。

**4. 增量思维贯穿始终。** 领域模型是扩展的，API 是新增的，数据库是加表的——在已有基础上生长。

### 产出物清单

| 文件 | 说明 |
|------|------|
| `.spec/features/002-todo-tags/spec.md` | 标签功能规范 |
| `.spec/features/002-todo-tags/plan.md` | 标签功能技术方案 |
| `.spec/features/002-todo-tags/tasks.md` | 标签功能任务列表 |
| 增量代码和测试 | Task 1-5 的实现 |

下一章，我们跳出具体的 Feature 开发，聊聊 SDD 的工具生态和实践中的最佳经验。
