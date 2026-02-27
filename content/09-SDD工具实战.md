# 第九章：SDD 工具实战

## 9.1 引言

**本章目标**：用工具走完一遍 SDD 流程，把前八章的知识整合为完整的开发工作流。

Ch6 讲了 SDD 方法论，你用文本编辑器手写了需求规范和设计文档。现在用工具自动化这个过程——工具不改变方法，只是把重复劳动交给 AI。

---

## 9.2 SDD 工具生态

| 工具 | 出品方 | 方式 | 特点 |
|------|--------|------|------|
| GitHub Spec Kit | GitHub 官方 | 斜杠命令（`/speckit.specify` 等） | 轻量，搭配现有 AI 助手使用 |
| OpenSpec | 开源社区 | 斜杠命令（`/opsx:propose` 等） | 工件驱动，结构清晰，基于 SKILL 实现 |
| Kiro | AWS | 图形界面 | 可视化工作流，IDE 集成；2025 年发布 |

本章以 **OpenSpec** 为主：它是基于 SKILL（Ch7）实现的——你在 Ch7 学会创建 `/code-review`，OpenSpec 的命令是完全相同的机制。

---

## 9.3 OpenSpec 是什么

OpenSpec 的核心是四个斜杠命令（v1.2.0）：

- `/opsx:propose`：输入变更意图，AI 生成需求文档 + 设计文档 + 任务清单，一步完成规划。人工审查设计文档的决策点不能跳过
- `/opsx:apply`：按任务清单逐条实现代码，完成一条打勾一条
- `/opsx:archive`：归档完成的变更，清理工作目录
- `/opsx:explore`（可选）：在 propose 之前先探索需求，适合边界不清晰的需求

进阶用法：`/opsx:new` + `/opsx:continue` 是拆分式工作流，适合需要增量推进的复杂变更。

工件驱动：每个阶段有明确的文档产出，不依赖对话记忆——这是与 Vibe Coding 的根本区别。

---

## 9.4 案例：完整走一遍 SDD 流程

**安装 OpenSpec**

OpenSpec 本质是几个 Skill 文件，安装方式就是把它们放到你的项目中：

```bash
# 在项目根目录执行，将 OpenSpec Skill 文件安装到 .claude/skills/
# 具体步骤见附录"工具与项目"链接的 GitHub 仓库 README
```

安装后你会在 `.claude/skills/` 下看到对应的 Skill 文件——和你在 Ch7 创建的 `code-review.md` 是完全一样的机制。（Ch7 示例放在 `.claude/commands/`，OpenSpec 使用 `.claude/skills/`，两者都是 Skill 文件，触发方式相同。）

---

以"为博客系统添加评论功能"为例，小明用 OpenSpec 完整走一遍。

**Step 1**：`/opsx:propose 为博客文章添加评论功能`

AI 自动生成三份文档并保存到 `.spec/` 目录：

`需求文档（req.md）` 示例：
```markdown
## 目标
用户可以在博客文章下方发表评论

## 范围内
- 登录用户可以发表评论
- 评论者可以删除自己的评论
- 文章作者可以删除任意评论

## 范围外（本期不做）
- 嵌套回复（评论的评论）
- 评论审核机制
- 匿名评论
```

`设计文档（design.md）` 示例：
```markdown
## 数据模型
新增 comments 表：id, article_id, user_id, content, created_at, deleted_at

## API 设计
- POST /articles/:id/comments   # 发表评论
- DELETE /comments/:id          # 删除评论（本人或文章作者）

## 技术决策
- 不支持嵌套，parent_id 字段暂不加（避免后续迭代被遗留字段干扰）
- 软删除：deleted_at 字段，不物理删除（保留审计追踪）
```

`任务清单（tasks.md）` 示例：
```markdown
- [ ] 建 comments 数据库迁移文件
- [ ] 实现 Comment 数据模型
- [ ] 实现 POST /articles/:id/comments 接口
- [ ] 实现 DELETE /comments/:id 接口（含权限校验）
- [ ] 在文章详情页展示评论列表
- [ ] 在文章详情页添加评论表单
- [ ] 编写接口测试
```

**Step 2（关键）**：人工审查设计文档

小明读了设计文档，发现一个问题：AI 没有提及"游客能否看评论"。他补上：

```markdown
## 读权限
- 评论列表公开可见，无需登录
```

这一步是 SDD 与 Vibe Coding 的核心区别——决策在文档里做，不是让 AI 自己猜。确认后进入下一步。

**Step 3**：`/opsx:apply`

AI 逐条实现任务清单，完成一条打勾一条：

```
- [x] 建 comments 数据库迁移文件
- [x] 实现 Comment 数据模型
- [x] 实现 POST /articles/:id/comments 接口
...
```

**Step 4（检查）**：对照规范文档 review 代码

```
根据 @.spec/design.md，检查实现代码：
1）任务清单每条是否都已实现？
2）设计决策是否正确落地（特别是软删除和权限校验）？
3）有没有超出规范范围的额外实现？
```

**Step 5**：`/opsx:archive`

归档，清理 `.spec/` 工作目录，准备下一个功能迭代。

---

## 9.5 要点

- **工具选择**：小团队/个人用 OpenSpec 或 Spec Kit；偏好可视化用 Kiro
- **关键认知**：`/opsx:propose` 等命令与 Ch7 的 `/code-review` 是同一种机制——都是 Skill 文件驱动的斜杠命令
- **迭代**：archive 之后再次 propose，就是下一轮迭代——这就是完整的 AI 辅助开发循环
- **延伸**：参考 OpenSpec 的结构，可以为自己的项目创建定制化的 SDD 工作流 Skill

---

## 9.6 本章小结

- SDD 工具 = 自动化规范文档生成 + 任务驱动实现
- 工具不改变方法论，只是把重复劳动交给 AI
- "人工审查设计文档"和"对照规范检查代码"，是流程中最不该跳过的两步

**练习**：安装 OpenSpec（参考附录"工具与项目"一节获取链接，或直接在附录链接的 GitHub 仓库查找安装说明），用 `/opsx:propose` 为你当前项目的一个真实小需求生成规划文档，完成人工审查设计文档这一步。

**讲师备注**：建议培训前预先验证 OpenSpec 安装流程，确认所用 Claude Code 版本支持相应 Skill 触发方式。

**结语**：你已经掌握了从提示词到 Agent、从规范到工具的完整 AI 编程工作流。工具会持续迭代，但"让 AI 按你的规矩干活"的核心方法不会变。
