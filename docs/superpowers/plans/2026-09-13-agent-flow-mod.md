# Agent Flow Mod Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship `mods/agent-flow`, a Claude Code mod whose `/flow` command opens a pane beside the transcript showing a live, read-only tree of the session's subagents and teammates, with a text fallback on surfaces that cannot draw panes.

**Architecture:** Engine events (`agent.spawn`, `tool.call`, `turn.*`, classic permission events) feed pure reducers over an immutable `FlowState`; `$.agent.list()` reconciles that state every 2 s while anything runs. Pure "row" selectors turn the state into lines; two thin views render those rows as a `RenderElement` tree (pane) or plain text (fallback). `hooks/register.ts` only wires hooks to reducers, timers and views.

**Tech Stack:** Claude Code function hooks (early access, `mods/types/claude-code.d.ts`), TypeScript, JSX with the `h` factory, `bun test` for unit tests (mapped onto the `claude-code/testing` API via tsconfig `paths`), `claude plugin validate`, `expect` for the smoke test.

**Spec:** `docs/superpowers/specs/2026-09-13-agent-flow-mod-design.md`

## Global Constraints

- Claude Code 2.1.270 or newer. Hooks modules are early access and off by default; every live `claude` run in this plan must first `source mods/agent-flow/scripts/local-env.sh` (git-ignored, already present on this machine), which exports the override. Never copy that variable's name into the repository, the README, or any public post.
- Everything lives under `mods/agent-flow/`; no upstream file is modified.
- `tsc -p mods/tsconfig.json` must stay clean (it already includes `*/hooks` and `*/tests`; it must never see a `bun:test` import, so the bun shim lives in `mods/agent-flow/bun/`, outside those globs).
- `claude plugin validate mods/agent-flow` must pass: `$.env.get` takes only literal names (we call none); `$` may only be passed to functions declared at the top level of a file; `$` is always spelled `$.noun.method`.
- Observation hooks call `next(e)` exactly once and return its result unchanged; never `{ deny }`.
- Only `Box`, `Text`, `Button` elements; Text props limited to `color`, `dimColor`, `bold`, `wrap`; Box props limited to `flexDirection`, `marginTop`, `paddingLeft`.
- Names: `PANE_ID = 'agent-flow'`, `PANE_TITLE = 'Agent flow'`, `COMMAND_NAME = 'flow'`, `STORE_OPEN_KEY = 'agent-flow.open'`, `ROOT_ID = 'main'`, `UNLISTED_KEY = 'unlisted'`.
- Limits: `OPEN_PROBE_MS 300`, `RECONCILE_MS 2000`, `TICK_MS 1000`, `INVALIDATE_DEBOUNCE_MS 100`, `SLOW_TOOL_MS 30000`, `QUIET_MS 120000`, `GONE_AFTER_MISSES 2`, `MAX_NODES 200`, `AUTO_OPEN_MIN_COLUMNS 144`, `KEPT_OPEN_MIN_COLUMNS 110`, `INLINE_MAX_ROWS 5`, `DESCRIPTION_MAX_CHARS 60`, `PROMPT_EXCERPT_CHARS 120`, `RECENT_TOOLS 5`, `EVENT_LOG_SIZE 50`.
- Every test file imports `describe`, `expect`, `test`, `tier` from `'claude-code/testing'` and never uses the `$`/`on` arguments, so the same file runs under `bun test` now and under `claude plugin test` later.
- Commit after every task on branch `agent-flow-mod`; commit messages end with `Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>`.
- Work from the repository root `/Users/charles/Desktop/ARRS-claude-code-main`; unit tests run with `cd mods/agent-flow && bun test`.

---

## File Structure

| Path | Responsibility |
| --- | --- |
| `mods/agent-flow/.claude-plugin/plugin.json` | Plugin manifest |
| `mods/agent-flow/hooks/hooks.json` | Names the hooks module |
| `mods/agent-flow/tsconfig.json` | Standalone typecheck; `paths` maps `claude-code/testing` to the bun shim; includes `.claude/types`, `hooks`, `tests`, `bun` |
| `mods/agent-flow/bunfig.toml` | `bun test` preload |
| `mods/agent-flow/bun/preload.ts` | Defines global `h` and `Fragment` producing plain `{type, props, children}` data |
| `mods/agent-flow/bun/kit.ts` | Re-exports `bun:test` under the `claude-code/testing` names |
| `mods/agent-flow/bun/bun-test.d.ts` | Ambient declaration of `bun:test` for the standalone tsconfig |
| `mods/agent-flow/hooks/names/ids.ts`, `texts.ts`, `index.ts` | Constants: ids and user-facing texts |
| `mods/agent-flow/hooks/limits/timing.ts`, `sizes.ts`, `columns.ts`, `index.ts` | Constants: timing, sizes, columns |
| `mods/agent-flow/hooks/model/flow-state.ts` | Types, initial state, node/event helpers |
| `mods/agent-flow/hooks/model/ensure-node.ts` | Creates an event-sourced node for an unknown agent id |
| `mods/agent-flow/hooks/model/prune.ts` | Caps the node count |
| `mods/agent-flow/hooks/model/on-spawn.ts` | `agent.spawn` reducer |
| `mods/agent-flow/hooks/model/on-tool.ts` | `tool.call` start/end reducers |
| `mods/agent-flow/hooks/model/on-turn.ts` | `turn.start` / `turn.complete` reducers |
| `mods/agent-flow/hooks/model/on-permission.ts` | Permission-request reducer |
| `mods/agent-flow/hooks/model/reconcile.ts` | `$.agent.list()` merge |
| `mods/agent-flow/hooks/model/signal-of.ts` | Stuck detection |
| `mods/agent-flow/hooks/model/elapsed-of.ts` | Elapsed-time text |
| `mods/agent-flow/hooks/model/counts-of.ts` | Header counts |
| `mods/agent-flow/hooks/model/rows-of.ts` | State to rows (full and inline) |
| `mods/agent-flow/hooks/model/index.ts` | Re-exports |
| `mods/agent-flow/hooks/views/kit/ui.ts` | `Ui` element table type |
| `mods/agent-flow/hooks/views/text-view.ts` | Rows to text |
| `mods/agent-flow/hooks/views/pane-view.tsx` | Rows to `RenderElement` |
| `mods/agent-flow/hooks/pane-toggle/pane-toggle-of.ts`, `should-auto-open.ts`, `index.ts` | Open/close and auto-open decisions |
| `mods/agent-flow/hooks/host/host.ts`, `index.ts` | The `$` subset the module uses |
| `mods/agent-flow/hooks/register.ts` | Wiring |
| `mods/agent-flow/tests/fixtures/index.ts`, `nodes.ts` | Test builders |
| `mods/agent-flow/tests/*.test.ts` | Unit tests, one per unit |
| `mods/agent-flow/tests/register.kit.ts` | Engine-kit test, activated when `claude plugin test` ships |
| `mods/agent-flow/scripts/smoke.exp`, `smoke.sh` | Interactive smoke test |
| `mods/agent-flow/README.md` | What it hooks, what it calls, how to try it |

---

### Task 1: Scaffold, constants and the test harness

**Files:**
- Create: `mods/agent-flow/.claude-plugin/plugin.json`
- Create: `mods/agent-flow/hooks/hooks.json`
- Create: `mods/agent-flow/hooks/register.ts` (empty registrar for now)
- Create: `mods/agent-flow/tsconfig.json`, `mods/agent-flow/bunfig.toml`
- Create: `mods/agent-flow/bun/preload.ts`, `mods/agent-flow/bun/kit.ts`, `mods/agent-flow/bun/bun-test.d.ts`
- Create: `mods/agent-flow/hooks/names/ids.ts`, `texts.ts`, `index.ts`
- Create: `mods/agent-flow/hooks/limits/timing.ts`, `sizes.ts`, `columns.ts`, `index.ts`
- Test: `mods/agent-flow/tests/names.test.ts`

**Interfaces:**
- Produces: `Names.PANE_ID`, `Names.PANE_TITLE`, `Names.COMMAND_NAME`, `Names.COMMAND_DESCRIPTION`, `Names.STORE_OPEN_KEY`, `Names.ROOT_ID`, `Names.UNLISTED_KEY`, `Names.UNLISTED_GROUP_LABEL`, `Names.NO_AGENTS_TEXT`, `Names.WIDEN_TEXT`, `Names.USAGE_TEXT`, `Names.REGISTER_FAILED_TEXT`, `Names.BUILTIN_HOLDS_PATTERN` (all `string` except the RegExp); every limit named in Global Constraints as `Limits.<NAME>: number`.

- [ ] **Step 1: Write the failing test**

`mods/agent-flow/tests/names.test.ts`:

```ts
import { describe, expect, test, tier } from 'claude-code/testing'

import Limits from '../hooks/limits'
import Names from '../hooks/names'

tier('user')

describe('names and limits', () => {
  test('the pane, command and store names are the ones the spec fixes', () => {
    expect(Names.PANE_ID).toBe('agent-flow')
    expect(Names.PANE_TITLE).toBe('Agent flow')
    expect(Names.COMMAND_NAME).toBe('flow')
    expect(Names.STORE_OPEN_KEY).toBe('agent-flow.open')
    expect(Names.ROOT_ID).toBe('main')
    expect(Names.UNLISTED_KEY).toBe('unlisted')
  })

  test('the limits are the ones the spec fixes', () => {
    expect(Limits.OPEN_PROBE_MS).toBe(300)
    expect(Limits.RECONCILE_MS).toBe(2000)
    expect(Limits.TICK_MS).toBe(1000)
    expect(Limits.MAX_NODES).toBe(200)
    expect(Limits.AUTO_OPEN_MIN_COLUMNS).toBe(144)
    expect(Limits.KEPT_OPEN_MIN_COLUMNS).toBe(110)
  })
})
```

- [ ] **Step 2: Create the harness files**

`mods/agent-flow/.claude-plugin/plugin.json`:

```json
{
  "name": "agent-flow",
  "version": "0.1.0",
  "description": "The agent flow pane: /flow opens a live tree of the session's subagents and teammates beside the transcript, fed by engine events, with a text fallback where no pane can be drawn.",
  "author": { "name": "Charles Tao" }
}
```

`mods/agent-flow/hooks/hooks.json`:

```json
{
  "description": "The agent flow pane: /flow toggles a pane drawing the session's subagent tree from agent.spawn, tool.call and turn.complete, reconciled with agent.list while agents run; /flow text prints the same tree",
  "modules": ["./register.ts"]
}
```

`mods/agent-flow/hooks/register.ts` (placeholder-free minimal module; Task 10 replaces it):

```ts
import type { On } from 'claude-code'

/**
 * Registers nothing yet: Task 10 wires the hooks. Kept so the plugin
 * validates and loads from the first commit on.
 *
 * @param on the engine's registrar
 */
export function register(on: On) {
  void on
}
```

`mods/agent-flow/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "es2023",
    "lib": ["es2023"],
    "types": [],
    "module": "esnext",
    "moduleResolution": "bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noEmit": true,
    "skipLibCheck": true,
    "jsx": "react",
    "jsxFactory": "h",
    "jsxFragmentFactory": "Fragment",
    "paths": { "claude-code/testing": ["./bun/kit.ts"] }
  },
  "include": [".claude/types", "../types/claude-code.d.ts", "hooks", "tests", "bun"]
}
```

`mods/agent-flow/bunfig.toml`:

```toml
[test]
preload = ["./bun/preload.ts"]
```

`mods/agent-flow/bun/preload.ts`:

```ts
/**
 * The JSX globals a hooks module has at run time, for bun test: `h` builds
 * plain data, calling a constructor tag with its props and flattened
 * children, and `Fragment` is a column Box, as the engine's is.
 */
type Element = { type: string; props: Record<string, unknown>; children: unknown[] }
type Constructor = (props: Record<string, unknown>) => unknown

const h = (
  tag: unknown,
  props: Record<string, unknown> | null | undefined,
  ...children: unknown[]
): unknown => {
  const flat = children
    .flat(Infinity)
    .filter(child => child !== null && child !== undefined && child !== false)

  if (typeof tag === 'function') {
    return (tag as Constructor)({ ...(props ?? {}), children: flat })
  }

  const element: Element = { type: String(tag), props: props ?? {}, children: flat }

  return element
}

const Fragment = (props: { children?: unknown[] }): Element => ({
  type: 'Box',
  props: { flexDirection: 'column' },
  children: props.children ?? [],
})

Object.assign(globalThis, { h, Fragment })
```

`mods/agent-flow/bun/kit.ts`:

```ts
import { describe as bunDescribe, expect as bunExpect, test as bunTest } from 'bun:test'

/**
 * The subset of `claude-code/testing` the unit tests use, over bun:test, so
 * the same files run here now and under `claude plugin test` later. `$` and
 * `on` are handed as undefined: a unit test never touches them.
 */
export const describe = bunDescribe
export const expect = bunExpect
export const test = (name: string, fn: ($: unknown, on: unknown) => unknown): void =>
  bunTest(name, () => fn(undefined, undefined))
export const tier = (_tier: string): void => undefined
```

`mods/agent-flow/bun/bun-test.d.ts`:

```ts
declare module 'bun:test' {
  export const describe: (name: string, body: () => void) => void
  export const test: (name: string, body: () => unknown) => void
  export const expect: (received: unknown, message?: string) => Record<string, (...args: unknown[]) => void>
}
```

- [ ] **Step 3: Create the constants**

`mods/agent-flow/hooks/names/ids.ts`:

```ts
/** The one pane the plugin opens; its `requestId` at `ui.render`. */
export const PANE_ID = 'agent-flow'
/** The pane's tab title while another pane is open too. */
export const PANE_TITLE = 'Agent flow'
/** The slash command, without the slash. */
export const COMMAND_NAME = 'flow'
/** The command's line in /help. */
export const COMMAND_DESCRIPTION = 'Toggle the agent flow pane; /flow text prints the tree'
/** The store key remembering whether the person keeps the pane open. */
export const STORE_OPEN_KEY = 'agent-flow.open'
/** The root node: the main conversation loop. */
export const ROOT_ID = 'main'
/** The expanded-set key of the "unlisted loops" group. */
export const UNLISTED_KEY = 'unlisted'
/** The group's label. */
export const UNLISTED_GROUP_LABEL = 'unlisted loops'
```

`mods/agent-flow/hooks/names/texts.ts`:

```ts
export const NO_AGENTS_TEXT = 'no subagents yet; ask Claude to use the Agent tool'
export const WIDEN_TEXT = 'widen to 110 columns for the full tree'
export const USAGE_TEXT = 'usage: /flow (toggle the pane) or /flow text (print the tree)'
export const REGISTER_FAILED_TEXT = 'agent flow: could not register /flow: '
/** What `$.command.register` says when another /flow already holds the name. */
export const BUILTIN_HOLDS_PATTERN = /already|holds|registered/i
```

`mods/agent-flow/hooks/names/index.ts`:

```ts
export * from './ids'
export * from './texts'

export * as default from '.'
```

`mods/agent-flow/hooks/limits/timing.ts`:

```ts
/** How long an open pane gets to draw before its surface is judged unable to. */
export const OPEN_PROBE_MS = 300
/** How often `$.agent.list()` corrects the tree while anything runs. */
export const RECONCILE_MS = 2000
/** How often the pane redraws to refresh elapsed times while anything runs. */
export const TICK_MS = 1000
/** How long state changes gather before one `$.ui.invalidate`. */
export const INVALIDATE_DEBOUNCE_MS = 100
/** A tool call longer than this is flagged slow. */
export const SLOW_TOOL_MS = 30000
/** A running agent with no event for this long is flagged quiet. */
export const QUIET_MS = 120000
```

`mods/agent-flow/hooks/limits/sizes.ts`:

```ts
/** Consecutive reconciles a running node may miss from the list before it is gone. */
export const GONE_AFTER_MISSES = 2
/** The most nodes kept; finished ones are pruned beyond it. */
export const MAX_NODES = 200
/** Rows drawn when the pane sits inline above the prompt. */
export const INLINE_MAX_ROWS = 5
/** A description longer than this is cut with an ellipsis. */
export const DESCRIPTION_MAX_CHARS = 60
/** How much of a spawn prompt the node keeps. */
export const PROMPT_EXCERPT_CHARS = 120
/** How many recent tool calls a node keeps. */
export const RECENT_TOOLS = 5
/** How many events the ring buffer keeps. */
export const EVENT_LOG_SIZE = 50
```

`mods/agent-flow/hooks/limits/columns.ts`:

```ts
/** The pane opens by itself on the first spawn from this width. */
export const AUTO_OPEN_MIN_COLUMNS = 144
/** ... or from this width when the person kept it open before. */
export const KEPT_OPEN_MIN_COLUMNS = 110
```

`mods/agent-flow/hooks/limits/index.ts`:

```ts
export * from './columns'
export * from './sizes'
export * from './timing'

export * as default from '.'
```

- [ ] **Step 4: Run everything**

Run: `cd mods/agent-flow && bun test`
Expected: `2 pass`, `0 fail`.

Run: `cd /Users/charles/Desktop/ARRS-claude-code-main && bunx tsc -p mods/tsconfig.json && bunx tsc -p mods/agent-flow/tsconfig.json`
Expected: no output from either.

Run: `claude plugin validate mods/agent-flow`
Expected: `✔ Validation passed`.

- [ ] **Step 5: Commit**

```bash
git add mods/agent-flow
git commit -m "agent-flow: scaffold, constants and the bun test harness

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

### Task 2: FlowState, node helpers, pruning and the spawn reducer

**Files:**
- Create: `mods/agent-flow/hooks/model/flow-state.ts`
- Create: `mods/agent-flow/hooks/model/ensure-node.ts`
- Create: `mods/agent-flow/hooks/model/prune.ts`
- Create: `mods/agent-flow/hooks/model/on-spawn.ts`
- Create: `mods/agent-flow/hooks/model/index.ts`
- Create: `mods/agent-flow/tests/fixtures/index.ts`, `mods/agent-flow/tests/fixtures/nodes.ts`
- Test: `mods/agent-flow/tests/flow-state.test.ts`, `mods/agent-flow/tests/on-spawn.test.ts`

**Interfaces:**
- Consumes: `Names.ROOT_ID`, `Limits.EVENT_LOG_SIZE`, `Limits.MAX_NODES`, `Limits.PROMPT_EXCERPT_CHARS`.
- Produces (all exported from `hooks/model/index.ts`):
  - types `NodeStatus`, `NodeSource`, `Activity`, `RecentTool`, `Usage`, `FlowNode`, `FlowEvent`, `PaneState`, `FlowState`, `SpawnInput`
  - `ZERO_USAGE: Usage`, `INITIAL_PANE: PaneState`, `isTerminal(status: NodeStatus): boolean`
  - `initialState(now: number): FlowState` (holds the root node `main`)
  - `newNode(seed: NodeSeed, now: number): FlowNode` where `NodeSeed = Pick<FlowNode, 'id' | 'parentId' | 'source' | 'type' | 'description'> & Partial<FlowNode>`
  - `nodeOf(state, id): FlowNode | undefined`, `withNode(state, node): FlowState`, `withEvent(state, event: FlowEvent): FlowState`, `withPane(state, patch: Partial<PaneState>): FlowState`
  - `ensureNode(state, agentId: string | undefined, now): FlowState`
  - `prune(state): FlowState`
  - `onSpawn(state, spawn: SpawnInput, now): FlowState`

- [ ] **Step 1: Write the failing tests**

`mods/agent-flow/tests/fixtures/nodes.ts`:

```ts
import type { SpawnInput } from '../../hooks/model'

/**
 * A spawn as the Agent tool raises it, with the fields a test cares about
 * overridden.
 */
export function spawnOf(id: string, overrides: Partial<SpawnInput> = {}): SpawnInput {
  return {
    agentId: id,
    subagentType: 'Explore',
    description: `task ${id}`,
    background: false,
    fork: false,
    prompt: `Do task ${id}`,
    ...overrides,
  }
}
```

`mods/agent-flow/tests/fixtures/index.ts`:

```ts
export * from './nodes'

export * as default from '.'
```

`mods/agent-flow/tests/flow-state.test.ts`:

```ts
import { describe, expect, test, tier } from 'claude-code/testing'

import Limits from '../hooks/limits'
import Model from '../hooks/model'
import Names from '../hooks/names'

tier('user')

describe('flow-state', () => {
  test('the initial state holds an idle root and nothing else', () => {
    const state = Model.initialState(1000)

    expect(state.nodes.size).toBe(1)
    expect(Model.nodeOf(state, Names.ROOT_ID)?.status).toBe('completed')
    expect(Model.nodeOf(state, Names.ROOT_ID)?.source).toBe('root')
    expect(state.events).toHaveLength(0)
    expect(state.pane.isBelievedOpen).toBe(false)
  })

  test('withNode replaces by id without touching the old state', () => {
    const state = Model.initialState(0)
    const node = Model.newNode(
      { id: 'a', parentId: null, source: 'spawn', type: 'Explore', description: 'x' },
      5,
    )
    const next = Model.withNode(state, node)

    expect(state.nodes.size).toBe(1)
    expect(next.nodes.size).toBe(2)
    expect(Model.nodeOf(next, 'a')?.firstSeenAt).toBe(5)
    expect(Model.nodeOf(next, 'a')?.status).toBe('running')
  })

  test('withEvent keeps only the newest EVENT_LOG_SIZE events', () => {
    let state = Model.initialState(0)

    for (let i = 0; i < Limits.EVENT_LOG_SIZE + 3; i += 1) {
      state = Model.withEvent(state, { at: i, kind: 'k', text: `e${i}` })
    }

    expect(state.events).toHaveLength(Limits.EVENT_LOG_SIZE)
    expect(state.events[0]?.text).toBe('e3')
  })

  test('ensureNode adds an event-sourced node for an unknown id and nothing for a known one', () => {
    const state = Model.ensureNode(Model.initialState(0), 'abcdef0123456789', 7)

    expect(Model.nodeOf(state, 'abcdef0123456789')?.source).toBe('event')
    expect(Model.nodeOf(state, 'abcdef0123456789')?.type).toBe('loop')
    expect(Model.nodeOf(state, 'abcdef0123456789')?.description).toBe('abcdef01')
    expect(Model.ensureNode(state, 'abcdef0123456789', 9)).toBe(state)
    expect(Model.ensureNode(state, undefined, 9)).toBe(state)
  })

  test('prune drops the oldest finished leaves once MAX_NODES is exceeded', () => {
    let state = Model.initialState(0)

    for (let i = 0; i < Limits.MAX_NODES + 5; i += 1) {
      state = Model.withNode(
        state,
        Model.newNode(
          { id: `n${i}`, parentId: null, source: 'spawn', type: 'Explore', description: '' },
          i,
        ),
      )
      const node = Model.nodeOf(state, `n${i}`)
      if (node && i < Limits.MAX_NODES) {
        state = Model.withNode(state, { ...node, status: 'completed', endedAt: i })
      }
    }

    const pruned = Model.prune(state)

    expect(pruned.nodes.size).toBe(Limits.MAX_NODES)
    expect(Model.nodeOf(pruned, 'n0')).toBe(undefined)
    expect(Model.nodeOf(pruned, `n${Limits.MAX_NODES + 4}`)?.status).toBe('running')
  })
})
```

`mods/agent-flow/tests/on-spawn.test.ts`:

```ts
import { describe, expect, test, tier } from 'claude-code/testing'

import Model from '../hooks/model'
import Fixtures from './fixtures'

tier('user')

describe('on-spawn', () => {
  test('a spawn under main becomes a running node with the spawn facts', () => {
    const state = Model.onSpawn(
      Model.initialState(0),
      Fixtures.spawnOf('a', { model: 'claude-sonnet-5', prompt: 'p'.repeat(200), name: 'scout' }),
      10,
    )
    const node = Model.nodeOf(state, 'a')

    expect(node?.status).toBe('running')
    expect(node?.parentId).toBe(null)
    expect(node?.source).toBe('spawn')
    expect(node?.spawnedAt).toBe(10)
    expect(node?.model).toBe('claude-sonnet-5')
    expect(node?.name).toBe('scout')
    expect(node?.promptExcerpt).toHaveLength(120)
    expect(state.events[0]?.kind).toBe('agent.spawn')
  })

  test('a spawn inside a subagent hangs under that parent', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)
    state = Model.onSpawn(state, Fixtures.spawnOf('b', { parentAgentId: 'a' }), 2)

    expect(Model.nodeOf(state, 'b')?.parentId).toBe('a')
  })

  test('a spawn of an id the list already added keeps the listed status', () => {
    let state = Model.initialState(0)
    state = Model.withNode(
      state,
      Model.newNode(
        { id: 'a', parentId: null, source: 'list', type: 'Explore', description: 'listed', status: 'unknown' },
        1,
      ),
    )
    state = Model.onSpawn(state, Fixtures.spawnOf('a', { description: 'spawned' }), 2)
    const node = Model.nodeOf(state, 'a')

    expect(node?.status).toBe('unknown')
    expect(node?.source).toBe('list')
    expect(node?.description).toBe('spawned')
    expect(node?.spawnedAt).toBe(2)
  })

  test('a refused spawn only logs an event', () => {
    const state = Model.onSpawn(
      Model.initialState(0),
      Fixtures.spawnOf('a', { agentId: undefined, deny: 'depth limit' }),
      3,
    )

    expect(state.nodes.size).toBe(1)
    expect(state.events[0]?.text).toContain('depth limit')
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd mods/agent-flow && bun test`
Expected: FAIL, `Cannot find module '../hooks/model'`.

- [ ] **Step 3: Write the model core**

`mods/agent-flow/hooks/model/flow-state.ts`:

```ts
import type { RenderSurface } from 'claude-code'

import Limits from '../limits'
import Names from '../names'

export type NodeStatus =
  | 'running'
  | 'completed'
  | 'failed'
  | 'killed'
  | 'aborted'
  | 'refusal'
  | 'gone'
  | 'unknown'

export type NodeSource = 'root' | 'spawn' | 'list' | 'event'

export type Activity =
  | { kind: 'idle' }
  | { kind: 'tool'; tool: string; since: number; toolUseId?: string }
  | { kind: 'permission'; tool?: string; since: number }

export type RecentTool = {
  tool: string
  startedAt: number
  durationMs?: number
  isError?: boolean
}

export type Usage = { input: number; output: number; cacheRead: number; cacheWrite: number }

export type FlowNode = {
  id: string
  parentId: string | null
  source: NodeSource
  type: string
  description: string
  name?: string
  model?: string
  background?: boolean
  fork?: boolean
  promptExcerpt?: string
  status: NodeStatus
  rawStatus?: string
  spawnedAt?: number
  firstSeenAt: number
  endedAt?: number
  lastEventAt: number
  activity: Activity
  toolCalls: number
  errors: number
  recentTools: readonly RecentTool[]
  turns: number
  usage: Usage
  misses: number
}

export type FlowEvent = { at: number; kind: string; agentId?: string; text: string }

export type PaneState = {
  isBelievedOpen: boolean
  wasDrawnSinceProbe: boolean
  columns: number | null
  placement: 'dock' | 'inline' | null
  expanded: ReadonlySet<string>
  hasAutoOpened: boolean
  closedByPerson: boolean
}

export type FlowState = {
  nodes: ReadonlyMap<string, FlowNode>
  events: readonly FlowEvent[]
  pane: PaneState
  startedSurface: RenderSurface | null
}

export type NodeSeed = Pick<FlowNode, 'id' | 'parentId' | 'source' | 'type' | 'description'> &
  Partial<FlowNode>

export const ZERO_USAGE: Usage = { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }

export const INITIAL_PANE: PaneState = {
  isBelievedOpen: false,
  wasDrawnSinceProbe: false,
  columns: null,
  placement: null,
  expanded: new Set<string>(),
  hasAutoOpened: false,
  closedByPerson: false,
}

const TERMINAL: readonly NodeStatus[] = ['completed', 'failed', 'killed', 'aborted', 'refusal', 'gone']

/**
 * Whether a status is an end state; `unknown` is neither running nor ended.
 *
 * @param status the node's status
 * @returns true for every end state
 */
export function isTerminal(status: NodeStatus): boolean {
  return TERMINAL.includes(status)
}

/**
 * A node from the facts a caller has, the rest defaulted: running, idle, no
 * calls, no usage.
 *
 * @param seed the identifying facts and any known field
 * @param now when it was first seen
 * @returns the node
 */
export function newNode(seed: NodeSeed, now: number): FlowNode {
  return {
    status: 'running',
    firstSeenAt: now,
    lastEventAt: now,
    activity: { kind: 'idle' },
    toolCalls: 0,
    errors: 0,
    recentTools: [],
    turns: 0,
    usage: ZERO_USAGE,
    misses: 0,
    ...seed,
  }
}

/**
 * The state at session start: the root node, idle, and nothing else.
 *
 * @param now the session's start time
 * @returns the state
 */
export function initialState(now: number): FlowState {
  const root = newNode(
    {
      id: Names.ROOT_ID,
      parentId: null,
      source: 'root',
      type: 'main',
      description: 'main conversation',
      status: 'completed',
    },
    now,
  )

  return {
    nodes: new Map([[root.id, root]]),
    events: [],
    pane: INITIAL_PANE,
    startedSurface: null,
  }
}

export function nodeOf(state: FlowState, id: string): FlowNode | undefined {
  return state.nodes.get(id)
}

export function withNode(state: FlowState, node: FlowNode): FlowState {
  const nodes = new Map(state.nodes)

  nodes.set(node.id, node)

  return { ...state, nodes }
}

export function withEvent(state: FlowState, event: FlowEvent): FlowState {
  const events = [...state.events, event].slice(-Limits.EVENT_LOG_SIZE)

  return { ...state, events }
}

export function withPane(state: FlowState, patch: Partial<PaneState>): FlowState {
  return { ...state, pane: { ...state.pane, ...patch } }
}
```

`mods/agent-flow/hooks/model/ensure-node.ts`:

```ts
import type { FlowState } from './flow-state'
import { newNode, nodeOf, withNode } from './flow-state'

/**
 * Makes sure an agent id has a node: an id no spawn and no list named gets
 * an event-sourced placeholder under the "unlisted loops" group.
 *
 * @param state the state
 * @param agentId the loop an event named, or undefined for the main loop
 * @param now when the event arrived
 * @returns the same state when nothing was missing
 */
export function ensureNode(state: FlowState, agentId: string | undefined, now: number): FlowState {
  if (agentId === undefined || nodeOf(state, agentId) !== undefined) {
    return state
  }

  return withNode(
    state,
    newNode(
      {
        id: agentId,
        parentId: null,
        source: 'event',
        type: 'loop',
        description: agentId.slice(0, 8),
      },
      now,
    ),
  )
}
```

`mods/agent-flow/hooks/model/prune.ts`:

```ts
import Limits from '../limits'
import type { FlowNode, FlowState } from './flow-state'
import { isTerminal } from './flow-state'

/**
 * Keeps the tree under MAX_NODES by dropping finished leaves, oldest ended
 * first, one at a time until it fits or no finished leaf remains. Running
 * nodes and nodes with children are never dropped.
 *
 * @param state the state
 * @returns the state, pruned
 */
export function prune(state: FlowState): FlowState {
  if (state.nodes.size <= Limits.MAX_NODES) {
    return state
  }

  const nodes = new Map(state.nodes)

  while (nodes.size > Limits.MAX_NODES) {
    const parents = new Set([...nodes.values()].map(node => node.parentId))
    const leaves = [...nodes.values()].filter(
      (node: FlowNode) => node.source !== 'root' && isTerminal(node.status) && !parents.has(node.id),
    )

    if (leaves.length === 0) {
      break
    }

    leaves.sort((a, b) => (a.endedAt ?? a.firstSeenAt) - (b.endedAt ?? b.firstSeenAt))
    nodes.delete((leaves[0] as FlowNode).id)
  }

  return { ...state, nodes }
}
```

`mods/agent-flow/hooks/model/on-spawn.ts`:

```ts
import Limits from '../limits'
import type { FlowNode, FlowState } from './flow-state'
import { newNode, nodeOf, withEvent, withNode } from './flow-state'
import { prune } from './prune'

/**
 * `agent.spawn` as the reducer takes it: the input's facts plus the result's
 * `agentId` and resolved `model`, or its `deny`.
 */
export type SpawnInput = {
  agentId?: string
  deny?: string
  parentAgentId?: string
  subagentType: string
  description: string
  model?: string
  name?: string
  background: boolean
  fork: boolean
  prompt: string
}

/**
 * Records a spawn: a new running node under its parent, or the spawn's facts
 * merged into a node the list added first (that node keeps the list's status).
 *
 * @param state the state
 * @param spawn the spawn
 * @param now when it happened
 * @returns the state
 */
export function onSpawn(state: FlowState, spawn: SpawnInput, now: number): FlowState {
  if (spawn.agentId === undefined) {
    return withEvent(state, {
      at: now,
      kind: 'agent.spawn',
      text: `refused: ${spawn.deny ?? 'no id'} "${spawn.description}"`,
    })
  }

  const facts = {
    type: spawn.subagentType,
    description: spawn.description,
    model: spawn.model,
    name: spawn.name,
    background: spawn.background,
    fork: spawn.fork,
    promptExcerpt: spawn.prompt.slice(0, Limits.PROMPT_EXCERPT_CHARS),
    spawnedAt: now,
    lastEventAt: now,
  }
  const existing = nodeOf(state, spawn.agentId)
  const node: FlowNode = existing
    ? { ...existing, ...facts, parentId: existing.parentId ?? spawn.parentAgentId ?? null }
    : newNode({ id: spawn.agentId, parentId: spawn.parentAgentId ?? null, source: 'spawn', ...facts }, now)

  return prune(
    withEvent(withNode(state, node), {
      at: now,
      kind: 'agent.spawn',
      agentId: spawn.agentId,
      text: `${spawn.subagentType} "${spawn.description}"`,
    }),
  )
}
```

`mods/agent-flow/hooks/model/index.ts` (grows in later tasks):

```ts
export * from './ensure-node'
export * from './flow-state'
export * from './on-spawn'
export * from './prune'

export * as default from '.'
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd mods/agent-flow && bun test`
Expected: all pass (`names`, `flow-state`, `on-spawn`).

Run: `cd /Users/charles/Desktop/ARRS-claude-code-main && bunx tsc -p mods/tsconfig.json && bunx tsc -p mods/agent-flow/tsconfig.json`
Expected: no output.

- [ ] **Step 5: Commit**

```bash
git add mods/agent-flow
git commit -m "agent-flow: flow state, node helpers, pruning and the spawn reducer

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

### Task 3: Tool-call and turn reducers

**Files:**
- Create: `mods/agent-flow/hooks/model/on-tool.ts`
- Create: `mods/agent-flow/hooks/model/on-turn.ts`
- Modify: `mods/agent-flow/hooks/model/index.ts`
- Test: `mods/agent-flow/tests/on-tool.test.ts`, `mods/agent-flow/tests/on-turn.test.ts`

**Interfaces:**
- Consumes: `initialState`, `newNode`, `nodeOf`, `withNode`, `withEvent`, `ensureNode`, `isTerminal`, `Names.ROOT_ID`, `Limits.RECENT_TOOLS`, `Fixtures.spawnOf`, `onSpawn`.
- Produces:
  - `ToolStart = { agentId?: string; tool: string; toolUseId?: string }`, `ToolEnd = { agentId?: string; tool: string; isError: boolean }`
  - `onToolStart(state, call: ToolStart, now): FlowState`, `onToolEnd(state, call: ToolEnd, now): FlowState`
  - `TurnComplete = { agentId?: string; reason: string; durationMs: number; usage?: TurnTokens }`, `TurnTokens = { input_tokens: number; output_tokens: number; cache_read_input_tokens: number; cache_creation_input_tokens: number }`
  - `onTurnStart(state, now): FlowState`, `onTurnComplete(state, turn: TurnComplete, now): FlowState`, `statusOfReason(reason: string): NodeStatus`

- [ ] **Step 1: Write the failing tests**

`mods/agent-flow/tests/on-tool.test.ts`:

```ts
import { describe, expect, test, tier } from 'claude-code/testing'

import Limits from '../hooks/limits'
import Model from '../hooks/model'
import Names from '../hooks/names'
import Fixtures from './fixtures'

tier('user')

describe('on-tool', () => {
  test('a call marks the agent busy in that tool, and its end records the call', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)
    state = Model.onToolStart(state, { agentId: 'a', tool: 'Bash', toolUseId: 't1' }, 10)

    expect(Model.nodeOf(state, 'a')?.activity).toEqual({ kind: 'tool', tool: 'Bash', since: 10, toolUseId: 't1' })

    state = Model.onToolEnd(state, { agentId: 'a', tool: 'Bash', isError: true }, 25)
    const node = Model.nodeOf(state, 'a')

    expect(node?.activity).toEqual({ kind: 'idle' })
    expect(node?.toolCalls).toBe(1)
    expect(node?.errors).toBe(1)
    expect(node?.recentTools).toEqual([{ tool: 'Bash', startedAt: 10, durationMs: 15, isError: true }])
    expect(node?.lastEventAt).toBe(25)
  })

  test('the main loop\'s calls land on the root node', () => {
    const state = Model.onToolStart(Model.initialState(0), { tool: 'Agent' }, 3)

    expect(Model.nodeOf(state, Names.ROOT_ID)?.activity).toEqual({ kind: 'tool', tool: 'Agent', since: 3 })
    expect(state.nodes.size).toBe(1)
  })

  test('a call from an unknown loop creates an event-sourced node', () => {
    const state = Model.onToolStart(Model.initialState(0), { agentId: 'wf-1234567890', tool: 'Read' }, 3)

    expect(Model.nodeOf(state, 'wf-1234567890')?.source).toBe('event')
    expect(Model.nodeOf(state, 'wf-1234567890')?.activity.kind).toBe('tool')
  })

  test('a call from a finished agent puts it back to running', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)
    state = Model.onTurnComplete(state, { agentId: 'a', reason: 'answer', durationMs: 5 }, 6)
    state = Model.onToolStart(state, { agentId: 'a', tool: 'Grep' }, 9)

    expect(Model.nodeOf(state, 'a')?.status).toBe('running')
    expect(Model.nodeOf(state, 'a')?.endedAt).toBe(undefined)
  })

  test('only the newest RECENT_TOOLS calls are kept', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)

    for (let i = 0; i < Limits.RECENT_TOOLS + 2; i += 1) {
      state = Model.onToolStart(state, { agentId: 'a', tool: `T${i}` }, i * 2)
      state = Model.onToolEnd(state, { agentId: 'a', tool: `T${i}`, isError: false }, i * 2 + 1)
    }

    const node = Model.nodeOf(state, 'a')

    expect(node?.recentTools).toHaveLength(Limits.RECENT_TOOLS)
    expect(node?.recentTools[0]?.tool).toBe('T2')
    expect(node?.toolCalls).toBe(Limits.RECENT_TOOLS + 2)
  })

  test('an end without a start is still counted', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)
    state = Model.onToolEnd(state, { agentId: 'a', tool: 'Bash', isError: false }, 4)

    expect(Model.nodeOf(state, 'a')?.toolCalls).toBe(1)
    expect(Model.nodeOf(state, 'a')?.recentTools[0]?.durationMs).toBe(0)
  })
})
```

`mods/agent-flow/tests/on-turn.test.ts`:

```ts
import { describe, expect, test, tier } from 'claude-code/testing'

import Model from '../hooks/model'
import Names from '../hooks/names'
import Fixtures from './fixtures'

tier('user')

describe('on-turn', () => {
  test('turn.start and turn.complete on the main loop toggle the root between busy and idle', () => {
    let state = Model.onTurnStart(Model.initialState(0), 1)

    expect(Model.nodeOf(state, Names.ROOT_ID)?.status).toBe('running')

    state = Model.onTurnComplete(state, { reason: 'answer', durationMs: 9 }, 10)

    expect(Model.nodeOf(state, Names.ROOT_ID)?.status).toBe('completed')
    expect(Model.nodeOf(state, Names.ROOT_ID)?.turns).toBe(1)
  })

  test('a subagent turn ends its node by reason and sums its usage', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)
    state = Model.onToolStart(state, { agentId: 'a', tool: 'Bash' }, 2)
    state = Model.onTurnComplete(
      state,
      {
        agentId: 'a',
        reason: 'answer',
        durationMs: 40,
        usage: { input_tokens: 10, output_tokens: 5, cache_read_input_tokens: 100, cache_creation_input_tokens: 7 },
      },
      42,
    )
    state = Model.onTurnComplete(
      state,
      {
        agentId: 'a',
        reason: 'answer',
        durationMs: 4,
        usage: { input_tokens: 1, output_tokens: 1, cache_read_input_tokens: 1, cache_creation_input_tokens: 1 },
      },
      50,
    )
    const node = Model.nodeOf(state, 'a')

    expect(node?.status).toBe('completed')
    expect(node?.endedAt).toBe(50)
    expect(node?.activity).toEqual({ kind: 'idle' })
    expect(node?.turns).toBe(2)
    expect(node?.usage).toEqual({ input: 11, output: 6, cacheRead: 101, cacheWrite: 8 })
  })

  test('reasons map to statuses', () => {
    expect(Model.statusOfReason('answer')).toBe('completed')
    expect(Model.statusOfReason('error')).toBe('failed')
    expect(Model.statusOfReason('aborted')).toBe('aborted')
    expect(Model.statusOfReason('refusal')).toBe('refusal')
    expect(Model.statusOfReason('something')).toBe('unknown')
  })

  test('a turn of an unknown loop creates an event-sourced node that is then finished', () => {
    const state = Model.onTurnComplete(Model.initialState(0), { agentId: 'wf-9', reason: 'answer', durationMs: 1 }, 5)

    expect(Model.nodeOf(state, 'wf-9')?.source).toBe('event')
    expect(Model.nodeOf(state, 'wf-9')?.status).toBe('completed')
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd mods/agent-flow && bun test`
Expected: FAIL, `Model.onToolStart is not a function` (and the turn equivalents).

- [ ] **Step 3: Write the reducers**

`mods/agent-flow/hooks/model/on-tool.ts`:

```ts
import Limits from '../limits'
import Names from '../names'
import { ensureNode } from './ensure-node'
import type { FlowState, RecentTool } from './flow-state'
import { isTerminal, nodeOf, withEvent, withNode } from './flow-state'

export type ToolStart = { agentId?: string; tool: string; toolUseId?: string }
export type ToolEnd = { agentId?: string; tool: string; isError: boolean }

/**
 * A tool call begins: the loop it runs in is busy in that tool from now. A
 * finished subagent that calls a tool is running again (SendMessage woke it).
 *
 * @param state the state
 * @param call the call, `agentId` absent for the main loop
 * @param now when it began
 * @returns the state
 */
export function onToolStart(state: FlowState, call: ToolStart, now: number): FlowState {
  const id = call.agentId ?? Names.ROOT_ID
  const ensured = ensureNode(state, call.agentId, now)
  const node = nodeOf(ensured, id)

  if (node === undefined) {
    return ensured
  }

  const isWoken = id !== Names.ROOT_ID && isTerminal(node.status)

  return withEvent(
    withNode(ensured, {
      ...node,
      activity: { kind: 'tool', tool: call.tool, since: now, toolUseId: call.toolUseId },
      status: isWoken ? 'running' : node.status,
      endedAt: isWoken ? undefined : node.endedAt,
      lastEventAt: now,
    }),
    { at: now, kind: 'tool.call', agentId: id, text: `${call.tool} started` },
  )
}

/**
 * A tool call ends: the loop is idle again, the call is counted and kept
 * among the recent ones with its duration and outcome.
 *
 * @param state the state
 * @param call the call, `agentId` absent for the main loop
 * @param now when it ended
 * @returns the state
 */
export function onToolEnd(state: FlowState, call: ToolEnd, now: number): FlowState {
  const id = call.agentId ?? Names.ROOT_ID
  const node = nodeOf(state, id)

  if (node === undefined) {
    return state
  }

  const startedAt = node.activity.kind === 'tool' ? node.activity.since : now
  const recent: RecentTool = { tool: call.tool, startedAt, durationMs: now - startedAt, isError: call.isError }

  return withEvent(
    withNode(state, {
      ...node,
      activity: { kind: 'idle' },
      toolCalls: node.toolCalls + 1,
      errors: node.errors + (call.isError ? 1 : 0),
      recentTools: [...node.recentTools, recent].slice(-Limits.RECENT_TOOLS),
      lastEventAt: now,
    }),
    { at: now, kind: 'tool.call', agentId: id, text: `${call.tool} ${call.isError ? 'failed' : 'done'} ${now - startedAt}ms` },
  )
}
```

`mods/agent-flow/hooks/model/on-turn.ts`:

```ts
import Names from '../names'
import { ensureNode } from './ensure-node'
import type { FlowState, NodeStatus } from './flow-state'
import { nodeOf, withEvent, withNode } from './flow-state'

export type TurnTokens = {
  input_tokens: number
  output_tokens: number
  cache_read_input_tokens: number
  cache_creation_input_tokens: number
}

export type TurnComplete = { agentId?: string; reason: string; durationMs: number; usage?: TurnTokens }

/**
 * Maps `turn.complete`'s reason to a node status.
 *
 * @param reason `answer`, `error`, `aborted` or `refusal`
 * @returns the status; `unknown` for anything else
 */
export function statusOfReason(reason: string): NodeStatus {
  switch (reason) {
    case 'answer':
      return 'completed'
    case 'error':
      return 'failed'
    case 'aborted':
      return 'aborted'
    case 'refusal':
      return 'refusal'
    default:
      return 'unknown'
  }
}

/**
 * The main loop starts a turn: the root is busy.
 *
 * @param state the state
 * @param now when the turn started
 * @returns the state
 */
export function onTurnStart(state: FlowState, now: number): FlowState {
  const root = nodeOf(state, Names.ROOT_ID)

  if (root === undefined) {
    return state
  }

  return withNode(state, { ...root, status: 'running', lastEventAt: now })
}

/**
 * A turn ends: a subagent's node takes the reason's status and its end time;
 * the root goes idle. Either sums the turn's tokens and counts the turn.
 *
 * @param state the state
 * @param turn the turn, `agentId` absent for the main loop
 * @param now when it ended
 * @returns the state
 */
export function onTurnComplete(state: FlowState, turn: TurnComplete, now: number): FlowState {
  const id = turn.agentId ?? Names.ROOT_ID
  const ensured = ensureNode(state, turn.agentId, now)
  const node = nodeOf(ensured, id)

  if (node === undefined) {
    return ensured
  }

  const isRoot = id === Names.ROOT_ID
  const status: NodeStatus = isRoot ? 'completed' : statusOfReason(turn.reason)
  const usage = turn.usage
    ? {
        input: node.usage.input + turn.usage.input_tokens,
        output: node.usage.output + turn.usage.output_tokens,
        cacheRead: node.usage.cacheRead + turn.usage.cache_read_input_tokens,
        cacheWrite: node.usage.cacheWrite + turn.usage.cache_creation_input_tokens,
      }
    : node.usage

  return withEvent(
    withNode(ensured, {
      ...node,
      status,
      endedAt: isRoot ? node.endedAt : now,
      turns: node.turns + 1,
      usage,
      activity: { kind: 'idle' },
      lastEventAt: now,
    }),
    { at: now, kind: 'turn.complete', agentId: id, text: `${turn.reason} ${turn.durationMs}ms` },
  )
}
```

Add to `mods/agent-flow/hooks/model/index.ts` (keep alphabetical):

```ts
export * from './on-tool'
export * from './on-turn'
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd mods/agent-flow && bun test`
Expected: all pass.

Run: `cd /Users/charles/Desktop/ARRS-claude-code-main && bunx tsc -p mods/tsconfig.json && bunx tsc -p mods/agent-flow/tsconfig.json`
Expected: no output.

- [ ] **Step 5: Commit**

```bash
git add mods/agent-flow
git commit -m "agent-flow: tool-call and turn reducers

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

### Task 4: Permission reducer and agent.list reconciliation

**Files:**
- Create: `mods/agent-flow/hooks/model/on-permission.ts`
- Create: `mods/agent-flow/hooks/model/reconcile.ts`
- Modify: `mods/agent-flow/hooks/model/index.ts`
- Test: `mods/agent-flow/tests/on-permission.test.ts`, `mods/agent-flow/tests/reconcile.test.ts`

**Interfaces:**
- Consumes: everything Task 2 and 3 produce, `Limits.GONE_AFTER_MISSES`.
- Produces:
  - `onPermission(state, req: { agentId?: string; tool?: string }, now): FlowState`
  - `Listed = { id: string; description: string; type: string; status: string; parentId?: string; name?: string }` (structurally what `AgentInfo` gives)
  - `statusOfListed(raw: string): NodeStatus`, `reconcile(state, listed: readonly Listed[], now): FlowState`

- [ ] **Step 1: Write the failing tests**

`mods/agent-flow/tests/on-permission.test.ts`:

```ts
import { describe, expect, test, tier } from 'claude-code/testing'

import Model from '../hooks/model'
import Names from '../hooks/names'
import Fixtures from './fixtures'

tier('user')

describe('on-permission', () => {
  test('a permission request marks the agent as waiting, naming the tool', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)
    state = Model.onPermission(state, { agentId: 'a', tool: 'Bash' }, 5)

    expect(Model.nodeOf(state, 'a')?.activity).toEqual({ kind: 'permission', tool: 'Bash', since: 5 })
    expect(state.events.at(-1)?.text).toBe('waiting for approval: Bash')
  })

  test('without an agent id the root waits', () => {
    const state = Model.onPermission(Model.initialState(0), {}, 5)

    expect(Model.nodeOf(state, Names.ROOT_ID)?.activity.kind).toBe('permission')
  })

  test('the next tool event clears the wait', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)
    state = Model.onPermission(state, { agentId: 'a', tool: 'Bash' }, 5)
    state = Model.onToolEnd(state, { agentId: 'a', tool: 'Bash', isError: false }, 9)

    expect(Model.nodeOf(state, 'a')?.activity).toEqual({ kind: 'idle' })
  })
})
```

`mods/agent-flow/tests/reconcile.test.ts`:

```ts
import { describe, expect, test, tier } from 'claude-code/testing'

import Limits from '../hooks/limits'
import Model from '../hooks/model'
import Fixtures from './fixtures'

tier('user')

const listed = (id: string, status = 'running', parentId?: string) => ({
  id,
  description: `listed ${id}`,
  type: 'general-purpose',
  status,
  parentId,
})

describe('reconcile', () => {
  test('agents the list names and the tree lacks are added as list-sourced', () => {
    const state = Model.reconcile(Model.initialState(0), [listed('a'), listed('b', 'completed', 'a')], 4)

    expect(Model.nodeOf(state, 'a')?.source).toBe('list')
    expect(Model.nodeOf(state, 'a')?.status).toBe('running')
    expect(Model.nodeOf(state, 'b')?.parentId).toBe('a')
    expect(Model.nodeOf(state, 'b')?.status).toBe('completed')
    expect(Model.nodeOf(state, 'b')?.endedAt).toBe(4)
  })

  test('the list has the last word on status and fills a missing parent', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)
    state = Model.onTurnComplete(state, { agentId: 'a', reason: 'answer', durationMs: 1 }, 2)
    state = Model.reconcile(state, [listed('a', 'running')], 3)

    expect(Model.nodeOf(state, 'a')?.status).toBe('running')
    expect(Model.nodeOf(state, 'a')?.source).toBe('spawn')
    expect(Model.nodeOf(state, 'a')?.description).toBe('task a')
  })

  test('an unmapped status is kept raw and shown as unknown', () => {
    const state = Model.reconcile(Model.initialState(0), [listed('a', 'parked')], 1)

    expect(Model.nodeOf(state, 'a')?.status).toBe('unknown')
    expect(Model.nodeOf(state, 'a')?.rawStatus).toBe('parked')
  })

  test('a running node missing from the list is gone after GONE_AFTER_MISSES reconciles', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)

    for (let i = 0; i < Limits.GONE_AFTER_MISSES - 1; i += 1) {
      state = Model.reconcile(state, [], 10 + i)
      expect(Model.nodeOf(state, 'a')?.status).toBe('running')
    }

    state = Model.reconcile(state, [], 20)

    expect(Model.nodeOf(state, 'a')?.status).toBe('gone')
    expect(Model.nodeOf(state, 'a')?.endedAt).toBe(20)
    expect(state.events.at(-1)?.kind).toBe('gone')
  })

  test('a miss streak resets when the list names the node again', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)
    state = Model.reconcile(state, [], 2)
    state = Model.reconcile(state, [listed('a')], 3)
    state = Model.reconcile(state, [], 4)

    expect(Model.nodeOf(state, 'a')?.status).toBe('running')
    expect(Model.nodeOf(state, 'a')?.misses).toBe(1)
  })

  test('event-sourced and root nodes never go missing, and a listed event node is upgraded', () => {
    let state = Model.onToolStart(Model.initialState(0), { agentId: 'wf-1', tool: 'Read' }, 1)
    state = Model.reconcile(state, [], 2)
    state = Model.reconcile(state, [], 3)

    expect(Model.nodeOf(state, 'wf-1')?.status).toBe('running')

    state = Model.reconcile(state, [listed('wf-1')], 4)

    expect(Model.nodeOf(state, 'wf-1')?.source).toBe('list')
    expect(Model.nodeOf(state, 'wf-1')?.type).toBe('general-purpose')
    expect(Model.nodeOf(state, 'wf-1')?.description).toBe('listed wf-1')
  })

  test('statuses map from the engine\'s words', () => {
    expect(Model.statusOfListed('running')).toBe('running')
    expect(Model.statusOfListed('pending')).toBe('running')
    expect(Model.statusOfListed('completed')).toBe('completed')
    expect(Model.statusOfListed('failed')).toBe('failed')
    expect(Model.statusOfListed('killed')).toBe('killed')
    expect(Model.statusOfListed('cancelled')).toBe('killed')
    expect(Model.statusOfListed('weird')).toBe('unknown')
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd mods/agent-flow && bun test`
Expected: FAIL, `Model.onPermission is not a function`, `Model.reconcile is not a function`.

- [ ] **Step 3: Write the reducers**

`mods/agent-flow/hooks/model/on-permission.ts`:

```ts
import Names from '../names'
import { ensureNode } from './ensure-node'
import type { FlowState } from './flow-state'
import { nodeOf, withEvent, withNode } from './flow-state'

/**
 * A permission request: the loop it came from waits for the person until its
 * next tool event or turn end clears it.
 *
 * @param state the state
 * @param req the request, `agentId` absent for the main loop
 * @param now when it was raised
 * @returns the state
 */
export function onPermission(
  state: FlowState,
  req: { agentId?: string; tool?: string },
  now: number,
): FlowState {
  const id = req.agentId ?? Names.ROOT_ID
  const ensured = ensureNode(state, req.agentId, now)
  const node = nodeOf(ensured, id)

  if (node === undefined) {
    return ensured
  }

  return withEvent(
    withNode(ensured, {
      ...node,
      activity: { kind: 'permission', tool: req.tool, since: now },
      lastEventAt: now,
    }),
    {
      at: now,
      kind: 'permission',
      agentId: id,
      text: `waiting for approval${req.tool ? `: ${req.tool}` : ''}`,
    },
  )
}
```

`mods/agent-flow/hooks/model/reconcile.ts`:

```ts
import Limits from '../limits'
import Names from '../names'
import type { FlowNode, FlowState, NodeStatus } from './flow-state'
import { isTerminal, newNode, withEvent } from './flow-state'
import { prune } from './prune'

/**
 * One agent as `$.agent.list()` describes it (structurally an AgentInfo).
 */
export type Listed = {
  id: string
  description: string
  type: string
  status: string
  parentId?: string
  name?: string
}

/**
 * Maps the engine's task status words onto node statuses.
 *
 * @param raw the list's status string
 * @returns the status; `unknown` for a word this mod does not know
 */
export function statusOfListed(raw: string): NodeStatus {
  switch (raw) {
    case 'running':
    case 'pending':
    case 'in_progress':
      return 'running'
    case 'completed':
    case 'done':
    case 'success':
      return 'completed'
    case 'failed':
    case 'error':
      return 'failed'
    case 'killed':
    case 'cancelled':
    case 'stopped':
      return 'killed'
    default:
      return 'unknown'
  }
}

/**
 * Merges what the engine lists into the tree: the list has the last word on
 * status and fills a parent the events never gave; a running spawn- or
 * list-sourced node absent from GONE_AFTER_MISSES lists in a row is gone.
 * Root and event-sourced nodes are never judged absent.
 *
 * @param state the state
 * @param listed the agents `$.agent.list()` answered
 * @param now when the list was read
 * @returns the state
 */
export function reconcile(state: FlowState, listed: readonly Listed[], now: number): FlowState {
  const nodes = new Map(state.nodes)
  const seen = new Set<string>()
  let next: FlowState = { ...state, nodes }

  for (const info of listed) {
    seen.add(info.id)

    const mapped = statusOfListed(info.status)
    const rawStatus = mapped === 'unknown' ? info.status : undefined
    const existing = nodes.get(info.id)

    if (existing === undefined) {
      nodes.set(
        info.id,
        newNode(
          {
            id: info.id,
            parentId: info.parentId ?? null,
            source: 'list',
            type: info.type,
            description: info.description,
            name: info.name,
            status: mapped,
            rawStatus,
            endedAt: isTerminal(mapped) ? now : undefined,
          },
          now,
        ),
      )
      continue
    }

    const isPlaceholder = existing.source === 'event'
    const merged: FlowNode = {
      ...existing,
      source: isPlaceholder ? 'list' : existing.source,
      type: isPlaceholder ? info.type : existing.type,
      description: isPlaceholder ? info.description : existing.description,
      name: existing.name ?? info.name,
      parentId: existing.parentId ?? info.parentId ?? null,
      status: mapped,
      rawStatus,
      endedAt: isTerminal(mapped) ? (existing.endedAt ?? now) : undefined,
      lastEventAt: mapped === existing.status ? existing.lastEventAt : now,
      misses: 0,
    }

    nodes.set(info.id, merged)
  }

  for (const node of [...nodes.values()]) {
    const isJudged =
      node.id !== Names.ROOT_ID && node.source !== 'event' && !seen.has(node.id) && node.status === 'running'

    if (!isJudged) {
      continue
    }

    const misses = node.misses + 1

    if (misses < Limits.GONE_AFTER_MISSES) {
      nodes.set(node.id, { ...node, misses })
      continue
    }

    nodes.set(node.id, {
      ...node,
      misses,
      status: 'gone',
      endedAt: now,
      activity: { kind: 'idle' },
      lastEventAt: now,
    })
    next = withEvent(next, { at: now, kind: 'gone', agentId: node.id, text: `${node.type} left the list` })
  }

  return prune({ ...next, nodes })
}
```

Add to `mods/agent-flow/hooks/model/index.ts` (alphabetical):

```ts
export * from './on-permission'
export * from './reconcile'
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd mods/agent-flow && bun test`
Expected: all pass.

Run: `cd /Users/charles/Desktop/ARRS-claude-code-main && bunx tsc -p mods/tsconfig.json && bunx tsc -p mods/agent-flow/tsconfig.json`
Expected: no output.

- [ ] **Step 5: Commit**

```bash
git add mods/agent-flow
git commit -m "agent-flow: permission reducer and agent.list reconciliation

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

### Task 5: Signals, elapsed time, counts and token text

**Files:**
- Create: `mods/agent-flow/hooks/model/signal-of.ts`
- Create: `mods/agent-flow/hooks/model/elapsed-of.ts`
- Create: `mods/agent-flow/hooks/model/counts-of.ts`
- Create: `mods/agent-flow/hooks/model/tokens-text-of.ts`
- Modify: `mods/agent-flow/hooks/model/index.ts`
- Test: `mods/agent-flow/tests/signal-of.test.ts`, `mods/agent-flow/tests/elapsed-of.test.ts`, `mods/agent-flow/tests/counts-of.test.ts`

**Interfaces:**
- Consumes: `FlowNode`, `FlowState`, `Usage`, `Limits.SLOW_TOOL_MS`, `Limits.QUIET_MS`, `Names.ROOT_ID`.
- Produces:
  - `Signal = 'waiting' | 'slow' | 'quiet' | 'none'`, `signalOf(node: FlowNode, now: number): Signal`
  - `durationTextOf(ms: number): string` (`12s`, `1m05s`), `elapsedOf(node: FlowNode, now: number): string` (`~` prefix when the node has no `spawnedAt`)
  - `Counts = { agents: number; running: number; waiting: number; unlisted: number }`, `countsOf(state: FlowState): Counts`
  - `tokensTextOf(usage: Usage): string` (`820 tok`, `4.1k tok`, `''` at zero)

- [ ] **Step 1: Write the failing tests**

`mods/agent-flow/tests/signal-of.test.ts`:

```ts
import { describe, expect, test, tier } from 'claude-code/testing'

import Limits from '../hooks/limits'
import Model from '../hooks/model'
import Fixtures from './fixtures'

tier('user')

const nodeAfter = (state: Model.FlowState) => Model.nodeOf(state, 'a') as Model.FlowNode

describe('signal-of', () => {
  test('a permission wait is the waiting signal', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)
    state = Model.onPermission(state, { agentId: 'a', tool: 'Bash' }, 2)

    expect(Model.signalOf(nodeAfter(state), 3)).toBe('waiting')
  })

  test('a tool call longer than SLOW_TOOL_MS is slow, a shorter one is nothing', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)
    state = Model.onToolStart(state, { agentId: 'a', tool: 'Bash' }, 10)

    expect(Model.signalOf(nodeAfter(state), 10 + Limits.SLOW_TOOL_MS)).toBe('none')
    expect(Model.signalOf(nodeAfter(state), 11 + Limits.SLOW_TOOL_MS)).toBe('slow')
  })

  test('a running idle agent with no event for QUIET_MS is quiet; a finished one is not', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)

    expect(Model.signalOf(nodeAfter(state), 2 + Limits.QUIET_MS)).toBe('quiet')

    state = Model.onTurnComplete(state, { agentId: 'a', reason: 'answer', durationMs: 1 }, 2)

    expect(Model.signalOf(nodeAfter(state), 3 + Limits.QUIET_MS)).toBe('none')
  })
})
```

`mods/agent-flow/tests/elapsed-of.test.ts`:

```ts
import { describe, expect, test, tier } from 'claude-code/testing'

import Model from '../hooks/model'
import Fixtures from './fixtures'

tier('user')

describe('elapsed-of', () => {
  test('durations read as seconds below a minute and m/ss above', () => {
    expect(Model.durationTextOf(0)).toBe('0s')
    expect(Model.durationTextOf(12400)).toBe('12s')
    expect(Model.durationTextOf(65000)).toBe('1m05s')
    expect(Model.durationTextOf(3600000)).toBe('60m00s')
  })

  test('a spawned node counts from its spawn to its end or now', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1000)
    const running = Model.nodeOf(state, 'a') as Model.FlowNode

    expect(Model.elapsedOf(running, 13000)).toBe('12s')

    state = Model.onTurnComplete(state, { agentId: 'a', reason: 'answer', durationMs: 1 }, 5000)

    expect(Model.elapsedOf(Model.nodeOf(state, 'a') as Model.FlowNode, 99000)).toBe('4s')
  })

  test('a node the list added counts from first sight with a ~ prefix', () => {
    const state = Model.reconcile(
      Model.initialState(0),
      [{ id: 'a', description: 'd', type: 't', status: 'running' }],
      2000,
    )

    expect(Model.elapsedOf(Model.nodeOf(state, 'a') as Model.FlowNode, 9000)).toBe('~7s')
  })
})
```

`mods/agent-flow/tests/counts-of.test.ts`:

```ts
import { describe, expect, test, tier } from 'claude-code/testing'

import Model from '../hooks/model'
import Fixtures from './fixtures'

tier('user')

describe('counts-of', () => {
  test('counts agents, running, waiting and unlisted, never the root', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)
    state = Model.onSpawn(state, Fixtures.spawnOf('b'), 2)
    state = Model.onTurnComplete(state, { agentId: 'b', reason: 'answer', durationMs: 1 }, 3)
    state = Model.onPermission(state, { agentId: 'a', tool: 'Bash' }, 4)
    state = Model.onToolStart(state, { agentId: 'wf-1', tool: 'Read' }, 5)

    expect(Model.countsOf(state)).toEqual({ agents: 3, running: 2, waiting: 1, unlisted: 1 })
  })

  test('a root waiting for approval counts as waiting', () => {
    const state = Model.onPermission(Model.initialState(0), { tool: 'Edit' }, 1)

    expect(Model.countsOf(state)).toEqual({ agents: 0, running: 0, waiting: 1, unlisted: 0 })
  })

  test('token totals read compactly', () => {
    expect(Model.tokensTextOf({ input: 0, output: 0, cacheRead: 0, cacheWrite: 0 })).toBe('')
    expect(Model.tokensTextOf({ input: 800, output: 20, cacheRead: 9, cacheWrite: 9 })).toBe('820 tok')
    expect(Model.tokensTextOf({ input: 4000, output: 120, cacheRead: 0, cacheWrite: 0 })).toBe('4.1k tok')
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `cd mods/agent-flow && bun test`
Expected: FAIL, `Model.signalOf is not a function` and the others.

- [ ] **Step 3: Write the selectors**

`mods/agent-flow/hooks/model/signal-of.ts`:

```ts
import Limits from '../limits'
import type { FlowNode } from './flow-state'

export type Signal = 'waiting' | 'slow' | 'quiet' | 'none'

/**
 * What deserves attention about a node right now: it waits for the person,
 * its tool call runs long, or it runs but has gone silent.
 *
 * @param node the node
 * @param now the time
 * @returns the strongest signal, `none` when all is well
 */
export function signalOf(node: FlowNode, now: number): Signal {
  if (node.activity.kind === 'permission') {
    return 'waiting'
  }

  if (node.activity.kind === 'tool' && now - node.activity.since > Limits.SLOW_TOOL_MS) {
    return 'slow'
  }

  if (node.status === 'running' && node.activity.kind === 'idle' && now - node.lastEventAt > Limits.QUIET_MS) {
    return 'quiet'
  }

  return 'none'
}
```

`mods/agent-flow/hooks/model/elapsed-of.ts`:

```ts
import type { FlowNode } from './flow-state'

/**
 * A duration as the pane prints it: whole seconds below a minute, `m` and
 * two-digit seconds above.
 *
 * @param ms the duration in milliseconds
 * @returns the text
 */
export function durationTextOf(ms: number): string {
  const seconds = Math.max(0, Math.round(ms / 1000))

  if (seconds < 60) {
    return `${seconds}s`
  }

  return `${Math.floor(seconds / 60)}m${String(seconds % 60).padStart(2, '0')}s`
}

/**
 * How long a node has run: from its spawn (or its first sight, marked `~` as
 * a lower bound) to its end or now.
 *
 * @param node the node
 * @param now the time
 * @returns the text
 */
export function elapsedOf(node: FlowNode, now: number): string {
  const from = node.spawnedAt ?? node.firstSeenAt
  const text = durationTextOf((node.endedAt ?? now) - from)

  return node.spawnedAt === undefined ? `~${text}` : text
}
```

`mods/agent-flow/hooks/model/counts-of.ts`:

```ts
import Names from '../names'
import type { FlowState } from './flow-state'

export type Counts = { agents: number; running: number; waiting: number; unlisted: number }

/**
 * The header's numbers: every node but the root is an agent; running and
 * unlisted likewise exclude the root; waiting includes it, since the main
 * loop's permission prompt is the person's to answer too.
 *
 * @param state the state
 * @returns the counts
 */
export function countsOf(state: FlowState): Counts {
  const counts: Counts = { agents: 0, running: 0, waiting: 0, unlisted: 0 }

  for (const node of state.nodes.values()) {
    if (node.activity.kind === 'permission') {
      counts.waiting += 1
    }

    if (node.id === Names.ROOT_ID) {
      continue
    }

    counts.agents += 1
    counts.running += node.status === 'running' ? 1 : 0
    counts.unlisted += node.source === 'event' ? 1 : 0
  }

  return counts
}
```

`mods/agent-flow/hooks/model/tokens-text-of.ts`:

```ts
import type { Usage } from './flow-state'

/**
 * Input plus output tokens, compact: `820 tok`, `4.1k tok`; nothing at zero.
 *
 * @param usage the node's summed usage
 * @returns the text
 */
export function tokensTextOf(usage: Usage): string {
  const total = usage.input + usage.output

  if (total === 0) {
    return ''
  }

  return total < 1000 ? `${total} tok` : `${(total / 1000).toFixed(1)}k tok`
}
```

Add to `mods/agent-flow/hooks/model/index.ts` (alphabetical):

```ts
export * from './counts-of'
export * from './elapsed-of'
export * from './signal-of'
export * from './tokens-text-of'
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd mods/agent-flow && bun test`
Expected: all pass.

Run: `cd /Users/charles/Desktop/ARRS-claude-code-main && bunx tsc -p mods/tsconfig.json && bunx tsc -p mods/agent-flow/tsconfig.json`
Expected: no output.

- [ ] **Step 5: Commit**

```bash
git add mods/agent-flow
git commit -m "agent-flow: signals, elapsed time, counts and token text

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

### Task 6: Rows: the state as lines, full and inline

**Files:**
- Create: `mods/agent-flow/hooks/model/rows-of.ts`
- Modify: `mods/agent-flow/hooks/model/index.ts`
- Test: `mods/agent-flow/tests/rows-of.test.ts`

**Interfaces:**
- Consumes: everything Tasks 2 to 5 produce, `Limits.DESCRIPTION_MAX_CHARS`, `Limits.INLINE_MAX_ROWS`, `Names.UNLISTED_KEY`, `Names.UNLISTED_GROUP_LABEL`, `Names.NO_AGENTS_TEXT`, `Names.WIDEN_TEXT`.
- Produces:
  - `RowKind = 'header' | 'root' | 'node' | 'group' | 'detail' | 'event' | 'hint' | 'empty'`
  - `Row = { kind: RowKind; text: string; id?: string; color?: string; dim?: boolean; bold?: boolean; isExpanded?: boolean }` (`id` set on `node` and `group` rows, which are the expandable ones)
  - `headerTextOf(state): string`, `nodeTextOf(node, now, prefix): string`, `detailsOf(node): string[]`
  - `rowsOf(state, now, expanded: ReadonlySet<string>): Row[]`, `inlineRowsOf(state, now): Row[]`

- [ ] **Step 1: Write the failing test**

`mods/agent-flow/tests/rows-of.test.ts`:

```ts
import { describe, expect, test, tier } from 'claude-code/testing'

import Limits from '../hooks/limits'
import Model from '../hooks/model'
import Names from '../hooks/names'
import Fixtures from './fixtures'

tier('user')

const NONE: ReadonlySet<string> = new Set()

function nested(): Model.FlowState {
  let state = Model.onTurnStart(Model.initialState(0), 0)
  state = Model.onToolStart(state, { tool: 'Agent' }, 500)
  state = Model.onSpawn(state, Fixtures.spawnOf('a', { subagentType: 'general-purpose', description: 'Refactor auth' }), 1000)
  state = Model.onSpawn(state, Fixtures.spawnOf('b', { parentAgentId: 'a', description: 'Find callers' }), 2000)
  state = Model.onToolStart(state, { agentId: 'b', tool: 'Grep' }, 2100)
  state = Model.onToolEnd(state, { agentId: 'b', tool: 'Grep', isError: false }, 2600)
  state = Model.onTurnComplete(
    state,
    { agentId: 'b', reason: 'answer', durationMs: 8000, usage: { input_tokens: 4000, output_tokens: 100, cache_read_input_tokens: 0, cache_creation_input_tokens: 0 } },
    10000,
  )
  state = Model.onSpawn(state, Fixtures.spawnOf('c', { description: 'Scan tests' }), 3000)
  state = Model.onPermission(state, { agentId: 'c', tool: 'Bash' }, 4000)

  return state
}

describe('rows-of', () => {
  test('the header counts and the root row say what main is doing', () => {
    const rows = Model.rowsOf(nested(), 12000, NONE)

    expect(rows[0]).toEqual({ kind: 'header', text: 'Agent flow · 3 agents · 2 running · 1 waiting', bold: true })
    expect(rows[1]?.kind).toBe('root')
    expect(rows[1]?.text).toBe('main · busy · Agent 12s')
  })

  test('the tree nests children under parents with box prefixes, in spawn order', () => {
    const texts = Model.rowsOf(nested(), 12000, NONE)
      .filter(row => row.kind === 'node')
      .map(row => row.text)

    expect(texts).toEqual([
      '├─ ● general-purpose "Refactor auth" 11s',
      '│  └─ ✓ Explore "Find callers" 8s ×1 · 4.1k tok',
      '└─ ◐ Explore "Scan tests" 9s · waiting for approval: Bash',
    ])
  })

  test('node rows carry their id, color and expandability; a waiting row is magenta and bold', () => {
    const rows = Model.rowsOf(nested(), 12000, NONE).filter(row => row.kind === 'node')

    expect(rows[0]).toMatchObject({ id: 'a', color: 'yellow', isExpanded: false })
    expect(rows[1]).toMatchObject({ id: 'b', color: 'green', dim: true })
    expect(rows[2]).toMatchObject({ id: 'c', color: 'magenta', bold: true })
  })

  test('an expanded node shows its details indented under it', () => {
    const rows = Model.rowsOf(nested(), 12000, new Set(['b']))
    const at = rows.findIndex(row => row.id === 'b')

    expect(rows[at]?.isExpanded).toBe(true)
    expect(rows[at + 1]).toMatchObject({ kind: 'detail', dim: true })
    expect(rows[at + 1]?.text).toBe('│     model ? · foreground')
    expect(rows[at + 2]?.text).toBe('│     prompt Do task b')
    expect(rows[at + 3]?.text).toBe('│     Grep 1s')
    expect(rows[at + 4]?.text).toBe('│     tokens in 4000 out 100 cache 0/0')
    expect(rows[at + 5]?.text).toBe('│     id b')
  })

  test('a slow tool call is marked, a failed node is red with its status', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 0)
    state = Model.onToolStart(state, { agentId: 'a', tool: 'Bash' }, 1000)
    const slow = Model.rowsOf(state, 2000 + Limits.SLOW_TOOL_MS, NONE).find(row => row.id === 'a')

    expect(slow?.text).toBe('└─ ● Explore "task a" 32s · Bash 31s !')
    expect(slow?.color).toBe('yellow')

    state = Model.onTurnComplete(state, { agentId: 'a', reason: 'error', durationMs: 1 }, 5000)
    const failed = Model.rowsOf(state, 9000, NONE).find(row => row.id === 'a')

    expect(failed?.text).toBe('└─ ✗ Explore "task a" 5s ×0 [failed]')
    expect(failed?.color).toBe('red')
  })

  test('with no agents the tree is one dim empty row', () => {
    const rows = Model.rowsOf(Model.initialState(0), 1, NONE)

    expect(rows.map(row => row.kind)).toEqual(['header', 'root', 'empty'])
    expect(rows[2]).toEqual({ kind: 'empty', text: `   ${Names.NO_AGENTS_TEXT}`, dim: true })
  })

  test('unlisted loops sit in a collapsed group that expands by its key', () => {
    let state = Model.onToolStart(Model.initialState(0), { agentId: 'wf-12345678', tool: 'Read' }, 1)
    const collapsed = Model.rowsOf(state, 2, NONE)
    const group = collapsed.find(row => row.kind === 'group')

    expect(group).toMatchObject({ id: Names.UNLISTED_KEY, text: `▸ ${Names.UNLISTED_GROUP_LABEL} (1)`, isExpanded: false })
    expect(collapsed.some(row => row.id === 'wf-12345678')).toBe(false)

    const expanded = Model.rowsOf(state, 2, new Set([Names.UNLISTED_KEY]))

    expect(expanded.find(row => row.kind === 'group')?.text).toBe(`▾ ${Names.UNLISTED_GROUP_LABEL} (1)`)
    expect(expanded.find(row => row.id === 'wf-12345678')?.text).toBe('   └─ ● loop "wf-12345" ~0s · Read 0s')
    void state
  })

  test('the last event closes the list, dim', () => {
    const rows = Model.rowsOf(nested(), 12000, NONE)

    expect(rows.at(-1)).toEqual({ kind: 'event', text: 'last: permission c waiting for approval: Bash', dim: true })
  })

  test('inline rows keep the header, the rows that need attention and a hint, at most INLINE_MAX_ROWS', () => {
    const rows = Model.inlineRowsOf(nested(), 12000)

    expect(rows[0]?.kind).toBe('header')
    expect(rows[1]?.text).toBe('◐ Explore "Scan tests" 9s · waiting for approval: Bash')
    expect(rows.at(-1)).toEqual({ kind: 'hint', text: Names.WIDEN_TEXT, dim: true })
    expect(rows.length).toBeLessThanOrEqual(Limits.INLINE_MAX_ROWS + 2)
  })

  test('inline rows fall back to running agents when nothing needs attention', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)
    state = Model.onSpawn(state, Fixtures.spawnOf('b'), 2)
    const rows = Model.inlineRowsOf(state, 3000)

    expect(rows.filter(row => row.kind === 'node').map(row => row.id)).toEqual(['a', 'b'])
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd mods/agent-flow && bun test tests/rows-of.test.ts`
Expected: FAIL, `Model.rowsOf is not a function`.

- [ ] **Step 3: Write the row selectors**

`mods/agent-flow/hooks/model/rows-of.ts`:

```ts
import Limits from '../limits'
import Names from '../names'
import { countsOf } from './counts-of'
import { durationTextOf, elapsedOf } from './elapsed-of'
import type { FlowNode, FlowState } from './flow-state'
import { isTerminal, nodeOf } from './flow-state'
import type { Signal } from './signal-of'
import { signalOf } from './signal-of'
import { tokensTextOf } from './tokens-text-of'

export type RowKind = 'header' | 'root' | 'node' | 'group' | 'detail' | 'event' | 'hint' | 'empty'

/**
 * One line of the pane or of the text tree: what it says and how it is
 * styled; `id` on the expandable rows (a node, the unlisted group).
 */
export type Row = {
  kind: RowKind
  text: string
  id?: string
  color?: string
  dim?: boolean
  bold?: boolean
  isExpanded?: boolean
}

const MAX_DEPTH = 8

export function headerTextOf(state: FlowState): string {
  const counts = countsOf(state)
  const unlisted = counts.unlisted > 0 ? ` · ${counts.unlisted} unlisted` : ''

  return `Agent flow · ${counts.agents} agents · ${counts.running} running · ${counts.waiting} waiting${unlisted}`
}

function activityTextOf(node: FlowNode, now: number, signal: Signal): string {
  switch (node.activity.kind) {
    case 'tool':
      return `${node.activity.tool} ${durationTextOf(now - node.activity.since)}${signal === 'slow' ? ' !' : ''}`
    case 'permission':
      return `waiting for approval${node.activity.tool ? `: ${node.activity.tool}` : ''}`
    default:
      return signal === 'quiet' ? `quiet ${Math.floor((now - node.lastEventAt) / 60000)}m` : ''
  }
}

function glyphOf(node: FlowNode, signal: Signal): string {
  if (signal === 'waiting') {
    return '◐'
  }

  switch (node.status) {
    case 'running':
      return '●'
    case 'completed':
      return '✓'
    case 'unknown':
      return '○'
    default:
      return '✗'
  }
}

function colorOf(node: FlowNode, signal: Signal): string | undefined {
  if (signal === 'waiting') {
    return 'magenta'
  }

  if (signal === 'slow') {
    return 'yellow'
  }

  if (signal === 'quiet') {
    return 'gray'
  }

  switch (node.status) {
    case 'running':
      return 'yellow'
    case 'completed':
      return 'green'
    case 'unknown':
      return undefined
    default:
      return 'red'
  }
}

function tagOf(node: FlowNode): string {
  if (node.status === 'unknown') {
    return node.rawStatus ? `[${node.rawStatus}]` : '[unknown]'
  }

  return isTerminal(node.status) && node.status !== 'completed' ? `[${node.status}]` : ''
}

function cut(text: string): string {
  return text.length > Limits.DESCRIPTION_MAX_CHARS ? `${text.slice(0, Limits.DESCRIPTION_MAX_CHARS - 1)}…` : text
}

/**
 * A node's line: prefix, glyph, type, name, description, elapsed, then its
 * activity, call count, tokens and status tag where each applies.
 *
 * @param node the node
 * @param now the time
 * @param prefix the tree prefix (`├─ `, `│  └─ `), or empty inline
 * @returns the text
 */
export function nodeTextOf(node: FlowNode, now: number, prefix: string): string {
  const signal = signalOf(node, now)
  const name = node.name ? ` ${node.name}` : ''
  const base = `${prefix}${glyphOf(node, signal)} ${node.type}${name} "${cut(node.description)}" ${elapsedOf(node, now)}`
  const activity = activityTextOf(node, now, signal)
  const calls = isTerminal(node.status) || node.toolCalls > 0 ? ` ×${node.toolCalls}` : ''
  const tokens = isTerminal(node.status) ? tokensTextOf(node.usage) : ''
  const tag = tagOf(node)

  return (
    base +
    (activity ? ` · ${activity}` : '') +
    (activity && node.activity.kind === 'tool' ? '' : calls) +
    (tokens ? ` · ${tokens}` : '') +
    (tag ? ` ${tag}` : '')
  )
}

/**
 * The lines under an expanded node: model and mode, prompt excerpt, recent
 * tool calls, tokens, id.
 *
 * @param node the node
 * @returns the lines, unindented
 */
export function detailsOf(node: FlowNode): string[] {
  const mode = node.fork ? 'fork' : node.background ? 'background' : 'foreground'
  const tools =
    node.recentTools.length === 0
      ? ['no tool calls yet']
      : node.recentTools.map(
          call => `${call.tool} ${durationTextOf(call.durationMs ?? 0)}${call.isError ? ' ✗' : ''}`,
        )

  return [
    `model ${node.model ?? '?'} · ${mode}`,
    ...(node.promptExcerpt ? [`prompt ${node.promptExcerpt}`] : []),
    ...tools,
    `tokens in ${node.usage.input} out ${node.usage.output} cache ${node.usage.cacheRead}/${node.usage.cacheWrite}`,
    `id ${node.id}`,
  ]
}

function nodeRowOf(node: FlowNode, now: number, prefix: string, expanded: ReadonlySet<string>): Row {
  const signal = signalOf(node, now)
  const isDim = isTerminal(node.status) && signal === 'none'

  return {
    kind: 'node',
    id: node.id,
    text: nodeTextOf(node, now, prefix),
    color: colorOf(node, signal),
    ...(isDim ? { dim: true } : {}),
    ...(signal === 'waiting' ? { bold: true } : {}),
    isExpanded: expanded.has(node.id),
  }
}

function byFirstSeen(a: FlowNode, b: FlowNode): number {
  return a.firstSeenAt - b.firstSeenAt
}

function treeRowsOf(
  nodes: readonly FlowNode[],
  now: number,
  expanded: ReadonlySet<string>,
  indent: string,
): Row[] {
  const known = new Set(nodes.map(node => node.id))
  const rows: Row[] = []

  const childrenOf = (parentId: string | null): FlowNode[] =>
    nodes
      .filter(node =>
        parentId === null ? node.parentId === null || !known.has(node.parentId) : node.parentId === parentId,
      )
      .sort(byFirstSeen)

  const walk = (parentId: string | null, prefix: string, depth: number): void => {
    if (depth > MAX_DEPTH) {
      return
    }

    const list = childrenOf(parentId)

    list.forEach((node, index) => {
      const isLast = index === list.length - 1

      rows.push(nodeRowOf(node, now, `${prefix}${isLast ? '└─ ' : '├─ '}`, expanded))

      const under = `${prefix}${isLast ? '   ' : '│  '}`

      if (expanded.has(node.id)) {
        for (const line of detailsOf(node)) {
          rows.push({ kind: 'detail', text: `${under}${line}`, dim: true })
        }
      }

      walk(node.id, under, depth + 1)
    })
  }

  walk(null, indent, 0)

  return rows
}

function rootRowOf(state: FlowState, now: number): Row {
  const root = nodeOf(state, Names.ROOT_ID)
  const activity = root ? activityTextOf(root, now, signalOf(root, now)) : ''
  const mood = root?.status === 'running' ? 'busy' : 'idle'

  return { kind: 'root', text: `main · ${mood}${activity ? ` · ${activity}` : ''}` }
}

function lastEventRowOf(state: FlowState): Row[] {
  const last = state.events.at(-1)

  return last ? [{ kind: 'event', text: `last: ${last.kind} ${last.agentId ?? 'main'} ${last.text}`, dim: true }] : []
}

/**
 * The whole document: header, root, the tree of listed agents (or an empty
 * note), the unlisted group, the last event.
 *
 * @param state the state
 * @param now the time
 * @param expanded the ids (and the unlisted key) the person expanded
 * @returns the rows
 */
export function rowsOf(state: FlowState, now: number, expanded: ReadonlySet<string>): Row[] {
  const all = [...state.nodes.values()].filter(node => node.id !== Names.ROOT_ID)
  const listed = all.filter(node => node.source !== 'event')
  const unlisted = all.filter(node => node.source === 'event')
  const tree = treeRowsOf(listed, now, expanded, '')
  const isGroupOpen = expanded.has(Names.UNLISTED_KEY)
  const group: Row[] =
    unlisted.length === 0
      ? []
      : [
          {
            kind: 'group',
            id: Names.UNLISTED_KEY,
            text: `${isGroupOpen ? '▾' : '▸'} ${Names.UNLISTED_GROUP_LABEL} (${unlisted.length})`,
            isExpanded: isGroupOpen,
          },
          ...(isGroupOpen ? treeRowsOf(unlisted, now, expanded, '   ') : []),
        ]

  return [
    { kind: 'header', text: headerTextOf(state), bold: true },
    rootRowOf(state, now),
    ...(tree.length === 0 ? [{ kind: 'empty' as const, text: `   ${Names.NO_AGENTS_TEXT}`, dim: true }] : tree),
    ...group,
    ...lastEventRowOf(state),
  ]
}

/**
 * The document for a pane seated inline above the prompt: the header, the
 * rows needing attention (waiting first, then slow) or else the running ones,
 * at most INLINE_MAX_ROWS, and a hint to widen the terminal.
 *
 * @param state the state
 * @param now the time
 * @returns the rows
 */
export function inlineRowsOf(state: FlowState, now: number): Row[] {
  const nodes = [...state.nodes.values()].filter(node => node.id !== Names.ROOT_ID).sort(byFirstSeen)
  const rank = (node: FlowNode): number => {
    const signal = signalOf(node, now)

    return signal === 'waiting' ? 0 : signal === 'slow' ? 1 : 2
  }
  const urgent = nodes.filter(node => rank(node) < 2).sort((a, b) => rank(a) - rank(b))
  const shown = (urgent.length > 0 ? urgent : nodes.filter(node => node.status === 'running')).slice(
    0,
    Limits.INLINE_MAX_ROWS,
  )
  const none: ReadonlySet<string> = new Set()

  return [
    { kind: 'header', text: headerTextOf(state), bold: true },
    ...shown.map(node => nodeRowOf(node, now, '', none)),
    { kind: 'hint', text: Names.WIDEN_TEXT, dim: true },
  ]
}
```

Add to `mods/agent-flow/hooks/model/index.ts` (alphabetical):

```ts
export * from './rows-of'
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd mods/agent-flow && bun test`
Expected: all pass. If a text assertion differs only by the exact wording the implementation produces, fix the implementation to match the test, not the reverse: the test strings are the spec's format.

Run: `cd /Users/charles/Desktop/ARRS-claude-code-main && bunx tsc -p mods/tsconfig.json && bunx tsc -p mods/agent-flow/tsconfig.json`
Expected: no output.

- [ ] **Step 5: Commit**

```bash
git add mods/agent-flow
git commit -m "agent-flow: rows: the state as lines, full and inline

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

### Task 7: The text view

**Files:**
- Create: `mods/agent-flow/hooks/views/text-view.ts`
- Create: `mods/agent-flow/hooks/views/index.ts`
- Test: `mods/agent-flow/tests/text-view.test.ts`

**Interfaces:**
- Consumes: `Row`, `rowsOf`.
- Produces: `textView(rows: readonly Row[]): string` (exported from `hooks/views/index.ts` as `Views.textView`).

- [ ] **Step 1: Write the failing test**

`mods/agent-flow/tests/text-view.test.ts`:

```ts
import { describe, expect, test, tier } from 'claude-code/testing'

import Model from '../hooks/model'
import Views from '../hooks/views'
import Fixtures from './fixtures'

tier('user')

describe('text-view', () => {
  test('the text tree is the rows, one per line, in order', () => {
    let state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1000)
    state = Model.onSpawn(state, Fixtures.spawnOf('b', { parentAgentId: 'a' }), 2000)
    const rows = Model.rowsOf(state, 5000, new Set())
    const text = Views.textView(rows)
    const lines = text.split('\n')

    expect(lines).toHaveLength(rows.length)
    expect(lines[0]).toBe('Agent flow · 2 agents · 2 running · 0 waiting')
    expect(lines[1]).toBe('main · idle')
    expect(lines[2]).toBe('└─ ● Explore "task a" 4s')
    expect(lines[3]).toBe('   └─ ● Explore "task b" 3s')
  })

  test('an empty tree still prints the header, the root and the note', () => {
    const text = Views.textView(Model.rowsOf(Model.initialState(0), 1, new Set()))

    expect(text).toContain('no subagents yet')
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd mods/agent-flow && bun test tests/text-view.test.ts`
Expected: FAIL, `Cannot find module '../hooks/views'`.

- [ ] **Step 3: Write the view**

`mods/agent-flow/hooks/views/text-view.ts`:

```ts
import type { Row } from '../model'

/**
 * The tree as plain text: what `/flow text` prints, and what `/flow` prints
 * where no pane can be drawn.
 *
 * @param rows the document
 * @returns one line per row
 */
export function textView(rows: readonly Row[]): string {
  return rows.map(row => row.text).join('\n')
}
```

`mods/agent-flow/hooks/views/index.ts`:

```ts
export * from './text-view'

export * as default from '.'
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd mods/agent-flow && bun test`
Expected: all pass.

Run: `cd /Users/charles/Desktop/ARRS-claude-code-main && bunx tsc -p mods/tsconfig.json && bunx tsc -p mods/agent-flow/tsconfig.json`
Expected: no output.

- [ ] **Step 5: Commit**

```bash
git add mods/agent-flow
git commit -m "agent-flow: the text view

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

### Task 8: The pane view

**Files:**
- Create: `mods/agent-flow/hooks/views/kit/ui.ts`, `mods/agent-flow/hooks/views/kit/index.ts`
- Create: `mods/agent-flow/hooks/views/pane-view.tsx`
- Modify: `mods/agent-flow/hooks/views/index.ts`
- Test: `mods/agent-flow/tests/pane-view.test.ts`

**Interfaces:**
- Consumes: `Row`, `rowsOf`; `ElementTable`, `RenderElement` from `claude-code`.
- Produces: `Ui = Pick<ElementTable, 'Box' | 'Text' | 'Button'>`, `PaneActions = { onToggle: (id: string) => void }`, `paneView(ui: Ui, rows: readonly Row[], actions: PaneActions): RenderElement`.

- [ ] **Step 1: Write the failing test**

`mods/agent-flow/tests/pane-view.test.ts`:

```ts
import { describe, expect, test, tier } from 'claude-code/testing'

import Model from '../hooks/model'
import Views from '../hooks/views'
import Fixtures from './fixtures'

tier('user')

type Fake = { type: string; props: Record<string, unknown>; children: unknown[] }

const ctor =
  (type: string) =>
  (props: Record<string, unknown>): Fake => {
    const { children, ...rest } = props

    return { type, props: rest, children: (children as unknown[]) ?? [] }
  }

const ui = { Box: ctor('Box'), Text: ctor('Text'), Button: ctor('Button') } as unknown as Views.Ui

describe('pane-view', () => {
  test('one column Box holds one child per row', () => {
    const rows = Model.rowsOf(Model.initialState(0), 1, new Set())
    const tree = Views.paneView(ui, rows, { onToggle: () => undefined }) as unknown as Fake

    expect(tree.type).toBe('Box')
    expect(tree.props).toEqual({ flexDirection: 'column' })
    expect(tree.children).toHaveLength(rows.length)
  })

  test('plain rows are Texts styled from the row', () => {
    const rows = Model.rowsOf(Model.initialState(0), 1, new Set())
    const tree = Views.paneView(ui, rows, { onToggle: () => undefined }) as unknown as Fake
    const header = tree.children[0] as Fake
    const empty = tree.children[2] as Fake

    expect(header.type).toBe('Text')
    expect(header.props).toEqual({ bold: true, wrap: 'truncate-end' })
    expect(header.children).toEqual([rows[0]?.text])
    expect(empty.props).toEqual({ dimColor: true, wrap: 'truncate-end' })
  })

  test('an expandable row is a row Box with its Text and a toggle Button that reports the id', () => {
    const state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)
    const rows = Model.rowsOf(state, 2, new Set())
    const pressed: string[] = []
    const tree = Views.paneView(ui, rows, { onToggle: id => pressed.push(id) }) as unknown as Fake
    const row = tree.children[2] as Fake
    const [text, button] = row.children as [Fake, Fake]

    expect(row.props).toEqual({ flexDirection: 'row' })
    expect(text.type).toBe('Text')
    expect(text.props).toEqual({ color: 'yellow', wrap: 'truncate-end' })
    expect(text.children).toEqual([`${rows[2]?.text} `])
    expect(button.type).toBe('Button')
    expect(button.props.key).toBe('toggle:a')
    expect(button.props.plain).toBe(true)
    expect(button.children).toEqual(['[+]'])
    ;(button.props.onPress as () => void)()
    expect(pressed).toEqual(['a'])
  })

  test('an expanded row shows [-]', () => {
    const state = Model.onSpawn(Model.initialState(0), Fixtures.spawnOf('a'), 1)
    const rows = Model.rowsOf(state, 2, new Set(['a']))
    const tree = Views.paneView(ui, rows, { onToggle: () => undefined }) as unknown as Fake
    const row = tree.children[2] as Fake
    const button = row.children[1] as Fake

    expect(button.children).toEqual(['[-]'])
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd mods/agent-flow && bun test tests/pane-view.test.ts`
Expected: FAIL, `Views.paneView is not a function`.

- [ ] **Step 3: Write the view**

`mods/agent-flow/hooks/views/kit/ui.ts`:

```ts
import type { ElementTable } from 'claude-code'

/**
 * The element constructors the pane draws with, destructured from the table
 * `$.ui.resolve(e)` hands the render hook. Every surface carries all three.
 */
export type Ui = Pick<ElementTable, 'Box' | 'Text' | 'Button'>
```

`mods/agent-flow/hooks/views/kit/index.ts`:

```ts
export * from './ui'

export * as default from '.'
```

`mods/agent-flow/hooks/views/pane-view.tsx`:

```tsx
/* @jsxRuntime classic */
/* @jsx h */
/* @jsxFrag Fragment */
import type { RenderElement } from 'claude-code'

import type { Row } from '../model'
import type { Ui } from './kit'

export type PaneActions = {
  /** Expands or collapses the node (or the unlisted group) with this id. */
  onToggle: (id: string) => void
}

type TextStyle = { color?: string; dimColor?: boolean; bold?: boolean; wrap: 'truncate-end' }

function styleOf(row: Row): TextStyle {
  return {
    ...(row.color !== undefined ? { color: row.color } : {}),
    ...(row.dim ? { dimColor: true } : {}),
    ...(row.bold ? { bold: true } : {}),
    wrap: 'truncate-end',
  }
}

/**
 * The pane's body: one line per row; an expandable row is a row Box holding
 * the line and a `[+]` / `[-]` Button whose press toggles the id.
 *
 * @param ui the surface's elements
 * @param rows the document
 * @param actions what a press does
 * @returns the tree
 */
export function paneView(ui: Ui, rows: readonly Row[], actions: PaneActions): RenderElement {
  const { Box, Text, Button } = ui

  return (
    <Box flexDirection="column">
      {rows.map(row => {
        const id = row.id

        if (id === undefined) {
          return <Text {...styleOf(row)}>{row.text}</Text>
        }

        return (
          <Box flexDirection="row">
            <Text {...styleOf(row)}>{`${row.text} `}</Text>
            <Button key={`toggle:${id}`} plain onPress={() => actions.onToggle(id)}>
              {row.isExpanded ? '[-]' : '[+]'}
            </Button>
          </Box>
        )
      })}
    </Box>
  )
}
```

Update `mods/agent-flow/hooks/views/index.ts`:

```ts
export * from './kit'
export * from './pane-view'
export * from './text-view'

export * as default from '.'
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd mods/agent-flow && bun test`
Expected: all pass.

Run: `cd /Users/charles/Desktop/ARRS-claude-code-main && bunx tsc -p mods/tsconfig.json && bunx tsc -p mods/agent-flow/tsconfig.json`
Expected: no output. If tsc rejects `key` on `Button`, replace `key={...}` with `label` left unset and put the address in `props` through a spread: `{...{ key: \`toggle:${id}\` }}`.

- [ ] **Step 5: Commit**

```bash
git add mods/agent-flow
git commit -m "agent-flow: the pane view

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

### Task 9: Pane toggle and auto-open decisions

**Files:**
- Create: `mods/agent-flow/hooks/pane-toggle/pane-toggle-of.ts`, `should-auto-open.ts`, `index.ts`
- Test: `mods/agent-flow/tests/pane-toggle.test.ts`

**Interfaces:**
- Consumes: `Limits.AUTO_OPEN_MIN_COLUMNS`, `Limits.KEPT_OPEN_MIN_COLUMNS`, `RenderSurface` from `claude-code`.
- Produces:
  - `PaneBelief = { isBelievedOpen: boolean; wasDrawnWhenProbed: boolean }`, `paneToggleOf(pane: PaneBelief): 'open' | 'close'`
  - `AutoOpenFacts = { startedSurface: RenderSurface | null; hasAutoOpened: boolean; closedByPerson: boolean; storedOpen: unknown; columns: number | null }`, `shouldAutoOpen(facts: AutoOpenFacts): boolean`

- [ ] **Step 1: Write the failing test**

`mods/agent-flow/tests/pane-toggle.test.ts`:

```ts
import { describe, expect, test, tier } from 'claude-code/testing'

import Limits from '../hooks/limits'
import PaneToggle from '../hooks/pane-toggle'

tier('user')

describe('pane-toggle', () => {
  test('/flow closes only a pane believed open that still draws, else opens', () => {
    expect(PaneToggle.paneToggleOf({ isBelievedOpen: true, wasDrawnWhenProbed: true })).toBe('close')
    expect(PaneToggle.paneToggleOf({ isBelievedOpen: true, wasDrawnWhenProbed: false })).toBe('open')
    expect(PaneToggle.paneToggleOf({ isBelievedOpen: false, wasDrawnWhenProbed: false })).toBe('open')
  })

  const facts = {
    startedSurface: 'terminal' as const,
    hasAutoOpened: false,
    closedByPerson: false,
    storedOpen: undefined,
    columns: Limits.AUTO_OPEN_MIN_COLUMNS,
  }

  test('auto-open needs a terminal, a first time, no earlier close and enough columns', () => {
    expect(PaneToggle.shouldAutoOpen(facts)).toBe(true)
    expect(PaneToggle.shouldAutoOpen({ ...facts, startedSurface: null })).toBe(false)
    expect(PaneToggle.shouldAutoOpen({ ...facts, hasAutoOpened: true })).toBe(false)
    expect(PaneToggle.shouldAutoOpen({ ...facts, closedByPerson: true })).toBe(false)
    expect(PaneToggle.shouldAutoOpen({ ...facts, storedOpen: false })).toBe(false)
    expect(PaneToggle.shouldAutoOpen({ ...facts, columns: null })).toBe(false)
    expect(PaneToggle.shouldAutoOpen({ ...facts, columns: Limits.AUTO_OPEN_MIN_COLUMNS - 1 })).toBe(false)
  })

  test('a person who kept it open before gets it from the narrower width', () => {
    expect(PaneToggle.shouldAutoOpen({ ...facts, storedOpen: true, columns: Limits.KEPT_OPEN_MIN_COLUMNS })).toBe(true)
    expect(PaneToggle.shouldAutoOpen({ ...facts, storedOpen: true, columns: Limits.KEPT_OPEN_MIN_COLUMNS - 1 })).toBe(false)
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd mods/agent-flow && bun test tests/pane-toggle.test.ts`
Expected: FAIL, `Cannot find module '../hooks/pane-toggle'`.

- [ ] **Step 3: Write the decisions**

`mods/agent-flow/hooks/pane-toggle/pane-toggle-of.ts`:

```ts
export type PaneBelief = {
  /** The plugin opened the pane and has not closed it. */
  isBelievedOpen: boolean
  /** A redraw asked just now was answered by a `ui.render` of the pane. */
  wasDrawnWhenProbed: boolean
}

/**
 * What `/flow` does: close only when the plugin opened the pane and it still
 * draws (the person may have closed it with no event the plugin hooks);
 * otherwise open.
 *
 * @param pane the belief and the probe's answer
 * @returns `open` or `close`
 */
export function paneToggleOf(pane: PaneBelief): 'open' | 'close' {
  return pane.isBelievedOpen && pane.wasDrawnWhenProbed ? 'close' : 'open'
}
```

`mods/agent-flow/hooks/pane-toggle/should-auto-open.ts`:

```ts
import type { RenderSurface } from 'claude-code'

import Limits from '../limits'

export type AutoOpenFacts = {
  startedSurface: RenderSurface | null
  hasAutoOpened: boolean
  closedByPerson: boolean
  /** The store's `agent-flow.open`: true, false or anything else for unset. */
  storedOpen: unknown
  columns: number | null
}

/**
 * Whether the first spawn of the session opens the pane by itself: on a
 * terminal, once, unless the person closed it (now or in an earlier
 * session), and only when the terminal is wide enough: 144 columns, or 110
 * for a person who kept it open before.
 *
 * @param facts what is known at the spawn
 * @returns true to open
 */
export function shouldAutoOpen(facts: AutoOpenFacts): boolean {
  if (facts.startedSurface !== 'terminal' || facts.hasAutoOpened || facts.closedByPerson) {
    return false
  }

  if (facts.storedOpen === false || facts.columns === null) {
    return false
  }

  const minimum = facts.storedOpen === true ? Limits.KEPT_OPEN_MIN_COLUMNS : Limits.AUTO_OPEN_MIN_COLUMNS

  return facts.columns >= minimum
}
```

`mods/agent-flow/hooks/pane-toggle/index.ts`:

```ts
export * from './pane-toggle-of'
export * from './should-auto-open'

export * as default from '.'
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `cd mods/agent-flow && bun test`
Expected: all pass.

Run: `cd /Users/charles/Desktop/ARRS-claude-code-main && bunx tsc -p mods/tsconfig.json && bunx tsc -p mods/agent-flow/tsconfig.json`
Expected: no output.

- [ ] **Step 5: Commit**

```bash
git add mods/agent-flow
git commit -m "agent-flow: pane toggle and auto-open decisions

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

### Task 10: Host and the hooks module

**Files:**
- Create: `mods/agent-flow/hooks/host/host.ts`, `mods/agent-flow/hooks/host/index.ts`
- Modify: `mods/agent-flow/hooks/register.ts` (replace the Task 1 stub entirely)

**Interfaces:**
- Consumes: everything from Tasks 1 to 9: `Model.*` reducers and selectors, `Views.paneView`, `Views.textView`, `PaneToggle.paneToggleOf`, `PaneToggle.shouldAutoOpen`, `Names.*`, `Limits.*`.
- Produces: `Host` (the `$` subset), `register(on: On)` hooking `session.start`, `ui.render{PromptHint}`, `ui.render{Pane}`, `command.run{flow}`, `ui.close`, `agent.spawn`, `tool.call`, `turn.start`, `turn.complete`, `classic.PermissionRequest`, `classic.Notification{permission_prompt}`.

There is no unit test for this file: it is wiring over tested pure functions. Its checks are `claude plugin validate`, both tsc runs, and two live headless runs in Step 4.

- [ ] **Step 1: Write the host**

`mods/agent-flow/hooks/host/host.ts`:

```ts
import type { AgentInfo, CommandSpec, PaneCloseArgs, PaneOpenArgs, TimerCall } from 'claude-code'

/**
 * The engine as `session.start` bound it from its `$`, each member spelled
 * `$.noun.method(...)` there; used by every later hook, timer and press.
 */
export type Host = {
  /** `$.clock.now`. */
  now: () => Promise<number>
  /** `$.clock.after`. */
  after: TimerCall
  /** `$.clock.every`. */
  every: TimerCall
  /** `$.clock.sleep`, no signal. */
  sleep: (ms: number) => Promise<void>
  /** `$.agent.list`. */
  listAgents: () => Promise<AgentInfo[]>
  /** `$.store.get`. */
  storeGet: (key: string) => Promise<unknown>
  /** `$.store.set`. */
  storeSet: (key: string, value: unknown) => Promise<void>
  /** `$.ui.invalidate("ui.render")`: every pane instance draws again. */
  invalidate: () => void
  /** `$.ui.status`: the plugin's line under the prompt. */
  status: (text: string | undefined) => void
  /** `$.ui.log`: one line under the plugin's name. */
  uiLog: (text: string) => void
  /** `$.ui.open`. */
  openPane: (pane: PaneOpenArgs) => Promise<void>
  /** `$.ui.close`. */
  closePane: (pane: PaneCloseArgs) => Promise<void>
  /** `$.command.register`; rejects while another `/flow` is listed. */
  registerCommand: (spec: CommandSpec) => Promise<unknown>
}
```

`mods/agent-flow/hooks/host/index.ts`:

```ts
export * from './host'

export * as default from '.'
```

- [ ] **Step 2: Write the hooks module**

`mods/agent-flow/hooks/register.ts`:

```ts
import type { On, Timer } from 'claude-code'

import type { Host } from './host'
import Limits from './limits'
import Model from './model'
import Names from './names'
import PaneToggle from './pane-toggle'
import Views from './views'

const messageOf = (error: unknown): string => (error instanceof Error ? error.message : String(error))

/**
 * Registers the agent flow pane: `/flow` once the command is granted, the
 * pane's drawing, the reducers behind every agent event, the reconcile and
 * tick timers while agents run, the auto-open on the first spawn, and the
 * text fallback where no surface draws the pane.
 *
 * `session.start` binds the host every later hook reads through; until it
 * has (or when `/flow` is refused because another plugin holds it) every
 * hook passes its event on untouched.
 *
 * @param on the engine's registrar
 */
export function register(on: On) {
  let host: Host | null = null
  let state: Model.FlowState = Model.initialState(0)
  let storedOpen: unknown = undefined
  let lastStatus: string | undefined = undefined
  let redrawTimer: Timer | null = null
  const timers = new Map<'reconcile' | 'tick', Timer>()

  function isAnythingRunning(): boolean {
    for (const node of state.nodes.values()) {
      if (node.id !== Names.ROOT_ID && node.status === 'running') {
        return true
      }
    }

    return false
  }

  function redraw(): void {
    const engine = host

    if (engine === null || redrawTimer !== null) {
      return
    }

    redrawTimer = engine.after(Limits.INVALIDATE_DEBOUNCE_MS, () => {
      redrawTimer = null
      engine.invalidate()
    })
  }

  function syncTimers(): void {
    const engine = host

    if (engine === null) {
      return
    }

    const isWanted = isAnythingRunning()

    if (isWanted && !timers.has('reconcile')) {
      timers.set(
        'reconcile',
        engine.every(Limits.RECONCILE_MS, () => {
          void reconcileNow()
        }),
      )
      timers.set(
        'tick',
        engine.every(Limits.TICK_MS, () => {
          if (state.pane.isBelievedOpen) {
            engine.invalidate()
          }
        }),
      )
    }

    if (!isWanted && timers.size > 0) {
      for (const timer of timers.values()) {
        timer.cancel()
      }

      timers.clear()
    }
  }

  function syncStatus(): void {
    const engine = host

    if (engine === null) {
      return
    }

    const waiting = Model.countsOf(state).waiting
    const text =
      !state.pane.isBelievedOpen && waiting > 0
        ? `${waiting} agent${waiting === 1 ? '' : 's'} waiting for approval`
        : undefined

    if (text !== lastStatus) {
      lastStatus = text
      engine.status(text)
    }
  }

  function apply(reduce: (current: Model.FlowState) => Model.FlowState): void {
    state = reduce(state)
    syncTimers()
    syncStatus()
    redraw()
  }

  function noteFailure(kind: string, error: unknown, at: number): void {
    state = Model.withEvent(state, { at, kind, text: `failed: ${messageOf(error)}` })
  }

  async function reconcileNow(): Promise<void> {
    const engine = host

    if (engine === null) {
      return
    }

    try {
      const listed = await engine.listAgents()
      const now = await engine.now()

      apply(current => Model.reconcile(current, listed, now))
    } catch (error) {
      noteFailure('agent.list', error, await engine.now().catch(() => 0))
    }
  }

  async function probeDrawn(engine: Host): Promise<boolean> {
    state = Model.withPane(state, { wasDrawnSinceProbe: false })
    engine.invalidate()
    await engine.sleep(Limits.OPEN_PROBE_MS)

    return state.pane.wasDrawnSinceProbe
  }

  async function openPane(engine: Host, isFocused: boolean): Promise<boolean> {
    await engine.openPane({
      id: Names.PANE_ID,
      title: Names.PANE_TITLE,
      ...(isFocused ? { focus: true as const } : {}),
    })
    state = Model.withPane(state, { isBelievedOpen: true })

    const isDrawn = await probeDrawn(engine)

    if (!isDrawn) {
      await engine.closePane({ id: Names.PANE_ID }).catch(() => undefined)
      state = Model.withPane(state, { isBelievedOpen: false })
    }

    syncStatus()

    return isDrawn
  }

  async function closePane(engine: Host): Promise<void> {
    await engine.closePane({ id: Names.PANE_ID })
    state = Model.withPane(state, { isBelievedOpen: false })
    syncStatus()
  }

  function toggleExpanded(id: string): void {
    const expanded = new Set(state.pane.expanded)

    if (expanded.has(id)) {
      expanded.delete(id)
    } else {
      expanded.add(id)
    }

    state = Model.withPane(state, { expanded })
    redraw()
  }

  function textTreeOf(now: number): string {
    return Views.textView(Model.rowsOf(state, now, state.pane.expanded))
  }

  async function maybeAutoOpen(engine: Host): Promise<void> {
    const isWanted = PaneToggle.shouldAutoOpen({
      startedSurface: state.startedSurface,
      hasAutoOpened: state.pane.hasAutoOpened,
      closedByPerson: state.pane.closedByPerson,
      storedOpen,
      columns: state.pane.columns,
    })

    if (!isWanted || state.pane.isBelievedOpen) {
      return
    }

    state = Model.withPane(state, { hasAutoOpened: true })
    await openPane(engine, false)
  }

  on('session.start', async ($, e, next) => {
    const engine: Host = {
      now: () => $.clock.now(),
      after: (ms, fn) => $.clock.after(ms, fn),
      every: (ms, fn) => $.clock.every(ms, fn),
      sleep: ms => $.clock.sleep(ms),
      listAgents: () => $.agent.list(),
      storeGet: key => $.store.get(key),
      storeSet: (key, value) => $.store.set(key, value),
      invalidate: () => $.ui.invalidate('ui.render'),
      status: text => $.ui.status(text),
      uiLog: text => $.ui.log(text),
      openPane: pane => $.ui.open(pane),
      closePane: pane => $.ui.close(pane),
      registerCommand: spec => $.command.register(spec),
    }
    const now = await engine.now()

    state = { ...Model.initialState(now), startedSurface: e.surface }

    try {
      await engine.registerCommand({ name: Names.COMMAND_NAME, description: Names.COMMAND_DESCRIPTION })
    } catch (error) {
      const reason = messageOf(error)

      if (!Names.BUILTIN_HOLDS_PATTERN.test(reason)) {
        engine.uiLog(`${Names.REGISTER_FAILED_TEXT}${reason}`)
      }

      return next(e)
    }

    host = engine
    storedOpen = await engine.storeGet(Names.STORE_OPEN_KEY).catch(() => undefined)
    await reconcileNow()

    return next(e)
  })

  on('ui.render', { component: 'PromptHint' }, ($, e, next) => {
    const columns = e.viewport?.columns

    if (columns !== undefined && columns !== state.pane.columns) {
      state = Model.withPane(state, { columns })
    }

    return next(e)
  })

  on('ui.render', { component: 'Pane' }, async ($, e, next) => {
    if (e.requestId !== Names.PANE_ID || host === null) {
      return next(e)
    }

    try {
      const { Box, Text, Button } = await $.ui.resolve(e)
      const now = await $.clock.now()

      state = Model.withPane(state, {
        wasDrawnSinceProbe: true,
        columns: e.viewport?.columns ?? state.pane.columns,
        placement: e.props.placement,
      })

      const rows =
        e.props.placement === 'inline'
          ? Model.inlineRowsOf(state, now)
          : Model.rowsOf(state, now, state.pane.expanded)

      return Views.paneView({ Box, Text, Button }, rows, { onToggle: toggleExpanded })
    } catch (error) {
      noteFailure('ui.render', error, await $.clock.now().catch(() => 0))

      return next(e)
    }
  })

  on('command.run', { command: Names.COMMAND_NAME }, async ($, e, next) => {
    const engine = host

    if (engine === null) {
      return next(e)
    }

    const now = await engine.now()
    const args = e.args.trim()

    if (args === 'text') {
      return { text: textTreeOf(now) }
    }

    if (args !== '') {
      return { text: Names.USAGE_TEXT }
    }

    const wasDrawnWhenProbed = state.pane.isBelievedOpen && (await probeDrawn(engine))
    const toggle = PaneToggle.paneToggleOf({ isBelievedOpen: state.pane.isBelievedOpen, wasDrawnWhenProbed })

    if (toggle === 'close') {
      await closePane(engine)
      storedOpen = false
      await engine.storeSet(Names.STORE_OPEN_KEY, false).catch(() => undefined)

      return {}
    }

    const isDrawn = await openPane(engine, true)

    if (!isDrawn) {
      return { text: textTreeOf(now) }
    }

    storedOpen = true
    await engine.storeSet(Names.STORE_OPEN_KEY, true).catch(() => undefined)

    return {}
  })

  on('ui.close', ($, e, next) => {
    if (e.id === Names.PANE_ID) {
      const isByPerson = e.origin.kind === 'person'

      state = Model.withPane(state, { isBelievedOpen: false, ...(isByPerson ? { closedByPerson: true } : {}) })

      if (isByPerson) {
        storedOpen = false
        void host?.storeSet(Names.STORE_OPEN_KEY, false).catch(() => undefined)
      }

      syncStatus()
    }

    return next(e)
  })

  on('agent.spawn', async ($, e, next) => {
    const result = await next(e)
    const engine = host

    if (engine === null) {
      return result
    }

    try {
      const now = await engine.now()

      apply(current =>
        Model.onSpawn(
          current,
          {
            agentId: result.agentId,
            deny: result.deny,
            parentAgentId: e.parentAgentId,
            subagentType: e.subagentType,
            description: e.description,
            model: result.model,
            name: e.name,
            background: e.background,
            fork: e.fork,
            prompt: e.prompt,
          },
          now,
        ),
      )
      await maybeAutoOpen(engine)
    } catch (error) {
      noteFailure('agent.spawn', error, await engine.now().catch(() => 0))
    }

    return result
  })

  on('tool.call', async ($, e, next) => {
    const engine = host

    if (engine !== null) {
      try {
        const now = await engine.now()

        apply(current => Model.onToolStart(current, { agentId: e.agentId, tool: e.tool, toolUseId: e.tool_use_id }, now))
      } catch (error) {
        noteFailure('tool.call', error, 0)
      }
    }

    const result = await next(e)

    if (engine !== null) {
      try {
        const now = await engine.now()
        const isError = 'isError' in result && result.isError === true

        apply(current => Model.onToolEnd(current, { agentId: e.agentId, tool: e.tool, isError }, now))
      } catch (error) {
        noteFailure('tool.call', error, 0)
      }
    }

    return result
  })

  on('turn.start', async ($, e, next) => {
    const engine = host

    if (engine !== null) {
      try {
        const now = await engine.now()

        apply(current => Model.onTurnStart(current, now))
      } catch (error) {
        noteFailure('turn.start', error, 0)
      }
    }

    return next(e)
  })

  on('turn.complete', async ($, e, next) => {
    const engine = host

    if (engine !== null) {
      try {
        const now = await engine.now()

        apply(current =>
          Model.onTurnComplete(
            current,
            { agentId: e.agentId, reason: e.reason, durationMs: e.durationMs, usage: e.usage },
            now,
          ),
        )
      } catch (error) {
        noteFailure('turn.complete', error, 0)
      }
    }

    return next(e)
  })

  on('classic.PermissionRequest', async ($, e, next) => {
    const engine = host

    if (engine !== null) {
      try {
        const now = await engine.now()

        apply(current => Model.onPermission(current, { agentId: e.agent_id, tool: e.tool_name }, now))
      } catch (error) {
        noteFailure('permission', error, 0)
      }
    }

    return next(e)
  })

  on('classic.Notification', { notification_type: 'permission_prompt' }, async ($, e, next) => {
    const engine = host

    if (engine !== null) {
      try {
        const now = await engine.now()

        apply(current => Model.onPermission(current, { agentId: e.agent_id }, now))
      } catch (error) {
        noteFailure('permission', error, 0)
      }
    }

    return next(e)
  })
}
```

If `tsc` reports that `tool_use_id` does not exist on the `tool.call` input, delete `toolUseId: e.tool_use_id` from the `onToolStart` call: the field is informational only. If `tsc` rejects the `classic.Notification` matcher, register without it and test `e.notification_type === 'permission_prompt'` inside the hook before applying.

- [ ] **Step 3: Static checks**

Run: `claude plugin validate mods/agent-flow`
Expected: `✔ Validation passed`, and the notes list the hooks `session.start, ui.render{component=PromptHint}, ui.render{component=Pane}, command.run{command=flow}, ui.close, agent.spawn, tool.call, turn.start, turn.complete, classic.PermissionRequest, classic.Notification{notification_type=permission_prompt}` and the calls `$.agent.list, $.clock.after, $.clock.every, $.clock.now, $.clock.sleep, $.command.register, $.store.get, $.store.set, $.ui.close, $.ui.invalidate, $.ui.log, $.ui.open, $.ui.resolve, $.ui.status`. If validation names a rule this module breaks, restructure to satisfy it (the two known rules are in Global Constraints) and re-run.

Run: `cd /Users/charles/Desktop/ARRS-claude-code-main && bunx tsc -p mods/tsconfig.json && bunx tsc -p mods/agent-flow/tsconfig.json`
Expected: no output.

Run: `cd mods/agent-flow && bun test`
Expected: all pass (nothing changed for the unit tests).

- [ ] **Step 4: Live headless checks**

Run (from the repository root; `source` first, in the same shell):

```bash
source mods/agent-flow/scripts/local-env.sh
mkdir -p /tmp/agent-flow-check && cd /tmp/agent-flow-check
claude --plugin-dir /Users/charles/Desktop/ARRS-claude-code-main/mods/agent-flow -p "/flow text" --output-format json --model haiku --debug-file /tmp/agent-flow-check/debug-text.txt | python3 -c 'import json,sys; print(json.load(sys.stdin)["result"])'
```

Expected output (the `-p` session has no pane surface, so `/flow text` and `/flow` both print):

```
agent-flow: Agent flow · 0 agents · 0 running · 0 waiting
main · idle
   no subagents yet; ask Claude to use the Agent tool
```

and `grep -c "hooks module agent-flow loaded" /tmp/agent-flow-check/debug-text.txt` prints `1`.

Run:

```bash
claude --plugin-dir /Users/charles/Desktop/ARRS-claude-code-main/mods/agent-flow -p "Use the Agent tool with subagent_type Explore to count how many .md files are in the current directory. Reply with only the number." --output-format json --max-turns 8 --debug-file /tmp/agent-flow-check/debug-agent.txt | python3 -c 'import json,sys; d=json.load(sys.stdin); print(d["result"], d["subagent_stats"]["spawned"])'
grep -o -E "hooks module agent-flow (agent\.spawn|tool\.call|turn\.complete) settled" /tmp/agent-flow-check/debug-agent.txt | sort | uniq -c
```

Expected: the second line shows at least one `agent.spawn settled`, several `tool.call settled` and at least one `turn.complete settled`; no `[agent-flow]` error lines (`grep -i "agent-flow.*error" /tmp/agent-flow-check/debug-agent.txt` prints nothing).

- [ ] **Step 5: Commit**

```bash
cd /Users/charles/Desktop/ARRS-claude-code-main
git add mods/agent-flow
git commit -m "agent-flow: host and the hooks module

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

### Task 11: Kit test, smoke test, README and the final verification

**Files:**
- Create: `mods/agent-flow/tests/fixtures/answers-engine.ts`; Modify: `mods/agent-flow/tests/fixtures/index.ts`
- Create: `mods/agent-flow/tests/register.kit.ts`
- Modify: `mods/agent-flow/tsconfig.json` (exclude the kit test from the standalone typecheck)
- Create: `mods/agent-flow/scripts/smoke.exp`, `mods/agent-flow/scripts/smoke.sh`
- Create: `mods/agent-flow/README.md`
- Modify: `docs/superpowers/specs/2026-09-13-agent-flow-mod-design.md` (status line)

**Interfaces:**
- Consumes: `register`, `Names.*`, `Limits.OPEN_PROBE_MS`, the `claude-code/testing` kit (`describe`, `expect`, `mock`, `test`, `tier`).
- Produces: nothing new in code; the verification artifacts.

- [ ] **Step 1: Write the engine-kit test (typechecks now, runs once `claude plugin test` ships)**

`mods/agent-flow/tests/fixtures/answers-engine.ts`:

```ts
import type { On } from 'claude-code'
import { mock } from 'claude-code/testing'
import type { MockClock } from 'claude-code/testing'

/**
 * Answers what an agent-flow session asks the engine beneath the plugin:
 * its start, each command it registers, an empty agent list, the store, and
 * the ui calls that draw nothing in a test. Returns the clock the session
 * reads, at 0 until the test moves it.
 *
 * @param on the test's `on`
 * @returns the mock clock
 */
export function answersEngine(on: On): MockClock {
  on('session.start', ($, e) => ({ cwd: e.cwd }))
  on('command.register', ($, e) => ({ value: { command: e.name } }))
  on('agent.list', () => ({ value: [] }))
  on('ui.invalidate', () => ({ value: undefined }))
  on('ui.status', () => ({ value: undefined }))
  on('ui.log', () => ({ value: undefined }))
  on('ui.close', ($, e, next) => next(e))
  mock.store(on)

  return mock.clock(on)
}
```

Add to `mods/agent-flow/tests/fixtures/index.ts`: `export * from './answers-engine'` (alphabetically first).

`mods/agent-flow/tests/register.kit.ts`:

```ts
import { describe, expect, test, tier } from 'claude-code/testing'

import Limits from '../hooks/limits'
import Names from '../hooks/names'
import Fixtures from './fixtures'

tier('user')

const FLOW = { command: Names.COMMAND_NAME, args: '', origin: { kind: 'composer' as const } }
const SESSION = { surface: 'terminal' as const, isInteractive: true, cwd: '/work' }

describe('register', () => {
  test('/flow opens the pane; with no surface drawing it, the tree is printed instead', async ($, on) => {
    const opened: string[] = []
    const clock = Fixtures.answersEngine(on)

    on('ui.open', ($, e, next) => {
      opened.push(e.id)

      return next(e)
    })

    await $.session.start(SESSION)

    const ran = $.command.run(FLOW)

    await clock.settle()
    await clock.advance(Limits.OPEN_PROBE_MS)

    const { text } = await ran

    expect(opened).toEqual([Names.PANE_ID])
    expect(text).toContain('Agent flow · 0 agents')
    expect(text).toContain(Names.NO_AGENTS_TEXT)
  })

  test('/flow text prints the tree without opening anything', async ($, on) => {
    const opened: string[] = []

    Fixtures.answersEngine(on)
    on('ui.open', ($, e, next) => {
      opened.push(e.id)

      return next(e)
    })

    await $.session.start(SESSION)

    const { text } = await $.command.run({ ...FLOW, args: 'text' })

    expect(opened).toEqual([])
    expect(text).toContain('main · idle')
  })

  test('when another plugin holds /flow, this one stands down', async ($, on) => {
    const logged: string[] = []

    Fixtures.answersEngine(on)
    on('command.register', () => ({ deny: 'a command named flow is registered already' }))
    on('command.run', () => ({ text: 'the other /flow ran' }))
    on('ui.log', ($, e) => {
      logged.push(e.text)

      return { value: undefined }
    })

    await $.session.start(SESSION)

    expect(await $.command.run(FLOW)).toEqual({ text: 'the other /flow ran' })
    expect(logged).toEqual([])
  })
})
```

In `mods/agent-flow/tsconfig.json` add, as a sibling of `include`:

```json
  "exclude": ["tests/*.kit.ts"]
```

Run: `cd /Users/charles/Desktop/ARRS-claude-code-main && bunx tsc -p mods/tsconfig.json && bunx tsc -p mods/agent-flow/tsconfig.json`
Expected: no output. (The first run typechecks the kit test against the real kit types; the second skips it. If the real kit's `on('agent.list', ...)` or `mock.store` signatures differ from what the fixture assumes, adjust the fixture to the declaration in `mods/types/claude-code.d.ts`, not the other way round.)

Run: `cd mods/agent-flow && bun test`
Expected: all pass, and `register.kit.ts` is not listed (bun runs `*.test.ts` only).

- [ ] **Step 2: Write the smoke test**

`mods/agent-flow/scripts/smoke.exp`:

```tcl
#!/usr/bin/expect -f
# Drives an interactive claude session with the mod loaded, in a 170-column
# pseudo terminal: opens /flow, dispatches a general-purpose agent that in
# turn dispatches an Explore agent, waits, closes the pane, exits. Every wait
# keeps reading the pty (a Claude Code session blocks on a full terminal
# buffer). Text and Enter go separately so a burst is not read as a paste.
#
# usage: expect smoke.exp <plugin dir> <work dir> <out dir>
set plugin [lindex $argv 0]
set work [lindex $argv 1]
set out [lindex $argv 2]
set env(TERM) "xterm-256color"
foreach v {CLAUDE_CODE_CHILD_SESSION CLAUDECODE CLAUDE_CODE_ENTRYPOINT CLAUDE_CODE_SSE_PORT CLAUDE_CODE_REMOTE CLAUDE_CODE_SESSION_ATTENDED CLAUDE_CODE_MESSAGING_SOCKET CLAUDE_CODE_MESSAGING_TOKEN CLAUDE_CODE_SESSION_ID CLAUDE_PID CLAUDE_CODE_EXECPATH CLAUDE_CODE_ENABLE_TASKS CLAUDE_EFFORT CLAUDE_AGENT_SDK_VERSION CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING} {
  catch {unset env($v)}
}
proc drain {secs} {
  expect -timeout $secs timeout {} eof { puts "\n>>> eof during drain"; exit 0 }
}
proc type_line {text} {
  send -- $text
  drain 1
  send "\r"
}
proc type_command {text} {
  type_line $text
  drain 2
  send "\r"
}
cd $work
log_file -noappend "$out/smoke.raw"
spawn -noecho claude --plugin-dir $plugin --debug-file "$out/smoke-debug.txt"
stty rows 45 columns 170 < $spawn_out(slave,name)
set timeout 8
expect {
  -re {(?i)trust} { sleep 1; send "\033\[B"; sleep 1; send "\r"; puts "\n>>> trust dialog answered" }
  timeout { puts "\n>>> no trust dialog" }
}
drain 12
puts "\n>>> /flow"
type_command "/flow"
drain 8
puts "\n>>> prompt"
type_line "Use the Agent tool with subagent_type general-purpose and this exact prompt: Use the Agent tool with subagent_type Explore to count how many .ts files are under the hooks directory, then reply with only that number. When it returns, reply with only the number."
drain 120
puts "\n>>> closing"
type_command "/flow"
drain 4
puts "\n>>> exiting"
type_command "/exit"
set timeout 20
expect {
  eof { puts "\n>>> session exited" }
  timeout { send "\003"; sleep 1; send "\003"; expect { eof {} timeout {} }; puts "\n>>> forced exit" }
}
```

`mods/agent-flow/scripts/smoke.sh`:

```bash
#!/usr/bin/env bash
# Runs the interactive smoke test and judges its output. Costs real API
# calls (about half a dollar). usage: scripts/smoke.sh [out dir]
set -euo pipefail

here="$(cd "$(dirname "$0")" && pwd)"
plugin="$(cd "$here/.." && pwd)"
out="${1:-/tmp/agent-flow-smoke}"
work="$(mktemp -d /tmp/agent-flow-work.XXXXXX)"

if [ -f "$here/local-env.sh" ]; then
  # shellcheck source=/dev/null
  source "$here/local-env.sh"
fi

mkdir -p "$out"
cp -R "$plugin/hooks" "$work/hooks"
rm -f "$out/smoke.raw" "$out/smoke-debug.txt"

expect "$here/smoke.exp" "$plugin" "$work" "$out" > "$out/expect.log" 2>&1 || true
grep '>>>' "$out/expect.log" || true

python3 - "$out/smoke.raw" "$out/smoke-debug.txt" <<'PY'
import re, sys

raw = open(sys.argv[1], 'rb').read().decode('utf-8', 'replace')
clean = re.sub(r'\x1b\[[0-?]*[ -/]*[@-~]|\x1b\][^\x07]*\x07|\x1b[=>]|\r', '', raw)
debug = open(sys.argv[2], encoding='utf-8', errors='replace').read()
checks = {
    'pane header drawn': 'Agent flow ·' in clean,
    'a general-purpose node': re.search(r'└─ [●✓✗◐○] general-purpose', clean) is not None,
    'a nested Explore node': re.search(r'   └─ [●✓✗◐○] Explore', clean) is not None,
    'a node finished': '✓' in clean,
    'module loaded': 'hooks module agent-flow loaded' in debug,
    'command ran': 'hooks module agent-flow command.run settled' in debug,
    'pane rendered': 'key=agent-flow' in debug,
    'no event-loop stall': '[event-loop-stall] blocked' not in debug,
    'no hook failure': re.search(r'agent-flow.*(threw|failed|error)', debug, re.I) is None,
}
ok = True
for name, passed in checks.items():
    print(('PASS ' if passed else 'FAIL ') + name)
    ok = ok and passed
sys.exit(0 if ok else 1)
PY
```

Run: `chmod +x mods/agent-flow/scripts/smoke.sh mods/agent-flow/scripts/smoke.exp`

- [ ] **Step 3: Write the README**

`mods/agent-flow/README.md`:

```markdown
# agent-flow

The agent flow pane as a plugin: `/flow` opens a live tree of the session's
subagents and in-process teammates beside the transcript, and closes it
again. Each row is one agent: status, type, name, description, elapsed time,
what it is doing right now (a tool call and how long it has run, or a wait
for the person's approval), its call count, and its tokens once it finished.
A `[+]` on a row expands its details: model, prompt excerpt, recent tool
calls, tokens. Agents waiting for approval are highlighted; a tool call over
30 seconds and a running agent silent for two minutes are marked too.

The tree comes from engine events (`agent.spawn`, `tool.call`,
`turn.complete`, the classic permission events) and is reconciled with
`$.agent.list()` every two seconds while anything runs. Loops the engine
never listed (a Workflow tool's agents, the engine's own forks) appear under
a collapsed "unlisted loops" group so they never vanish silently.

Where the surface cannot draw a pane (a `-p` run, the VS Code extension as
of 2.1.270) `/flow` prints the same tree as text; `/flow text` always prints
it. On a terminal narrower than 110 columns the pane sits inline above the
prompt and shows only the rows that need attention. The first spawn of a
session opens the pane by itself on a terminal of 144 columns or more (110
when you kept it open before), unless you closed it.

Hooks modules are early access and load only where function hooks are
enabled; see `mods/README.md`.

## What it hooks

| event | what the hook does |
| --- | --- |
| `session.start` | Binds the engine, registers `/flow` (stands down when another `/flow` is listed), reads the open preference, reconciles once. |
| `ui.render` of `PromptHint` | Reads the terminal's width for the auto-open decision. |
| `ui.render` of `Pane` | Draws the pane: header, root, tree, unlisted group, last event; the inline summary when seated above the prompt. |
| `command.run` of `flow` | Toggles the pane, printing the text tree where no surface draws it; `text` prints it outright. |
| `ui.close` | Forgets an open pane the person closed, and remembers not to auto-open again. |
| `agent.spawn` | Adds the new agent under its parent; opens the pane on the session's first spawn. |
| `tool.call` | Marks the loop busy in the tool, then counts the call and its duration. |
| `turn.start`, `turn.complete` | The root's busy state; a subagent's end status, duration and tokens. |
| `classic.PermissionRequest`, `classic.Notification` | Marks the loop waiting for approval. |

## What it calls on `$`

`agent.list`, `clock.after`, `clock.every`, `clock.now`, `clock.sleep`,
`command.register`, `store.get`, `store.set`, `ui.close`, `ui.invalidate`,
`ui.log`, `ui.open`, `ui.resolve`, `ui.status`.

## Try it

    claude --plugin-dir /path/to/agent-flow

then `/flow`, and ask Claude to use the Agent tool.

## Tests

    cd mods/agent-flow && bun test          # unit tests over the pure model and views
    bunx tsc -p mods/tsconfig.json          # types, with the rest of the mods
    claude plugin validate mods/agent-flow  # the engine's static checks
    mods/agent-flow/scripts/smoke.sh        # interactive smoke test, costs API calls

`tests/register.kit.ts` is written for `claude plugin test`; rename it to
`register.test.ts` once that command ships.
```

- [ ] **Step 4: Final verification**

Run, from the repository root:

```bash
cd mods/agent-flow && bun test && cd ../..
bunx tsc -p mods/tsconfig.json && bunx tsc -p mods/agent-flow/tsconfig.json
claude plugin validate mods/agent-flow
mods/agent-flow/scripts/smoke.sh /tmp/agent-flow-smoke
```

Expected: every unit test passes; tsc prints nothing twice; validation passes; the smoke script prints nine `PASS` lines and exits 0. If `a nested Explore node` fails but everything else passes, open `/tmp/agent-flow-smoke/smoke.raw` through the same ANSI-stripping regex and read the tree: a general-purpose agent that answered without spawning is a model choice, not a defect; rerun the smoke test once before treating it as one.

The VS Code check (`/flow` in the extension prints the text tree) needs a person at the keyboard: leave it for Charles and say so in the final report.

- [ ] **Step 5: Mark the spec implemented and commit**

In `docs/superpowers/specs/2026-09-13-agent-flow-mod-design.md` change the status line `- 状态：待 Charles 审阅` to `- 状态：已批准；第一阶段已实现（见 mods/agent-flow）`.

```bash
git add mods/agent-flow docs/superpowers/specs/2026-09-13-agent-flow-mod-design.md
git commit -m "agent-flow: kit test, smoke test and README

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```
