# 04 — 可选模块：Supabase auth / sync / payments

前提：用户已建好 Supabase 项目（问 URL + anon key）。**DB migration 永不自动跑**——生成 SQL、列出执行步骤，等用户明确点头。

## 0. Env 管理

- `apps/mobile/.env.local`（gitignored）+ `.env.example`（文档化全部必需 var，这是 env 的活文档）。
- `app.config.ts` 读 `EXPO_PUBLIC_*` 进 `extra`；运行时唯一入口 `Config`（见 02 §3）。

## 1. packages/backend-client（平台无关 Supabase 客户端包）

自建 `@<scope>/backend-client`，**零 RN/DOM import**——平台差异（存储、深链）全部由调用方注入。模块与其职责：

### client.ts — 客户端工厂

```ts
export function createSupabaseClient({ supabaseUrl, supabaseAnonKey, storage }: Options) {
  return createClient(supabaseUrl, supabaseAnonKey, {
    auth: {
      autoRefreshToken: true,
      detectSessionInUrl: false,            // mobile 深链自己处理，不让 SDK 猜
      persistSession: Boolean(storage),
      storage,                              // 平台注入 SupportedStorage
    },
  });
}
```

### functions.ts — Edge Function 调用包装（核心价值：401 语义）

```ts
export async function invokeFunction<T>(supabase, name, options): Promise<T> {
  const started = Date.now();
  let result = await supabase.functions.invoke(name, options);

  if (isUnauthorized(result.error) && hasSession()) {
    const refreshed = await refreshTokenOnce(supabase);   // 并发去重，见下
    if (refreshed) result = await supabase.functions.invoke(name, options); // 恰好一次重试
  }
  emitMetrics({ name, ms: Date.now() - started, ok: !result.error }); // 可插拔 handler

  if (isUnauthorized(result.error)) handleAuthError(result.error);    // 仍 401 才走全局登出
  if (result.error) throw normalizeError(result.error);
  return result.data;
}
```

规则（每条都是生产事故换来的）：
- 401 时**恰好一次** token refresh + retry；refresh 因网络/5xx 失败**绝不登出**（瞬时故障登出用户 = 灾难级体验）。
- 只有 401 触发全局 auth-error handler，403 不触发（403 是权限问题不是会话问题）。
- 每个业务域一个小文件导出类型化请求函数，全部构建在 `invokeFunction` 上。

### 其余模块

- `auth-error-handler.ts` — 全局 401 handler 注册表（`onAuthError(cb)` / `handleAuthError(err)`）；区分 `session_expired` vs `account_deleted`（后者要清本地数据）。
- `auth-otp.ts` — Email OTP（`signInWithOtp` / `verifyOtp`）。
- `token-refresh.ts` — `refreshTokenOnce`：模块级 in-flight promise 去重并发刷新。
- `token-manager.ts` — 存储 token 的加载/应用（`setSession`）与刷新回写（有 legacy token 迁移需求时）。
- `clock-skew.ts` — 设备时钟漂移 vs JWT expiry 校正（用户手机时间不准比你想象的常见）。
- `sync-engine.ts` — SyncQueueEngine（装 sync 时，见 §4）。

## 2. Mobile auth 接线

1. `services/storage/supabase-auth-storage.ts`：用加密 MMKV 实现 `SupportedStorage` → Supabase SDK 自管 session 持久化/刷新。
2. `features/auth/supabase-client.ts`：`createSupabaseClient({ ...Config, storage: supabaseAuthStorage })`。
3. `AuthProvider`：mount 时 load session；注册全局 `onAuthError`（401 强制登出；`account_deleted` 时清 SQLite+文件+MMKV）；session 变化同步观测 user（只 id）与计费 SDK 登录态。
4. Gate 纯函数 + 瘦组件（见 02 §9）。
5. 登录方式：**Email OTP 最小可行**（无密码，省掉整套密码重置流）；Apple / Google 集成见 §2.1 / §2.2。注意 App Review Guideline 4.8：iOS 上提供任何第三方登录（含 Google），就**必须**同时提供 Apple 登录。
6. **删号功能必备**（App Store 硬性要求）：edge function 级联删（DB+storage+auth.users）+ 客户端该调用跳过全局 401 处理（删完必 401，属预期）+ 本地全清。

### 2.1 Apple 登录（iOS only）

原生 SDK 拿 identity token → Supabase `signInWithIdToken` 换 session，**不走 OAuth 网页跳转**（体验和成功率都远好于 web flow）。

依赖与配置：`npx expo install expo-apple-authentication`；app.config.ts 设 `ios.usesAppleSignIn: true`（生成 entitlement，无需额外 plugin）；Apple Developer 后台给 App ID 勾选 Sign in with Apple capability。

```ts
import * as AppleAuthentication from 'expo-apple-authentication';
import * as Crypto from 'expo-crypto';

const rawNonce = Crypto.randomUUID();
const hashedNonce = await Crypto.digestStringAsync(Crypto.CryptoDigestAlgorithm.SHA256, rawNonce);

const credential = await AppleAuthentication.signInAsync({
  requestedScopes: [
    AppleAuthentication.AppleAuthenticationScope.FULL_NAME,
    AppleAuthentication.AppleAuthenticationScope.EMAIL,
  ],
  nonce: hashedNonce,               // Apple 拿哈希
});
const { error } = await supabase.auth.signInWithIdToken({
  provider: 'apple',
  token: credential.identityToken!,
  nonce: rawNonce,                  // Supabase 拿原文校验
});
```

生产坑：
- Supabase Dashboard → Auth → Apple provider 的 **Client IDs 必须把所有环境的 bundle id 都列上**（含 `.dev`/`.preview` 后缀变体），漏一个对应环境登录必挂且报错含混。
- `fullName`/`email` **只在首次授权返回一次**，当场存后端；用户在系统设置里撤销授权后重来才会再给。
- 用户可选「隐藏邮箱」（private relay 地址），业务不要假设拿到真实邮箱。
- UI 按钮用 `AppleAuthentication.AppleAuthenticationButton`（Apple 对按钮样式有审核要求）。

资料：
- https://docs.expo.dev/versions/latest/sdk/apple-authentication/
- https://supabase.com/docs/guides/auth/social-login/auth-apple?platform=react-native

### 2.2 Google 登录

同样走原生 SDK 拿 idToken → `signInWithIdToken`。依赖：`@react-native-google-signin/google-signin`（原生模块，需 dev client/EAS 构建，Expo Go 不可用）。

Google Cloud Console 要建**三个 OAuth Client ID**（最常见的翻车点）：
1. **Web client** — 它的 client ID 作为 `webClientId` 传给 SDK（idToken 的 audience，Supabase 用它校验）；
2. **iOS client** — bundle id 对应；其「reversed client ID」配进 app.config 的 plugin：`['@react-native-google-signin/google-signin', { iosUrlScheme: 'com.googleusercontent.apps.xxx' }]`；
3. **Android client** — package name + **SHA-1 指纹**。SHA-1 要把三把钥匙都注册：本地 debug keystore、EAS 构建凭据（`eas credentials` 查看）、**Play App Signing 的签名密钥**（Play Console → Setup → App integrity 里拿）——漏最后一个 = 本地全好、上架后全挂。

```ts
import { GoogleSignin } from '@react-native-google-signin/google-signin';

GoogleSignin.configure({ webClientId: Config.GOOGLE_WEB_CLIENT_ID }); // app 启动时一次
// 登录按钮 onPress:
await GoogleSignin.hasPlayServices({ showPlayServicesUpdateDialog: true });
const { data } = await GoogleSignin.signIn();
const { error } = await supabase.auth.signInWithIdToken({
  provider: 'google',
  token: data!.idToken!,
});
```

生产坑：
- `DEVELOPER_ERROR` ≈ 永远是 SHA-1/package name 与 Android client 不匹配，别的方向不用查。
- Supabase Dashboard → Google provider 的 Authorized Client IDs 要把 web/iOS/Android 三个都填上。
- `webClientId` 必须是 **Web** 类型那个，填成 Android/iOS 的拿不到可校验 idToken。

资料：
- https://react-native-google-signin.github.io/docs/setting-up/expo
- https://supabase.com/docs/guides/auth/social-login/auth-google?platform=react-native

### 2.3 App Review 专用登录（提审必备）

审核员需要 demo 账号，但 OTP 登录收不到验证码。方案：一个 `review-login` edge function——识别预留的审核邮箱（env 配置），对它返回**固定 OTP**（其余邮箱走正常流程），客户端零改动。提审信息里填该邮箱 + 固定码。注意：该账号进的是真实生产环境，里面放好演示内容；固定码只对白名单邮箱生效，不构成后门。

## 3. Edge Functions 骨架

`supabase/functions/_shared/` 六件套（各 30-80 行，一次写好全项目复用）：

| 文件 | 职责 |
|---|---|
| `auth.ts` | `getAuthenticatedUser(req)`（完整 SDK 校验）+ `getAuthenticatedUserId(req)`（本地 JWKS `getClaims` 快路径，高频端点用） |
| `supabase.ts` | `getServiceRoleClient()` 缓存单例 + `createUserClient(authHeader)` per-request RLS 客户端 |
| `request.ts` | `ensureRequestMethod` / `parseJsonBody` |
| `response.ts` | `toJsonResponse` / `toErrorResponse` / `toUnexpectedResponse` |
| `errors.ts` | `AppError` + 封闭 error code union |
| `cors.ts` | CORS 头 |

规范函数结构：`index.ts`（纯 HTTP）+ `logic.ts`（纯业务可测）+ `*.test.ts`。index 恒定流程：

```ts
ensureRequestMethod(req, 'POST');
const body = await parseJsonBody(req);
const supabase = getServiceRoleClient();
const user = await getAuthenticatedUser(req);
const result = await doLogic(supabase, user, body);   // logic.ts
return toJsonResponse(result);
```

部署：单函数 `supabase functions deploy <name>`（别全量）；默认 JWT 保护，`--no-verify-jwt` 仅 webhook/authless 端点。

### RLS 模板（每张 user-owned 表）

```sql
create table public.user_<name> (
  id uuid primary key,
  user_id uuid not null references auth.users(id) on delete cascade,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  deleted_at timestamptz          -- soft delete，同步协议依赖它
);
create index on public.user_<name> (user_id);
create index on public.user_<name> (user_id) where deleted_at is null;
-- handle_updated_at() trigger
alter table public.user_<name> enable row level security;
create policy "user_own" on public.user_<name> for all
  using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "service_role_all" on public.user_<name> to service_role
  for all using (true) with check (true);   -- edge functions 走 service role，需显式授权
```

Migrations：时间戳前缀、单一关注点、幂等（`IF NOT EXISTS` / `DROP POLICY IF EXISTS` 先删后建）。**含 RPC/PL-pgSQL 函数的 migration 必须真库回放写路径**（mock 单测碰不到 PG 解析器，见 06 §7）；大 JSON 列查询必带 limit（06 §8）；queue/claim 类端点必带 max_attempts（06 §9）。

## 4. Sync engine（可选子模块）

两层架构：

**第一层 — SyncQueueEngine（backend-client，离线优先 patch 队列）**：

```ts
class SyncQueueEngine<P> {
  async dispatchPatch(patch: P): Promise<'sent' | 'queued'> {
    if (!(await this.hasSession())) { await this.enqueue(patch); return 'queued'; }
    try {
      await this.invokePatch(patch);       // 域注入的闭包，调一个 edge function
      await this.flushQueue();             // 成功顺带清积压
      return 'sent';
    } catch { await this.enqueue(patch); return 'queued'; }
  }
  // flushQueue: 按序走持久化队列，首个失败即停并保序重排，maxAttempts=5 后丢弃并上报
}
```

**第二层 — SyncDomain + Coordinator（core 包，跨域调度）**：`SyncDomain = { id, isReady?, sync(options) }`；coordinator 并行跑各域 + in-flight 去重（同 key 复用进行中的 promise）+ per-user 节流。

新增同步实体四步：
1. patch 类型（discriminated union：`ADD_X`/`UPDATE_X`/`DELETE_X`）+ 两个 edge function：`sync-<entity>`（幂等，服务端 content-hash 去重）/ `get-<entity>`（`{since}` cursor，量大则分页 loop until `hasMore`，守卫重复 cursor）。
2. `XSyncEngine` 包 SyncQueueEngine（`invokePatch` 闭包）。
3. `createXSyncDomain()` 闭包住 store 的 `syncFromCloud`。
4. 加入 coordinator 的 domains 数组。

**铁律**：sync engine 在 storage 类里**惰性构造**（getter 内 `??= new`），module load 期禁止实例化——否则破坏测试 partial-mock，且登录前就建了网络对象。冲突解决默认 last-writer-wins on `updated_at` + 软删除级联——简单、可预测，真需要 CRDT 时再说。

## 5. Payments（可选）

选型：IAP 用托管方案（RevenueCat）而非裸 StoreKit/Play Billing——收据校验、续订状态机、跨平台订阅状态合并不值得自己写。

**计费策略抽象**：`getBillingStrategy(): 'iap' | 'stripe' | 'none'` 按分发渠道+平台分发（商店包走 IAP，官网 sideload 包走 Stripe Checkout——商店包里不能出现外部支付入口）。

### 5.1 RevenueCat 集成

依赖：`react-native-purchases`（autolinking，无需 config plugin；原生模块，需 dev client/EAS 构建）。

后台侧（一次性）：RC 项目 → 每平台一个 app → 拿**每平台独立的 public API key**（`EXPO_PUBLIC_REVENUECAT_API_KEY_IOS/ANDROID` 走 env）→ App Store Connect / Play Console 建内购商品 → RC 里映射为 Products → 归入 **Entitlement**（如 `'plus'`，代码只认它）→ 组织成 Offering/Packages（价格实验/改品不发版的关键：代码只展示 current offering，不硬编码商品 ID）。

```ts
import Purchases from 'react-native-purchases';

// AuthProvider mount 时，任何购买调用之前:
Purchases.configure({ apiKey: Platform.OS === 'ios' ? Config.RC_KEY_IOS : Config.RC_KEY_ANDROID });
// session 变化时桥接身份（appUserID ↔ 后端 user id）:
await Purchases.logIn(userId);      // 登出时 Purchases.logOut()

// 付费墙: 展示 current offering
const offerings = await Purchases.getOfferings();
const pkg = offerings.current?.availablePackages[0];
const { customerInfo } = await Purchases.purchasePackage(pkg);
const isPlus = customerInfo.entitlements.active['plus'] !== undefined;
```

生产坑：
- `configure` 失败要**非致命**处理（降级为免费态继续跑），别让计费 SDK 初始化失败挡住 app 启动。
- 身份桥接放 AuthProvider：登录 `logIn(后端 user id)`、登出 `logOut()`——不桥接的话匿名购买和账号对不上，跨设备恢复全乱。
- 检查 entitlement（`entitlements.active['plus']`），别检查具体商品 ID——换品/加年付月付时代码零改动。
- `addCustomerInfoUpdateListener` 监听变化刷新 UI（续订/退款/家庭共享变更是异步到达的）。
- 恢复购买按钮必须有（`Purchases.restorePurchases()`，iOS 审核要求）。
- 沙盒测试：iOS 用 Sandbox tester/TestFlight，Android 用 License testing 名单；沙盒续订周期是加速的（月=5分钟级），测状态机很方便。

**服务端真相**：客户端 entitlement 只用于 UI 门禁；服务端资源授权以 `profiles` 表为准，由 **RC Webhook**（`INITIAL_PURCHASE`/`RENEWAL`/`CANCELLATION`/`EXPIRATION`/`BILLING_ISSUE` 事件 → edge function 更新 tier/status/expires）维护。**客户端只展示，永不计算扣减**——客户端可被篡改，webhook 才是账本。

资料：
- https://www.revenuecat.com/docs/getting-started/installation/reactnative
- https://www.revenuecat.com/docs/getting-started/configuring-sdk
- https://www.revenuecat.com/docs/integrations/webhooks
- Expo 专项指引：https://www.revenuecat.com/docs/getting-started/installation/expo

### 5.2 Stripe（sideload 渠道）

官网分发包走 Stripe Checkout（web 页）而非 SDK 内购：edge function `create-checkout-session` 生成 URL → 系统浏览器打开 → 支付完成深链回 app（`<scheme>://checkout-result`，在 `+native-intent.ts` 排除出路由）→ `stripe-webhook` edge function 按事件类型模块化处理（`checkout.session.completed`、`customer.subscription.updated/deleted`、`invoice.paid/payment_failed`…）更新 `profiles`。订阅管理用 Customer Portal（`create-portal-session`），自己不写改卡/取消 UI。

资料：https://docs.stripe.com/checkout/quickstart 、https://docs.stripe.com/customer-management

### 5.3 Credits 计量（AI 类功能按量计费时）

tier→月度额度表；扣减顺序 base 先于 bonus；月度重置锚定原始购买日（处理短月/DST）；重置用 **lazy reset-on-read**（get-profile 时顺手检查），免 cron。

## 6. 有意不做的（除非明确需要）

多区域网关路由/竞速、per-call geo 解析、双云镜像部署——这是特定分发区域（如中国大陆）的需求，复杂度极高，默认单 Supabase 区域起步。
