# 第八章：MCP（模型上下文协议）

## 8.1 引言

**本章目标**：理解 MCP 是什么，知道什么时候需要安装它。

Claude Code 已经具备相当强的行动能力：读写文件、执行命令、访问网络。但有一类系统它无法直接操作——你的数据库、公司的私有仓库、内部 API。MCP 解决的是这个问题。

---

## 8.2 MCP 是什么

Claude Code 的能力分两层：

**内置工具**（开箱即用，无需配置）：文件读写、执行 Bash 命令、访问网络（WebFetch）。

**MCP 扩展**（按需安装）：数据库、GitHub API、Slack、自定义内部系统——这些不在内置范围内，需要安装对应的 MCP 才能操作。

**MCP（Model Context Protocol，模型上下文协议）**：Anthropic 于 2024 年 11 月发布的标准协议，让 AI 通过标准化接口连接外部工具和数据源。类比：MCP 相当于给 AI 装插件——内置工具是出厂配置，MCP 是按需扩展。

MCP 是让 Agent（Ch3）能力边界持续扩展的关键机制。

---

## 8.3 案例：小明从手动到自动

### 场景一：网页访问（内置能力，无需安装）

小明需要分析竞品的 API 文档，整理出和自己系统的差异。文档在对方的官网上，有十几个页面。

**Before（手动复制粘贴）**

小明的流程：打开对方网站 → Ctrl+A → Ctrl+C → 切到 Claude Code → Ctrl+V → 问问题。每个页面重复一次，十几个页面下来，手腕开始酸。而且粘贴过来的内容格式乱，图片缺失，AI 有时还会抱怨"这段内容格式不对，能重新提供吗"。

**After（直接给 URL）**

> 分析 https://example-api.com/docs/auth，和我们的 @app/routers/auth.py 做对比，列出我们缺少的功能

Claude 操作记录：

```
● WebFetch(https://example-api.com/docs/auth)
  获取页面内容，提取 API 端点列表

● Read(app/routers/auth.py)
  读取本地实现

● 分析对比结果：
  对方有：token 刷新接口、设备管理、多因素认证
  我们缺：token 刷新接口（/auth/refresh）
  我们多：自定义权限角色（对方没有）
```

整个分析从"给 URL"到"拿到对比结果"，不超过 30 秒。十几个页面的分析工作，一下午缩成了半小时。

这里不需要安装任何 MCP——WebFetch 是 Claude Code 的内置工具，直接可用。

---

### 场景二：数据库查询（MCP 的真正用武之地）

分析完竞品，小明还想查一下自己系统最近的用户数据——注册趋势、活跃度分布。数据在开发库里，他需要打开数据库客户端、写 SQL、跑查询、把结果粘回来再分析……

这也是重复劳动。但这次 Claude Code 没有内置能力直接操作数据库，这正是 MCP 出场的场景。

**安装 MySQL MCP（一条命令）**

```bash
claude mcp add mysql
```

配置开发库连接信息（只配开发库，不连生产环境），重启 Claude Code。

**After（直接问数据）**

> users 表里注册时间在最近 7 天的用户有多少？按天分布是怎样的？

Claude 操作记录：

```
● mysql(SELECT DATE(created_at), COUNT(*) FROM users
         WHERE created_at >= NOW() - INTERVAL 7 DAY
         GROUP BY DATE(created_at))

  返回结果：
  2025-01-20: 142 人
  2025-01-21: 156 人
  2025-01-22: 134 人
  ...
```

不需要打开数据库客户端，不需要写 SQL，不需要复制结果——直接问，直接拿答案。

装了 MySQL MCP，Claude 就能直接操作你的数据库。你用什么工具，AI 就能操作什么工具——每新增一个 MCP，Agent 的能力边界就扩展一次。

---

## 8.4 要点

- **内置 vs MCP**：WebFetch、文件操作、Bash 是内置的，无需配置；数据库、外部 API、私有系统需要安装对应 MCP
- **安全边界**：数据库类 MCP 只配置开发库，不要连接生产环境
- **适用场景**：调试、数据分析、信息检索；不适用：生产环境写操作
- **MCP vs SKILL（Ch7）**：SKILL 扩展 AI 的"思维方式"（提示词层面）；MCP 扩展 AI 的"行动能力"（连接外部系统）
- **MCP 生态**：Anthropic、各大云厂商和社区已发布大量现成 MCP，可在 Claude Code 官方文档的 MCP 章节或社区目录中查找

---

## 8.5 本章小结

- Claude Code 内置工具（WebFetch、文件操作、Bash）开箱即用，无需配置
- MCP = 给 AI 装插件，连接内置工具覆盖不到的外部系统
- 安装方式：`claude mcp add <name>`，配置连接信息，重启即可
- 每新增一个 MCP，Agent 的能力边界就扩展一次

**练习**：向 Claude Code 直接提一个需要访问网络的问题（如总结某个官方文档页面），确认内置 WebFetch 能正常工作。如果你有本地开发数据库，尝试安装对应的 MCP，让 Claude 直接查询一条数据。

**下一章预告**：现在你已经掌握了所有核心工具——Agent、规范、SKILL、MCP。最后一章把它们整合起来，用 SDD 工具完整走一遍开发流程。
