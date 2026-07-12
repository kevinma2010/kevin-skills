# create-app

一个 Claude Code skill：从零创建生产级移动应用（Expo/React Native 独立仓库）。

沉淀自一个成熟上架应用（iOS/Android 双端、App Store + Google Play + 官网 sideload 三分发渠道、Supabase 后端）的架构经验。里面不是代码模板，是**决策、规范、模式和真实事故换来的教训**——让 AI 按一套生产验证过的架构帮你搭新应用，而不是重新发明和重新踩坑。

## 里面有什么

| 文件 | 内容 |
|---|---|
| `SKILL.md` | 工作流（收集参数 → 分步搭建 → 验证首提交）、核心/可选模块分层、工程纪律、10 条不可违背的架构原则 |
| `references/01-scaffold.md` | pnpm monorepo + Expo 全套工具链配置（app.config / metro / eslint / jest，含 SDK 57 gotchas）、husky+CI、AGENTS.md 规范 |
| `references/02-app-architecture.md` | Expo Router 路由骨架、provider 栈顺序、主题 token 单源、Themed 组件族、feature 模块内形 |
| `references/03-state-data.md` | zustand 薄店（无 persist 的理由）、MMKV 分层存储、SQLite 单连接管理器 + repository + 手写 migrations、全套测试模式 |
| `references/04-backend-supabase.md` | Supabase auth（Email OTP / Apple / Google / 审核专用账号）、Edge Functions 骨架 + RLS 模板、离线同步引擎、RevenueCat / Stripe 付费 |
| `references/05-cross-cutting.md` | i18n、Sentry PII 纪律、PostHog 事件规范、推送选型、版本更新检查、Maestro E2E、发布、SDK 升级核对清单 |
| `references/06-hard-lessons.md` | 14 条硬教训，每条来自真实生产事故，按「现象 → 根因 → 规则」写（WebView opacity 性能灾难、OS 备份撕裂同步游标、大文件 OOM、content:// URI、RPC 必须真库回放……） |

技术栈：Expo SDK 57 / React Native 0.86 / TypeScript / pnpm monorepo / NativeWind / zustand / MMKV / expo-sqlite / Supabase（可选）/ RevenueCat（可选）。

版本策略：skill 标注校准基线（当前 SDK 57），但不做版本真相——搭建时以 `npx expo install` 实际解析为准，所以基线过期也不影响使用。

## 安装

### Claude Code（推荐）

```bash
npx skills add kevinma2010/kevin-skills
```

### 手动安装

把本目录拷到 Claude Code 的 skills 目录即可：

```bash
# 全局（所有项目可用）
git clone https://github.com/kevinma2010/kevin-skills /tmp/kevin-skills
cp -R /tmp/kevin-skills/skills/create-app ~/.claude/skills/create-app

# 或只装到某个项目
cp -R /tmp/kevin-skills/skills/create-app <your-repo>/.claude/skills/create-app
```

## 使用

装好后，在 Claude Code 里直接说：

> 创建一款新应用 / scaffold a new mobile app / 用 create-app 起一个新项目

它会先一次性问清参数（产品名、bundle id、品牌色、需要哪些可选模块——Supabase auth+同步 / 付费 / i18n / 观测 / EAS / E2E），然后按 references 分步搭建，最后跑 lint + typecheck + test 全绿并完成首提交。

也可以只把它当手册用：比如接 Apple 登录前读 `references/04` §2.1，能少翻很多官方文档和 Stack Overflow。

## 适合谁

- 用 Claude Code / AI agent 开发，想从第一天就有生产级架构的独立开发者
- 正在做 Expo + Supabase 全栈应用，想参考一套完整落地过的约定
- 不用 AI 也没关系——references 本身就是一份可读的架构决策记录

## 迭代

设计决策与已知缺口见仓库 [`worklogs/2026-07-create-app.md`](../../worklogs/2026-07-create-app.md)。欢迎提 issue 分享你的踩坑经验。
