## ADDED Requirements

### Requirement: 第六章标题与定位
第六章标题 SHALL 改为「SDD 方法与工具介绍」，定位为方法论 + 工具的一体化介绍章节。

#### Scenario: 标题反映内容范围
- **WHEN** 读者看到第六章标题
- **THEN** 标题为「第六章：SDD 方法与工具介绍」

### Requirement: 保留引言与 SDD 核心概念
引言（小明案例）和「SDD 是什么」两节 SHALL 保持不变。

#### Scenario: 引言案例完整保留
- **WHEN** 读者阅读 6.1 引言
- **THEN** 小明评论功能返工案例完整呈现，无删减

#### Scenario: SDD 核心理念完整保留
- **WHEN** 读者阅读 6.2 SDD 是什么
- **THEN** 「人决策，AI 执行」核心理念及 Vibe Coding 对比完整呈现

### Requirement: PDCA 内容压缩为概览
PDCA 一节 SHALL 压缩至半页左右，保留四步骨架（Plan/Do/Check/Act），去掉详细提示词示例，结尾引出 OpenSpec 工具。

#### Scenario: PDCA 保留骨架
- **WHEN** 读者阅读 PDCA 节
- **THEN** 能看到 Plan/Do/Check/Act 四步的简要说明

#### Scenario: PDCA 引出工具
- **WHEN** 读者读完 PDCA 节
- **THEN** 最后一句说明「上述手动流程可由 OpenSpec 自动化」，自然过渡到工具章节

### Requirement: 新增 OpenSpec 工具实战章节
第六章 SHALL 新增一节介绍 OpenSpec，涵盖安装配置和扩展工作流。

#### Scenario: 安装与配置步骤完整
- **WHEN** 读者阅读安装配置部分
- **THEN** 包含：① npm 安装命令 ② `openspec config profile` 启用扩展工作流 ③ `openspec init --tools claude` 或 `openspec update` 初始化项目

#### Scenario: 扩展工作流命令全部介绍
- **WHEN** 读者阅读扩展工作流部分
- **THEN** 依次介绍：explore → new → continue（×3，分别生成 proposal/specs/design/tasks）→ apply → archive

#### Scenario: 案例演示完整走一遍流程
- **WHEN** 读者阅读案例部分
- **THEN** 使用「为博客系统添加评论功能」案例，完整展示 explore 到 archive 的每一步命令和输出

### Requirement: 删除第九章
`content/09-SDD工具实战.md` SHALL 被删除，其核心内容已迁移至第六章。

#### Scenario: 第九章文件不存在
- **WHEN** 改动完成后检查 content/ 目录
- **THEN** 不存在 `09-SDD工具实战.md` 文件

### Requirement: 其他章节交叉引用更新
所有引用「Ch9」或「第九章」的文字 SHALL 更新为指向第六章或直接删除。

#### Scenario: 无悬空的 Ch9 引用
- **WHEN** 全文搜索「Ch9」或「第九章」
- **THEN** 不存在指向已删除第九章的引用

### Requirement: 附录文件去除章节编号
附录文件 SHALL 从 `10-附录-延伸学习资源.md` 重命名为 `附录-延伸学习资源.md`，不再带数字前缀。

#### Scenario: 附录文件名无数字前缀
- **WHEN** 改动完成后检查 content/ 目录
- **THEN** 存在 `附录-延伸学习资源.md`，不存在 `10-附录-延伸学习资源.md`

#### Scenario: 附录内部标题与引用同步更新
- **WHEN** 检查附录文件内容及其他章节对附录的引用
- **THEN** 附录标题不含「第十章」字样，其他章节引用附录时无章节编号
