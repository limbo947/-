# dsh-recall-plugin 重构计划（JS → TypeScript）

> 状态：待评审
>
> 目标形态：同形态复刻——功能与对外契约保持不变，仅将源码整体迁移为 TypeScript，用编译期类型锁死目前仅靠注释与测试维持的契约一致性。

---

## 一、背景与目标

### 1.1 项目现状

[dsh-recall-plugin](https://github.com/limbo947/dsh-recall-plugin) 是 DeepSeek Harness（DSH）的官方消息撤回插件：在用户消息旁挂「撤回」按钮，用独立影子 git 仓库保存工作区快照，配合官方 `sessions.fork` 把文件与对话一起回退到消息发送之前。

全仓约 1.2 万行，测试占近一半（17 个单测文件 227 例 + 2 个探针文件）。三层结构：

| 层 | 位置 | 技术约束 |
| --- | --- | --- |
| Host 服务端 | `lib/` | 必须是 cordis 4 插件（`apply(ctx)`），运行于 DSH 的 Node 进程 |
| Client 浏览器端 | `src/client/` | 必须走 `window.__ModuleLoader__.load` + React 单文件 CJS bundle |
| 跨平台执行层 | `lib/scripts.pwsh.js` / `lib/scripts.posix.js` | PowerShell/bash 脚本模板函数，与语言无关 |

### 1.2 重构动机

当前代码高度依赖「两处必须人工对齐」的不变量，全部只靠注释与测试维持：

- 双平台脚本模板必须导出同名接口（漏导出一个只在另一平台炸，运行时靠 `checkScriptParity` 手工比对兜底）
- 脚本哨兵字符串（`ROLLBACK_OK` / `RESCUE_OK` / `SNAP_SKIP` / `TREE` 等）与解析函数逐字呼应
- `index.json` / `lineage.json` / `exclude.txt` 的兼容字段结构（旧版本插件读新索引要忽略未知字段）
- DSH 各版本 API 依赖面字段（[docs/dsh-contract.md](./dsh-contract.md) 建档的几十个字段）只活在注释里

类型化之后，这些约束从「靠测试发现」升级为「编译期保证」。

### 1.3 不变量（重构期间不可改变的事）

1. `package.json` 的 `main` / `exports`（`lib/index.js`、`lib/client.js`）路径不变
2. `cordis.patch.yml`、`peerDependencies` 不变
3. npm 发布 `files` 仍只含 `lib/`，包布局零变化
4. 既有 17 个单测 + 2 个探针全部保留为回归网
5. `docs/`、README 双语、CHANGELOG、LICENSE 原样保留
6. 行为零变化：快照/回退/维护/设置页功能与现状逐项一致

---

## 二、目标目录结构

文件粒度保持 1:1 迁移，不新增抽象模块，最小化重构面。

```
dsh-recall-plugin/
├── src/
│   ├── types/                        # 跨域共享类型（本规划的核心增量）
│   │   ├── dsh-contract.ts           # DSH 依赖面类型（shell/sessions/sessionQuery/
│   │   │                             #   agents/webServer/settings/事件信封）
│   │   ├── payloads.ts               # index.json / lineage.json / exclude.txt / root.txt 结构
│   │   ├── state.ts                  # 共享 state（stores/snapshots/queue/feedback/缓存 holder）
│   │   ├── api.ts                    # /api/recall/* 各端点请求/响应类型
│   │   └── config.ts                 # Config 全类型（schemastery schema 的运行时镜像）
│   ├── host/                         # 原 lib/ 业务，1:1 迁移
│   │   ├── index.ts                  # 装配入口（端点组装/事件接线/预热）
│   │   ├── runtime.ts                # 执行与存储层（runShell/store 解析/ensureGit/迁移）
│   │   ├── snapshots.ts              # 快照域（模块级纯逻辑 + createSnapshots 工厂）
│   │   ├── maintenance.ts            # 维护域（gc/条数上限/保留天数/会话清理）
│   │   ├── routes-core.ts            # init/snapshot-info/preview/execute/status/lineage-record
│   │   ├── routes-manage.ts          # exclude/config/manage 管理端点
│   │   ├── config.ts                 # schemastery schema + createConfig 解析
│   │   ├── diagnostics.ts            # buildFeedbackError/classifyEnvError/ENV_HINTS
│   │   ├── errors.ts                 # 错误码常量
│   │   ├── dump-parse.ts             # stores/exclude dump 解析
│   │   ├── session-info.ts           # 会话标题/消息文本两段式读取
│   │   └── scripts/                  # 双平台脚本模板
│   │       ├── contract.ts           # 两套模板统一接口类型 + 哨兵/常量声明
│   │       ├── pwsh.ts               # PowerShell 模板实现
│   │       └── posix.ts              # bash 模板实现（macOS 3.2 兼容约束保留）
│   └── client/                       # 原 src/client/，整体搬入统一源码根
│       ├── entry.ts                  # __ModuleLoader__ 注册（id 字面量不变）
│       ├── app.ts                    # 装配（CSS/两个 keyed slot/设置卡片挂钩）
│       ├── recall-node.tsx           # 撤回按钮 + 确认面板
│       ├── settings-cards.tsx        # 设置卡片（配置表单/排除表/快照管理树）
│       ├── util.ts                   # API client（返回类型绑 src/types/api.ts）
│       └── css.ts                    # 手写 CSS 常量（原样）
├── lib/                              # 构建产物目录（对外契约不变）
│   ├── index.js / client.js / …      # build-host 产物 + client bundle
├── scripts/
│   ├── build-host.mjs                # 新增：esbuild 编 src/host → lib/
│   ├── build-client.mjs              # 现有，改入口为 src/client/entry.ts，产物断言保留
│   ├── check-dsh-version.mjs         # 不动
│   └── verify-host.mjs               # 装配门禁，import 路径更新后不动
├── tests/
│   ├── unit/                         # 现有 17 文件，import 路径从 lib/ 改指 src/
│   ├── probe/                        # 探针，同上
│   └── types/                        # 新增：编译期契约断言
│       ├── scripts-parity.test.ts    # 两套模板 satisfies ScriptsContract
│       ├── parse-contracts.test.ts   # parse 函数返回结构 satisfies 载荷类型
│       └── endpoints.test.ts         # routes 返回字面量 satisfies api.ts 类型
├── tsconfig.json                     # 类型门禁配置（strict，noEmit）
├── tsconfig.build.json               # 产物构建（declaration 供类型消费）
├── package.json                      # 增 typecheck/build:host 脚本；main/exports 不变
├── .github/workflows/ci.yml          # 增 tsc --noEmit 与产物新鲜度双门禁
└── cordis.patch.yml / README* / CHANGELOG.md / LICENSE   # 不动
```

---

## 三、关键设计决策

| 决策 | 理由（为什么这样做） |
| --- | --- |
| 源码 `lib/` → `src/host/`，产物仍输出 `lib/` | 保持 npm `files`/`main`/`exports` 指向不变，cordis patch 与已安装的 DSH 消费端零感知 |
| 文件 1:1 迁移，不拆模块 | 原域拆分已健康且被测试直接钉住，重构目标是加类型而不是改结构，拆分只会放大回归面 |
| scripts 模板带 `contract.ts` 接口 | 运行时手工 `checkScriptParity` 是编译期就该保证的约束；`satisfies` 锁死后，运行时比对降级为双保险 |
| 重建 `types/dsh-contract.ts` | dsh-contract.md 里几十个依赖面字段从注释收敛为单一类型源，DSH 升级核查从「读 diff 改注释」变成「diff 类型文件」 |
| 端点返回字面量绑定 `types/api.ts` | routes 返回与 client util 手写请求/响应形状，双向绑定同一类型后联调断裂在编译期暴露 |
| client 用 `.tsx` | JSX 形态不变，esbuild `platform: 'browser'` 原生支持 TS/TSX，构建链不加新依赖 |
| vitest 直接消费 `src/` TS | vitest 内置 esbuild 转译，单测无需先构建，`npm test` 保持秒级迭代 |
| 保留 `@deepseek-ai/schemastery` 的 schema 作为唯一默认值源 | config schema 与 createConfig 的默认值目前双份维护（`DEFAULTS` 镜像），类型化阶段收敛为单源，改默认值不再两处同步 |

---

## 四、构建与测试流程变化

### 4.1 构建

- 新增 `build:host`：`esbuild src/host/index.ts --format=esm --platform=node --outdir=lib`。现项目 `"type": "module"` 已是 ESM，产物格式与现状连续。
- `build:client` 入口改指 `src/client/entry.ts`，产物包裹格式 / `__ModuleLoader__` 注册 / react-only require 白名单断言原样保留。
- 新增 `tsconfig.build.json`：生成 `.d.ts` 声明，供下游类型消费与包内自检。

### 4.2 测试

- 既有 227 例单测与探针全部保留，仅更新 import 路径（`lib/` → `src/`）。
- 新增 `tests/types/`：`expectTypeOf` / `satisfies` 断言充当编译期契约测试，覆盖脚本模板同名导出、parse 返回结构、端点载荷形状三类高风险点。

### 4.3 CI 门禁

现 CI（.github/workflows/ci.yml）只有 client 产物新鲜度门禁。重构后新增两道：

1. `tsc --noEmit`（类型门禁），置于单测 job 之前
2. Host 产物新鲜度门禁（`node scripts/build-host.mjs && git diff --exit-code lib/`），与 client 同机制，防止手改 src 忘 rebuild 静默流出

---

## 五、迁移里程碑

| 阶段 | 内容 | 验收标准 |
| --- | --- | --- |
| M1 | 落 tsconfig + `build-host.mjs` + CI 类型门禁，源码仍 `.js`（`checkJs` 兜底） | 现有测试全绿，产物可纯净重建 |
| M2 | `types/` 全部类型先建（自 dsh-contract.md 与现状反推） | `tsc --noEmit` 通过 |
| M3 | 纯函数域转 `.ts`：config/errors/diagnostics/dump-parse/session-info + snapshots 的模块级纯逻辑 | 单测 1:1 通过 |
| M4 | scripts 三件套 `.ts` + `satisfies` 契约；scripts-contract 测试保留 | 双平台模板类型锁死 |
| M5 | 工厂与接线层 `.ts`：runtime/snapshots/maintenance/routes/index | `verify:host` 装配门禁全绿 |
| M6 | client 侧 `.ts`/`.tsx` + 产物流程 | 浏览器实弹冒烟（撤回按钮/确认面板/设置卡片） |

---

## 六、验收标准与兼容性承诺

迁移完成的标志是三个零变化：

1. **行为零变化**：快照/回退/维护/设置页功能与重构前逐项一致（对照 README 功能清单逐条走查）
2. **契约零变化**：`cordis.patch.yml` insert 行、npm 包布局（`package-layout.test.js` 断言）、DSH API 字段探针（`test:probe`）、装配门禁（`verify:host`）均与重构前一致
3. **工程质量提升**：`tsc --noEmit` 全绿，无 `@ts-ignore` 逃生舱残留（允许极少数经评审的豁免并逐个建档）

---

## 七、风险与对策

| 风险 | 对策 |
| --- | --- |
| DSH 依赖面升级导致类型过期 | `dsh-contract.ts` 是唯一类型源，升级核查流程（dsh-contract.md 第七节）改为 diff 此文件 |
| 产物与源码脱节（改 src 忘 rebuild 流出） | CI 产物新鲜度门禁扩展到 lib/ 全部输出 |
| 迁移过程回归 | 17 个单测 + 2 个探针全程保持绿色，M3–M5 每个阶段独立验收 |
| 旧版兼容分支（0.1.1-rc.2 ↔ 0.1.2-alpha.2）类型化时误删 | 双版本分支持续由兼容分支注释标注，类型上用联合/可选字段显式建模，探针继续钉真实运行实例 |
| PowerShell/bash 模板字符串在 TS 中可读性下降 | 模板生成保持字符串拼接风格，`contract.ts` 只声明签名不重写实现；模板常量（哨兵/`FIDELITY_ATTRS`）抽为带注释的命名常量便于两侧对照 |

---

## 八、开始条件与产物

开始条件：本计划评审通过。

阶段产物：

- M1 产出：`tsconfig.json`、`scripts/build-host.mjs`、CI 类型门禁
- M2 产出：`src/types/` 五个类型文件
- M3–M6 产出：逐域迁移提交，每阶段一个独立 commit 便于回滚
- 最终产物：`npm run build` 可纯净重建 `lib/`，`npm test` / `test:probe` / `verify:host` 全绿