## Context

当前课程第六章介绍 SDD 方法论（手动 PDCA），第九章介绍 OpenSpec 工具实战。两章内容高度相关但被 Ch7（SKILL）和 Ch8（Agent）隔开，读者需等到第九章才能看到工具落地。

此次改动将两章合并，第九章删除。

## Goals / Non-Goals

**Goals:**
- 第六章成为 SDD 方法 + 工具的完整介绍
- 读者在一章内建立概念（Why）并掌握工具（How）
- 保留 PDCA 作为概念脚手架，压缩为过渡段落

**Non-Goals:**
- 不讲 SKILL 底层机制（Ch7 负责）
- 不讲 OpenSpec 自定义 Schema 等进阶用法
- 不覆盖 GitHub Spec Kit、Kiro 等其他工具

## Decisions

**决策 1：保留 PDCA 而非直接删除**
- 理由：PDCA 建立「规范先行」直觉，解释工具存在的 Why。无 PDCA 则工具章节缺乏铺垫，读者可能机械使用工具而不理解原理。
- 方案：压缩至半页，保留四步骨架，删除详细提示词示例，结尾加一句过渡语引出 OpenSpec。

**决策 2：介绍扩展工作流（explore/new/continue），不介绍 core 工作流（propose）**
- 理由：扩展工作流逐工件审查，更能体现「人决策」的 SDD 理念；propose 一步生成所有文档，决策点不够清晰。
- 方案：工作流从 explore 开始，new 创建变更，continue × 3 逐步生成 proposal/specs/design/tasks。

**决策 3：不在 Ch6 解释 SKILL 机制**
- 理由：Ch7 专门讲 SKILL，Ch6 提前讲会造成内容重叠；读者不需要理解 SKILL 就能使用 OpenSpec 命令。
- 方案：Ch6 只讲命令用法，在结尾或旁注说明「这些命令底层是 SKILL 文件，详见 Ch7」。

## Risks / Trade-offs

- **风险**：Ch6 篇幅增加，可能过长 → 缓解：PDCA 严格压缩，案例只展示关键步骤和输出，不逐行展开
- **风险**：读者在 Ch6 看到斜杠命令，Ch7 才讲 SKILL，可能困惑 → 缓解：加一行注释「斜杠命令的机制将在 Ch7 详细介绍」
- **风险**：其他章节引用 Ch9 的文字遗漏更新 → 缓解：tasks 中列为独立检查项

## Migration Plan

1. 改写 `content/06-SDD方法论.md`：调整标题、压缩 PDCA、新增 OpenSpec 节
2. 删除 `content/09-SDD工具实战.md`
3. 全局搜索「Ch9」「第九章」，更新或删除相关引用
4. 检查目录/前言文件是否需要更新章节列表
