# 06 — 硬教训清单（生产事故与深坑）

每条来自真实生产事故或数天级排查。格式：现象 → 根因 → 规则。搭建和后续迭代时全程对照。

## RN / UI

### 1. WebView 禁用 `opacity: 0`

**现象**：大文档 WebView 渲染 16s+，去掉一个样式后 <1s。**根因**：RN 层给 WebView 设 `opacity: 0` 触发 iOS 全量离屏合成（off-screen compositing）。**规则**：隐藏 WebView 用 `display: 'none'`、零尺寸或屏外定位；fade-in 动画（opacity 0→1）过渡期间同样触发；WebView **内部** HTML 元素的 CSS opacity 不受影响。

### 2. setState updater 状态未变必须返回原引用

**现象**：Jest 下渲染含（甚至不可见的）modal 的组件 OOM/SIGTERM。**根因**：`setX((prev) => ({ ...prev, flag: false }))` 在状态未变时也返回新对象；生产只是多一次渲染，但 reanimated 的 jest mock 下变成微任务级死循环。**规则**：所有 updater/patch-reducer 状态未变时 `return prev` 原引用；测试里渲染树可能含 modal 的组件，把 modal mock 掉。

## 数据与备份

### 3. OS 备份恢复会撕裂「DB + 同步游标」的一致性（最难排查的一条）

**现象**：换机/恢复备份后，云端有数据但客户端永不回填，无任何报错。**根因**：Android 云备份/换机迁移恢复了 MMKV（含 sync 游标/队列），但 SQLite DB 文件不在备份范围——客户端拿着旧游标自认「已同步到 T 时刻」，增量同步永久 no-op。**规则**：凡是「本地 DB + KV 存同步游标」的组合，必须用 config plugin 写 Android `backup_rules.xml`（≤API 30）**和** `data_extraction_rules.xml`（API 31+，云备份与 device-transfer 都要写）把游标/队列文件排除出备份；恢复后无游标 = 自动退化为全量同步，正确。auth session、偏好等无一致性依赖的 KV 照常备份。

### 4. 大文件全链路传路径，不传内容

**现象**：导入 100MB+ 文件闪退。**根因**：把文件整块读进 JS 内存（base64 后再膨胀 1/3）。**规则**：picker→校验→解析→存储全链路传 URI/路径；解析用流式/分块；native 模块接口设计成收路径不收 buffer。

### 5. Android `content://` URI 不是文件路径

**现象**：分享/「打开方式」进来的文件，直接当路径用各种失败且偶现。**根因**：content URI 只保证通过 ContentResolver 可读，没有稳定文件路径，授权还有时效。**规则**：收到 content URI 第一步 copy 进自己的 documentDirectory，之后只操作自己的副本。intent filter 要同时声明 `file` 和 `content` 两种 scheme。

### 6. 下载/导入内容用魔数嗅探，不信 URL 和 Content-Type

**现象**：同一功能对某些站点的文件永远识别失败。**根因**：脏 URL（无后缀/带 query）、重定向、错误 MIME 遍地都是。**规则**：读文件头魔数判断真实类型（ZIP/PDF/HTML…），URL 后缀和 Content-Type 只做提示不做依据。

## 后端（Supabase/Postgres）

### 7. 含 RPC/函数的 migration 必须真库回放写路径

**现象**：全套单测绿、冒烟绿，上线后写路径 100% 失败：`column reference "x" is ambiguous`。**根因**：PL/pgSQL 对 `RETURNS TABLE` 输出列做变量替换，与函数体引用的表列同名即撞车；mock 单测永远碰不到 PG 解析器。**规则**：输出列不与表列同名（避不开就 `ON CONFLICT ON CONSTRAINT xxx_pkey` 绕开列名推断）；migration push 后用真实参数经 PostgREST 直调回放**全部代码路径尤其写路径**，验证后清理测试行；冒烟只打到参数校验层等于没测。

### 8. 大 JSON 列查询必须加 limit

**现象**：项目进入 Unhealthy 需手动重启。**根因**：对含大 JSON 列（转写文本、内容块等）的表做无 limit 查询，带宽瞬间耗尽。**规则**：大 JSON 列查询必带 `limit`（批量走 10-20 行分页）；只要元数据就只 select 需要的列，禁 `select *`。

## 定时任务 / 付费 API

### 9. queue/claim 类任务必须有 max_attempts；付费 API 批处理必须有熔断

两次真实烧钱事故：① claim 函数无重试上限 + 一处 LLM 返回格式改坏 → 失败记录被无限重捞，800+ 次无效调用；② 下游存储宕机但脚本继续调付费转录 API，白烧数小时。**规则**：claim/queue 必带 `max_attempts`（默认 3，超过不再捞）；批处理连续 N 次下游写入失败**立即停**（默认 N=3），调付费 API 前先探测下游健康；改 LLM 调用配置（模型/参数/格式）时 grep 全部调用点确认无遗漏；**定时任务上线后 24h 必须查一次日志**确认无持续失败模式。

## 测试与工作流

### 10. vitest + Node 22：global localStorage 遮蔽 jsdom

Node 22 自带的 global `localStorage` 会遮蔽 jsdom 的实现，行为诡异。**规则**：vitest `setupFiles` 里显式 polyfill storage。

### 11. git worktree 里 husky pre-commit 假失败

git 给 linked worktree 的 hook 注入绝对 `GIT_DIR`，monorepo 子目录里的 lint-staged 误判 toplevel、路径翻倍报错。**规则**：`.husky/pre-commit` 第一行 `unset GIT_DIR GIT_WORK_TREE`（主工作区无副作用）；禁用 `--no-verify` 绕过。

### 12. `git reset --hard` 前必查 `git status`

有未提交改动先 `git stash`，否则 tracked 文件的修改永久丢失、git 无法恢复。

## 原生能力集成机制

### 13. 原生定制的两条合法通道（原生目录不入库时）

- **本地 config plugin**（`plugins/*.js`，`withDangerousMod` 等）：改 manifest/plist/资源文件——上面 #3 的备份排除、依赖版本 pin 都是这类。
- **本地 expo module**（`modules/<name>/` + `app.plugin.js`）：需要原生代码的能力——iOS widget extension、share extension、自定义原生 API。

两条通道都在 `expo prebuild` 时生效、可进版本库、可 code review——**永远不要手改 ios/android 目录**（下次 prebuild 全部丢失）。

### 14. 文件类型关联全套（做文件型 app 时）

- iOS：`CFBundleDocumentTypes` + `UTImportedTypeDeclarations`（Open-In / 用其他应用打开）。
- Android：VIEW/BROWSABLE intent filters，**`file` 和 `content` 双 scheme**、后缀大小写变体都要列。
- 配合 #5 的 content URI copy 规则；进入 app 后统一走 `+native-intent.ts` 排除出路由、交给专门的导入 handler。
