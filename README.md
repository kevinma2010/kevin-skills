# kevin-skills

Kevin 的个人 Skill 集合，用于 Claude Code 等 AI 助手。

## Skills

| Skill | 说明 |
|-------|------|
| [gemini-researcher](./skills/gemini-researcher/) | 用 Gemini CLI 进行研究任务的小助手 |

## 安装

### Claude Code

```bash
npx skills add kevinma2010/kevin-skills
```

### OpenClaw

在聊天中告诉 OpenClaw：

> 帮我安装这个 skill：https://github.com/kevinma2010/kevin-skills

或使用命令行：

```bash
npx clawdhub@latest install kevinma2010/kevin-skills
```

## 目录结构

```
kevin-skills/
├── README.md
├── SKILL.md
└── skills/
    └── <skill-name>/
        ├── SKILL.md      # skill 文档
        └── bin/          # 可执行文件
```

## 添加新 Skill

1. 在 `skills/` 下创建目录
2. 添加 `SKILL.md` 说明文档
3. 可执行文件放入 `bin/`
