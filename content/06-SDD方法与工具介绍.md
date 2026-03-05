# 第六章：SDD 方法与工具介绍

## 6.1 引言

**本章目标**：理解 SDD 的核心理念，建立"规范先行"的开发直觉，并用工具走完一遍完整流程。

接到一个新需求：为内部系统添加评论功能。小明想着 AI 这么强，直接说需求就行了：

> 帮我做一个评论功能

AI 很快给出了实现：数据库表、API 接口、前端组件，洋洋洒洒两百行。小明合并进主分支。

第二天产品来问：评论支持嵌套吗（回复某条评论）？

小明去问 AI：给现有评论功能加嵌套支持。

AI 改了一版，但架构需要大调整——原来的表结构没有预留父评论字段，现在要加字段、迁移数据、改接口。改完后，产品又问：评论能 @人 吗？

再次大改。三轮下来，评论模块的代码改了又改，没有一致的设计逻辑，测试也断断续续。

小明后来复盘：问题不在于 AI 写的代码质量——问题在于他从来没有想清楚"评论功能到底要做什么"就让 AI 开始写了。需求没想清楚，AI 只能猜。每次 AI 猜错，就是一次大返工。

---

## 6.2 SDD 是什么

**SDD（Specification-Driven Development，规范驱动开发）**：先写清楚规范，再让 AI 按规范执行。

核心理念只有一句话：**人决策，AI 执行。**

架构怎么设计、边界在哪里、这期不做什么——这些决策由人来做，写进规范文档。AI 负责的是执行：按规范生成代码。

Vibe Coding 的问题不在于"用 AI"，而在于把决策权也交给了 AI——AI 只能猜，猜错就返工。SDD 把决策权还给人，把执行交给 AI。

---

## 6.3 最小化实践：PDCA 循环

SDD 是理念，不绑定工具。用文本编辑器手写规范文档就能落地：

```
Plan  → 写需求规范（做什么、不做什么）；和 AI 澄清边界
Do    → 技术设计 + 拆分任务 + AI 执行
Check → 对照规范检查代码
Act   → 修复偏差，归档，进入下一轮
```

关键是在 **Plan 阶段**写清楚"不做什么"——范围不清是翻车最常见的原因。先写规范的过程本身会暴露需求漏洞，比如"游客能否查看评论"这类遗漏，在动手前澄清比实现后返工代价小得多。

上面这套流程可以纯手动执行。下一节用 OpenSpec 工具自动化它。

---

## 6.4 OpenSpec 工具实战

### 安装与配置

OpenSpec 是一个 CLI 工具，核心是一套 Skill 文件（斜杠命令的底层机制详见 Ch7）。

```bash
# 全局安装
npm install -g @fission-ai/openspec

# 配置扩展工作流（交互式，选择包含 new/continue 等命令的配置）
openspec config profile

# 在项目根目录初始化，将 Skill 文件安装到 .claude/skills/
openspec init --tools claude

# 如果项目已初始化过，更新 Skill 文件
openspec update
```

重启 IDE 后斜杠命令生效。

---

### 扩展工作流

OpenSpec 的扩展工作流把 PDCA 拆成七个动作：

| 命令 | 作用 |
|------|------|
| `/opsx:explore` | 探索阶段：与 AI 思考需求边界，不产生文档 |
| `/opsx:new` | 创建变更目录，初始化工件结构 |
| `/opsx:continue` | 生成下一个工件（依次：proposal → specs → design → tasks） |
| `/opsx:apply` | 按任务清单逐条实现代码，完成一条打勾一条 |
| `/opsx:archive` | 归档完成的变更，清理工作目录 |

工件驱动的关键：每个阶段产出明确的文档，下一步以上一步的文档为输入——不依赖对话记忆。

**工件依赖关系：**

```
proposal
    ├── specs
    └── design
         └── tasks → apply → archive
```

---

### 案例：为博客系统添加评论功能

以小明的案例为例，用扩展工作流重来一遍。

**Step 1：探索需求**

```
/opsx:explore
```

AI 作为思考伙伴，帮小明梳理边界：评论的读写权限、是否支持嵌套、删除规则。探索结束后，思路清晰，进入下一步。

**Step 2：创建变更**

```
/opsx:new add-comment-feature
```

在 `openspec/changes/add-comment-feature/` 创建变更目录，等待生成工件。

**Step 3：逐步生成工件（continue × 4）**

```
/opsx:continue   # 生成 proposal.md（变更意图与范围）
/opsx:continue   # 生成 specs（需求规范）
/opsx:continue   # 生成 design.md（技术设计）
/opsx:continue   # 生成 tasks.md（任务清单）
```

`specs` 示例片段：

```markdown
### 做什么
- 登录用户可发表评论
- 评论者可删除自己的评论
- 文章作者可删除任意评论
- 评论列表公开可见，无需登录

### 不做（本期范围外）
- 嵌套回复
- 评论审核机制
- 匿名评论
```

`design.md` 示例片段：

```markdown
## 数据模型
comments 表：id, article_id, user_id, content, created_at, deleted_at

## 技术决策
- 软删除（deleted_at 字段）：保留审计追踪
- API：POST /articles/:id/comments、DELETE /comments/:id（含权限校验）
```

**Step 4（关键）：人工审查规范和设计**

小明读 specs 和 design.md，确认范围和技术决策没有遗漏。发现问题直接编辑文档——决策在文档里做，不让 AI 猜。这是 SDD 与 Vibe Coding 的核心区别。

**Step 5：实现**

```
/opsx:apply
```

AI 按任务清单逐条实现，完成一条打勾一条：

```
- [x] 建 comments 数据库迁移文件
- [x] 实现 Comment 数据模型
- [x] 实现 POST /articles/:id/comments 接口
- [x] 实现 DELETE /comments/:id 接口（含权限校验）
- [x] 在文章详情页展示评论列表
- [x] 编写接口测试
```

**Step 6：归档**

```
/opsx:archive
```

归档变更，清理工作目录。下一期要加嵌套回复时，重新从 `/opsx:explore` 开始。

---

## 6.5 要点

- **规范的最低标准**：写了"不做什么" = 合格。范围不清是翻车最常见的原因
- **先写规范的价值**：写规范的过程本身会暴露需求漏洞，比代码写完再返工代价低得多
- **人工审查不能跳过**：`/opsx:continue` 生成 design.md 后，必须人工确认技术决策再进入实现阶段
- **工件驱动 vs 对话驱动**：每个阶段有文档产出，下一步以文档为输入，不依赖对话记忆——这是与 Vibe Coding 的根本区别
- **工具选择**：小团队/个人用 OpenSpec；偏好可视化用 Kiro（AWS 出品，2025 年发布）
- **何时用 SDD**：需要长期维护的功能、多人协作、需求可能扩展的场景。一次性脚本不必

---

## 6.6 本章小结

- SDD = 先写规范，人决策，AI 执行
- 最简实践：Plan（需求规范 + 澄清边界）→ Do（技术设计 + 拆任务 + AI 执行）→ Check（对照规范）→ Act（归档迭代）
- OpenSpec 把上述流程工具化：explore → new → continue × 4 → apply → archive
- 核心不变：决策显式化，不依赖对话记忆，不让 AI 猜

**练习**：用 OpenSpec 为你当前项目的一个真实小需求走一遍流程：`/opsx:explore` 探索需求，`/opsx:new` 创建变更，`/opsx:continue` 生成规范文档，完成人工审查这一步。

**下一章预告**：`/opsx:*` 这类斜杠命令底层是 SKILL 文件。下一章介绍 SKILL 机制，你将学会为自己的项目创建定制化的斜杠命令。
