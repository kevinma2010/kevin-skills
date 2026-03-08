# kevin-skills

Kevin 的个人 Skill 集合，用于 Claude Code 等 AI 助手。

## Skills

| Skill | 说明 |
|-------|------|
| [gemini-researcher](./skills/gemini-researcher/) | 用 Gemini CLI 进行研究任务的小助手 |

## 安装

```bash
git clone https://github.com/kevinma2010/kevin-skills.git
cd kevin-skills

# 将 skill 加入 PATH（以 gemini-researcher 为例）
ln -s $(pwd)/skills/gemini-researcher/bin/gemini-researcher /usr/local/bin/
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
