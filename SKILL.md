# kevin-skills

Kevin 的个人 Skill 集合。

## 包含的 Skills

| Skill | 说明 |
|-------|------|
| [gemini-researcher](./skills/gemini-researcher/) | 用 Gemini CLI 进行研究任务的小助手 |

## 安装

```bash
# 克隆仓库
git clone https://github.com/kevinma2010/kevin-skills.git

# 将 skill 的 bin 目录加入 PATH，或创建软链接
ln -s $(pwd)/kevin-skills/skills/gemini-researcher/bin/gemini-researcher /usr/local/bin/
```

## 目录结构

```
kevin-skills/
├── SKILL.md              # 本文件
└── skills/
    └── <skill-name>/
        ├── SKILL.md      # skill 文档
        └── bin/          # 可执行文件
```

## 添加新 Skill

1. 在 `skills/` 下创建新目录
2. 添加 `SKILL.md` 文档
3. 可执行文件放入 `bin/`
