# Agent Skills

个人 AI Agent 技能集合，跨平台复用。

## 使用方式

每个技能是一个包含 `SKILL.md` 的文件夹，按类别组织在 `skills/` 目录下。

要使用某个技能，把对应的文件夹复制到你的 Agent 平台的 skills 目录即可：

| 平台 | 目标路径 |
|------|----------|
| QoderWork CN | `~/.qoderworkcn/skills/<技能名>/` |
| QoderCLI CN | `~/.qoder/skills/<技能名>/` |
| Claude Code | `~/.claude/skills/<技能名>/` |
| WorkBuddy | `~/.workbuddy/skills/<技能名>/` |
| Codex | `~/.codex/skills/<技能名>/` |

Codex 安装后可通过 `$<技能名>` 主动调用；符合 `SKILL.md` 中的触发描述时也可以自动启用。

## 当前技能

| 技能 | 类别 | 说明 |
|------|------|------|
| [progressive-teaching](skills/teaching/progressive-teaching/) | 教学 | 根据问题难度和用户反馈自适应讲解深度的循序渐进教学 |
| [reverse-design](skills/teaching/reverse-design/) | 教学 | 逆向工程式学习——先从零设计再对比源码，深入理解项目架构 |
| [csdn-publish](skills/publish/csdn-publish/) | 发布 | 通过浏览器自动化把本地 Markdown 发布到 CSDN 草稿箱 |
| [technical-article-analyzer](skills/analysis/technical-article-analyzer/) | 分析 | 从系统架构视角分析技术文章的定位、流程、设计选择与工程风险 |

## 添加新技能

在 `skills/<类别>/<技能名>/` 下创建 `SKILL.md`：

```markdown
---
name: my-skill
description: 一句话描述这个技能做什么、什么时候触发
---

# 技能标题

## 指令
具体内容...
```

## License

MIT
