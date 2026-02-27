# 第三章：Agent（让 AI 真正干活）

## 3.1 引言

**本章目标**：理解 Agent 模式，让 AI 从"回答问题"变成"完成任务"。

AI 编程工具不只是一个聊天窗口。当你给它正确的授权，它可以自主读代码、写代码、运行测试、修复报错，直到任务完成——全程无需人工干预。这就是 Agent 模式，也是本书所有后续技巧的基础。

---

## 3.2 Agent 是什么

**传统 AI 使用方式**：你问 → AI 答 → 你复制 → 你手动执行。每一步都需要人介入。

**Agent 模式**：你描述目标 → AI 自主循环（读 → 写 → 运行 → 看结果 → 修复）→ 任务完成。

Agent = 大模型 + 工具 + 自主循环。工具赋予 AI 真正的"行动能力"：读写文件、执行终端命令、运行测试、查看输出。

**Claude Code 本身就是一个 Agent**：它能读写文件、在终端运行命令、编译代码、执行测试、修复报错，全程自主。本书后续所有技巧——Token 管理、规范文件、SKILL、MCP——本质上都是在让这个 Agent 更好用。

---

## 3.3 案例：小明的测试修复

小明在公司电商系统里负责维护折扣计算模块。周五下午，CI 流水线突然报红——`calculate_discount()` 的测试挂掉了。

**Before（以前的做法）**

小明切到终端，跑 `pytest`：

```
FAILED tests/test_discount.py::test_bulk_discount - AssertionError: 0.15 != 0.2
```

他盯着错误看了一会儿，不确定是测试写错了还是函数本身有 bug。打开 `discount.py`，翻了一遍逻辑，又打开 `test_discount.py` 对比。发现是批量折扣的边界条件——数量 ≥10 件时折扣率计算有问题。改了代码，再跑，又挂了另一个测试。再查，再改，来回三轮，20 分钟过去了。

**After（Agent 模式）**

同样的问题，小明在 Claude Code 里说了一句话：

> 为 `calculate_discount()` 函数补全单元测试，确保全部通过。

然后他去倒了杯水。回来看到 Claude 的操作记录：

```
● Read(discount.py)
  读取折扣计算函数，发现批量折扣逻辑：数量 ≥10 时应返回 20% 折扣

● Read(tests/test_discount.py)
  现有 3 个测试，缺少批量折扣边界条件的覆盖

● Edit(tests/test_discount.py)
  新增 test_bulk_discount_boundary 测试用例

● Bash(pytest tests/test_discount.py)
  FAILED test_bulk_discount - 期望 0.20，实际返回 0.15

● Read(discount.py)
  定位问题：第 23 行条件判断用了 > 而非 >=，漏掉了恰好 10 件的情况

● Edit(discount.py)
  将 if quantity > 10 改为 if quantity >= 10

● Bash(pytest tests/test_discount.py)
  5 passed in 0.31s ✓
```

整个过程 47 秒。

Claude 的终端不是"显示输出"的窗口——它是 Claude 的工作台。它读文件、改代码、跑命令、看结果，这个循环完全自主进行。你只需要说清楚目标，不需要告诉它每一步怎么做。

---

## 3.4 要点

- **描述目标，而非步骤**：目标清晰（"确保测试通过"）比步骤清晰（"先写测试，再运行"）更能激发 Agent 能力
- **设置合理边界**：给 Agent 授权前想清楚它可以操作哪些文件、能否执行破坏性命令
- **能力上限取决于工具**：后续章节将介绍如何通过 SKILL（Ch7）和 MCP（Ch8）进一步扩展 Agent 能力

---

## 3.5 本章小结

- Agent = 大模型 + 工具 + 自主循环，核心是让 AI 自主执行，而非只生成文本
- 目标驱动优于步骤驱动
- Claude Code 是你的第一个 Agent

**练习**：在你自己的项目中，交给 Agent 一个明确的小任务（如"为某个函数写测试并确保通过"），观察它的自主执行过程，记录它完成了哪些步骤。

**下一章预告**：Agent 自主运行的代价是 token 消耗——下一章讲清楚 token 是什么，以及如何用最低成本让 Agent 持续工作。
