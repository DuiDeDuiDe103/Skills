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

ChatGPT / Codex 可直接复制 SKILL.md 内容到 Custom Instructions。

## 当前技能

| 技能 | 类别 | 说明 |
|------|------|------|
| [progressive-teaching](skills/teaching/progressive-teaching/) | 教学 | 循序渐进分步教学，每次只教一个知识点 |

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
