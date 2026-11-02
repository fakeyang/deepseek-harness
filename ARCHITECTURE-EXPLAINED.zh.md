# DeepSeek Harness 源码全景讲解（中文）

> 本文是面向源码阅读者的架构导览，由对仓库的全面探索整理而成（基于 `caiyang10` 分支，dsh 0.2.2-alpha.2）。
> 它不是官方文档——官方权威文档在 `docs/`（均有中英双语），本文每个主题都标注了对应的权威出处，用于"先建立地图，再按图索骥"。
> 阅读建议：第 1–3 章是理解一切的前提；第 4–8 章按子系统展开；第 9–10 章讲工程体系；第 11 章给出源码学习路径。

## 0. 一句话理解这个项目

**DeepSeek Harness（dsh）是一个"全插件化"的 AI Agent 运行时**：模型适配器、工具注册表、会话日志、甚至 agent 循环本身，统统是挂在 Cordis 插件框架上的可替换插件（`docs/architecture.md:11`）。没有特权核心可改——你要扩展它，就在旁边挂一个自己的插件。

它对外以几种形态分发：浏览器 GUI（`dsh web`）、一次性命令行（`dsh --profile headless`）、TS/Python SDK、自动化协议（ACP）、Electron 桌面应用——这些形态共享同一棵插件树，只是组合（composition）不同。

## 1. 全景地图

### 1.1 分层视图

```
┌─────────────────────────────────────────────────────────┐
│ 应用形态层   web(GUI) │ headless(CLI) │ sdk │ acp │ desktop │
├─────────────────────────────────────────────────────────┤
│ 组合层       profile + bundle + cordis.patch.yml 层叠组合    │
│              （apps/cli → boot/app-boot → plugin-manager）  │
├─────────────────────────────────────────────────────────┤
│ 编排层       subagent │ jobs │ schedule │ goal │ plan │ todo │
│              workflow │ webhook │ agent-team │ preset        │
├─────────────────────────────────────────────────────────┤
│ 核心运行时   agent-loop(turn/step 驱动) │ session(事件日志)   │
│              tools(工具管线) │ system-prompt │ llm(适配器缝)  │
├─────────────────────────────────────────────────────────┤
│ 能力缝层     shell/subprocess/sandbox/fs/terminal/ssh/lsp/   │
│              ptc/web/skill/mcp/computer-use/browser-use      │
├─────────────────────────────────────────────────────────┤
│ 持久化层     session-persistence-jsonl │ sqlite │ storage │  │
│              attachment │ spill │ workspace │ compaction     │
├─────────────────────────────────────────────────────────┤
│ 框架层       vendor/cordis（Context/Fiber/Events/Effect）    │
└─────────────────────────────────────────────────────────┘
        全部数据落在 ~/.dsh（DSH_HOME）之下
```

### 1.2 顶层目录速查（根 `AGENTS.md:17-79`）

| 目录 | 内容 |
|---|---|
| `vendor/` | Cordis 框架及基础库的**源码级 vendored 副本**（非 npm 依赖） |
| `packages/` | 约 60 组 `@deepseek-ai/dsh-*` workspace 包，按 `packages/<组>/<包>/` 组织 |
| `apps/` | `cli`（唯一启动器）、`web`（浏览器前端壳）、`desktop` + `desktop-host`（Electron） |
| `python/` | Python SDK 与运行时 |
| `native/` | Node addon |
| `docs/` | 权威文档（architecture.md 是地图，subsystems/ 是各子系统参考，全部中英双语） |
| `scripts/` | 约 50 个 `verify-*` 静态门禁 + 生成器 + 发布管线 |
| `snapshots/` | 无密钥录制会话回放测试 |
| `benchmarks/` | 用户路径级性能门禁 |
| `.agents/` | Agent 工作流：`skills/`（可复用流程）+ `notes/`（ADR 式决策记录） |
| `website/` | VitePress 文档站投影 |

### 1.3 数据根目录 `~/.dsh`

所有运行时数据的根：`DSH_HOME`，解析优先级「显式配置 > `$DSH_HOME` 环境变量 > `~/.dsh`」（`packages/util/home-paths/src/index.ts:12-18,79-88`）。重要子目录：

```
~/.dsh/
├── profiles/<name>/        # 每个 profile 的组合定义与第三方插件
│   ├── package.json        #   dependencies(插件) + dsh.profile.bundles(层列表)
│   ├── cordis.patch.yml    #   用户 patch 层
│   └── compatibility.json  #   插件版本豁免记录
├── sessions/<项目编码>/<session-id>/session.vN.jsonl.zstd   # 会话事件日志（权威数据）
├── attachments/v1/         # 二进制附件（内容寻址）
├── storages/               # 非会话 KV
└── logs/                   # 启动失败诊断
```

## 2. Cordis 框架层：一切的地基

### 2.1 vendor/ 是什么

`vendor/` 是 Cordis 框架的**源码副本而非 npm 依赖**（`vendor/README.md:1-7`）——目的是让 dsh 完全拥有自己的框架层：可审计、可打补丁、可锁版本。核心 `cordis` 上游是 Koishi 生态的 `cordiverse/cordis`（v4.0.0-rc.7），全部包 rescope 到 `@deepseek-ai/*`。已登记 23 条与上游的本地分歧（`vendor/README.md:29-59`），其中重要的有：fiber 生命周期加固（#6）、惰性 `!!js` 配置求值（#15）。

vendor 全家福：核心 `cordis`（约 2700 行，9 个源文件）+ `loader`（从 cordis.yml 挂插件）+ `include`（解析配置/patch）+ `schemastery`（配置 schema 校验）+ `cosmokit`（工具底座）+ `hmr`/`group`/`timer`/`logger-console`。

### 2.2 核心四件套

Cordis 核心可以归纳为四个概念（源文件都在 `vendor/cordis/src/`）：

**① Context——服务仓库 + 原型链作用域**（`context.ts`）
运行时是 Proxy（`context.ts:74`），`ctx.tools`、`ctx.llm`、`ctx.sessions` 都是按 key 取服务而不是 import 实现。`extend()/isolate()/intercept()`（:99/:121/:141）派生原型继承的子 context——这是"每个插件有自己独立 ctx 视图"的基础。类型面上，`Context` 接口是开放的，各包用 TypeScript **declaration merging** 往上合并键，例如 `packages/core/tools/src/index.ts:137-141` 把 `tools: ToolRuntime` 合并进 `Context`。

**② Fiber——插件生命周期**（`fiber.ts`，754 行，全框架最重的文件）
插件的三种形态：函数（带 `apply`）、类（Service 子类）、对象（带 `apply` 方法）（`registry.ts:92-133`）。每个插件实例是一个 fiber，有六态状态机（PENDING/LOADING/ACTIVE/FAILED/DISPOSED/UNLOADING，`fiber.ts:147-154`）。配置在挂载前经 Standard Schema 校验，失败即抛——这就是 AGENTS.md 约定"misconfiguration fails loud"的实现（`fiber.ts:50-62`）。

**③ Events——五种派发模式**（`events.ts:32`）
`emit`（同步不等）、`parallel`（allSettled）、`serial`（按序 await）、`bail`（同步首个真值）、**`waterfall`（环绕中间件）**。Waterfall 是 dsh 最重要的扩展机制：监听器收到 `(...args, next)`，**必须调 `next()` 委托，不调即短路否决**（`events.ts:234-243`；语义详解 `docs/cordis-primer.md:29-36`）。工具执行、请求拦截、prompt 组装全靠它。

**④ Effect——可逆注册**（`fiber.ts:415-418`）
`ctx.effect(execute)` 立即执行并返回 disposer；disposer 在手动调用或 fiber 卸载时（二者先到）逆序执行。`ctx.on()` 注册监听器内部就是调 effect（`events.ts:254-260`）——这就是 AGENTS.md 铁律 **"Registrations are effects"** 的字面实现：一切注册可逆，插件卸载即清理干净，HMR 和配置热更都建立在这上面。

### 2.3 inject：依赖注入即加载顺序

插件声明 `inject = ['tools', 'sessionProjections']` 后，fiber 在依赖未就绪时保持 PENDING；服务被 provide 时 `reflect.ts:319-336` 唤醒等待中的 fiber 重算依赖并激活。**cordis.yml 里插件行的书写顺序不决定加载顺序，服务依赖关系才决定**（教程 `docs/cordis-tutorial/01-first-plugin.md:31` 特意强调）。依赖消失（服务被替换）还会触发 fiber 自动卸载重载。

### 2.4 一个真实的最小插件

以 `tool-todo`（212 行）为模板（`packages/todo/tool-todo/src/index.ts`）：

```ts
export const name = 'tool-todo'                        // :22  loader 元数据
export const inject = ['tools', 'sessionProjections']  // :23  依赖声明
export const Config = z.object({ ... })                // :41  部署侧配置 schema
export function apply(ctx: Context, config: Config) {  // :117 插件体
  // 注册工具、事件投影……所有注册返回 disposer
}
```

服务形态插件的模板是 `ToolRuntime extends Service`（`packages/core/tools/src/index.ts:807-859`）：`static inject` + `static Config` + `super(ctx, 'tools')` 把自身挂为 `ctx.tools`，随 fiber 卸载自动注销。

### 2.5 SessionEventMap：声明合并的范式样本

dsh 把 declaration merging 用到了极致——**会话事件类型本身是 merge-extensible 的**。基础接口 `SessionEventMap`（`packages/core/session/src/types.ts:281`）声明"事件名 → 负载类型"；任何包都能贡献新事件，例如 `packages/todo/tool-todo/src/types.ts:28-33` 合并 `'todo/write'`（全仓库 39 处此类合并）。写入点 `Session.append<T>(type: T, data: SessionEventMap[T])`（`index.ts:718-722`）让事件名与负载在编译期绑定。配套治理：事件默认 required-on-read（旧版本 harness 读到不认识的事件会拒绝而不是静默误读），信息性事件需显式标 `ignorable: true`。

## 3. 启动、组合与分发：一棵插件树是怎么长出来的

### 3.1 唯一启动路径

仓库有硬性约定：**应用启动只走 `dsh` profile**（`verify-application-entrypoints` 门禁强制），进程启动只有一条路：

```
bin.ts → args.ts（选 profile）→ profile-boot.runProfile()
  → app-boot 按层叠顺序组合 patch → 兼容性 preflight
  → cordis Loader 挂载整棵树 → 必需条目审计 → 应用就绪
```

- `apps/cli/src/args.ts:202-204`：`dsh web` ≡ `dsh --profile web`（首个裸参数自动补 `--profile`）；launcher 只解析自己的 flag，**第一个不认识的 token 起全部原样交给 app**（args.ts:5-14）。
- `apps/cli/src/profile-boot.ts:244-326`：`runProfile()` 全链路——装代理、组合层、装 fail-loud 守卫与信号处理、boot、把剩余参数以 `ctx.cmdlineArgs` 交给 app 插件。
- profile 目录不存在时按模板自动初始化（`profile-boot.ts:167-173`）。

### 3.2 profile / bundle / patch：三层组合模型

**层叠顺序**（`docs/architecture.md:27`，实现在 `profile-context.ts:63-74`）：

```
空根 cordis.yml（每次启动重写）
 ← bundle 层（按 profile package.json 的 dsh.profile.bundles 顺序）
 ← profile 的 cordis.patch.yml（你的定制）
 ← ~/.dsh/cordis.patch.yml（home 层，机器级偏好）
 ← --patch overlay（按 argv 顺序）
 ← telemetry 开关补丁
```

- **bundle** = Cordis 配置行 + 所挂代码的分发格式（包 `package.json` 的 `dsh.bundle.patch` 声明）。
- **patch** = 顶层 YAML 数组：按 `id` 整行替换 config（**不深合并**）、`insert` 新行、`disabled: true` 禁用。`!!js` 表达式只允许出现在 `config` 和 `disabled` 下，且在 inject 激活后惰性求值（如 `!!js ctx.webRuntime.trustedHosts`）。
- 想看你机器实际启动的树：`dsh --profile web --dump-config`。

**五个出厂 profile**（`profile.ts:179-195`）：

| profile | 组成 | 用途 |
|---|---|---|
| `web` | dsh-base + dsh-web-app | 浏览器 GUI |
| `headless` | dsh-base + dsh-headless | 一次性任务，无服务器 |
| `sdk` | dsh-base + dsh-sdk-app | SDK JSON-RPC over stdio |
| `sdk-minimal` | 仅 dsh-sdk-minimal（刻意不叠 base） | 最小 SDK 树 |
| `acp` | dsh-base + dsh-acp-app | 自动化 ACP 协议 |

**dsh-base**（`packages/bundle/base/cordis.patch.yml`）是共享第一层：模型适配器、工具、持久化、sandbox/approval、settings、credentials、遥测、plugin-manager，默认模型 deepseek-flash，默认权限 `workspace-write` + ask。**dsh-web-app** 再加 webserver/connection/API controller 和浏览器客户端行，然后**禁用 base 上的 per-agent 工具行**——因为 Web 把 agent 平面挪到了 per-session 的 agent preset 机制（`agent-preset-registry`）。

### 3.3 插件管理器：第三方插件的安装与兼容性

`dsh plugin --profile web add <pkg>` 背后的机制（`packages/boot/plugin-manager`）。新版另有 `packages/boot/plugin-market`：读取用户显式配置的插件索引，为 Web「发现」页和只读 `plugin_market` 工具提供搜索、版本兼容性与安装状态；实际安装仍交给 `pluginManager.installBundle`，市场服务本身不修改插件。

- **兼容性检查**：只查插件 `peerDependencies` 里的 `@deepseek-ai/dsh*` 范围对当前运行时版本做 semver 匹配（prerelease 参与！`plugin-compatibility.ts:61-88`）。三个检查边界：安装前 preflight（不兼容直接拒绝）、装后核对、**启动/重组 preflight**（被拒行改成 `disabled: true` 的 detached 行，插件模块根本不会被 import，`compatibility-preflight.ts`）。
- **精确版本豁免**：`dsh plugin allow-version <pkg@ver> --dsh-version <exact> --accept-risk` 写入 profile 的 `compatibility.json`（独立于 manifest，文件锁+原子写 0600，`profile-compatibility.ts:113-141`）。**版本绑定**——dsh 或插件任一升级后需重新豁免（我们在这台机器上经历过：`0.2.0-rc.1 → 0.2.1-alpha.1` 后豁免失效，插件被跳过）。
- 第三方插件与内置包共享同一个 cordis 实例：profile 的 `pnpm-workspace.yaml` 设 `autoInstallPeers: false`，缺失 peer 走**运行时解析拦截**（`PluginPackages` 服务挂在 Node 内部解析器上，需 `--expose-internals`，`profile-resolution/service.ts:60-93`）。

### 3.4 HMR：运行时的配置与代码热更

`packages/boot/hmr`：watch profile 的 `package.json`、`cordis.patch.yml` 和 home 层 patch → 变化即重读全部层 → `reconcileProfilePatches` 更新根 Include → 等 fiber settle。模块替换与配置重载共用一条串行队列；命中 CLI 入口依赖树的变化直接全量重启。

### 3.5 Desktop：同一台启动机器的另一种安装锚点

Electron 壳（`apps/desktop`）**不跑插件树**；它用 `ELECTRON_RUN_AS_NODE` spawn `apps/desktop-host` 子进程，后者调同一个 `runProfile()`（profile 名 `desktop`，`--no-open --port 0`），跑的就是 Web 那套组合（`apps/desktop-host/src/index.ts:18-42`）。Electron 窗口加载 `dsh-app://` 特权协议：静态文件本地直出，API 请求转发给带认证的 Host（`apps/desktop/src/main.ts:662-677`）。桌面与 npm CLI 共享 `~/.dsh` 但包管理各自独立；npm CLI 对 `desktop` profile 一律拒绝。

## 4. 核心运行时：一次提问的完整数据流

这是全仓库的心脏。权威描述在 `docs/architecture.md:84-119`（turn flow），实现主体在 `packages/core/agent-loop/src/agent.ts`（`ReactLoopAgent`）。

### 4.1 Turn / Step 模型

- **step** = 一次模型请求 + 它调用的工具。
- **turn** = 零或多个 step：认领输入前开启，"不再欠任何东西"时关闭。
- 关键不变量 **"Model-visible ⟺ logged"**：任何到达模型请求的内容必须能从事件日志重建；`deriveMessages()` 从日志投影模型历史，请求体深冻结。

### 4.2 完整链路（附实现行号）

以 `agent.followup(message)` 为起点：

```
send()/followup()/steer()/inject()
 → ReactLoopAgent.send()        写入 durable inbox（agent/inbox/spliced 事件）
 → wakeDriver()                 idle → running，启动 driver
 → turn()                       agent.ts:327
   ├─ append 'turn/start'
   ├─ preStep()                                  agent.ts:283
   │   ├─ inbox.claim()                          认领 next-turn/next-step 消息
   │   ├─ 快照 Session 模型选择                  重试沿用同一次选择
   │   ├─ ctx.systemPrompt.assemble()            组装 prompt sections + 本 step 准入的工具 schema + 变量
   │   ├─ RuntimeContextProjection.project()     动态上下文快照（sandbox/approval 策略等）
   │   └─ waterfall 'agent/pre-step'             ★扩展点：可改写/拒绝本步输入
   ├─ append 'step/start'
   ├─ step()                                     agent.ts:435
   │   ├─ prepareRequest()                       agent.ts:613
   │   │   ├─ waterfall 'agent/request'          已弃用；只能调整控制项，不能改 provider/model/effort
   │   │   └─ ctx.llm.prepareCall()              绑定 adapter 生成代际（防 HMR 串代）
   │   ├─ append 'system/message'（prompt 协调）
   │   ├─ 首次尝试 append 'user/message'
   │   ├─ buildRequest()                         agent.ts:656
   │   │   ├─ append 'request/header'/'request/context'
   │   │   └─ session.deriveMessages() → 深冻结 GenerateOptions
   │   ├─ preparedCall.stream() → waterfall 'llm/stream' → adapter
   │   │   └─ AssistantStreamAttempt 发 'agent/assistant-stream' start/chunk/end 帧
   │   ├─ 结算 append 'assistant/message'（内嵌完整 stream + usage）
   │   │   失败则 append 'assistant/attempt' → 'agent/request-error' waterfall（可重试）
   │   └─ 当前作用域的 ToolExecutor 执行原始工具调用批次
   │       ├─ append 'tool/call'
   │       ├─ tools/pre-execute → ask(approval) → guards
   │       ├─ tools/execute → 工具体
   │       ├─ tools/post-execute
   │       └─ append 'tool/result'
   ├─ append 'step/end'
   ├─ 工具欠下一轮 / 新输入到达 → 下一 step
   ├─ serial 'agent/turn-stopping'               ★扩展点：可要求继续
   └─ append 'turn/end', reason
```

其中 `turn/*`、`step/*`、`*\/message`、`tool/*` 是 **durable 会话事件**；`agent/*`、`llm/stream`、`tools/*` 是**活扩展点**。`agent/pre-step`、`agent/request`、`llm/stream`、`tools/*` 是 waterfall（监听器必须 `next()`）；`agent/turn-stopping` 是 serial。

### 4.3 四个核心包

**`core/agent`（`ctx.agents`）——接口与活注册表**
`Agent` 公共句柄（`packages/core/agent/src/types.ts:15`）+ 运行态面孔经 declaration merging 扩充（`runtime-types.ts:163-243`：`send/followup/steer/inject/cancel/whenIdle`）。`AgentRegistry extends Service`（`index.ts:245`）管活注册表、工厂委托，以及基于 AsyncLocalStorage 的 **initiator 因果链**（同进程归因用，不是授权）。全部 `agent/*` 事件声明在 `runtime-types.ts:245-394`。

**`core/agent-loop`（`ctx.agentLoop`）——默认驱动**
`ReactLoopAgent implements Agent`（`agent.ts:112`）实现上述状态机；`AgentLoop` 负责创建事务、初始化失败回滚与 resume。`agent/pre-step` 前快照的模型选择会贯穿本步及其重试；Agent 工厂初始化发生在 `session/created` 之后、`agent/created` 之前。工具批次交给当前作用域的执行器，不再由 agent-loop 自己调度每个工具。

**`core/session`（`ctx.sessions`）——append-only 事件日志**
类型体系（`types.ts`）：`SessionId` 等 branded 类型、`SESSION_FORMAT_VERSION = 4`（:90）、`SessionEventMap`（:287）、`SurfaceOp`（append 或 replace——compaction 的遮蔽机制就靠 replace）。`Session.append()`（`index.ts:806`）完成校验、入 log 并同步发布 `session/event`；`deriveMessages()`（:999）按 surface 节点投影模型历史。fork 通过 `ctx.agents.create({ sessionId, seed, inheritedEventCount, meta: { isSeeded: true } })` 在 turn 边界创建带继承前缀的新会话。

**`core/tools`（`ctx.tools`）——scoped 注册表 + 执行器与守卫管线**
`defineTool()` 负责参数 DSL 声明、类型推断和执行前校验。Scoped registry（`ToolLayer`，`index.ts:63`）让全局与 agent 作用域注册互相遮蔽；`ctx.tools.execute()`（:375）接收原始批次和并发限制。`tool-executor-native` 直接调用各工具，并行安全调用可重叠、独占调用构成屏障；`tool-executor-ptc` 只让模型直接调用 `run_code`，在程序内生成 SDK 调用其他工具。两者共用以下守卫管线：

```
tools/pre-execute(waterfall: allow/deny/cancel/ask)
  → serviceAsk() → ctx.approval（无服务时 fail-closed 为 deny）
  → guards（只有拒绝没有允许，顺序无法翻案）
  → tools/execute(waterfall) → 工具体
  → tools/post-execute(waterfall: accept/block)
  → tools/result(emit)
```

**`core/system-prompt`（`ctx.systemPrompt`）——prompt 组装**
贡献者注册 `PromptSection`（带 name/order）、`PromptContext`（动态快照）、工具 schema、变量；`assemble()`（`packages/core/system-prompt/src/index.ts:621`）做 scope 链遮蔽合并 → 排序（中央顺序表 `SECTION_ORDERS` :136）→ `{{variable}}` 严格插值（未定义即抛错）→ waterfall `system-prompt/assemble`。模型选择通过 `AssembleContext.modelSelection` 进入组装；系统提示只以 `system/message` 历史的形式存在，能力路由支持增量更新。

**`llm/llm`（`ctx.llm`）——消息词汇表 + 适配器缝**
`LlmAdapter` 抽象类（`packages/llm/llm/src/index.ts:208`）：唯一必需方法 `stream(options)`。`LlmRuntime.registerAdapter()`（:401）all-or-nothing 注册，返回带 `replace()` 的句柄支持原子路由替换（HMR 安全）。`prepareCall()`（:960）把模型元数据与派发绑定到同一代际。Provider 实例包括 `llm-pi-ai` 与 `llm-deepseek`；凭证缺失响亮失败（`MISSING_CREDENTIAL`）。

### 4.4 Approval / Sandbox / Guard：人类在环

一次典型的 bash 沙箱升级批准流：

```
模型发起 bash 调用（沙箱拒绝后带 sandbox_permissions 重试）
 → tool/call 落日志
 → tools/pre-execute waterfall（hooks 可 deny/ask）
 → serviceAsk()（tools/index.ts:1727）
 → ctx.approval.request()（interaction/user-approval/src/index.ts:215）
    ├─ append 'approval/asked'（durable 审计对的一半）
    ├─ 策略 'never'? → 确定性拒绝
    ├─ 'approval/request' waterfall → 远端 UI（Web 客户端）应答
    └─ append 'approval/decided'
 → allowed-once → guards → tools/execute → 工具体（提升仅此一次）
 → tool/result 落日志，模型在下一 step 读到结果
```

- `ctx.approval` 的四种结果：`allowed-once / rejected / cancelled / unavailable`（`user-approval/src/types.ts:32`），审计事件必须成对且在开放 turn 内（:217）。
- 三档 sandbox 模式：`read-only / workspace-write / danger-full-access`；`permission-presets` 把 sandbox mode + approval policy 打包成用户预设。
- 策略陈述都进运行时上下文快照（模型知道自己处于什么权限环境）。

## 5. 持久化与会话数据：append-only 的世界观

### 5.1 会话日志：事件源（event sourcing）

- **契约**：`ctx.sessionPersistence`（`packages/session/session-persistence/src/index.ts:135`）：`create/open/stat/list/flush`。语义核心：append-only、seq 连续、事件永不重写、撕裂尾部（torn tail）对读者不可见、`flush` 才是耐久屏障。
- **JSONL 后端**：磁盘布局 `~/.dsh/sessions/<projectKey>/<sessionId>/session.vN.jsonl.zstd`（`session-persistence-jsonl/src/format.ts:199,225,267`）。zstd 压缩带校验和；live 事件经 200ms 有界批量窗口写穿；跨进程写锁（lease）。
- **世代（generation）与迁移**：历史世代文件**不可变，永不移动/覆盖/删除**。`SESSION_FORMAT_VERSION = 4`；迁移是静态相邻链 `v0→v1→v2→v3→v4`（每包一条边，`packages/session/session-format-*/`），流式逐事件转换，在独立 worker 里验证后排他发布（POSIX 硬链接）。打开旧会话 = 选最高世代 → 必要时一次性组合迁移链 → 返回当前逻辑事件。
- **词汇增长护栏**：新事件类型默认 required-on-read，旧版本 harness 读到不认识的类型会**拒绝**而不是静默误读；信息性事件标 `ignorable: true` 豁免。

### 5.2 为什么没有"删除会话"

持久化契约没有 delete API——会话日志是审计/回放依据，只增不减。Web UI 的「归档会话」只是把 id 写进 workspace registry 的全局归档集（`packages/workspace/workspace/src/index.ts:363`），**只隐藏不删除**，可原位恢复。物理删除只能手动 `rm` 会话目录（我们在这台机器上做过）。

### 5.3 Compaction：不改写历史的上下文压缩

压缩不是删除旧消息，而是**追加**：

1. `compaction/start`（事务锁）→ `compaction/summary`（摘要 + 被遮蔽范围 + token 计量）→ `compaction/end`（释放锁）
2. 紧接着追加一条**带 `surfaceOp: {op:'replace', startSeq, endSeq}` 的 `user/message`**，遮蔽旧范围（`compaction-basic/src/region.ts:491-509`）

触发方式三选一：压力触发（`agent/pre-step` 监听，token 总量 ≥ `thresholdRatio 0.8` 才动手）、溢出恢复（请求报 context-window-exceeded 后强制压缩并重试）、手动 `/compact` 命令。事务不变式：范围必须 tool call/result 配对平衡；**摘要必须比被遮蔽内容便宜，否则拒绝**。

### 5.4 其余存储家族

| 子系统 | 机制 | 要点 |
|---|---|---|
| session-query | `ctx.sessionQuery` + SQLite FTS5 派生索引（`SCHEMA_VERSION = 8`，版本不符**原地重置**而非迁移） | "live 优先"读模型；组合默认 `:memory:` 不落盘 |
| attachment | 内容寻址（`sha256:<hex>`），`DSH_HOME/attachments/v1` | 字节永不进 JSONL，日志只存引用 |
| spill | 大工具输出全文外溢到进程私有临时目录，内联保留下头尾 | `maxInlineTokens` 策略控制 |
| storage | hub `ctx.storage` + domain 层 `ctx.storageDomain`，后端 sqlite/json | 非会话 KV（workspace 记录等） |
| session-projection | `ctx.sessionProjections`：注册单元 fold 已提交事件；carrier 通过 `snapshot()` 读取裁剪后的客户端视图 | 领域服务等待注册、保留私有句柄并暴露业务方法；`AgentRegistry.turnBoundaryOf(session)` 读 turn/step，`AgentLoop` 拥有模型选择和 inbox 状态 |
| workspace | 实体注册表；归档=隐藏；pin 与归档互斥 | 活跃会话默认拒档 |

## 6. 能力缝（Capability Seam）：最重要的设计模式

### 6.1 三角色定义

权威定义（`docs/architecture.md:131-137`）：

> 一个缝 = **Service Definition**（声明接口）+ **Service Provider**（实现）+ **Consumer**（消费，通常是面向模型的工具）。一个包可以身兼多角色，但只有一个角色不构成缝；添加能力 = 设计全部三个角色。

机制上：Definition 包用 declaration merging 往 `Context` 合并一个键 + 抽象 `Service` 类；Provider 子类化并注册；Consumer 用 `static inject` 声明依赖。**接线是 YAML 层的事**——换一个 provider 就是改 `cordis.patch.yml` 里的一行 `name:`，上层消费者零改动。

### 6.2 执行世界（execution world）：缝套缝的连锁反应

命令执行链路是四缝叠放：**tool-bash → `ctx.shell`（执行器）→ `ctx.subprocess`（起进程）→ `ctx.sandbox`（包裹 argv）**。关键设计决策（`.agents/notes/implemented/architecture/2026-07-28-portable-execution-world-consumers.md`）：

> `ctx.fs` 和 `ctx.subprocess` 共同定义一个 execution world；bash、LSP、PTT 等通用消费者只建立在这两个接口上。于是**把 fs+subprocess 换成远程 provider，Bash/PTY/LSP 就整体搬到远程沙箱，不需要任何 provider 分叉**。

实证是 `packages/ssh/`：四个包分别注册到**已有**缝上（`fs-ssh`→`ctx.fs`、`subprocess-ssh`→`ctx.subprocess`、`sandbox-ssh`→`ctx.sandbox`、`ssh`→`ctx.ssh` 共享连接），不新增任何 Definition——换上它们，`bash-local` 的 spawn、`lsp-stdio` 的 fs+spawn 原封不动就跑在远程 POSIX 主机上。

### 6.3 模式的三科变体

1. **标准三角**：shell / subprocess / sandbox / fs / terminal / web(search+fetch) / skill / lsp / ptc-runtime。
   - shell 契约是"显式 > 隐式"的全仓模板：`resolve(request)` 显式补默认值，`execute(spec)` 无隐藏默认（`packages/shell/shell/README.md:52`）。
   - **lsp 纪律**："Providers register capabilities, not tools"——模型可见的名字/schema 只归 tool-lsp，换 provider 永不变更模型请求（`packages/lsp/README.md:31`）。
   - **ptc-runtime**：模型不再逐个发工具调用，而是**写一段程序**，把工具当普通 async 函数调用，只回传打印输出和返回值——省中间上下文。Consumer 已独立为 `core/tool-executor-ptc`，与 `tool-executor-native` 一样挂载在最近作用域，同一作用域只能选择一个执行器。
   - **apply_patch**：`fs/fs-patch` 是纯字符串库，解析 Codex 风格补丁并应用文本 hunk，不直接做文件 I/O；文件系统 provider 负责路径、权限和行尾风格。
2. **注册器变体**：computer-use / browser-use 的 Definition 缩成"排他命名注册器"，Provider 兼任 Consumer（自带工具注册到 `ctx.tools`）。
3. **嫁接与事件**：mcp-client 不做自己的三角，而是把外部 MCP server 的工具以 `mcp__<server>__<tool>` 限定名嫁接进已有的 `ctx.tools`；hooks（Claude Code/Codex 桥接）纯走 `agent/*` 和 `tools/*` 事件扩展点，不立新服务——**能用事件解决的就不要立缝**。

## 7. Web 客户端栈：一套自给自足的 UI 架构

权威文档：`docs/subsystems/web-client.md`、`slots.md`、`client-modules.md`；规则总纲 `packages/client/AGENTS.md`（158 行，全是硬规则）。

### 7.1 三层架构（单向依赖，不可越层）

```
第1层 数据对象层（React-free）
  client/connection（传输世代、rpcId 铸造、认证）
  api/session-controller/client（ClientSessions → SessionManager → Session 三级模型）
  client/store（快照 store 引擎：zustand vanilla + immer，产物是"裸可观测源"）
        ↓ 裸可观测源
第2层 渲染机制层（ui-renderer，动态插件）
  唯一做 ctx↔React 集成的包：slot 渲染器、SessionProvider、uSES 适配器
  全应用唯一的 useSyncExternalStore 使用点（scoped-slots.tsx:1066-1092）
        ↓ 五份派生 props
第3层 展示组件层（ui-* 插件包的 src/client/）
  纯 props 的"消耗品"，永远见不到 ctx、transport、别的插件实现
```

核心红线：业务数据住对象层**永不进 store**；store 只装共享视图/交互状态（选中、草稿、面板宽度）；组件连 React context 都看不见。

### 7.2 Slot 系统：声明即授权

整个应用树 = **唯一一次** `ctx.slots.renderSlot('root', {})`（`ui-renderer/src/client/app.tsx:21`）。Slot 是类型化、可组合、生命周期级联的 UI 扩展点：

- **类型声明**：各包用 `declare module '@deepseek-ai/dsh-client-ui-slots'` 往 `SlotMap` 合并键（`ui-slots/src/index.ts:26`）。每个条目两轴：`kind`（single/list/keyed/chain）× `scope`（root/session-maybe/session）。
- **`register(options, component)`**（:1157-1312）一次调用完成四件事：向某 slot 贡献组件 + 声明子 slot（`children` 表 = **声明+授权+运行时 spec 三合一**——你只能渲染自己声明过的子键）+ 声明 store 席位 + inject 业务面。加载期校验失败即抛。
- **遮蔽与 fallback**：同格条目按 priority 共存、最低者渲染；无赢家时渲染 owner 在 renderSlot 处给的 `fallback`。
- **级联生命周期**：disposer 移除贡献并**递归坍塌**其声明的全部子 slot。
- **`ctx.slots.inject(key, callback)`**：跨包贡献的等待机制——声明存在即跑 callback，声明坍塌即移除，重新声明时重跑。

实战案例（我们改 logo 时走过的路）：`sidebar.brand.mark` slot 由 ui-sidebar 声明（`contract/slots.ts:16-25`），`SidebarRoot.tsx:225` 渲染时给 `fallback: <FishLogo/>`，official 构建 profile 下 ui-brand-official 用 `slots.inject` 注册官方标志占位——三方互不知道对方存在。

### 7.3 Host 侧：webserver / frontend-static / 认证

- **`dsh-host-webserver`**（`packages/host/webserver/src/index.ts`）：纯 `node:http` 路由注册服务，**不认识任何 harness 概念**。三类注册：具名路由、HTTP upgrade（WebSocket）、**唯一 fallback 席位**（第二注册者抛——"两个 fallback 无法组合"）。请求异常只记日志 + 400，绝不杀进程。
- **`dsh-host-frontend-static`**：认领 fallback 席位 serve `apps/web/dist`；index 响应**先过认证再渲染**。
- **认证**（`packages/client/connection/README.md:36-58`）：每进程铸造随机 launch token → 启动打印 `?token=...` URL → `GET /` 接受 token 换**签名 cookie**（HttpOnly/SameSite=Strict，密钥持久化在 `$DSH_HOME/.credentials.yaml`，默认 30 天）→ 302 到干净地址。请求信任前置：Host 必须 loopback 或 trusted、Origin 必须等于 Host、拒绝 `sec-fetch-site: cross-site`——防 DNS rebinding。
- **传输**：一元 RPC 走 HTTP POST JSON；流式走 `/api/remote.mux` WebSocket；连接世代由 `$events` 逻辑流的首帧 `ready` 发布，断线重连 500ms→10s 抖动退避。

### 7.4 客户端构建链与模块共享表

三段式：tsc（Client face）→ tsdown（产出 **CJS 闭包工厂** 的 `lib/client.js`：banner 调 `window.__ModuleLoader__.load({id, factory})`，外部模块经注入的同步 `require` 从模块表解析——无全局、无 import map）→ Vite 打 `apps/web/dist`。

模块共享表（**module graph 在 cordis DI 之下**，对照表在 `packages/client/AGENTS.md:87-97`）：

- 基线 `PLATFORM_MODULES`（`packages/client/web/src/platform.ts:8-14`）：react、react-dom、cordis、client-store、ui-slots、ui-primitives、ui-dockkit——shell 静态 import 种子，**所有动态 bundle 看到同一实例**（hook/element 身份不分裂）。
- `dsh.client.external`：非基线模块请求，**不是功能插件依赖机制**，只有基础设施/传输/生成装配可用；未声明的 workspace 值导入在构建时就是错误（`dsh-client-bundle-purity` 纯度门）。
- 对比：cordis `inject` 等服务（运行时、可替换、允许环）；module `external` 等模块（物化期同步 require、唯一身份、拒绝环）。

`apps/web` **没有独立 dev server**（`vite.config.ts:47-54` 硬抛错）——裸 `vite dev` 无法注入 `__DSH_BOOT__`，必须 `pnpm dsh web` 或 `pnpm run dev:web`。

### 7.5 i18n：typed dictionary

命名空间即类型（`LocaleNamespaceMap` 同款声明合并）；**zh 是键集事实源**（`packages/client/locale/src/locales/zh.ts:47`）；注册即双语平衡（缺键/多键编译错误）；查找链：条目命名空间 → 共享 common 命名空间 → 显示 key 本身。`verify-client-ui-i18n` 门禁拒绝一切硬编码文案——我们改 "DSH 本地构建" → "DataAgent" 走的就是字典 key `brand.localBuild`。

### 7.6 Desktop：同一份 Web 入口代码

`apps/web/src/main.ts:11-36` 检测 `globalThis.dshDesktopBoot` 存在即进入桌面路径：IPC 拿回 boot injections + streamBaseUrl → 手动插 `<script>` → 放行门闩 → 同一个 `AppWebEntry.run()`。Electron 侧 `dsh-app://` 协议分流：静态文件本地直出（不过 Host）、API 转发带 cookie、WebSocket 连认证过的 Host。关闭窗口=隐藏不关（Host 继续跑任务）。

## 8. 自动化与编排：agent 之上的那些包

### 8.1 subagent：委托的光谱（`ctx.subagents`）

按隔离/异构程度排列（`packages/subagent/`）：

- **进程内**：`spawn`（全新 child）、`fork`（带父历史前缀的 child）
- **进程外同构**：`subagent-dsh-sdk`（全新 dsh 子进程跑完整运行时）
- **进程外标准化协议**：`subagent-acp`（驱动任何 ACP 服务器）
- **另一个产品**：`subagent-codex`、`subagent-claude-code`——把一轮真正委托给别的 agent 产品

形态分 one-shot vs continuable；治理参数 `maxDepth` 默认 1、`maxActiveSubagents` 默认 8。shell 后台命令、PTY、subagent 都注册为 **job**。

### 8.2 四个"任务管理"包的区分（极易混淆）

| 包 | 一句话 | 服务 |
|---|---|---|
| **schedule** | **时间触发**：一次性/间隔/daily/weekly/cron，Host 重启保留，到期作为 follow-up 消息进原会话 | `ctx.schedule` |
| **goal** | **完成判据**：每会话一个持久目标，`goal-round-driver` 驱动连续 Round | `ctx.goals` |
| **plan** | **批准闸门**：计划模式，`exit_plan_mode` 把计划呈交用户批准；是引导非限制 | `ctx.planMode` |
| **todo** | **模型自维护的清单**：`todo_write` 整体替换，持久在会话日志 | 仅工具 |

### 8.3 其余编排件

- **jobs**（`ctx.jobs`）：后台任务注册表；模型工具 `job_output/job_list/job_kill`；完成通知按"繁忙注入/空闲唤醒"投递，`maxConsecutiveWakes` 封顶自激链。
- **workflow**（`ctx.workflowEngine`）：运行**模型写的纯 JavaScript 编排脚本**（`agent(prompt)` 扇出、`parallel()/pipeline()` 组合），执行后端复用 PTC 沙箱运行时。
- **webhook**（`ctx.webhookRuntime`）：外部事件触发 Session。刻意 **fire-and-forget**：不拥有交付数据库/队列/重试。
- **acp**：Agent Client Protocol（JSON-RPC over stdio），锁定标准 ACP v1，**刻意省略** DSH 专有呈现（无计划卡片/todo/终端）——"automation-only"的含义。
- **SDK（TS + Python）**：整个 agent loop 经换行分帧 JSON-RPC 投影到进程外——方法面极小（`initialize/session/prompt/shutdown` + `session.event` 全量通知）。Python SDK 捆绑版本锁死的运行时 wheel，必须显式 `dsh_home`。仓库规则：**agent-loop、会话生命周期、SessionEventMap 变更必须在同一 PR 更新两套 SDK 的预期输出**。
- **experimental/agent-team**（`ctx.agentTeams`）：单会话多 agent 团队——Lead 创建具名 teammate，持久 mailbox + CAS 版本化共享任务板；Lead 会话日志是唯一真源。
- **preset**（`ctx.agentPresets`）：**Host 共享一份 agent loop；每个 agent 用自己所选代际的工具/提示词/技能**。代际机制保证"最后一个引用释放后才销毁旧插件树"；Web 的 per-session 能力集就靠它。
- **extensions**：运行时自我修改两层——`tool-cordis` 让模型只读发现服务；`cordis-host-runner` 在 `node:vm` 里当场求值运行新插件源码（按会话隔离、重启即消失）；持久自我修改走 plugin-manager 安装 bundle（Creator 模式）。

## 9. 工程体系：约定如何被机械化

这个仓库最特别的部分。闭环是：**AGENTS.md 声明约定 → verify-* 门禁把约定机械化 → Agent Notes 记录 why → docs 双语承载知识 → snapshots/benchmarks 把行为变成可回放资产 → 自研发布管线保证版本从仓库可读**。

### 9.1 测试体系（`docs/testing.md`）

- **分层 lane**：unit（vitest）/ `test:coverage`（CI 门禁：**`packages/*/*/src` 逐文件 100% 行覆盖**，理念是"未覆盖的行往往是该删的死代码"）/ `test:e2e`（真 API，"We are DeepSeek — do not ration real-API tests"）/ `test:expected`（无密钥 CLI 期望输出）/ `test:snapshot`（录制会话回放）/ Web 三层（`test:gui` 内循环、`test:web` 浏览器比对）。
- **贯穿原则**：mock 只打昂贵/非确定性边界（LLM adapter、网络、时钟），下游全真；**验证世界而非自述**（e2e 断言靠重跑命令/重读文件，不对 agent 输出做关键词探测）；产品可见插件必须有经 Loader boot 的 REAL 组合测试；"单独跑才过"的 spec 视为 spec 缺陷。
- **snapshots/**：提交会话 JSONL 既是回放输入又是期望输出，被测进程一律经 `dsh` CLI + shipped profile 启动；CI 强制只读 replay，record/refresh 只能本地做。任何非平凡的模型可见/用户可见变更必须同 PR 更新回放场景。
- **benchmarks/**：用户路径级性能门禁（session-open、terminal-io 等），被计时代码在纯 Node worker 里跑（不经 tsx），预算写死在源常量里，环境变量不得覆盖。

### 9.2 verify-* 静态门禁（约 50 个）

`scripts/run-gates.ts` 是带依赖图的调度器，`check:*` 脚本都是它的薄壳。代表性门禁（每个都有同名 `.spec.ts`，门禁自己也被测试）：

| 门禁 | 贯彻的约定 |
|---|---|
| `verify-cordis-config` | `!!js` 只在 `config`/`disabled` 下允许；bare 插件必须在 manifest `dependencies` |
| `verify-client-ui-i18n` | Client 文案全部来自 locale 字典（450 文件语料下限防空扫） |
| `verify-application-entrypoints` | 应用启动只走 dsh profile |
| `verify-package-dependencies` | 从发布面反推并修复 npm dependency 段；`workspace:*` vs `workspace:~` |
| `verify-export-jsdoc` | 每个导出有 JSDoc，函数含 `@param`/`@returns` |
| `verify-no-unknown-casts` | 棘轮式禁新增 `as unknown`（baseline 只减不增） |
| `verify-doc-budgets` | 文档词数上限（根 AGENTS.md ≤1960 词） |
| `gen-* --check` 家族 | 生成物新鲜度（手改生成物会被打回） |

### 9.3 构建与双平面

当前要求 Node.js `>=24.9`。`pnpm run build` 并发执行 `build:artifacts` 与 `build:check`：前者完成 Host/Client 编译、tsdown 运行时 bundle 和 Web 构建，后者基于最新产物检查聚合测试与脚本；日常开发用 `build:artifacts`，完工前再跑 `build:check` 或完整 `build`。Host/Client 两个编译 face 分开（两边对 cordis `Context` 做同键不同服务的声明合并，放一个 Program 会撞车）。**source plane vs artifact plane** 铁律：静态门禁和测试经 `tsconfig.base.json` 的 `paths` 把 workspace import 解析到 `src`，消费构建产物的门禁必须显式声明。`pnpm dsh` 源码启动走 tsx 的 ESM-only hook——能到达的模块必须保持 ESM。

**排障经验：包残骸与悬空 symlink。** 当一个包从 git 树中删除（如 v0.2.0-rc.2 移除 `schedule-bundle`），它在本地的 `lib/`、`node_modules/` 残留不会被 git 清理（gitignore 覆盖），各级 `node_modules` 里的 symlink 会变成悬空链接，后续构建/桌面端同步时报 `ENOENT: no such file or directory, open '.../package.json'`。特征识别：包目录下只剩 `lib/` 和 `node_modules/`、没有 `package.json`。排查与清理：

```sh
# 找出所有悬空 symlink（目标已不存在）
find node_modules apps/*/.desktop-build -type l ! -exec test -e {} \; -print 2>/dev/null
# 确认目录是残骸（无 package.json）后删除，再重跑 pnpm install / 构建
```

桌面端的 `.desktop-build/` 损坏时可直接 `rm -rf` 重建，`pnpm run dev:desktop` 会重新生成。

### 9.4 文档与 Agent Notes

- **docs/ 双语机制**：一对文档 = `foo.md` + `foo.zh.md` + `foo.i18n.yaml` 三元组，i18n.yaml 逐节记录中英内容 hash，`verify-translation-pairing` 机械校验一致性。分层税法："一个事实一个家"——根 AGENTS.md 放命令、architecture.md 是地图、subsystems/ 放类型、cookbook/ 放步骤、Agent Notes 放 why、postmortem/ 放事故。**`docs/upgrade-guide/`** 按版本目录组织外部可感知的破坏性变更指南（根 AGENTS.md 规定此类变更必须立即记录），每个指南给出"变更内容 + 迁移步骤 + 验证方法"三段式——例如 `v0.2.0-rc.2/schedule-bundle-retired/` 记录了"自动化任务"可选 bundle 的移除及 profile 自动迁移逻辑。升级 dsh 后行为异常时，先查对应版本的 upgrade-guide 目录往往直接有答案。
- **.agents/notes/**：agent 写的 RFC/ADR，路径即元数据 `{lifecycle}/{class}/yyyy-mm-dd-topic.md`。生命周期 `proposed → implemented / rejected → archived`（archived 冻结永不可编辑）；六个封闭类别（刻意无 `refactor`）；`## Alternatives considered` 必填（记录被否决方案防翻案）。仅 `implemented/architecture/` 就有约 140 篇——**这是理解架构决策"为什么"的最宝贵素材库**。

### 9.5 发布

不用 changesets，自研三条独立发布序列（dsh / vendor / native）。核心原则：**发布的版本必须能从仓库读出来，CI 永不写仓库**。管线：`bump.ts`（本地）→ `verify.ts` → `pack.ts`（每个 PR 演练打包）→ `publish.ts`（仅 Actions 从 tag 手动 dispatch；同 integrity 跳过使重跑安全）。

## 10. 源码学习路径建议

按依赖深度排序，每条路径给出入口文件：

1. **先读文档**：根 `AGENTS.md`（约定总纲）→ `docs/architecture.md`（架构地图）→ `docs/cordis-primer.md`（框架速成）。这三份是整个仓库的索引。
2. **理解框架**：`vendor/cordis/src/` 按 `context.ts → service.ts → registry.ts → fiber.ts → events.ts` 顺序读（约 2000 行）。读完你就懂了 dsh 的"插件世界观"。
3. **跟踪一次请求**：从 `packages/bundle/headless/src/index.ts:365` 的 `agent.followup()` 出发，对照本文 §4.2 的行号表走一遍 `agent.ts` 的 turn/step。
4. **读懂一条能力缝**：推荐 shell 链（tool-bash → bash-local → subprocess-local → sandbox-local），它最完整地展示了 Definition/Provider/Consumer 三角色 + YAML 接线（`packages/bundle/base/cordis.patch.yml:220-287`）。
5. **按需深入子系统**：`docs/subsystems/*.zh.md` 每个主题一份权威参考；想找"为什么这样设计"就去 `.agents/notes/implemented/architecture/` 按主题搜 Agent Note。
6. **Web 客户端**：先读 `packages/client/AGENTS.md`（规则总纲）+ `docs/subsystems/slots.md`，再从 `ui-renderer/src/client/app.tsx:21` 的唯一 `renderSlot('root')` 出发。

## 11. 关键文件速查表

| 主题 | 文件 |
|---|---|
| 架构地图 | `docs/architecture.md` |
| Cordis 核心 | `vendor/cordis/src/{context,fiber,events,registry,service,reflect}.ts` |
| CLI 入口 | `apps/cli/src/{bin,args,profile-boot}.ts` |
| 启动组合 | `packages/boot/app-boot/src/index.ts`（boot :973-1039） |
| Agent 循环 | `packages/core/agent-loop/src/agent.ts`（turn :327 / step :435） |
| 会话类型 | `packages/core/session/src/types.ts`（SessionEventMap :287） |
| 工具注册与执行 | `packages/core/tools/src/index.ts`（execute :375）+ `pipeline.ts`（serviceAsk :432） |
| 工具执行器 | `packages/core/tool-executor-{native,ptc}/src/index.ts` |
| Prompt 组装 | `packages/core/system-prompt/src/index.ts`（assemble :621） |
| LLM 适配器 | `packages/llm/llm/src/index.ts`（LlmAdapter :208 / registerAdapter :401） |
| 审批 | `packages/interaction/user-approval/src/index.ts`（request :242） |
| 沙箱 | `packages/sandbox/sandbox-local/src/`（macOS Seatbelt profiles.ts） |
| 会话持久化 | `packages/session/session-persistence-jsonl/src/{format,storage,generation}.ts` |
| 格式迁移链 | `packages/session/session-format*/`（v0→v4 四包） |
| Compaction | `packages/compaction/compaction-basic/src/region.ts` |
| Slot 核心 | `packages/client/ui-slots/src/index.ts`（register :1157） |
| 渲染机制 | `packages/client/ui-renderer/src/client/scoped-slots.tsx` |
| Host webserver | `packages/host/webserver/src/index.ts` |
| 基础层组合 | `packages/bundle/base/cordis.patch.yml`（529 行） |
| Web 层组合 | `packages/bundle/web-app/cordis.patch.yml`（583 行） |
| 事件全表 | `docs/event-producer-consumer.md` |
| 测试政策 | `docs/testing.md` |
| 决策素材库 | `.agents/notes/implemented/architecture/`（约 140 篇） |

---

*本文基于 8 路并行代码探索整理，行号对应 `caiyang10` 分支当前 HEAD。仓库演进很快（pre-stable API），行号会漂移，但模块职责与设计模式相对稳定；发现出入时以 `docs/architecture.md` 为准。*
