---
name: create-app
description: 从零创建一款生产级移动应用（Expo/React Native 独立仓库）。沉淀自一个成熟上架应用的架构经验：pnpm monorepo + Expo Router + 主题 token 单源 + zustand/MMKV/SQLite 数据层 + 可选 Supabase auth/sync/payments/i18n/观测。Use when the user asks to 创建一款新应用 / 新 app / 起一个新项目 / scaffold a new mobile app / create-app.
---

# create-app — 生产级移动应用脚手架

本 skill 沉淀的是**经验、规范、模式与架构选型**，不是代码库：所有决策都在真实上架应用（iOS/Android 双端、App Store + Google Play + 自分发三渠道）中经过生产验证。产出物 = 一个全新的独立 monorepo 仓库。

**版本基线**：Expo SDK 57 / RN 0.86 / React 19.2 / TS 6.0（2026-07 校准）。搭建时用 `npx expo install` 解析实际版本，不要硬编码本 skill 中出现的版本号；SDK 升级时的核对清单见 `references/05-cross-cutting.md` §9。

## 工作流

### Step 0 — 收集参数（一次问齐，再动手）

| 参数 | 说明 |
|---|---|
| 产品名 / slug / 仓库名 | displayName、expo slug、repo 目录名 |
| bundle id / package | 如 `com.xxx.app`（dev/preview 环境自动加 `.dev`/`.preview` 后缀） |
| URL scheme | 深链 scheme |
| 品牌主色 | 建议「单一品牌色 + 12 级灰阶」哲学；确认一个 hex |
| 深色模式策略 | 跟随系统 vs 固定浅色+手动切换（显式决策，见原则 9） |
| 可选模块（多选） | ① Supabase auth+云同步 ② 付费（IAP/Stripe + credits）③ i18n 多语 ④ 观测（Sentry+PostHog）⑤ EAS 构建/发布 ⑥ Maestro E2E |

### Step 1 — 仓库与工具链骨架 → `references/01-scaffold.md`

pnpm monorepo（`apps/mobile` + `packages/core`）、Expo app 创建与改造（app.config.ts / metro / babel / tsconfig / eslint / jest）、husky+commitlint、CI quality-gate、AGENTS.md 规范。

### Step 2 — 应用架构 → `references/02-app-architecture.md`

Expo Router 路由骨架、root layout 与 provider 栈、design tokens 单源 + Themed 组件族、screen 工厂与布局原语、icon 注册表、字体策略、feature 模块内形。

### Step 3 — 状态与数据层 → `references/03-state-data.md`

zustand 薄店（无 persist）、MMKV 分层存储、SQLite 单连接管理器 + repository + 手写 migrations、per-user 文件存储、错误双通道、全套测试模式。

### Step 4 — 可选模块

- Supabase auth / sync / payments → `references/04-backend-supabase.md`
- i18n / Sentry / PostHog / 版本更新检查 / Maestro / 发布 → `references/05-cross-cutting.md`

**全程对照** `references/06-hard-lessons.md`——生产事故与深坑清单（WebView opacity、OS 备份撕裂同步游标、大文件 OOM、content:// URI、真库回放 migration 等），涉及对应场景时先读再写。

### Step 5 — 验证与首提交

1. `pnpm install` → `pnpm mobile:lint && pnpm mobile:typecheck && pnpm mobile:test` 全绿（lint 从 `--max-warnings 0` 起步，不给未来留 warning 债）。
2. `pnpm --filter <core-package> test` 绿。
3. 不启动 dev server / 模拟器（除非用户明确要求）。
4. `git init` + 初始 commit（`chore: scaffold <app> from create-app template`）。

## 分层清单

**核心必装**（每个新 app 都要）：monorepo 骨架、Expo Router + typedRoutes、design tokens 单源 + Themed 组件、zustand+MMKV、lint/typecheck/jest 全链、husky+CI、AGENTS.md 规范、错误双通道骨架（观测 wrapper 可先 no-op）。

**按需可选**：SQLite（有结构化本地数据才装）、Supabase auth+sync、payments、i18n、观测、EAS、Maestro。宁可后补也不预装空壳——空模块 = 噪音。

## 工程纪律（写进新 repo 的 AGENTS.md，机制而非口号）

- **bug 先写失败测试再修**——测试转绿即验收信号；偶现 bug 必须先做出确定性复现，禁止「看起来修好了」。
- **PR 验收清单**：自动测试覆盖了什么 + 剩余手动 QA 项（含真机项），写进 PR body。
- **pitfalls.md 活文档制度**：每次踩坑（≥半天排查或真实事故）立刻记一条「现象→根因→规则」；新 repo 第一天就建这个文件——本 skill 的 `references/06` 就是它的种子。
- **定时任务/批处理上线后 24h 查一次日志**，确认无持续失败模式。
- lint warning 零容忍从第一天开始（`--max-warnings 0`），债务只会单向增长。

## 不可违背的架构原则（生产教训提炼）

1. **路由文件是瘦包装**，UI/逻辑全在 `src/features/<name>/`。
2. **品牌色单源**在 `packages/core/src/design/colors.ts`；tailwind config 不硬编码品牌色（生产教训：config 残留模板默认蓝与陈旧色值，和 token 真相漂移数月无人察觉）。
3. **zustand 不用 persist middleware**：store = 内存投影；真持久化在 MMKV/SQLite 服务层，action 内 write-through。
4. **依赖单向**：UI → stores → services → packages → SDK。services 不 import React/zustand/stores。
5. **SQLite 单连接 + 写串行**（promise 链 + exclusive transaction + busy retry），per-user db 文件。
6. **网络同步引擎惰性构造**（getter 内 new），module load 期禁止实例化——否则破坏测试 partial-mock，且在用户登录前就创建了网络/存储对象。
7. **观测唯一入口 wrapper** + `setUser({id})` only（PII 铁律）+ 封闭 ErrorCategory；分析事件用 snake_case typed event map。
8. **Auth gate = 纯决策函数 + 瘦 effect 组件**（永远 `router.replace`）；splash 等 fonts+storage+auth 全 ready 才 hide。
9. **决策点显式过，不盲抄**：OTA（expo-updates vs 自建版本检查+强更能力）、跟随系统深色与否、特殊显示模式（如 eink）支持与否。
10. **AGENTS.md canonical + CLAUDE.md symlink** + verify 脚本进 CI，从第一天开始。
