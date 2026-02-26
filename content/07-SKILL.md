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

`!` 前缀：在 Skill 内嵌入 bash 命令，让 Skill 能读取运行时信息。
```
! git diff HEAD
```
适用于需要获取当前状态的场景（如 git 状态、文件列表）。

Anthropic 官方维护了一个 Skills 仓库（[github.com/anthropics/skills](https://github.com/anthropics/skills)），现成的 Skill 可以直接复用。

---

## 7.3 案例

创建 `/code-review` Skill：

**第一步**：新建 `.claude/commands/code-review.md`，写入：

```
请对以下文件进行 Code Review，重点检查：
1. 安全漏洞（权限校验、SQL 注入、XSS）
2. 异常处理是否完整
3. 代码可读性

文件：$ARGUMENTS
```

**第二步**：使用：`/code-review src/auth.py`

AI 立即按标准格式审查指定文件——不需要重新描述需求，不需要记住格式。

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

**下一章预告**：SKILL 扩展了 AI 的"思维方式"，但 AI 还是无法访问外部系统。下一章介绍 MCP，让 AI 真正能"动手"操作外部工具。
