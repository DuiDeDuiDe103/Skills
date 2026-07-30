# 项目维护规范

这份文档用于维护、迁移和交接本仓库。接手者不需要知道最初的创建过程，但必须理解这里的组织规则：本仓库保存的是可迁移的 Agent Skill 源文件，不是某一个平台的私有配置备份。

## 项目定位

本仓库是个人 AI Agent 技能集合，目标是让同一批 Skill 可以在 Codex、Claude Code、Qoder、WorkBuddy 等平台之间迁移复用。

核心资产是每个 Skill 目录中的 `SKILL.md`。平台本地安装目录只是运行副本，仓库中的 `skills/` 才是长期维护源。

## 目录规则

所有 Skill 必须放在 `skills/<类别>/<技能名>/` 下。

当前类别约定：

| 类别 | 用途 |
|------|------|
| `analysis` | 文章、代码、系统分析类 Skill |
| `design` | 前端、视觉、产品设计类 Skill |
| `publish` | 发布、平台自动化类 Skill |
| `teaching` | 教学、学习、理解辅助类 Skill |

每个 Skill 至少包含：

```text
skills/<类别>/<技能名>/
└── SKILL.md
```

如 Skill 需要 Codex UI 元数据，可以包含：

```text
agents/openai.yaml
```

如 Skill 来自外部仓库或第三方项目，必须保留其许可证文件，例如：

```text
LICENSE.txt
```

不要在 Skill 目录里放无关说明、临时脚本、调试输出或下载缓存。确实需要辅助资料时，优先使用 `references/`、`scripts/`、`assets/` 这些语义清晰的目录名。

## Skill 编写规则

`SKILL.md` 顶部必须有 frontmatter：

```markdown
---
name: skill-name
description: 清楚说明 Skill 做什么，以及什么时候应该触发
---
```

维护时重点检查三件事：

1. `name` 是否与目录名一致；
2. `description` 是否包含真实触发场景；
3. 正文是否写的是可执行规则，而不是泛泛的理念。

好的 Skill 应该告诉 Agent：

- 什么场景启用；
- 先做什么，后做什么；
- 什么时候不要做；
- 需要读取哪些文件或工具；
- 输出应该长什么样；
- 如何验证结果。

不要把 Skill 写成普通文章。Skill 是运行时指令，核心是让另一个 Agent 能稳定复现行为。

## 来源和授权规则

README 的技能表必须维护“来源”列。

来源分为三类：

| 来源类型 | README 写法 | 额外要求 |
|----------|-------------|----------|
| 本仓库原创 | `本仓库` | 默认按仓库 License 管理 |
| 外部仓库引入 | 上游仓库链接和许可证 | 保留上游许可证文件 |
| 外部内容改写 | 标明参考来源 | 在 Skill 中避免大段照搬原文 |

引入第三方 Skill 时必须做到：

- 保留原始许可证；
- README 标明上游链接；
- 不删除上游版权或授权信息；
- 如果做了修改，在提交说明中写明修改点；
- 不把不确定授权的内容同步到公开远程仓库。

例如 `frontend-design` 来自：

```text
https://github.com/anthropics/skills/tree/main/skills/frontend-design
```

因此仓库中保留了它的 `LICENSE.txt`，README 中也标明了 `anthropics/skills` 和 `Apache-2.0`。

## 本地安装和同步规则

仓库路径：

```text
D:\Skills
```

Codex 本地 Skill 安装路径：

```text
C:\Users\ASUS\.codex\skills\<技能名>\
```

维护原则：

- 修改源文件时，先改 `D:\Skills\skills/...`；
- 需要立即在 Codex 中使用时，再同步到 `C:\Users\ASUS\.codex\skills/...`；
- 同步后用哈希或文件对比确认一致；
- 不要只改本地安装目录，否则后续迁移和 Git 历史会丢失。

PowerShell 同步示例：

```powershell
Copy-Item -Recurse -Force `
  D:\Skills\skills\teaching\progressive-teaching `
  C:\Users\ASUS\.codex\skills\progressive-teaching
```

校验示例：

```powershell
Get-FileHash `
  D:\Skills\skills\teaching\progressive-teaching\SKILL.md, `
  C:\Users\ASUS\.codex\skills\progressive-teaching\SKILL.md
```

两个哈希一致，才说明仓库版和 Codex 安装版同步成功。

## 迁移规则

迁移到新机器时，以 GitHub 远程仓库为准。

基本步骤：

1. 克隆仓库到目标机器；
2. 检查 `README.md` 的技能表和来源；
3. 将需要的 Skill 目录复制到目标平台的 skills 目录；
4. 保留每个 Skill 目录内的附加文件，例如 `agents/`、`LICENSE.txt`；
5. 在目标平台中用 `$<技能名>` 或自然触发语测试一次。

常见平台路径：

| 平台 | 目标路径 |
|------|----------|
| Codex | `~/.codex/skills/<技能名>/` |
| Claude Code | `~/.claude/skills/<技能名>/` |
| QoderWork CN | `~/.qoderworkcn/skills/<技能名>/` |
| QoderCLI CN | `~/.qoder/skills/<技能名>/` |
| WorkBuddy | `~/.workbuddy/skills/<技能名>/` |

迁移时不要只复制 `SKILL.md`。如果 Skill 带有 `agents/openai.yaml`、许可证、脚本或引用资料，必须整个目录复制。

## 新增 Skill 流程

新增 Skill 时按以下顺序：

1. 确定类别和目录名；
2. 创建 `skills/<类别>/<技能名>/SKILL.md`；
3. 如果是 Codex 个人 Skill，优先使用官方 `skill-creator` 初始化；
4. 如果需要 UI 元数据，生成 `agents/openai.yaml`；
5. 如果来自外部，保留 `LICENSE.txt` 并记录来源；
6. 更新 README 技能表；
7. 同步到本地平台目录；
8. 做一次最小触发测试；
9. 提交 Git，提交说明写清楚来源和行为变化；
10. 推送到远程仓库。

提交说明建议格式：

```text
feat: add <skill-name> skill

- add <skill-name> under skills/<category>
- document source/license in README
- sync local Codex install when applicable
```

修改已有 Skill 时，提交说明要写行为变化，而不只是写“update skill”。

## 修改已有 Skill 的判断标准

修改前先判断这次变化属于哪一种：

| 类型 | 处理方式 |
|------|----------|
| 行为规则变化 | 修改 `SKILL.md`，并在提交说明写明新行为 |
| 触发范围变化 | 同步修改 frontmatter 的 `description` |
| Codex 展示变化 | 检查或更新 `agents/openai.yaml` |
| 外部 Skill 更新 | 对比上游内容，保留许可证和来源 |
| 文档变化 | 更新 README 或本文件 |

如果 Skill 已经被安装到本地平台，修改后必须同步安装目录，否则当前 Codex 会继续使用旧版本。

## Git 维护规则

远程仓库是项目迁移和交接的权威来源。

日常检查：

```powershell
git status --short --branch
git remote -v
```

提交前必须确认：

- 只包含本次任务相关文件；
- README 技能表已更新；
- 外部来源和许可证已标明；
- 本地安装目录已按需同步；
- 没有临时文件、缓存、验证依赖被提交。

推荐提交粒度：

- 一个新 Skill 一个提交；
- 一个明确行为改动一个提交；
- README 和对应 Skill 可以放在同一个提交；
- 不把无关 Skill 的格式化修改混进同一个提交。

## 交接检查清单

交给别人维护前，至少确认：

- `README.md` 能说明项目用途、安装位置和当前 Skill 列表；
- `MAINTENANCE.md` 是最新的；
- 每个 Skill 都有 `SKILL.md`；
- 外部 Skill 保留许可证和来源；
- Git 工作区干净；
- 本地分支已经推送到远程；
- 接手者知道仓库源文件和平台安装副本的区别。

一句话记忆：

> `D:\Skills` 是源，平台 skills 目录是副本；README 说明有什么，MAINTENANCE 说明怎么维护，GitHub 负责迁移和交接。
