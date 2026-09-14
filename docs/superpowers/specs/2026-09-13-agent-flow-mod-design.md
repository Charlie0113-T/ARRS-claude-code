# Agent flow mod 设计（第一阶段）

- 日期：2026-09-13
- 状态：待 Charles 审阅
- 范围：Claude Code 的一个 mod，`/flow` 在 transcript 旁边打开一个面板，实时显示本会话的子 agent 和 teammates 树。第一阶段只读加展开。

## 0. 背景与决策

上游 anthropics/claude-code 的 issue #24537 提出了 agent 层级仪表盘，七个月没有官方回应。本仓库 `mods/` 目录是内置 mod（diff、telemetry、sec-default）的源码，它们用 early access 的 function hooks API 实现。2026-09-13 的 spike 在 Claude Code 2.1.270 上证明：一个 mod 可以注册 `/flow`、用 `$.ui.open` 在 transcript 旁 dock 一个面板、通过 `agent.spawn`、`tool.call`、`turn.complete` 拿到 agent id、父 id、工具调用和耗时，并在每个事件后重绘。

已做的决定：

| 决定 | 选择 |
| --- | --- |
| 范围 | 第一阶段覆盖 Agent 工具的子 agent 和进程内 teammates；动态 workflow 的 agent 留到第二阶段 |
| 位置 | 本仓库 `mods/agent-flow`，与 diff 并列，结构照搬 diff |
| 交互 | 只读加展开；停止、发消息等操作留到 1.1 |
| 界面 | 终端 fullscreen 是主界面；VS Code 扩展 2.1.270 不画插件组件，必须有文本降级 |
| 架构 | 事件为主加定期校准（方案 C） |
| 仓库 | 工作区是 Charles 的 fork Charlie0113-T/ARRS-claude-code 的克隆；上游 anthropics/claude-code 作为 `upstream` 远端，偶尔同步 |

非目标：不做可拖拽的节点图（patoles/agent-flow 已经做了，定位是事后调试）；不做回放；不给 `$` 增加名词；不改上游任何文件。

## 1. 目录与模块划分

```text
mods/agent-flow/
  .claude-plugin/plugin.json
  README.md              # 挂了哪些事件、调了 $ 的哪些名词、怎么试
  tsconfig.json          # 独立于本仓库做类型检查时用；paths 把 claude-code/testing 映射到 bun/kit.ts
  bunfig.toml            # bun test 的 preload
  bun/
    preload.ts           # 定义全局 h / Fragment，产出 {type, props, children} 纯数据
    kit.ts               # 用 bun:test 实现 claude-code/testing 的 describe/test/expect/tier
    bun-test.d.ts        # bun:test 的环境声明，只给独立 tsconfig 用；mods/tsconfig.json 不包含 bun/
  hooks/
    hooks.json           # {"description": "...", "modules": ["./register.ts"]}
    register.ts          # 入口：session.start 建 host，注册 /flow，把事件接到 reducer 和视图
    host/                # Host 接口：模块用到的 $ 子集，session.start 时用闭包封一次
    names/               # 常量：pane id、标题、命令名、store 键、提示文本
    limits/              # 常量：探测超时、校准间隔、tick 间隔、阈值、上限
    model/               # 纯函数：FlowState、每个事件的 reducer、校准、裁剪、行序、卡住判定
    views/
      pane-view.tsx      # 模型到 RenderElement 树，面板用
      text-view.ts       # 模型到纯文本树，降级输出和 /flow text 用
      kit/               # 视图用到的元素类型：Box、Text、Button
    pane-toggle/         # /flow 该开、该关还是太窄，纯函数
  tests/
    fixtures/
    *.test.ts            # 纯函数测试，按所覆盖的单元命名，从 claude-code/testing 导入，bun test 直接跑
    register.kit.ts      # 按 claude-code/testing 写的入口测试，等 claude plugin test 上线后改名为 register.test.ts
  scripts/
    smoke.exp            # expect 冒烟脚本，手动跑
```

规则：

- `register.ts` 和 `host/` 只做接线，不含业务逻辑。所有逻辑在 `model/` 和 `views/` 的纯函数里。
- 每个单元一个文件夹，`index.ts` 转发导出，和 diff 一致。
- 类型来自仓库的 `mods/types/claude-code.d.ts`；`mods/tsconfig.json` 已经 include 了 `*/hooks` 和 `*/tests`，不复制声明文件。
- 不新增 `types/` 契约目录，因为不给 `$` 加名词。

## 2. 数据流与节点状态

### 2.1 事件与 reducer

reducer 全部是 `(state, event, now) => state` 的纯函数，返回新对象。register 在每个钩子里做三件事：调 reducer、按需触发重绘、按需同步计时器。

| 事件 | 钩子做什么 | 对 `next(e)` 的处理 |
| --- | --- | --- |
| `session.start` | 建 host；`$.command.register` 注册 `/flow`；记录 `e.surface`；调一次 `$.agent.list()` 做初始校准 | 返回 `next(e)` |
| `command.run{command:"flow"}` | 见 3.4 | 自己回答 `{ text }`，不调 next |
| `ui.close` | id 是本面板时把 `isBelievedOpen` 置 false，`origin` 为 person 时置 `closedByPerson` | 返回 `next(e)` |
| `ui.render{component:"PromptHint"}` | 记录 `viewport.columns`，供自动打开和宽度判断用 | 返回 `next(e)` |
| `ui.render{component:"Pane"}` | `requestId` 不是本面板时直接 `next(e)`；否则用 `$.ui.resolve(e)` 拿元素，画树 | 自己回答元素树 |
| `agent.spawn` | 先 `await next(e)`；结果带 `agentId` 时建节点；被拒绝时只记事件日志 | 原样返回结果 |
| `tool.call` | 调 next 之前把 `e.agentId` 对应节点的 activity 标成 tool；返回后清成 idle，累计次数、错误、最近工具 | 原样返回结果 |
| `turn.start` | 只有主循环会发；根节点标 busy | 返回 `next(e)` |
| `turn.complete` | 按 `e.reason` 改状态；记 `endedAt`；累加 `e.usage`；主循环则根节点标 idle | 返回 `next(e)` |
| `classic.PermissionRequest`、`classic.Notification` | 载荷带 agent_id 时把该节点 activity 标成 permission，否则标根节点 | 返回 `next(e)` |

观察型钩子永远恰好调用一次 `next(e)` 并原样返回它的结果，绝不返回 `deny`。

### 2.2 节点与状态

```ts
type NodeStatus =
  | 'running' | 'completed' | 'failed' | 'killed'
  | 'aborted' | 'refusal' | 'gone' | 'unknown'

type Activity =
  | { kind: 'idle' }
  | { kind: 'tool'; tool: string; since: number; toolUseId?: string }
  | { kind: 'permission'; tool?: string; since: number }

type FlowNode = {
  id: string                       // 根节点固定为 'main'
  parentId: string | null          // null 表示挂在根下
  source: 'root' | 'spawn' | 'list' | 'event'
  type: string                     // subagentType，teammate 为 'teammate'，来源 event 为 'loop'
  description: string
  name?: string                    // SendMessage 用的地址
  model?: string
  background?: boolean
  fork?: boolean
  promptExcerpt?: string           // prompt 前 120 字
  status: NodeStatus
  rawStatus?: string               // agent.list 给的原始字符串，映射不了时展示它
  spawnedAt?: number               // 只有 source 为 spawn 的节点有
  firstSeenAt: number
  endedAt?: number
  lastEventAt: number
  activity: Activity
  toolCalls: number
  errors: number
  recentTools: { tool: string; startedAt: number; durationMs?: number; isError?: boolean }[]  // 最多 5 条
  turns: number
  usage: { input: number; output: number; cacheRead: number; cacheWrite: number }
  misses: number                   // 连续几次校准时列表里没有它
}

type FlowEvent = { at: number; kind: string; agentId?: string; text: string }

type PaneState = {
  isBelievedOpen: boolean
  wasDrawnSinceProbe: boolean
  columns: number | null
  placement: 'dock' | 'inline' | null
  expanded: ReadonlySet<string>
  hasAutoOpened: boolean
  closedByPerson: boolean
}

type FlowState = {
  nodes: ReadonlyMap<string, FlowNode>
  events: readonly FlowEvent[]     // 环形，最多 50 条
  pane: PaneState
  startedSurface: RenderSurface | null   // session.start 的 e.surface
}
```

状态迁移：

- `agent.spawn` 成功：建 `running` 节点，来源 spawn。若校准已先建了同 id 的节点，则合并字段，保留列表给的状态。
- `tool.call` 开始：activity 变 tool；若节点处于终态，改回 `running`（被 SendMessage 唤醒的情况）。
- `tool.call` 结束：activity 变 idle，`toolCalls` 加一，结果 `isError` 时 `errors` 加一，`recentTools` 追加并保留最近 5 条。
- `turn.complete`：`answer` 变 `completed`，`error` 变 `failed`，`aborted` 变 `aborted`，`refusal` 变 `refusal`；记 `endedAt`；`turns` 加一；`usage` 累加；activity 变 idle。
- permission 事件：activity 变 permission，工具名已知时带上。该节点下一次 `tool.call` 开始或结束、或 `turn.complete` 时清除。
- 校准（`agent.list()`）：列表里的每一项 upsert；列表对状态和父 id 有最终解释权，映射不到枚举的原样存进 `rawStatus` 并把状态记为 `unknown`；`misses` 清零。来源为 spawn 或 list、状态为 `running` 而列表里没有的，`misses` 加一，达到 2 时变 `gone` 并记 `endedAt`。根节点和来源为 event 的节点不参与缺席判定。
- 根节点：`turn.start` 到 `turn.complete` 之间状态为 `running`，其余时间为 `completed`，视图分别显示为 busy 和 idle。

来源不明的 id：`tool.call` 或 `turn.complete` 带来 spawn 和列表都没见过的 agentId 时，建来源为 event 的节点，`parentId` 为 null，`type` 为 `loop`，描述为 id 前 8 位。视图把这类节点归到"unlisted loops"一组，默认折叠。workflow agent 和引擎自己的 fork（压缩、记忆）会走这条路，第一阶段只保证它们不静默消失。

### 2.3 计时器与重绘

- 有任何非根节点处于 `running` 时启动两个计时器，没有时全部停止，空闲会话零开销：每 2000 毫秒调一次 `$.agent.list()` 做校准；每 1000 毫秒触发一次重绘刷新耗时。
- 每个 reducer 之后调用 `syncTimers(state)`，用一个 `Map<'reconcile' | 'tick', Timer>` 持有句柄，和 diff 一样。
- 重绘请求经 100 毫秒合并：已排队则不再排，到期调一次 `$.ui.invalidate('ui.render')`。

### 2.4 卡住的判定

在 selector 里算，不存进状态：

| 判定 | 条件 | 显示 |
| --- | --- | --- |
| 等待批准 | activity 为 permission | 品红加粗，`◐`，文字 `waiting for approval: <tool>` |
| 工具跑慢 | activity 为 tool 且 `now - since > 30000` | 黄色加 `!`，文字 `<tool> <秒数>s` |
| 安静 | 状态 running、activity 为 idle、`now - lastEventAt > 120000` | 灰色，文字 `quiet <分钟>m` |

### 2.5 排序与容量

- 兄弟节点按 `firstSeenAt` 排，不按状态重排。
- 终态节点留在树上变暗。
- 节点超过 200 个时裁剪：只删终态且没有 running 后代的节点，按 `endedAt` 从早到晚删，删到 200 为止。

## 3. 渲染与降级

### 3.1 面板布局

从上到下：

1. 标题行：`Agent flow · 3 agents · 2 running · 1 waiting`，有未列出的循环时追加 `· 2 unlisted`。
2. 根行：`main · busy · Agent 12s` 或 `main · idle`。
3. 树：每行由树前缀（`├─`、`└─`、`│`）、状态符号、类型、name、描述、耗时、当前活动、调用次数组成；终态节点追加 token 总量。描述超过 60 字截断。
4. 每行末尾一个 `[+]` 按钮（展开后为 `[-]`）。展开显示四到六行细节：模型和是否后台、prompt 摘要、最近五次工具调用及各自耗时和是否出错、输入输出 token。展开集合记在 `PaneState.expanded`。
5. "unlisted loops" 组：有来源为 event 的节点时出现一行组标题加 `[+]`，默认折叠，在 `expanded` 集合里用键 `unlisted` 表示展开。
6. 耗时：来源为 spawn 的节点从 `spawnedAt` 起算；来源为 list 或 event 的节点从 `firstSeenAt` 起算并加 `~` 前缀，表示只是下限。
7. 底部一行变暗的最近事件。

状态符号与颜色：

| 状态 | 符号 | 颜色 |
| --- | --- | --- |
| running | `●` | yellow |
| completed | `✓` | green，dimColor |
| failed、killed、gone、aborted、refusal | `✗` | red |
| 等待批准 | `◐` | magenta，bold |
| unknown | `○` | 默认 |

文本示例：

```text
Agent flow · 3 agents · 2 running · 1 waiting
main · busy · Agent 1m12s
├─ ● general-purpose "Refactor auth" 1m12s · Edit 3s ×7                    [+]
│  └─ ✓ Explore "Find callers" 8s ×3 · 4.1k tok                            [+]
└─ ◐ Explore "Scan tests" 40s · waiting for approval: Bash                  [+]
last: tool.call Edit a50bb062 00:04:56
```

键盘、滚动、焦点都归引擎，mod 不处理按键。只用 Box、Text、Button 三种元素，不用 Select，desktop surface 能画时不用改代码。

### 3.2 宽度与 inline

- dock 时用 `e.props.bodyColumns` 决定每行截断，Text 用 `wrap="truncate-end"`。
- `e.props.placement` 为 `inline` 时画精简版：标题行，加正在等待批准或跑慢了的行，都没有就取前 5 个 running 行，最多 5 行，再加一行提示 `widen to 110 columns for the full tree`。

### 3.3 自动打开与偏好

- 本会话第一个来源为 spawn 的节点出现时，若满足全部条件则自动打开一次：`startedSurface` 为 `terminal`；`hasAutoOpened` 为 false；`closedByPerson` 为 false；store 里 `agent-flow.open` 不为 false；列数不小于 144，或 store 里 `agent-flow.open` 为 true 时不小于 110。
- `/flow` 手动开关时把 `agent-flow.open` 写进 `$.store`。
- 列数来自 `ui.render` 的 `PromptHint` 事件的 `viewport.columns`，和 diff 一样。

### 3.4 `/flow` 命令

- 无参数：按 `pane-toggle` 判定。相信面板已开且探测时确实画了，就关；否则开。
- 开：`$.ui.open({ id: 'agent-flow', title: 'Agent flow', focus: true })`，然后触发重绘并等 300 毫秒。期间收到本面板的 `ui.render` 即认定画出来了，返回 `{ text: undefined }`。没收到则认定这个 surface 画不了面板，`$.ui.close` 收回，返回 `{ text: <文本树> }`。
- `text`：不碰面板，直接返回 `{ text: <文本树> }`。
- 其他参数：返回一行用法说明。
- `-p` 会话和 VS Code 扩展会自然落到文本输出这条路。

### 3.5 状态行（可选，实现时可砍）

面板关闭而有节点在等待批准时，`$.ui.status('1 agent waiting for approval')`；等待解除时 `$.ui.status(undefined)` 清掉。面板打开时不用状态行。

## 4. 错误处理

- 观察型钩子内部的任何异常都被捕获，记进事件日志，不影响 `next(e)` 的调用和返回。
- `ui.render` 自己画失败时返回 `next(e)`，让引擎画默认内容。
- `command.run` 失败时返回 `{ text: 'agent flow: <原因>' }`。
- `$.agent.list()` 失败时保留旧状态，下一次校准再试。
- `$.store` 读写失败当作没有偏好。
- `$.command.register` 被拒绝（另一个 `/flow` 已存在）时用 `$.ui.log` 说一次，随后整个 mod 保持空闲，和 diff 的退让一致。
- reducer 对缺字段的事件宽容处理，永不抛出。

## 5. 测试策略

四层，从便宜到贵：

1. 静态：`tsc -p mods/tsconfig.json`；`claude plugin validate mods/agent-flow`。后者会检查 `$.env.get` 必须传字面量、`$` 只能传给文件顶层函数等规则，并列出模块挂的事件和调用的名词。
2. 单元：`cd mods/agent-flow && bun test`。覆盖每个 reducer、校准合并规则、来源不明 id 归组、裁剪、三种卡住判定、行序和前缀、`pane-toggle`、两个视图。视图测试通过 `bunfig.toml` 的 preload 注入极简的 `h` 和 `Fragment`，产出 `{type, props, children}` 纯数据，断言文本、颜色、按钮出现在预期位置。
3. 引擎套件：`tests/register.kit.ts` 按 `claude-code/testing` 写三条：`/flow` 开面板；spawn 后树里出现一行；别人已占 `/flow` 时退让。现在只做类型检查，`claude plugin test` 上线后改名启用。
4. 冒烟：`scripts/smoke.exp`。在 170 列的伪终端里启动 `claude --plugin-dir mods/agent-flow`，等待期间持续读取 pty，文字和回车分开发送。流程：`/flow`，派一个会再派 Explore 的 general-purpose agent，等待，`/flow` 关闭，`/exit`。断言屏幕流里出现标题、嵌套的两层节点、running 变 completed；再对照 `--debug-file` 里 `hooks module agent-flow loaded`、`command.run settled`、`ui.render settled ... key=agent-flow`。mod 本身不写调试日志。

## 6. 实现前必须验证的点

每条用一个小探针确认后再依赖：

1. `classic.PermissionRequest` 和 `classic.Notification` 的载荷是否带 agent_id。
2. Box 和 Text 允许的 props 到底有哪些（至少需要 `flexDirection`、`color`、`dimColor`、`bold`、`wrap`）。
3. 引擎的 `h` 产出的元素形状，测试用的 preload 要与之一致。
4. `$.store.get`、`$.store.set`、`$.ui.status` 在 2.1.270 可用。
5. VS Code 扩展里 `/flow` 确实打印文本树。
6. Button 的 `onPress` 闭包在面板重绘后仍然有效。

## 7. 完成标准

- 单元测试全绿；静态检查干净。
- 冒烟测试在 fullscreen 终端里看到两层嵌套树从 running 到 completed。
- VS Code 扩展里 `/flow` 打印文本树。
- README 写清挂了哪些事件、调了哪些名词、怎么试。
- 合并到 fork 的 main。

## 8. 后续阶段

- 1.1 操作：行内加 `[stop]` 和 `[msg]` 按钮，通过 `$.tool.call` 调 TaskStop 和 SendMessage，需要先验证权限确认在面板里怎么弹。
- 第二阶段 workflow：目前 `$.agent.list()` 不含 workflow agent，它们只会以来源为 event 的匿名节点出现。候选数据源：上游把它们列进 `agent.list`；或 run journal 对插件开放；或 OpenTelemetry 里的 `workflow.run_id` 属性。等其中一条可用再设计。

## 9. 仓库与同步

- 工作区就是 fork Charlie0113-T/ARRS-claude-code 的克隆，`origin` 指向它，主分支 `main` 与上游历史一致。
- 添加 `upstream` 远端指向 anthropics/claude-code。同步方式：`git fetch upstream && git merge upstream/main`。mod 只住在 `mods/agent-flow`，设计文档只住在 `docs/superpowers/`，上游都没有这两个路径，合并基本不会冲突。
- 开发在分支 `agent-flow-mod` 上进行，完成后合并到 fork 的 `main`。

## 附录 A：常量

| 名字 | 值 |
| --- | --- |
| OPEN_PROBE_MS | 300 |
| RECONCILE_MS | 2000 |
| TICK_MS | 1000 |
| INVALIDATE_DEBOUNCE_MS | 100 |
| SLOW_TOOL_MS | 30000 |
| QUIET_MS | 120000 |
| GONE_AFTER_MISSES | 2 |
| MAX_NODES | 200 |
| AUTO_OPEN_MIN_COLUMNS | 144 |
| KEPT_OPEN_MIN_COLUMNS | 110 |
| INLINE_MAX_ROWS | 5 |
| DESCRIPTION_MAX_CHARS | 60 |
| PROMPT_EXCERPT_CHARS | 120 |
| RECENT_TOOLS | 5 |
| EVENT_LOG_SIZE | 50 |

## 附录 B：名字

| 名字 | 值 |
| --- | --- |
| PANE_ID | `agent-flow` |
| PANE_TITLE | `Agent flow` |
| COMMAND_NAME | `flow` |
| COMMAND_DESCRIPTION | `Toggle the agent flow pane; /flow text prints the tree` |
| STORE_OPEN_KEY | `agent-flow.open` |
| ROOT_ID | `main` |
| UNLISTED_GROUP_LABEL | `unlisted loops` |

## 附录 C：参考

- 类型声明：`mods/types/claude-code.d.ts`（由 `/plugin-types` 生成，随版本变化）。
- 结构范本：`mods/diff`，特别是 `hooks/register.ts` 的 host 封装、探测开面板、退让逻辑。
- spike：会话暂存目录 `agent-flow-spike/`，一次性代码，验证了本设计依赖的全部事件。
- 上游讨论：anthropics/claude-code#24537。
- 相关工具：patoles/agent-flow，外部节点图可视化，与本 mod 互补。
