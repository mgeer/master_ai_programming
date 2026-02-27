# 第七章：SKILL（可复用提示词）

## 7.1 引言

**本章目标**：把好用的提示词固化为命令，一次编写，永久复用。

每次 Code Review 都要重新写提示词，每次生成 SDD 文档都要重新描述格式要求——这些重复劳动完全可以消除。SKILL 就是解决方案：把提示词保存为文件，用斜杠命令一键触发。

---

## 7.2 SKILL 是什么

**SKILL 的本质是一个 Markdown 文件**，放在 `.claude/commands/` 目录下，用斜杠命令触发。

```
~/.claude/commands/     # 用户级，所有项目可用
.claude/commands/       # 项目级，仅当前项目可用
```

触发方式：在 Claude Code 中输入 `/命令名` 即可执行对应的 Skill 文件内容。

**两个关键机制：**

`$ARGUMENTS`：让 Skill 接受参数，变成可复用的"函数"。
```
/code-review src/auth.py
```
其中 `src/auth.py` 就是 `$ARGUMENTS`，在 Skill 文件里用 `$ARGUMENTS` 占位即可引用。

`!` 前缀：在 Skill 内嵌入 bash 命令，让 Skill 能读取运行时信息。Bash 命令在发送给 Claude 之前执行，输出内容直接注入提示词。
```
! git diff HEAD
```
适用于需要获取当前状态的场景（如 git 状态、文件列表）。

Anthropic 官方维护了一个 Skills 仓库（[github.com/anthropics/skills](https://github.com/anthropics/skills)），现成的 Skill 可以直接复用。

---

## 7.3 案例：小明的提示词资产

小明团队引入了 code review 流程，每个 PR 合并前都要 AI 审查一遍。第一周，小明每次都现写提示词：

> 帮我 review 这段代码，检查安全漏洞，看看有没有 SQL 注入、权限校验问题，异常处理是否完整，代码可读性怎么样……

写到第三天，他意识到这段话一字不差地打了十几遍。

**Before（临时提示词）**

每次 review 都要：
1. 打开文件，复制文件路径（或 `@` 引用）
2. 想一下这次 review 要查什么
3. 重新打那段 200 字的描述
4. 粘进对话，提交

**After（创建 /code-review Skill）**

小明花了 5 分钟创建了 `.claude/commands/code-review.md`：

```markdown
请对以下文件进行 Code Review，重点检查：
1. 安全漏洞（权限校验、SQL 注入、XSS）
2. 异常处理是否完整
3. 代码可读性和命名规范

给出具体的问题行号和修改建议，不要泛泛而谈。

文件：$ARGUMENTS
```

现在他的 review 流程变成了：

```
/code-review src/auth.py
```

一句话，回车，完成。`$ARGUMENTS` 自动替换为 `src/auth.py`，Skill 内容作为提示词完整发送给 Claude。

**进阶：用 `!` 前缀读取运行时信息**

小明后来创建了 `/pre-commit` Skill，每次提交前自动 review 将要改动的文件，不用手动指定文件名：

```markdown
我要提交以下改动，请检查是否有遗漏的问题：

! git diff --staged

重点检查：
1. 新加代码是否有对应测试
2. 是否有调试用的 print/console.log 残留
3. 变量和函数命名是否清晰
```

`! git diff --staged` 是 bash 命令，Skill 执行时会自动把当前暂存区的 diff 内容注入到提示词里。每次提交前跑 `/pre-commit`，Claude 看的就是那次提交实际要改动的代码。

好的提示词不应该是一次性的。写一个 Skill 文件，等于给自己的工作流加了一个永久生效的专业工具——用的次数越多，沉淀的价值越大。

---

## 7.4 要点

- **好 Skill 的标准**：单一职责、上下文充分、输出格式明确
- **与 CLAUDE.md 的关系**：规范（Ch5）约束 Skill 的行为；Skill 是执行规范的快捷方式
- **Skill vs MCP**：Skill 扩展 AI 的"思维方式"（提示词层面）；MCP 扩展 AI 的"行动能力"（连接外部系统，详见 Ch8）

---

## 7.5 本章小结

- SKILL = 保存在文件里的提示词，用斜杠命令触发
- `$ARGUMENTS` 让 Skill 参数化，`!` 前缀让 Skill 读取运行时信息
- 把常用 prompt 固化为 Skill，是提升 AI 工作效率的最低成本方式

**练习**：创建一个你在日常工作中真正用得上的 Skill（不必是 Code Review，可以是你重复最多的任何 AI 任务），保存到 `.claude/commands/`，测试它能否正常触发。

**下一章预告**：SKILL 扩展了 AI 的"思维方式"；MCP 则扩展 AI 的"行动能力"——让它能连接文件、Bash 之外的外部系统。
