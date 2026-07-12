# 01 — 仓库与工具链骨架

## 1. Monorepo 骨架

即使只有一个 app 也建 monorepo：共享包 TS 直出零构建成本，为未来 desktop/extension 留位。

```
<repo>/
├── apps/mobile/
├── packages/core/
├── scripts/           # validate-changed.mjs, verify-agent-instructions.mjs
├── .github/workflows/quality-gate.yml
├── pnpm-workspace.yaml
├── .prettierrc  .prettierignore  .gitignore
├── AGENTS.md  CLAUDE.md -> AGENTS.md
└── package.json
```

**pnpm-workspace.yaml**（关键：hoisted）：

```yaml
packages:
  - apps/*
  - packages/*
nodeLinker: hoisted
onlyBuiltDependencies:
  - esbuild
```

选型理由：`nodeLinker: hoisted` 是 Metro monorepo 配置的前提（依赖平铺在根 node_modules，Metro 无需理解 pnpm symlink 结构）；副作用是共享 bin（prettier/eslint/tsc）都在根 `node_modules/.bin`，根目录 `npx` 直接可用。

**根 package.json**：`packageManager: pnpm@10.x`；scripts 只放薄包装：`mobile:lint|typecheck|test` = `pnpm --filter <app-name> <script>`；`prepare: husky`；`verify:agent-instructions`。**不放 build/dev/deploy**——根 scripts 是「验证命令目录」，不是操作面板。

**.gitignore** 必含：`apps/mobile/android/`、`apps/mobile/ios/`。原生目录不入库（Expo CNG，`expo prebuild` 随时再生），一切原生定制走 config plugins——杜绝原生工程手工漂移。

## 2. 创建 Expo app 并改造

```bash
pnpm create expo-app@latest apps/mobile --template default
```

### app.config.ts（删掉 app.json，全 TS 动态配置）

```ts
import packageJson from './package.json';

const withEnvSuffix = (base: string) => {
  const env = process.env.EXPO_PUBLIC_ENV;
  if (env === 'development') return `${base}.dev`;
  if (env === 'preview') return `${base}.preview`;
  return base;
};

export default {
  expo: {
    name: withDisplaySuffix(packageJson.displayName),   // "(Dev)" / "(Preview)"
    version: packageJson.version,                        // 单一版本真相
    ios: { bundleIdentifier: withEnvSuffix(BASE_BUNDLE_ID) },
    android: { package: withEnvSuffix(BASE_BUNDLE_ID) },
    scheme: withEnvSuffix(BASE_SCHEME),
    experiments: { typedRoutes: true },
    extra: { /* 集中透传 EXPO_PUBLIC_*，运行时唯一入口见 02 的 Config */ },
    plugins: [
      'expo-router',
      ['expo-splash-screen', { image: './assets/images/splash-icon.png', imageWidth: 120, backgroundColor: '#FFFFFF' }],
      ['expo-build-properties', { ios: { deploymentTarget: '16.4' } }],
      'expo-font',
      'expo-secure-store',
    ],
  },
};
```

要点：
- dev/preview/prod 三环境靠 bundle id + scheme 后缀共存于同一台设备——上线后你一定需要。
- `main: "expo-router/entry"`（package.json）。
- **SDK 57**：不写 `newArchEnabled`（已是唯一架构）、不写 `android.edgeToEdgeEnabled`（已强制默认）。

### metro.config.js（monorepo 三件套 + NativeWind）

```js
const { getDefaultConfig } = require('expo/metro-config');
const { withNativeWind } = require('nativewind/metro');
const path = require('path');

const projectRoot = __dirname;
const monorepoRoot = path.resolve(projectRoot, '..', '..');
const config = getDefaultConfig(projectRoot);

config.watchFolders = [...config.watchFolders, path.join(monorepoRoot, 'packages')]; // merge, don't replace
config.resolver.nodeModulesPaths = [
  path.resolve(projectRoot, 'node_modules'),
  path.resolve(monorepoRoot, 'node_modules'),
];
config.resolver.unstable_enablePackageExports = true;

module.exports = withNativeWind(config, {
  input: './src/global.css',
  configPath: path.resolve(__dirname, 'tailwind.config.js'),
});
```

### babel.config.js

只 `presets: ['babel-preset-expo']`。NativeWind 走 Metro 不走 Babel。

### tsconfig.json

```jsonc
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "skipLibCheck": true,
    "jsx": "react-native",
    "types": ["jest", "node"],        // TS 6 必需
    "paths": { "@/*": ["./src/*", "./*"] }
  },
  "include": ["**/*.ts", "**/*.tsx", ".expo/types/**/*.ts", "expo-env.d.ts", "nativewind-env.d.ts"]
}
```

伴随文件：`nativewind-env.d.ts`（`/// <reference types="nativewind/types" />`）、`global.d.ts`（`declare module '*.css';`）。

### eslint.config.js（flat, ESLint 9）

`eslint-config-expo/flat` + `eslint-config-prettier`，再加两组：

```js
// 质量监控（warn 不 error——是雷达不是墙；测试文件里关掉）
rules: {
  complexity: ['warn', { max: 15 }],
  'max-lines': ['warn', { max: 500, skipBlankLines: true, skipComments: true }],
  'max-lines-per-function': ['warn', { max: 100, skipBlankLines: true, skipComments: true, IIFEs: true }],
  'max-depth': ['warn', 4],
  'max-params': ['warn', 5],
}
```

未启用 React Compiler 时，把 `eslint-config-expo` 57 新带的 `react-hooks/globals|immutability|preserve-manual-memoization|refs|set-state-in-effect` 显式 `off`。

**lint script = `expo lint --max-warnings 0`**。经验：warning 债一旦开始积累就只能靠痛苦的 ratchet 机制往回收，新项目从第一天零容忍。

### jest（SDK 57 形态）

```js
module.exports = {
  preset: 'jest-expo',                                    // SDK 57：不再是 jest-expo/node
  resolver: 'react-native-worklets/jest/resolver',        // reanimated 4.5 worklets 拆包必需
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/jest.setup.js'],
  moduleNameMapper: { '^@/(.*)$': '<rootDir>/src/$1' },
  transformIgnorePatterns: [
    'node_modules/(?!(jest-)?react-native|@react-native|expo(nent)?|@expo(nent)?/.*|react-navigation|@react-navigation/.*|react-native-svg)',
  ],
  clearMocks: true,
  watchman: false,
  coverageThreshold: { global: { branches: 75, functions: 80, lines: 80, statements: 80 } },
};
```

`jest.setup.js` 要覆盖的 mock 面（jsdom 下没有原生桥，缺一个就是一类诡异报错）：

- env shims：`global.__DEV__ = true`、假 backend env vars、`window.matchMedia` polyfill、`TextEncoder/TextDecoder`（Node util）、**`setImmediate/clearImmediate` → setTimeout fallback**（SDK 57 jest-expo 不再提供）。
- `jest.mock('expo', ...)`：stub `requireNativeModule`/`requireOptionalNativeModule`/`requireNativeView`。
- 逐包 mock 用到的 expo-*：constants/font/splash-screen/haptics/linking/secure-store/localization…（用到什么 mock 什么）。
- 第三方原生：`react-native-reanimated`（官方 mock）、`react-native-safe-area-context`（固定 insets）、`@gorhom/bottom-sheet`（假 modal ref）、`react-native-gesture-handler`（`__mocks__/` 手写 manual mock）。
- 观测 SDK 全 no-op mock，analytics 假 client 暴露给测试断言。

Mock 陷阱（生产踩过）：partial-mock `react-native` 的 `AppState` 必须 `...actual.AppState` 先展开再覆盖单个方法——jest-expo 57 需要 `currentState` 等字段存在。测 `await import()` 动态导入的代码时，需要一个 babel 插件把 `import(x)` 重写为 `Promise.resolve().then(() => require(x))`（且只对白名单文件生效，避免影响 `jest.mock` 语义）。

## 3. packages/core（TS 直出）

```jsonc
// packages/core/package.json
{
  "name": "@<scope>/core",
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "sideEffects": false,
  "exports": { ".": "./src/index.ts" },
  "scripts": { "test": "vitest --run", "typecheck": "tsc --noEmit" }
}
```

无 build step；app 用 `workspace:*` 依赖，改动即时生效。公共 API 全走 `src/index.ts` barrel，未导出即 private。初始只放 `design/`（tokens）+ `utils/`，其余按需长出来。vitest 注意：Node 22 的 global `localStorage` 会遮蔽 jsdom，`setupFiles` 里显式 polyfill（见 06 §10）。

**core/app 边界**：core 只放解析/模型/类型/纯业务逻辑/常量/settings schema/design tokens——凡是能脱离 RN 在 vitest 里测的。UI、导航、平台 API、样式框架留 app。

## 4. Git hooks + CI

- husky：`pre-commit` = agent-instructions 校验 + `lint-staged`（prettier+eslint --fix）；`commit-msg` = commitlint（`<type>(<scope>): <subject>` 小写无句号）；`pre-push` = 变更范围 lint + 禁直推 main（escape hatch: `ALLOW_PUSH_TO_MAIN=1`）。**`pre-commit` 第一行写 `unset GIT_DIR GIT_WORK_TREE`**——否则 linked git worktree 里 lint-staged 误判 toplevel 假失败（见 06 §11）。
- **validate-changed.mjs**（自写，~100 行）：CI 与 pre-push 共用的 monorepo 变更检测器。核心逻辑：`git diff merge-base` 拿变更文件 → 按前缀映射到 workspace（`apps/mobile/` → mobile 的 lint+typecheck+test）→ **共享包变更只级联 typecheck 到 dependents**（不跑全量测试，省 CI 时间又抓类型断裂）→ 无匹配则 exit 0。
- `.github/workflows/quality-gate.yml`：PR 触发 + per-ref concurrency cancel，`fetch-depth: 0`（merge-base 需要），pnpm install --frozen-lockfile → agent-instructions 校验 → `node scripts/validate-changed.mjs --base origin/<base>`。

## 5. AGENTS.md 规范

- 根 + apps/mobile + packages/core 各一份 `AGENTS.md`（嵌套只写 delta，不重复根规则），同目录 `CLAUDE.md -> AGENTS.md` symlink——一份内容适配所有 agent harness。
- **verify-agent-instructions.mjs**（自写，~60 行）：遍历全仓，任何含 AGENTS.md/CLAUDE.md 的目录必须两者都在、CLAUDE.md 是 symlink 且指向同目录 AGENTS.md，否则 fail。接进 CI + pre-commit，文档漂移零成本防住。
- 根 AGENTS.md 内容 = 索引不是百科：结构图、验证命令（只 lint/typecheck/test）、代码规范要点、git 约定。

## 6. EAS（可选模块 ⑤）

`eas.json`：`cli.appVersionSource: "remote"`（EAS 管 build number，本地 package.json 只管语义版本）；profiles：`development`（dev client, internal, iOS simulator）/ `preview`（internal, Android apk）/ `production`（store, `autoIncrement: true`）。多分发渠道（appstore / googleplay / 官网 sideload apk）需要时给每个渠道独立 production profile + `EXPO_PUBLIC_DISTRIBUTION_CHANNEL` env，运行时据此切换计费方式与更新提示文案。
