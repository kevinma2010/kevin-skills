# 05 — 可选模块：i18n / 观测 / 更新检查 / E2E / 发布

## 1. i18n（可选模块 ③）

`react-i18next` + `expo-localization`。结构：

```
src/i18n/
├── index.ts        # init + changeLanguage
├── provider.tsx    # I18nProvider
└── locales/{en,zh-Hans,...}.json
```

- 跨端共享字符串放 `packages/core/src/i18n/shared.ts`，init 时 merge（平台键覆盖共享键）。
- Key 按 feature 嵌套；`{{var}}` 插值。
- 规则：**禁动态拼 key**（`t('error.' + type)` 会让「找出所有未翻译 key」变成不可能）；禁 HTML in strings（用 `<Trans>`）；新增字符串必须全 locale 齐上——宁可机翻占位也不留 key 缺失。
- 语言解析优先级：显式设置 → 引导期偏好 → 设备 locale → `'en'`；解析纯函数放 core（可脱 RN 测试）。

## 2. 错误观测（Sentry 类，可选模块 ④）

- init 在 `_layout.tsx` module scope（越早越好，才能抓到启动期崩溃）：`{dsn, environment, enabled: !__DEV__}` + 低基数 tags（渠道、地区等）。
- **唯一入口 wrapper** `utils/sentry.ts`：`captureError/captureWarning/addBreadcrumb/setUser`；app 代码禁止直接 import 观测 SDK（写进 AGENTS.md）。不装观测时同名 wrapper 先 no-op——调用面从第一天统一，后装 SDK 零改动。
- **PII 铁律**：`setUser()` 只发 `{id}`，永不发 email/名字；其余属性走 `setTag()` 低基数非 PII。
- `ErrorCategory` 封闭 union（按 app 的域定义：storage/auth/sync/import/…）；每次 capture 必带 `{category, operation}`——没有这两个字段的错误在后台就是垃圾流。
- 基数纪律：tags 禁文件名/URL/用户输入（高基数 tag 会毁掉聚合）；extra 可放 debug 上下文但禁 token/密码/PII。

```ts
captureError(error, {
  category: 'sync',              // 封闭 union
  operation: 'flushQueue',
  extra: { queueLength },        // debug 上下文，不可检索
  tags: { entity: 'notes' },     // 低基数，可检索
});
```

## 3. 产品分析（PostHog 类，可选模块 ④）

- Provider 挂 app 根（配置缺失时 disabled 优雅降级，dev 默认关）+ Effects 组件：AuthTracker（identify/reset，用 `distinctId + JSON(props)` 去重避免重复 identify）、ScreenTracker（路由变化自动 screen 事件）。
- **typed event map 约定**：

```ts
export const ONBOARDING_EVENTS = {
  setupStarted: 'onboarding_setup_started',
  // camelKey: 'domain_snake_case_event'
} as const;

export function captureAnalyticsEvent(client, event, properties) {
  // strips undefined; no-ops when client is null — 唯一的 capture 调用点
}
```

- 事件与属性都 snake_case；事件名按域分组常量对象，杜绝裸字符串散落。
- 不可信来源（webview 注入、用户内容）的遥测字段/值必须 allowlist 白名单校验后才 capture——防用户内容/PII 混进分析流。
- Fire-once 里程碑：先持久化 flag 再 capture（跨重挂载恰好一次）。
- 非 React 服务代码要用 client 时走 module-level setter 注入，不 import hook。

## 4. 版本更新检查（可选，先过 OTA 决策）

**决策点**：expo-updates OTA vs 纯商店发版 + 自建检查。自建路线（生产验证过的取舍：换取「强制升级」能力——OTA 更不了 native，出安全事故时你需要能硬拦旧版本）：

- `features/update/`：`use-update-check.ts` 调后端 `check-app-update`（`{platform, channel, version}`），节流 + per-version dismissal 持久化（用户忽略过的版本不再骚扰）。
- 软提示 modal + 硬阻断 modal（服务端 `force_update: true` 触发），root 挂载。
- 多渠道时 `channel` 参与判断（商店审核期与官网包节奏不同）。

选 expo-updates 则这套全不需要，但记住它对 native 变更无能为力。

## 5. 推送通知（可选，仅选型指引）

选型：Expo CNG 项目默认走 **`expo-notifications` + Expo Push Service**——一个 ExpoPushToken 同时覆盖 APNs/FCM，服务端只对接 Expo 一个 HTTP API（免费），凭据（iOS APNs key / Android FCM V1 service account）全部由 `eas credentials` 托管，零原生配置漂移。除非有厂商级直连需求（国内厂商通道、精细化送达统计），不要直接接 FCM/APNs 或引入 OneSignal。

接入要点：需要 dev build（Expo Go 不支持推送）；token 注册走后端 `register-push-token` 端点存 per-device 行；服务端发送用官方 server SDK 并处理 receipts（票据里才有送达失败原因）。

资料：
- 总览：https://docs.expo.dev/push-notifications/overview/
- 客户端接入：https://docs.expo.dev/push-notifications/push-notifications-setup/
- 服务端发送 + receipts：https://docs.expo.dev/push-notifications/sending-notifications/
- FCM V1 凭据：https://docs.expo.dev/push-notifications/fcm-credentials/

## 6. Maestro E2E（可选模块 ⑥）

- `apps/mobile/.maestro/*.yaml`，每 flow 独立可跑：`appId: ${APP_ID}` env 化（多环境 bundle id）+ `tags` 分平台。
- 选择器只用 `TEST_IDS` 注册表的 `id:`（见 02 §10），不用 locale 文本。
- **刻意不进 CI**——真机/模拟器冒烟是发版前手动验收层，进 CI 的性价比（设备农场成本+flake）不划算；CI 守 lint/typecheck/jest。
- Gotchas（真机踩过）：dev build 的 Metro LogBox toast 会挡点击（flow 启动后条件 dismiss）；Android 根 tab 硬件返回是退后台不是导航。

## 7. 质量工具（低成本，建议装）

- knip（死代码）：per-workspace entry/project globs——mobile entry 含 `src/app/**`（路由是隐式入口）+ 各 config 文件；core 就是 `src/index.ts`。结果当**triage 清单**用（路由/动态 import 有假阳性），不当硬门禁。
- jscpd（重复代码）：`minLines 10 / minTokens 70 / exitCode 0`——非阻断信息流，只为 review 时有个地图。
- 各包一个 `pnpm deadcode` / `pnpm dupcode` runner 脚本，日志统一进 `.tool-logs/`（gitignore）。

## 8. 发布（可选模块 ⑤，上店时再做）

- EAS profiles 见 01 §6；`appVersionSource: remote` + production `autoIncrement`。
- 商店元数据/提审自动化：App Store Connect API 与 Google Play Developer API 都可脚本化（pull/push 元数据、建版本、绑 build、提审、promote track）——上店后值得投入一次，此后发版全 CLI。
- 坑（生产事故换来的）：EAS submit 进行中**永不**并发调用 Play API 的 edits（会把提交工具持有的 edit 会话顶掉，提审直接失败）；EAS 云构建要求 lockfile 与 package.json 严格同步（CI frozen install）。

## 9. Expo SDK 升级注意（模板维护）

本模板校准于 SDK 57。升级到新 SDK 时重点核对：
- react-navigation 导入路径（57 起全走 `expo-router/react-navigation`，`@react-navigation/*` 不进 package.json）。
- jest preset 名 + resolver（57：`jest-expo` + `react-native-worklets/jest/resolver`）+ `setImmediate` polyfill。
- app.config 字段增删（57 删了 `newArchEnabled`/`edgeToEdgeEnabled`）。
- eslint-config-expo 新增规则（57 带 React Compiler 规则，未启用 Compiler 就显式 off）。
- 全套 mock 面重新过一遍测试（partial-mock 的模块最容易被 SDK 升级打碎）。
