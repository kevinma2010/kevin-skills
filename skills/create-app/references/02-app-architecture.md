# 02 — 应用架构：路由 / 主题 / 组件 / feature 组织

## 1. src/ 总体结构

```
apps/mobile/src/
├── app/            # Expo Router 路由（全部瘦包装）
├── features/       # 业务模块（UI+逻辑真正所在）
├── components/     # themed/ ui/ layout/ 三层通用组件
├── stores/         # zustand（见 03）
├── services/       # 数据/业务服务（见 03）
├── hooks/          # 跨 feature 通用 hooks（use-app-startup, use-color-scheme…）
├── providers/      # app-providers.tsx 等
├── constants/      # config.ts（env 唯一入口）、theme-colors.ts（re-export core）
├── i18n/           # 可选
├── utils/          # 纯函数 + 观测 wrapper + test-selectors/
└── global.css      # @tailwind base/components/utilities
```

文件/目录名一律 kebab-case。

## 2. 路由骨架（最小版）

```
src/app/
├── _layout.tsx          # 根 Stack + Gate + providers
├── +not-found.tsx
├── (tabs)/
│   ├── _layout.tsx      # Tabs
│   ├── index.tsx
│   └── settings.tsx
└── (auth)/              # 装 auth 时
    ├── _layout.tsx
    └── signin.tsx
```

约定：
- **路由文件 = 一行包装** `features/` 的组件；presentation 选项走工厂函数。
- 扁平 settings/detail 屏直接挂根 Stack 用 form-sheet 呈现，不嵌套导航器——导航树保持一层，深链和返回语义都简单。
- **SDK 57**：react-navigation API 一律从 `expo-router/react-navigation` 导入（`useFocusEffect`、`ThemeProvider`、`DarkTheme` 等）；`@react-navigation/*` 不进 package.json（expo-router 已内置）。
- 深链过滤：需要拦截非路由 URL（文件打开/支付回调）时建 `+native-intent.ts`，在 `redirectSystemPath` 里把它们从路由系统排掉，交给专门的 Linking listener。

**screen option 工厂**（`components/layout/navigation/`）：`createModalScreenOptions()`（全屏 modal 无 header）、`createFormSheetScreenOptions()`（带 header 的 modal，最常用）、`createStackScreenOptions()`。根 Stack `screenOptions` 集中供给：返回键、`headerTitleAlign: 'center'`、主题色、`headerShadowVisible: false`——所有屏幕视觉一致性来自这一处。

## 3. 根 layout 模板（import 顺序敏感）

```tsx
// src/app/_layout.tsx — order matters
import '@/utils/polyfills';        // 1. polyfills first
import '@/global.css';             // 2. NativeWind global stylesheet
// 3. Sentry.init(...) at module scope (装观测时)
// 4. void SplashScreen.preventAutoHideAsync() at module scope

export default function RootLayout() {
  const { fontsLoaded, storageReady, storageError } = useAppStartup();
  if (storageError) return <StartupStorageErrorScreen />; // 裸 RN 组件，不依赖任何 provider
  return (
    <AppProviders>
      <RootLayoutContent fontsLoaded={fontsLoaded} storageReady={storageReady} />
    </AppProviders>
  );
}
```

**Provider 栈**（`providers/app-providers.tsx`，外→内，规则：infra → 导航主题 → analytics → i18n → auth → UI overlay。内层 provider 常需要外层 context，顺序错了就是运行时炸）：

```
GestureHandlerRootView > SafeAreaProvider > KeyboardProvider > ActionSheetProvider
  > ThemeProvider(expo-router/react-navigation) > [AnalyticsProvider] > [I18nProvider]
  > [AuthProvider] > BottomSheetModalProvider > ModalProvider > {children} + AlertHost
```

**启动序列**（`hooks/use-app-startup.ts`）：`useFonts(APP_FONTS)` → `storageManager.initialize()`（MMKV）→ ready 标志。**Splash 只在 Gate 内 `fonts && storage && auth` 全 ready 后 hide**（首帧前完成全部初始化，无闪白）；storage-error 路径立即 hide 防卡死在 splash。

**全局 overlay hosts** 挂根（Stack 外）：跨切 overlay 的统一模板 = 「root 挂一次 host 组件 + store 暴露命令式 `triggerX()`」——付费墙、更新提示、alert 队列全用这个形状。

**env 唯一入口**：`constants/config.ts::Config` 读 `Constants.expoConfig?.extra` 并 fallback `process.env.EXPO_PUBLIC_*`（EAS 构建与 Metro dev 双路径都通）。feature 代码禁止裸读 `process.env`。

## 4. Theming（单源 tokens + Themed 组件族）

### tokens 单源

```ts
// packages/core/src/design/colors.ts — 平台无关，纯 hex，无 RN/DOM import
export const THEME_COLORS = {
  light: {
    color: '#1A1A1A', background: '#FAF9F7', primary: '<brand>',
    destructive: '#E0524C', error: '#FF3B30',
    gray1: '#FCFCFC', /* ... gray12 — 12 级灰阶 */
  },
  dark: { /* ... */ },
} as const;
export type ThemeColorKey = keyof typeof THEME_COLORS.light;
```

mobile 侧 `constants/theme-colors.ts` **纯 re-export**（文件头注释明令禁止在此重定义）。品牌哲学：单一品牌色 + 12 级灰阶做层次，不引入第二品牌色；`destructive`（危险按钮）≠ `error`（表单校验）是两个语义、两个红。

### hooks

```ts
// use-theme-color.ts 核心逻辑
export function useThemeColor(
  token: ThemeColorKey | SemanticAlias,        // 语义别名: text→color, separator→gray4…
  overrides?: { light?: string; dark?: string } // 调用点覆盖
) {
  const scheme = useColorScheme();              // 解析顺序: 特殊模式 → 手动覆盖 → 默认
  return overrides?.[scheme] ?? THEME_COLORS[scheme][resolveAlias(token)];
}
```

`useColorScheme()` 的默认值是**显式产品决策**（Step 0 问过）：固定 `'light'`+手动切换，或跟随系统。预留特殊模式（如 eink）抢占位。

### Themed 组件族（`components/themed/`，强制替代裸 RN 原语，写进 AGENTS.md）

最小集起步：

- `ThemedView` — `card` prop → surface 样式（背景+圆角+阴影，特殊模式下可换 hairline border）。
- `ThemedText` — `type`（h1-h6/body1/body2/caption/label 排版表：fontSize+lineHeight+weight 一次定死）+ `weight` 覆盖 + `variant` 语义色（default/muted/error/success/link）。`allowFontScaling={false}` + 自建 app 级 fontScale——OS 字体缩放对精排 UI 是布局炸弹，自己管。
- `ThemedButton` — `variant`（primary/secondary/outline/ghost/danger）×`size`。
- `ThemedTextInput`、`ThemedScrollView`。

其余（Switch/Checkbox/Modal/FlatList…）需要时再加。要点：**排版和颜色的自由度收在组件 props 里**，屏幕代码不出现裸 fontSize/hex。

### NativeWind 用途界定（防漂移）

- className 只管**布局/间距/排版工具类**（`flex-1 p-4 rounded-xl`）。
- 品牌/scheme 相关颜色一律走 `useThemeColor`/Themed 组件。
- **tailwind.config.js 不硬编码品牌色**。生产教训：config 里残留的模板默认色和 global.css 陈旧色值与 token 真相漂移数月才被发现——因为恰好没人消费。要么从 colors.ts 生成，要么不写。

## 5. 布局原语（`components/layout/`）

- `ScreenWrapper` — 屏幕壳一件套：safe-area edges + 可选 scroll + 下拉刷新 + loading/error/empty 状态切换 + iOS 键盘规避。每个屏幕 body 从它开始，杜绝每屏手拼这五件事。
- `common/`：`EmptyState` / `ErrorState`（props: `error, title, message, onRetry`，缺省 fallback 到 i18n 通用 key）/ `LoadingState`。
- 规则：能用系统 `Alert.alert` / `ActionSheetIOS` 就不自建 modal——OS 已经做得更好的交互不要重造。

## 6. Icon 注册表

`components/ui/icons.ts` 单文件：项目自有的 `IconName` union → 图标库组件映射；`icon.tsx` 薄包装统一 size/color 接口。全 app 禁止 ad-hoc import 图标库——换图标库时只改一个文件，且 IconName 是类型安全的。

## 7. 字体策略

- UI 字体与内容字体（阅读正文类）分离；内容衬线字体绝不用于 UI chrome。
- iOS 不捆 UI 字体（系统栈 SF Pro + 各语言 fallback 已最优）；Android 视情况捆（系统 Roboto 不够时，如 Inter 4 个字重）。
- 平台选择集中一个 helper：`uiFont(weight) → {fontFamily?, fontWeight}`（fontFamily 只在 Android 设），tailwind `fontFamily` 镜像同一逻辑。

## 8. Feature 模块内形

```
features/<name>/
├── index.ts          # barrel，只导出公共面
├── components/       # 大屏加 components/sections/（每组一个 *-section.tsx + orchestrator 组合）
├── hooks/
├── screens/          # 可选：feature 拥有整页组件时
└── __tests__/        # 每层共置
```

- 业务逻辑扁平在 feature 根，展示进 `components/`，横切 effect 进 `hooks/`。
- 「很多设置行」模式：通用 `setting-item.tsx` 行原语（toggle/navigate/value 三形态）+ sections 组合。
- 跨 feature 只许 import 对方 barrel。

## 9. Gate 模式（无 auth 也适用 onboarding gate）

决策 = 纯函数（无 React、可穷举单测）：

```ts
// features/auth/gate-redirect.ts
export function resolveGateRedirect(input: {
  ready: boolean;
  hasUser: boolean;
  inAuthFlow: boolean;
  onboardingPending: boolean;
}): Route | null {
  if (!input.ready) return null;
  if (!input.hasUser) return input.inAuthFlow ? null : '/(auth)/signin';
  if (input.onboardingPending) return '/onboarding';
  return input.inAuthFlow ? '/(tabs)' : null;
}
```

执行 = 瘦组件：`<Gate>` 包住根 Stack，not-ready 时 `return null`（撑住 splash），effect 里 **永远 `router.replace` 不 `push`**（gate 重定向不该进返回栈）。启动副作用（深链、分享导入、前台同步触发）也统一挂在 Gate 里——app 里唯一的「启动完成后做事」的地方。

## 10. TEST_IDS 注册表（装 E2E 时必须，不装也建议）

`utils/test-selectors/`：集中的 `TEST_IDS` 常量对象，kebab-case `domain-surface-element` 命名，动态 id 走 sanitize helper。E2E 选择器优先 `id:` 不用 locale 文本——多语言 app 里文本选择器就是 flaky 之源。
