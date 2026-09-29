# Taskute · 项目长期约定

## 项目定位

任务记录与执行软件。**桌面端 Tauri 2 + React 19 + SQLite**。本地单机，无服务端、无网络依赖。
工作区 `D:/Projects/Taskute`，代码在 `desktop/`，文档在 `docs/`。

## 技术选型（已定，不要擅改）

| 层 | 选型 | 备注 |
|---|---|---|
| 后端 | Rust + Tauri **2.12** | **不用 3.0.0-alpha**，`tauri` crate / `@tauri-apps/cli` / `@tauri-apps/api` 三者版本同步 |
| 数据库 | SQLite，**sqlx 0.9 + `sqlx::migrate!()`** | 不用 ORM、不用 diesel、不用 `tauri-plugin-sql` |
| SQL 写法 | `query_as` + 手写 `FromRow` | **不用 `sqlx::query!` 编译期宏**（需 DATABASE_URL，卡首次 clone/CI） |
| 前端 | React 19 + TS + Vite 8 + antd 6 | |
| 作图 | React Flow（DAG/脉络图）+ ECharts（日历/矩阵） | 分工见 `docs/03-前端设计.md` |
| 状态 | zustand 5 | 不引 Redux |

## 数据模型（三层，已定）

**任务 `task` → 阶段 `stage` → 步骤 `step`**，加 `step_dependency`（DAG 边）、`step_progress`（完成事件）、`app_meta`。

关键约定：

- 主键 **TEXT + UUID v7**（为未来多端同步留合并键）
- 时间统一 **UTC 毫秒 i64**；`ddl` 可空
- **软删只用于 `task` / `stage`**；**`step` 硬删**（软删会让依赖边残留成 DAG 幽灵节点）。`step.deleted_at` 字段保留但恒为 NULL，**查询不要依赖它过滤**
- `order_index` 间隔 10，插入取中点，耗尽触发全组重排
- 完成度（task / stage 的 progress）**不落库**，实时按 `est_minutes` 加权算

### 默认依赖规则（用户原话："不添加默认依赖于当前阶段"）

新增步骤未手动指定依赖时 → 向前回溯找到**第一个非空阶段**，依赖其**全部**步骤，`origin='auto'`。
第一个阶段的新步骤无依赖；前面全空阶段时无依赖。
用户手动指定依赖后，原 `auto` 边被清除。

### 今日排序公式

`score = 0.45·urgency + 0.35·importance + 0.20·quick_win`

`urgency` 分段映射（逾期封顶 1.0 不再上升；`ddl IS NULL` 给 0.30；`quick_win` 以 240 分钟为上限）。
常量在 `services/schedule.rs` 顶部。权重不做用户可配置。

## 架构约定

**分层单向依赖**：`commands → services → repo → db`。反向禁止。

- `services/` **禁止引用 tauri 类型**（`AppHandle`/`Emitter`/`Window`），只用领域类型 + `AppError` —— 这是为未来抽 `taskute-core` crate 支持 Android 留的门（`docs/01-架构设计.md` §3.3）
- 前端**唯一** invoke 入口是 `src/api/client.ts`，组件里禁止直接 `import { invoke }`
- 写操作**返回最新快照**，前端直接替换 store，不二次拉取
- 所有写命令参数**包一个对象**（`req` / `filter`），不传扁平多参
- `SqlitePool` `max_connections = 1`（SQLite 写锁库级，多连接只会 `SQLITE_BUSY`）
- PRAGMA 一律放 `after_connect`（连接级设置），**不进迁移文件**。`foreign_keys = ON` 最易漏

## 视觉约定

主色竹绿 `#4A7C59`，次级表面米色 `#F5F0E6`，与 Bambook 保持一致。
危险色 `#C0453C`（非纯红）。**重要程度用绿色深浅 + 填充数量双重编码**，不用红黄绿（避免"1星=绿=好"误读）。
所有色彩变量在 `src/styles/tokens.css`，是唯一色彩来源。

## 工作方式

- "实现 → 跑起来 → 对照 `docs/05-开发路线图.md` 的对齐检查逐条核对 → 满足则勾掉" —— 每个里程碑末尾都有检查清单，**逐条核对不凭记忆**
- 决策记 `docs/DECISION_LOG.md`，踩坑记 `docs/ISSUES.md`，都是活文档
- **已合并进主干的迁移文件永不修改**（包括注释格式），否则 sqlx 校验和会导致用户应用启动失败
- `0001_init.sql` 在打出第一个安装包（v0.1.0）前可随意改（删库重建）；之后冻结，一律追加迁移

## 已知环境坑

- **MSVC 链接器 `cl`/`link` 不在 bash PATH**。`tauri build` 若报 `link.exe not found`，用 "x64 Native Tools Command Prompt for VS" 执行
- 数据库路径 `%APPDATA%\com.halley.taskute\taskute.db`，由 `identifier` 决定。**`identifier` 发布后不可改**（改了等于换数据目录）
- WAL 模式下备份不能直接 `fs::copy`，必须先 `PRAGMA wal_checkpoint(TRUNCATE)`
- NSIS bundle 必须显式设 `deleteAppDataOnUninstall: false`，否则卸载/重装丢全部用户数据

## 未决问题（影响表结构，动手前定）

| # | 问题 | 截止 |
|---|---|---|
| Q1 | 任务是否需要手动拖拽排序（加 `task.sort_order`）？ | M2 前 |
| Q2 | 步骤是否支持进度百分比？ | M3 前 |
| Q3 | `step` 是否需要回收站？ | M3 前 |
| Q6 | 完成日历统计口径：步骤数还是耗时分钟？ | M6 前 |
