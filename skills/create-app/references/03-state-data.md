# 03 — 状态与数据层

## 0. 分层铁律

```
components/features (React) → stores (zustand) → services (I/O 单例) → packages → SDK/native
```

- **service** = 有 I/O 的业务/数据类，模块级单例导出，**永不 import React/zustand/stores**。
- **store** = React 可订阅的内存投影 + loading/error 标志，唯一允许 import zustand 的层。
- **util** = 无 I/O 纯函数。
- 同域 service 用 barrel 控制公共面（repository 层不 re-export，保持 service 内部私有）。

## 1. zustand 薄店模式（核心心法）

**不用 `persist` middleware / 不用 immer**。选型理由：persist 中间件把「什么时候写、写什么、怎么迁移」藏进黑盒；而把持久化放服务层，写入时机显式、可测、可跨 store 复用，store 永远只是投影。

```ts
// stores/use-preferences-store.ts
const loadFromStorage = (): PreferencesState => ({
  theme: PreferencesStorage.getInstance().getTheme(),
  // ...每个字段从 MMKV 读
});

export const usePreferencesStore = create<PreferencesStore>((set) => ({
  ...loadFromStorage(),                      // 同步 hydration（MMKV 是同步的）
  setTheme: (theme) => {
    PreferencesStorage.getInstance().setTheme(theme);  // 先写存储
    set({ theme });                                     // 再更内存
  },
  reloadFromStorage: () => set(loadFromStorage()),      // 云拉取后显式重水合
}));
```

重 I/O store（SQLite 后备）模式要点：
- state 含 `{ items, loading, syncing, error: string | null, isBootstrapped }`。
- `bootstrap(userId)` 幂等（per-user 守卫）+ **每个 `await` 后检查 `bootstrapUserId !== userId` 即中止**——用户换号时 stale continuation 必须自杀，否则 A 的数据写进 B 的界面。
- **瞬态并发守卫放 store 旁的模块级闭包变量**（`activeSyncPromise`、`lastSyncAttempt`……），不进 zustand state——它们不需要触发渲染。
- `reset()` 清守卫 + `set(initialState)`，登出/换用户时调。

**跨 store 通信**（三种，够用，不引入事件总线）：
1. 专用 `useAuthSubscription()` hook 挂 Gate：auth 变化时命令式 `useXStore.getState().reset()` 逐个重置 + service 单例 `setActiveUser`；用持久化的 `lastLoggedInUserId` 区分同用户重登（不重置）vs 换用户（全重置，防数据串号）。
2. 派生 store：`useSourceStore.subscribe((s, prev) => ...)`，root 挂一次、幂等守卫、返回 unsubscribe。
3. 多 store 派生 hook：`useMemo` + `useRef` 自定义相等性稳定数组引用，防重渲染风暴。

## 2. MMKV 分层存储（`services/storage/`）

不用 AsyncStorage——MMKV 同步、快、支持加密实例，同步读写让 store hydration 变成一行。

```
MMKVAdapter (抽象基类)
├── PreferencesStorage   # 用户设置
├── CacheStorage         # TTL + 驱逐
├── SecureStorage        # 加密实例（token 用）
└── TempStorage          # 每次启动清空
StorageManager 单例: .preferences .cache .secure .temp; initialize() 启动时必调;
performStartupCleanup(); clearUserData()
```

```ts
// mmkv-adapter.ts 核心形状
export abstract class MMKVAdapter {
  private storage: MMKV | null = null;
  protected constructor(private readonly options: MMKVConfiguration) {}
  private get mmkv(): MMKV {
    return (this.storage ??= new MMKV(this.options));   // lazy
  }
  protected get<T>(key: string): T | null {
    const raw = this.mmkv.getString(key);
    if (raw == null) return null;
    try { return JSON.parse(raw) as T; } catch (e) { throw new DeserializationError(key, e); }
  }
  protected set<T>(key: string, value: T): void {
    this.mmkv.set(key, JSON.stringify(value));           // 失败抛 IOError/SerializationError
  }
}
```

规范：
- 每个设置独立 key + `getX()/setX()` 对，getter 内联默认值（`getTheme() ?? 'system'`）——不搞一个大 JSON blob（否则每次写全量、迁移地狱）。
- 抛类型化错误不吞错；小域各建薄 storage 类（onboarding-flow-storage 等），不全塞 Preferences。
- 本地存储与云同步分离：`preferences-sync.ts` 之类的同步逻辑在 storage 类**外面**。

## 3. SQLite（可选：有结构化本地数据才装，expo-sqlite）

### DatabaseManager 单例（不可简化的不变量）

```ts
class DatabaseManagerImpl {
  private writeQueue: Promise<unknown> = Promise.resolve();

  async runWrite<T>(fn: (db: SQLiteDatabase) => Promise<T>, label: string): Promise<T> {
    const task = this.writeQueue.then(async () => {
      const db = await this.getConnection();
      return runWithRetry(() => db.withExclusiveTransactionAsync(fn), label); // busy 退避重试
    });
    this.writeQueue = task.catch(() => undefined);   // 队列不因单次失败断链
    return task;
  }
}
```

- **一个连接**：per-user db 文件 `<app>_<sanitizedUserId>.db`（未登录用 `guest`）；`setActiveUser` 关旧开新。
- open 时 `PRAGMA foreign_keys=ON; journal_mode=WAL; busy_timeout=5000;` + `applyAllMigrations(db)`。
- **写串行**：所有写走单一 promise 链 + exclusive transaction——跨 repository 也不交错，这是多店共用一个连接还能安全的唯一前提。
- **busy 重试**：`SQLITE_BUSY`/locked 指数退避 3 次（50ms 基数）；只有耗尽/不可重试才上报观测 + rethrow。
- 生命周期切换（setActiveUser/close/delete）走独立 promise 链 + 排空 in-flight reads 再关——防换号瞬间 "database closed" 竞态。
- 惰性：模块级单例但首个 `getConnection()` 才真开，不在 app boot eager init。
- 测试口：`_resetForTesting()` + 导出 impl 类。

### 手写 migrations（无框架，选型理由：RN 端没有值得引入的迁移框架，版本表 + 内省 50 行搞定）

每域一个文件：`<DOMAIN>_SCHEMA_VERSION` 常量 + `__<domain>_meta` 版本表 + `ensureXSchema(db)`：读版本 → 已最新则 no-op → 否则 `BEGIN/COMMIT/ROLLBACK` 包裹，`PRAGMA table_info` 内省做**加法迁移**（ADD COLUMN / rename-and-recreate + `INSERT…SELECT` 回填）。索引幂等 `CREATE INDEX IF NOT EXISTS`（含 `WHERE deleted_at IS NULL` 部分索引）。`migrations/index.ts::applyAllMigrations` 按序调各域。

### Repository 模式

普通 class、无 ORM、每聚合一个：

```ts
async getById(id: string) {
  return databaseManager.runRead(async (db) => {
    const row = await db.getFirstAsync<Row>('SELECT ... WHERE id = ?', [id]);
    return row ? this.mapRowToItem(row) : null;
  }, 'library.getById');           // 点分 label 供日志/错误上下文
}
```

- **永不直接 touch SQLite**，全走 manager `runRead`/`runWrite` + label。
- 批量：`prepareAsync` + 循环 `executeAsync` + `finally finalizeAsync`；`IN (...)` 按 ~900 参数分块（SQLite 绑定参数上限）。
- Upsert：`INSERT … ON CONFLICT(id) DO UPDATE SET`，非平凡合并语义写注释（如 `COALESCE(old.imported_at, excluded.imported_at)` 保住本地原始时间不被服务端 payload 冲掉）。
- 部分更新：按 `hasOwn()` 动态拼 SET 子句，只更新调用方真正传了的字段。
- JSON 列：serialize/parse helper 吞错 + 观测 warning，不抛——一条脏数据不该炸整个列表。
- 行→域映射集中 `mapRowToItem(row)`。

## 4. 文件存储（per-user 分区）

```
<documentDirectory>/<app-slug>/<userId|guest>/
  <domain>/<entityId>/...
```

- 与 SQLite per-user 分区镜像；单一 FileStorage 类拥有整棵树。
- **DB 只存相对路径**，运行时 `resolveAbsolutePath()` 拼当前 user root——iOS 容器路径会变，绝对路径入库 = 升级后全部失效。
- 删除 `{ idempotent: true }`；目录重命名/合并要在 move 成功**后**再改指针（防悬挂引用）。
- `deleteUserFiles(userId)` 整树递归删，登出清数据/删号时与 `deleteUserDatabase` 一起调。

## 5. 错误双通道

1. **观测通道**：观测 wrapper（见 05）——services/stores 每个 try/catch 必带 `{category, operation, extra}`。
2. **用户通道**：store 的 `error: string | null` 短文案 → 屏幕渲染共享 `<ErrorState onRetry>`（inline 可重试态）。**不用 toast 传数据层错误**——toast 会消失，错误态要可重试、可停留。瞬态 SQLITE_BUSY 重试内部消化，用户无感。
3. 启动关键失败（MMKV init）短路到专用 `StartupStorageErrorScreen`。

## 6. 测试模式

- **jest.setup.js 全局 mock**：`expo-sqlite`（共享 fake db 对象，方法全 jest.fn 返回 `[]`/`null`/`{changes:0}`，`withExclusiveTransactionAsync` 回调自身）、`expo-file-system`、secure-store、观测 SDK。
- **MMKV 不全局 mock**——每个测试文件本地 `jest.mock('react-native-mmkv', ...)`，对 `set` 调用参数断言、按测试 stub `getString`。
- **repository 测试**：mock manager 边界（`runRead`/`runWrite` = jest.fn 且 `mockImplementation(async (cb) => cb(fakeDb))`），断言三件事：SQL 文本片段 + 绑定参数数组 + 映射后的域对象。
- **manager 测试**：mock `expo-sqlite` 本身 + `jest.requireActual` 真 manager 类；并发验证 = 不 await 连发多个写/换号调用，用 mock 调用顺序断言串行性。
- **store 测试**：文件顶部 `jest.mock` 兄弟 store/service（`/* eslint-disable import/first */`），`renderHook`/`act` 驱动真 store；纯派生函数另做无 React 的表驱动单测。
- 单例重置：`(XStorage as any).instance = null`（beforeEach）。
