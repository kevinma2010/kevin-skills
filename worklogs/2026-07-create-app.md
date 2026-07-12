# Worklog — create-app skill（2026-07）

记录本 skill 的产出过程与决策，供后续迭代参考。

## 目标

把一个成熟上架的 Expo/React Native 应用（双端、三分发渠道、Supabase 后端）的架构经验抽象成可复用的 create-app skill，让未来创建新应用时不重复踩坑。

## 产出方法

1. **多维度并行扫描源码库**（6 个只读探查 agent）：① 构建配置与工具链 ② 路由与 UI 主题体系 ③ 状态管理与本地数据层 ④ 后端集成（auth/sync/payments）⑤ 横切关注点（i18n/观测/E2E/monorepo 规范）⑥ Expo SDK 54→57 升级 delta。
2. 扫描报告蒸馏为 6 个 reference 文件，每个非常规决策补 why（选型理由），生产事故转为「现象→根因→规则」三段式。
3. 原始扫描报告归档在本地（`~/.claude/create-app-research/`），**不随本仓库发布**——含源项目内部细节，仅作 skill 维护时的溯源材料。

## 关键决策

| 决策 | 结论 | 理由 |
|---|---|---|
| 新应用形态 | 独立新仓库（非源 monorepo 内新 app） | 与源项目零耦合，通用性最大 |
| 模板深度 | 分层：核心必装 + 可选模块按需勾选 | 空模块 = 噪音；宁可后补 |
| **开源定位**（中途转向） | skill 自包含：不引用源仓库路径/代码，只保留经验、规范、模式、选型理由 + 骨架级代码示例 | 原版含「从源仓库拷贝文件」指令，开源后对外部用户无意义 |
| UI/功能模块复用深度 | 只复用模式 + 关键示例，不做代码拷贝清单 | 源项目组件与业务/品牌深度耦合；开源形态下模式才是可迁移的 |
| 版本策略 | skill 不做版本真相，标注校准基线（SDK 57），搭建时以 `npx expo install` 实际解析为准 | 版本号必然过期，机制不会 |
| 推送通知 | 只给选型指引 + 官方文档地址，不写集成细节 | 源项目未实际接线，没有真经验可沉淀，硬写就是抄文档 |

## 文件结构与分工

```
skills/create-app/
├── SKILL.md                        # 工作流、分层清单、工程纪律、10 条架构原则
└── references/
    ├── 01-scaffold.md              # monorepo + 全套工具链配置（含 SDK 57 gotchas）
    ├── 02-app-architecture.md      # 路由/provider 栈/主题 token/Themed 组件/feature 内形
    ├── 03-state-data.md            # zustand 薄店/MMKV 分层/SQLite 管理器+repository/测试模式
    ├── 04-backend-supabase.md      # auth（含 Apple/Google/审核账号）/Edge Functions/RLS/sync/付费（RevenueCat/Stripe）
    ├── 05-cross-cutting.md         # i18n/Sentry/PostHog/推送选型/更新检查/Maestro/发布/SDK 升级清单
    └── 06-hard-lessons.md          # 14 条生产事故与深坑（现象→根因→规则）
```

## 迭代提醒

- **SDK 升级时**：按 `references/05` §9 的核对清单过一遍（react-navigation 导入路径、jest preset/resolver、app.config 字段、eslint 新规则、mock 面），并更新 SKILL.md 的版本基线行。
- **新踩坑**：符合「≥半天排查或真实事故」标准的，按三段式补进 `06-hard-lessons.md` 对应分组。
- **外链健康**：references 里的官方文档地址（Expo/Supabase/RevenueCat/Stripe/google-signin）发布前抽查有效性。
- **已知缺口**（未来可补）：onboarding funnel 完整模式只带过一句；paywall 呈现层只有 overlay host 模式；深链 Universal Links 无实战经验（源项目未启用）。
- **待验证**：skill 尚未被真实「创建第二个 app」的任务完整走过一遍；首次实战后按暴露的问题修订工作流粒度。

## 时间线

- 2026-07-12：六维度扫描完成四个后暂停（等 Expo SDK 57 升级）。
- SDK 57 合并后：补齐状态/数据层扫描 + 升级 delta，产出 skill 初版（含源仓库拷贝清单）。
- 同期：确定开源定位，重写为自包含版；补 Apple/Google/RevenueCat 集成手册、硬教训清单、工程纪律、推送选型；迁入本仓库。
