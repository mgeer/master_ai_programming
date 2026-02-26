# 第八章：MCP（模型上下文协议）

## 8.1 引言

**本章目标**：理解 MCP 是什么，学会安装和使用第一个 MCP 工具。

默认情况下，AI 的能力边界是"生成文本"——它无法访问网络、查询数据库、操作文件系统之外的系统。MCP 打破了这个边界。

---

## 8.2 MCP 是什么

**MCP（Model Context Protocol，模型上下文协议）**：Anthropic 于 2024 年 11 月发布的标准协议，让 AI 通过标准化接口连接外部工具和数据源。

类比：MCP 相当于给 AI 装插件——装了什么插件，AI 就能操作什么系统。

- 没有 MCP 的 AI：生成文本、读写本地文件
- 有了 MCP 的 AI：访问网络、查询数据库、调用 API、操作任意外部系统

MCP 是让 Agent（Ch3）能力边界扩展的关键机制。

---

## 8.3 案例：安装 fetch MCP

**安装前**：
> 你：帮我总结 https://docs.anthropic.com 的最新内容
> AI：我无法直接访问网络链接。

**安装 fetch MCP**：

Claude Code 用户（终端执行）：
```bash
claude mcp add fetch
```

Claude Desktop 用户：在 `~/.claude/claude_desktop_config.json` 中添加 fetch MCP 配置，重启生效。

**安装后**：
> 你：帮我总结 https://docs.anthropic.com 的最新内容
> AI：（实时抓取页面，返回摘要）

同一个问题，安装 MCP 前后是两种完全不同的体验。

**进阶示例（供参考）**：MySQL MCP 让 AI 直接查询数据库——直接问"users 表有多少条记录"，AI 返回真实数据。适合已有本地 MySQL 的读者自行探索。

---

## 8.4 要点

- **安全边界**：数据库类 MCP 只开读权限到开发库，不暴露生产环境
- **适用场景**：调试、数据分析、信息检索；不适用：生产环境写操作
- **MCP vs SKILL（Ch7）**：SKILL 扩展 AI 的"思维方式"，MCP 扩展 AI 的"行动能力"
- **MCP 生态**：Anthropic、各大云厂商和社区已发布大量现成 MCP，可直接使用（可在 Claude Code 官方文档的 MCP 章节或社区目录中查找）

---

## 8.5 本章小结

- MCP = 给 AI 装插件，装了什么就能操作什么
- 安装方式：Claude Code 用 `claude mcp add`，Claude Desktop 用配置文件
- 每新增一个 MCP，Agent 的能力边界就扩展一次

**练习**：安装 fetch MCP，向 AI 提一个需要访问网络才能回答的问题（如总结某个官方文档页面），确认它能正常返回结果。

**下一章预告**：现在你已经掌握了所有核心工具——Agent、规范、SKILL、MCP。最后一章把它们整合起来，用 SDD 工具完整走一遍开发流程。
