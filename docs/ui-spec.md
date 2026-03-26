# acpx-grid UI Specification

> **Repo:** https://github.com/sisutuulenisa/acpx-grid
> **Product:** acpx-grid — universal ACPX worker/session monitor
> **Positioning:** AO-independent. Shows ACPX sessions started by AO, other orchestrators, or manual usage.

---

## Design checklist

- [x] Active-only dynamic grid behavior defined
- [x] Pane enter/exit behavior defined
- [x] Process-info strip placement defined
- [x] Omarchy-friendly theme tokens defined
- [x] SSH terminal constraints documented
- [x] Optional Ghostty and tmux companion scenarios documented
- [x] Grid repack algorithm defined
- [x] Keyboard navigation defined
- [x] State transitions documented

---

## 1. Core UX principle

**Wall of live workers.** The screen is a fullscreen dynamic grid of active ACPX session panes. No sidebars, no dashboards, no management chrome. Panes appear when sessions become active and disappear after an inactivity timeout. The grid auto-repacks to fill available space.

---

## 2. Architecture overview

```
┌─────────────────────────────────────────────────────────────┐
│                    ACPX SESSION SOURCES                     │
│                                                             │
│   ┌──────────┐   ┌──────────────┐   ┌───────────────────┐  │
│   │  Manual   │   │     AO       │   │ Other orchestrator│  │
│   │  acpx use │   │  orchestrated│   │   (future)        │  │
│   └─────┬─────┘   └──────┬───────┘   └────────┬──────────┘  │
│         │                │                     │             │
│         └────────────────┼─────────────────────┘             │
│                          │                                   │
│                    ┌─────▼─────┐                             │
│                    │ acpx CLI  │                             │
│                    │ sessions  │                             │
│                    │ list/show │                             │
│                    └─────┬─────┘                             │
└──────────────────────────┼──────────────────────────────────┘
                           │
                    ┌──────▼───────┐
                    │   Polling    │
                    │  Discovery   │
                    │  (2s cycle)  │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │   Session    │
                    │    Store     │
                    │              │
                    │ • snapshots  │
                    │ • history    │
                    │ • liveness   │
                    │ • idle age   │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │   OpenTUI    │
                    │  Dynamic     │
                    │   Grid       │
                    │              │
                    │ • auto-pack  │
                    │ • pane cards │
                    │ • info strip │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
        ┌─────▼─────┐ ┌───▼────┐ ┌────▼─────┐
        │ Any SSH   │ │ Ghostty│ │  tmux    │
        │ terminal  │ │(local  │ │(optional │
        │ (primary) │ │ only)  │ │companion)│
        └───────────┘ └────────┘ └──────────┘
```

### Component roles

| Component | Role | Required? |
|-----------|------|-----------|
| **acpx CLI** | Source of truth for session metadata and history | Yes |
| **Polling Discovery** | Scans configured roots + agents on interval | Yes |
| **Session Store** | Maintains snapshots, computes idle age, evicts stale | Yes |
| **OpenTUI Grid** | Renders fullscreen dynamic pane grid in terminal | Yes |
| **Any SSH terminal** | Primary runtime environment | Yes |
| **Ghostty** | Local terminal emulator for nicer desktop experience | No — client-side only |
| **tmux** | Remote persistence, session survival across disconnects | No — optional companion |

---

## 3. Main screen layout

### Fullscreen active-only grid (primary state: 4 active sessions)

```
┌─────────────────────────────────────────────────────────────────────┐
│ acpx-grid                                              4 active  q │
├────────────────────────────────┬────────────────────────────────────┤
│ pi ▸ feature-auth    pid:2341 │ pi ▸ fix-login       pid:2387     │
│ ~/projects/api    idle:12s  ● │ ~/projects/web    idle:3s   ●     │
│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │
│                                │                                   │
│ > Analyzing auth middleware... │ > Running test suite...            │
│ > Found 3 files to modify     │ > 47/52 tests passing             │
│ > Editing src/auth/jwt.ts     │ > Fixing assertion in login.test  │
│ > Writing unit tests...       │ > Re-running failed tests...      │
│ >                             │ >                                  │
│                                │                                   │
│                                │                                   │
│                                │                                   │
├────────────────────────────────┼────────────────────────────────────┤
│ codex ▸ refactor-db  pid:2402 │ pi ▸ docs-update     pid:2415     │
│ ~/projects/core   idle:45s  ● │ ~/projects/docs   idle:1s   ●     │
│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │
│                                │                                   │
│ > Mapping table relationships  │ > Generating API reference...     │
│ > Creating migration file...   │ > Processing 12 endpoints         │
│ > Testing rollback scenario    │ > Writing examples for /users     │
│ >                              │ > Formatting markdown output      │
│                                │ >                                  │
│                                │                                   │
│                                │                                   │
│                                │                                   │
├────────────────────────────────┴────────────────────────────────────┤
│ ↑↓←→ navigate  enter expand  r refresh  q quit                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Process-info strip (per pane, top position)

```
┌──────────────────────────────────────┐
│ {agent} ▸ {session}    pid:{pid}     │  ← line 1: identity
│ {cwd_tail}          idle:{age}  {●}  │  ← line 2: location + status
│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│  ← separator (dashed)
│                                      │
│ > {recent history lines...}          │  ← pane body: last N output lines
│                                      │
└──────────────────────────────────────┘
```

**Strip fields:**
- `agent`: ACPX agent name (e.g. `pi`, `codex`)
- `session`: session name
- `pid`: process ID
- `cwd_tail`: last 2 path segments of working directory
- `idle`: time since last activity (e.g. `3s`, `2m`, `1h`)
- `●`: liveness indicator (LIVE=green/●, STALE=yellow/◐, EXITED=red/○)

---

## 4. Grid repack behavior

### Layout algorithm

The grid uses a simple row-major flow layout:
1. Count active sessions: `n`
2. Compute columns: `cols = ceil(sqrt(n))` (biased toward wider-than-tall)
3. Compute rows: `rows = ceil(n / cols)`
4. Distribute terminal width/height evenly across cells
5. Last row may have fewer cells — remaining space is distributed equally

### Repack table

| Active sessions | Columns | Rows | Layout |
|----------------|---------|------|--------|
| 1 | 1 | 1 | Full screen |
| 2 | 2 | 1 | Side by side |
| 3 | 2 | 2 | 2+1 (last row centered or left-aligned) |
| 4 | 2 | 2 | 2×2 grid |
| 5–6 | 3 | 2 | 3×2 grid |
| 7–9 | 3 | 3 | 3×3 grid |
| 10–12 | 4 | 3 | 4×3 grid |
| 13–16 | 4 | 4 | 4×4 grid |

### Repack animation

When a session appears or disappears:
1. New grid dimensions are computed immediately
2. Panes reflow into new positions
3. No animation — instant repack (terminal limitation)
4. Newly appearing panes start with empty history that fills on next poll
5. Disappearing panes are simply removed from the grid

---

## 5. State transitions

### Session lifecycle

```
   [not discovered]
         │
         ▼
   ┌───────────┐
   │  ACTIVE   │ ← session found via acpx sessions list
   │   (●)     │    pid is live, recent history activity
   └─────┬─────┘
         │ no activity for >60s (configurable)
         ▼
   ┌───────────┐
   │   IDLE    │ ← still in session list, pid live
   │   (◐)     │    but no new history entries
   └─────┬─────┘
         │ idle > inactivityTimeout (default: 5min)
         ▼
   ┌───────────┐
   │  EVICTED  │ ← pane removed from grid
   │  (hidden) │    grid repacks
   └───────────┘

   At any point if pid dies:
   ┌───────────┐
   │  EXITED   │ ← pid no longer in /proc
   │   (○)     │    shown briefly, then evicted
   └─────┬─────┘
         │ after exitGracePeriod (default: 30s)
         ▼
   ┌───────────┐
   │  EVICTED  │
   └───────────┘
```

### Grid repack on session enter/exit

**State A: 4 active sessions (2×2 grid)**
```
┌──────────┬──────────┐
│ session1 │ session2 │
├──────────┼──────────┤
│ session3 │ session4 │
└──────────┴──────────┘
```

**State B: session3 becomes inactive → evicted → 3 sessions (2×2 grid, one empty)**
```
┌──────────┬──────────┐
│ session1 │ session2 │
├──────────┼──────────┘
│ session4 │
└──────────┘
```

**State C: new session5 appears → back to 4 sessions (2×2 grid)**
```
┌──────────┬──────────┐
│ session1 │ session2 │
├──────────┼──────────┤
│ session4 │ session5 │
└──────────┴──────────┘
```

**State D: 3 more sessions appear → 7 sessions (3×3 grid)**
```
┌────────┬────────┬────────┐
│ sess1  │ sess2  │ sess4  │
├────────┼────────┼────────┤
│ sess5  │ sess6  │ sess7  │
├────────┼────────┼────────┘
│ sess8  │
└────────┘
```

---

## 6. Keyboard navigation

| Key | Action |
|-----|--------|
| `↑` / `k` | Move selection up |
| `↓` / `j` | Move selection down |
| `←` / `h` | Move selection left |
| `→` / `l` | Move selection right |
| `Enter` | Expand selected pane to detail overlay |
| `Esc` | Close detail overlay / back to grid |
| `r` | Force refresh now |
| `q` | Quit |

### Selection behavior
- One pane is always selected (highlighted border)
- Arrow keys wrap within grid bounds (no wrap-around)
- When grid repacks, selection stays on same session if it still exists, otherwise moves to nearest neighbor

---

## 7. Detail overlay (expanded pane)

When pressing Enter on a selected pane:

```
┌─────────────────────────────────────────────────────────────────────┐
│ ◄ ESC                                                              │
├─────────────────────────────────────────────────────────────────────┤
│ Session: feature-auth                                              │
│ Agent:   pi                                                        │
│ PID:     2341                   Status: LIVE ●                     │
│ CWD:     /home/sisu/.openclaw/workspace/local/projects/api         │
│ Idle:    12 seconds                                                │
│ Started: 2026-03-26 12:04:31 UTC                                   │
├─────────────────────────────────────────────────────────────────────┤
│ Recent history (last 20 entries):                                   │
│                                                                     │
│ [12:04:45] > Analyzing auth middleware in src/auth/                 │
│ [12:04:47] > Found 3 files to modify: jwt.ts, session.ts, guard.ts │
│ [12:04:52] > Reading src/auth/jwt.ts (245 lines)                   │
│ [12:05:01] > Editing src/auth/jwt.ts: adding refresh token logic   │
│ [12:05:15] > Reading src/auth/session.ts (128 lines)               │
│ [12:05:22] > Editing src/auth/session.ts: session expiry handling  │
│ [12:05:30] > Writing unit tests for jwt refresh                    │
│ [12:05:45] > Running vitest src/auth/jwt.test.ts                   │
│ [12:05:48] > 4/4 tests passing                                     │
│ [12:05:50] > Writing unit tests for session expiry                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. Theme tokens (Omarchy-friendly)

The app uses semantic theme tokens mapped to terminal ANSI/indexed colors. No hardcoded RGB values.

| Token | Purpose | Default ANSI mapping |
|-------|---------|---------------------|
| `fg.primary` | Main text | Terminal default fg |
| `fg.secondary` | Dimmed text (idle age, hints) | ANSI dim / color 8 |
| `fg.accent` | Session name, agent name | ANSI cyan (6) |
| `bg.pane` | Pane background | Terminal default bg |
| `bg.selected` | Selected pane highlight | ANSI color 0 (bright) |
| `border.default` | Pane borders | ANSI dim white |
| `border.selected` | Selected pane border | ANSI bold cyan |
| `status.live` | Live indicator ● | ANSI green (2) |
| `status.idle` | Idle indicator ◐ | ANSI yellow (3) |
| `status.exited` | Exited indicator ○ | ANSI red (1) |
| `strip.bg` | Process-info strip background | Slightly different from pane bg |
| `strip.fg` | Process-info strip text | ANSI bold |

### Theme loading priority
1. User config file (`theme` section)
2. Environment variable `ACPX_GRID_THEME`
3. Built-in defaults (ANSI indexed colors above)

The app **never assumes RGB values**. It uses ANSI color indices so that the active terminal theme (Omarchy, Catppuccin, Dracula, etc.) determines the actual colors.

---

## 9. Terminal constraints (SSH-first)

- **Minimum terminal size:** 80×24 (graceful degradation below this)
- **Preferred size:** 120×40 or larger for multi-pane grids
- **No true-color assumption:** Use ANSI 16/256 indexed colors only
- **No mouse dependency:** Full keyboard navigation
- **No Ghostty-specific escape sequences:** Standard VT100/xterm compatible
- **Unicode box-drawing:** Use standard box-drawing characters (supported by virtually all modern terminals)
- **Refresh rate:** Visual updates on poll cycle (default 2s), not continuous animation

---

## 10. Optional companion layers

### Ghostty (local terminal emulator)
- When running locally (not over SSH), Ghostty provides a nicer desktop experience
- Ghostty's own window/tab management can complement the in-app grid
- No Ghostty APIs are called by acpx-grid
- The app renders identically in Ghostty and any other terminal

### tmux (remote persistence)
- tmux can wrap acpx-grid for session persistence across SSH disconnects
- `tmux new-session -s acpx-grid 'npx acpx-grid'`
- The app's own grid is independent of tmux panes
- tmux is never a hard dependency — acpx-grid works without it
- Power users may use tmux alongside acpx-grid for additional terminal multiplexing

---

## 11. Empty states

### No active sessions
```
┌─────────────────────────────────────────────────────────────────────┐
│ acpx-grid                                              0 active  q │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                                                                     │
│                                                                     │
│                    No active ACPX sessions                          │
│                                                                     │
│              Watching: ~/projects, ~/.worktrees                     │
│              Agents: pi, codex                                      │
│              Polling every 2s...                                    │
│                                                                     │
│                                                                     │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│ r refresh  q quit                                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### Single active session (fullscreen pane)
```
┌─────────────────────────────────────────────────────────────────────┐
│ acpx-grid                                              1 active  q │
├─────────────────────────────────────────────────────────────────────┤
│ pi ▸ feature-auth                               pid:2341           │
│ ~/projects/api                               idle:3s   ●          │
│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │
│                                                                     │
│ > Analyzing auth middleware in src/auth/                            │
│ > Found 3 files to modify: jwt.ts, session.ts, guard.ts            │
│ > Reading src/auth/jwt.ts (245 lines)                              │
│ > Editing src/auth/jwt.ts: adding refresh token logic              │
│ > Writing unit tests for jwt refresh                                │
│ > Running vitest src/auth/jwt.test.ts                              │
│ > 4/4 tests passing                                                │
│                                                                     │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│ enter expand  r refresh  q quit                                     │
└─────────────────────────────────────────────────────────────────────┘
```
