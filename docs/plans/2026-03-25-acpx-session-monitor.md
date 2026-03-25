# ACPX Session Monitor Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a small standalone TypeScript terminal app that monitors active ACPX sessions, shows them in a live grid, and lets the operator inspect recent output/history for each session over SSH.

**Architecture:** The app runs entirely in the terminal on the remote Ubuntu host and polls `acpx` session metadata/history on an interval. The TUI should not depend on Ghostty features; Ghostty is only an optional local terminal emulator for nicer desktop-side window/grid usage. Use OpenTUI as the primary UI layer, with a clean abstraction around the ACPX CLI so the same monitor can later support additional data sources (AO session metadata, process stats, logs).

**Tech Stack:** TypeScript, Node.js, OpenTUI, ACPX CLI, execa (or child_process wrapper), zod, vitest, tsx

**Theme Requirement:** Support Omarchy-friendly theming by preferring terminal palette/indexed colors and configurable theme tokens instead of hardcoded RGB assumptions. Ghostty may provide the terminal theme locally, but the app must still render correctly in any SSH terminal.

---

## Constraints and product decisions

- This lives **outside AO core**.
- Working path: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor`
- Primary scenario: running on a remote Ubuntu server via SSH.
- Ghostty support is **client-side optional only**. Do not make the monitor require Ghostty-specific APIs.
- The first version should prefer **polling ACPX session data** over raw PTY mirroring.
- The first version should show **recent history/output** in a dynamic pane grid, not try to implement a full terminal multiplexer.
- The primary UX is **active-only**: panes appear when ACPX sessions become active and disappear after an inactivity timeout.
- Scope session identity by:
  - agent (`pi`, `codex`, etc.)
  - cwd
  - session name
- Design for a fast MVP: useful first, pretty second.

## Repository skeleton

### Task 0: Draw the interaction model and UI mockup in Figma

**Files:**
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/docs/figma-notes.md`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/docs/ui-spec.md`

**Step 1: Write the failing design checklist**

Create `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/docs/ui-spec.md` with unchecked items for:
- active-only dynamic grid behavior
- pane enter/exit behavior
- process-info strip placement
- Omarchy-friendly theme tokens
- SSH terminal constraints
- optional Ghostty and tmux companion scenarios

**Step 2: Draw before coding**

Using the available Figma MCP through Claude Code CLI, create:
- a simple architecture diagram showing ACPX -> polling store -> OpenTUI grid
- one main mockup of the full-screen active pane grid
- one state showing pane appearance/disappearance and auto-repack
- one detail for the thin process-info strip above or below each pane

Keep the mockup low-fidelity but structurally precise. Prioritize layout behavior over visual polish.

**Step 3: Write the design notes down locally**

Capture in `docs/figma-notes.md`:
- Figma file/link details
- chosen layout rules
- pane sizing behavior
- inactivity eviction behavior
- theme/token decisions
- what is intentionally deferred

**Step 4: Commit**

```bash
git add docs/figma-notes.md docs/ui-spec.md
 git commit -m "docs: define figma mockup and ui behavior"
```

### Task 1: Create standalone project scaffold

**Files:**
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/package.json`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/tsconfig.json`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/.gitignore`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/README.md`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/index.ts`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/app.ts`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/config.ts`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/vitest.config.ts`

**Step 1: Write the failing test**

Create `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/config.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { getDefaultConfig } from './config'

describe('getDefaultConfig', () => {
  it('returns SSH-safe defaults', () => {
    const cfg = getDefaultConfig()
    expect(cfg.refreshMs).toBe(2000)
    expect(cfg.historyLimit).toBe(20)
    expect(cfg.useGhostty).toBe(false)
  })
})
```

**Step 2: Run test to verify it fails**

Run:

```bash
cd /home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor
pnpm vitest run src/config.test.ts
```

Expected: FAIL because config module does not exist yet.

**Step 3: Write minimal implementation**

Create minimal project scaffold with:
- `package.json` scripts: `dev`, `build`, `test`, `typecheck`
- `config.ts` exporting `getDefaultConfig()`
- `index.ts` calling `runApp()`
- `app.ts` with placeholder render boot

Minimal config shape:

```ts
export type MonitorConfig = {
  refreshMs: number
  historyLimit: number
  useGhostty: boolean
}

export function getDefaultConfig(): MonitorConfig {
  return {
    refreshMs: 2000,
    historyLimit: 20,
    useGhostty: false,
  }
}
```

**Step 4: Run test to verify it passes**

Run:

```bash
pnpm vitest run src/config.test.ts
```

Expected: PASS

**Step 5: Commit**

```bash
git add .
git commit -m "chore: scaffold acpx session monitor"
```

### Task 2: Add ACPX CLI adapter with typed parsing

**Files:**
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/acpx/client.ts`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/acpx/types.ts`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/acpx/parse.ts`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/acpx/client.test.ts`

**Step 1: Write the failing test**

```ts
import { describe, expect, it } from 'vitest'
import { parseSessionShow } from './parse'

describe('parseSessionShow', () => {
  it('parses acpx sessions show text output', () => {
    const result = parseSessionShow(`id: 123\nname: demo\ncwd: /tmp\npid: 42\nhistoryEntries: 2`)
    expect(result.name).toBe('demo')
    expect(result.cwd).toBe('/tmp')
    expect(result.pid).toBe(42)
  })
})
```

**Step 2: Run test to verify it fails**

Run:

```bash
pnpm vitest run src/acpx/client.test.ts
```

Expected: FAIL because parser/client files do not exist.

**Step 3: Write minimal implementation**

Implement:
- typed `AcpxSessionSummary`
- `parseSessionShow(text)` for `key: value` output
- `parseSessionHistory(text)` for history lines
- `runAcpx(args: string[], cwd?: string)` wrapper using `execa`
- `showSession(agent, name, cwd)`
- `historySession(agent, name, cwd, limit)`

Do not overbuild yet. Keep parsing text-first; JSON can be added later if ACPX grows a stronger machine format.

**Step 4: Run test to verify it passes**

Run:

```bash
pnpm vitest run src/acpx/client.test.ts
```

Expected: PASS

**Step 5: Commit**

```bash
git add src/acpx
 git commit -m "feat: add typed acpx cli adapter"
```

### Task 3: Support session discovery across scopes

**Files:**
- Modify: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/acpx/client.ts`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/discovery.ts`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/discovery.test.ts`

**Step 1: Write the failing test**

```ts
import { describe, expect, it } from 'vitest'
import { buildScopeKey } from './discovery'

describe('buildScopeKey', () => {
  it('uses agent cwd and name', () => {
    expect(buildScopeKey({ agent: 'pi', cwd: '/repo', name: 'demo' }))
      .toBe('pi::/repo::demo')
  })
})
```

**Step 2: Run test to verify it fails**

Run:

```bash
pnpm vitest run src/discovery.test.ts
```

Expected: FAIL because discovery module does not exist.

**Step 3: Write minimal implementation**

Implement discovery rules:
- maintain a configured list of watched cwd roots
- for each root + agent, call `acpx <agent> sessions list`
- normalize sessions into a unique scope key
- include lightweight metadata:
  - agent
  - cwd
  - name
  - lastActivity
  - pid if available

Start with static config for watched roots, e.g.:
- `/home/sisu/.worktrees`
- `/home/sisu/.openclaw/workspace/local/projects`

**Step 4: Run test to verify it passes**

Run:

```bash
pnpm vitest run src/discovery.test.ts
```

Expected: PASS

**Step 5: Commit**

```bash
git add src/discovery*
git commit -m "feat: discover acpx sessions across configured scopes"
```

### Task 4: Build a polling session store

**Files:**
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/store/session-store.ts`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/store/session-store.test.ts`
- Modify: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/app.ts`

**Step 1: Write the failing test**

```ts
import { describe, expect, it } from 'vitest'
import { createSessionStore } from './session-store'

describe('createSessionStore', () => {
  it('keeps the latest session snapshot', async () => {
    const store = createSessionStore()
    store.setSessions([{ key: 'pi::/tmp::demo', name: 'demo' }])
    expect(store.getSessions()).toHaveLength(1)
  })
})
```

**Step 2: Run test to verify it fails**

Run:

```bash
pnpm vitest run src/store/session-store.test.ts
```

Expected: FAIL because store does not exist.

**Step 3: Write minimal implementation**

Implement a polling store that:
- refreshes on interval
- keeps latest sessions
- keeps per-session recent history buffer
- computes active/inactive state based on recent activity
- evicts inactive panes after a configurable timeout
- marks sessions stale if polling fails
- emits updates to the UI

Expose methods:
- `start()`
- `stop()`
- `subscribe(listener)`
- `getSnapshot()`

**Step 4: Run test to verify it passes**

Run:

```bash
pnpm vitest run src/store/session-store.test.ts
```

Expected: PASS

**Step 5: Commit**

```bash
git add src/store src/app.ts
git commit -m "feat: add polling session store"
```

### Task 5: Build the first OpenTUI screen

**Files:**
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/ui/root.ts`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/ui/session-card.ts`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/ui/root.test.ts`
- Modify: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/app.ts`

**Step 1: Write the failing test**

```ts
import { describe, expect, it } from 'vitest'
import { formatSessionTitle } from './root'

describe('formatSessionTitle', () => {
  it('shows agent name and session name', () => {
    expect(formatSessionTitle({ agent: 'pi', name: 'demo' })).toContain('pi')
    expect(formatSessionTitle({ agent: 'pi', name: 'demo' })).toContain('demo')
  })
})
```

**Step 2: Run test to verify it fails**

Run:

```bash
pnpm vitest run src/ui/root.test.ts
```

Expected: FAIL because UI helpers do not exist.

**Step 3: Write minimal implementation**

Build the first screen with:
- a flexible full-screen grid of active session panes
- panes auto-flow/repack as sessions appear and disappear
- each pane shows:
  - a thin process-info strip above or below the pane
  - session name
  - agent
  - cwd tail
  - pid
  - last activity / idle age
  - last 5–10 history lines
- no heavy chrome; keep the screen mostly grid
- minimal footer with key help only if it does not fight the grid

Do not attempt split panes or advanced non-grid sidebars yet. Focus on a readable SSH-safe dynamic grid.

**Step 4: Run test to verify it passes**

Run:

```bash
pnpm vitest run src/ui/root.test.ts
```

Expected: PASS

**Step 5: Commit**

```bash
git add src/ui src/app.ts
git commit -m "feat: add initial opentui session grid"
```

### Task 6: Add keyboard navigation and session focus

**Files:**
- Modify: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/ui/root.ts`
- Modify: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/ui/session-card.ts`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/ui/navigation.test.ts`

**Step 1: Write the failing test**

```ts
import { describe, expect, it } from 'vitest'
import { nextIndex } from './root'

describe('nextIndex', () => {
  it('moves right within bounds', () => {
    expect(nextIndex({ current: 0, direction: 'right', count: 4, columns: 2 })).toBe(1)
  })
})
```

**Step 2: Run test to verify it fails**

Run:

```bash
pnpm vitest run src/ui/navigation.test.ts
```

Expected: FAIL because helper does not exist.

**Step 3: Write minimal implementation**

Add keys:
- arrow keys / hjkl to move selected card
- Enter to open expanded detail overlay
- `r` to refresh now
- `q` to quit

Expanded detail view should show:
- full cwd
- full history window
- latest activity timestamps

**Step 4: Run test to verify it passes**

Run:

```bash
pnpm vitest run src/ui/navigation.test.ts
```

Expected: PASS

**Step 5: Commit**

```bash
git add src/ui
git commit -m "feat: add keyboard navigation and detail overlay"
```

### Task 7: Add process liveness enrichment

**Files:**
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/system/process.ts`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/system/process.test.ts`
- Modify: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/store/session-store.ts`

**Step 1: Write the failing test**

```ts
import { describe, expect, it } from 'vitest'
import { isPidLive } from './process'

describe('isPidLive', () => {
  it('returns false for clearly invalid pid', async () => {
    await expect(isPidLive(-1)).resolves.toBe(false)
  })
})
```

**Step 2: Run test to verify it fails**

Run:

```bash
pnpm vitest run src/system/process.test.ts
```

Expected: FAIL because process helper does not exist.

**Step 3: Write minimal implementation**

Implement:
- `isPidLive(pid)`
- optional `/proc/<pid>/stat` lookup
- optional command/cgroup capture for detail view

UI badge examples:
- `LIVE`
- `STALE`
- `EXITED`

This is important because `sessions show` can retain metadata after the process is gone.

**Step 4: Run test to verify it passes**

Run:

```bash
pnpm vitest run src/system/process.test.ts
```

Expected: PASS

**Step 5: Commit**

```bash
git add src/system src/store/session-store.ts
git commit -m "feat: enrich session cards with process liveness"
```

### Task 8: Add ghostty compatibility notes and Omarchy theme support strategy without coupling runtime to Ghostty

**Files:**
- Modify: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/README.md`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/docs/ghostty-notes.md`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/docs/runtime-scenarios.md`

**Step 1: Write the failing test**

Create a doc assertion test in `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/docs.test.ts`:

```ts
import { describe, expect, it } from 'vitest'
import { readFileSync } from 'node:fs'

describe('runtime docs', () => {
  it('documents that ghostty is optional', () => {
    const readme = readFileSync('README.md', 'utf8')
    expect(readme).toContain('Ghostty is optional')
  })
})
```

**Step 2: Run test to verify it fails**

Run:

```bash
pnpm vitest run src/docs.test.ts
```

Expected: FAIL because note is missing.

**Step 3: Write minimal implementation**

Document clearly:
- Ghostty is useful locally for arranging windows in a grid.
- Omarchy theme support should be implemented through app theme tokens that map cleanly onto terminal palettes, especially ANSI/indexed colors, instead of assuming a fixed RGB theme.
- Over SSH on Ubuntu server, the monitor still works in any terminal.
- The app itself owns the in-terminal grid; Ghostty does not.
- If later desired, Ghostty launch workflows can open multiple monitor instances or external logs, but that is a separate enhancement.

**Step 4: Run test to verify it passes**

Run:

```bash
pnpm vitest run src/docs.test.ts
```

Expected: PASS

**Step 5: Commit**

```bash
git add README.md docs src/docs.test.ts
git commit -m "docs: clarify ghostty and ssh runtime scenarios"
```

### Task 9: Add a sample config for watched roots and agents

**Files:**
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/config.monitor.example.json`
- Modify: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/config.ts`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/src/config-file.test.ts`

**Step 1: Write the failing test**

```ts
import { describe, expect, it } from 'vitest'
import { loadConfigFile } from './config'

describe('loadConfigFile', () => {
  it('loads watched agents and roots', async () => {
    const cfg = await loadConfigFile('config.monitor.example.json')
    expect(cfg.agents).toContain('pi')
    expect(cfg.watchRoots.length).toBeGreaterThan(0)
  })
})
```

**Step 2: Run test to verify it fails**

Run:

```bash
pnpm vitest run src/config-file.test.ts
```

Expected: FAIL because file loader/config file do not exist.

**Step 3: Write minimal implementation**

Example config should include:

```json
{
  "agents": ["pi", "codex"],
  "watchRoots": [
    "/home/sisu/.worktrees",
    "/home/sisu/.openclaw/workspace/local/projects"
  ],
  "refreshMs": 2000,
  "historyLimit": 20
}
```

**Step 4: Run test to verify it passes**

Run:

```bash
pnpm vitest run src/config-file.test.ts
```

Expected: PASS

**Step 5: Commit**

```bash
git add config.monitor.example.json src/config*
git commit -m "feat: add file-based monitor configuration"
```

### Task 10: Verify end-to-end on the real host

**Files:**
- Modify: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/README.md`
- Create: `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/docs/verification.md`

**Step 1: Write the failing verification checklist**

Create `/home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor/docs/verification.md` with unchecked items for:
- launch on Ubuntu over SSH
- detect named `pi` session
- render grid with multiple sessions
- show history lines
- mark dead pid as stale
- run without Ghostty
- optional desktop run inside Ghostty

**Step 2: Run manual verification to produce evidence**

Run:

```bash
cd /home/sisu/.openclaw/workspace/local/projects/tools/acpx-session-monitor
pnpm install
pnpm typecheck
pnpm test
pnpm build
pnpm dev
```

Expected:
- tests pass
- app opens in terminal
- at least one real ACPX session appears

**Step 3: Capture real-world findings**

Document:
- what worked on SSH
- whether OpenTUI behaved correctly in the server terminal
- whether polling intervals feel acceptable
- any ACPX command bottlenecks
- whether Ghostty adds value only locally

**Step 4: Commit**

```bash
git add README.md docs/verification.md
git commit -m "docs: add real-host verification notes"
```

## Architecture notes for implementation

### Data source hierarchy

Use this order:
1. ACPX session list/show/history as source of truth
2. Process liveness from pid as validation layer
3. Optional AO metadata later, not in MVP

### Why OpenTUI over Ghostty-specific behavior

- Ghostty is not reliably available on the remote Ubuntu server over SSH.
- The monitor must render inside any terminal.
- OpenTUI gives the app its own layout/grid logic.
- Ghostty can still be useful on the laptop as the terminal emulator hosting the app.
- tmux is a strong optional companion for saved screen setups, remote persistence, and operator workflows, but it should not replace the app's own primary rendering model.

### Explicit non-goals for MVP

Do **not** implement yet:
- raw terminal stream mirroring per ACPX process
- true PTY attachment to agent subprocesses
- direct AO-core integration
- multi-user auth or remote API server
- screenshot/export features
- tmux-driven pane orchestration as a hard requirement for core functionality

### MVP success criteria

The MVP is successful when:
- it runs over SSH in a plain Ubuntu terminal
- it discovers ACPX sessions by agent/cwd/name
- it renders a flexible active-only pane grid that repacks as sessions start/stop
- each pane includes a lightweight process-info strip
- it hides inactive panes after a timeout
- it can highlight stale/exited processes
- it remains useful without Ghostty

Plan complete and saved to `docs/plans/2026-03-25-acpx-session-monitor.md`. Two execution options:

**1. Subagent-Driven (this session)** - I dispatch fresh subagent per task, review between tasks, fast iteration

**2. Parallel Session (separate)** - Open new session with executing-plans, batch execution with checkpoints

Which approach?
