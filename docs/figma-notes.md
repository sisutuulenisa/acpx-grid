# acpx-grid Design Notes

> **Repo:** https://github.com/sisutuulenisa/acpx-grid
> **Product:** acpx-grid — universal ACPX worker/session monitor
> **Date:** 2026-03-26

---

## Figma status

**Blocker:** No Figma MCP tools are available in the Claude Code CLI environment. The only MCP servers connected are WordPress (roxtar) endpoints, which are unrelated to Figma. Figma diagrams and mockups could not be created programmatically.

**Fallback:** All design specifications, architecture diagrams, UI mockups, and state transitions are documented as ASCII art and structured specifications in `docs/ui-spec.md`. These serve as a complete Figma brief for manual creation if desired.

---

## Layout rules

### Grid auto-packing
- Uses row-major flow: `cols = ceil(sqrt(n))`, `rows = ceil(n / cols)`
- Biased toward wider-than-tall layouts (more columns than rows)
- Terminal width/height distributed evenly across cells
- Last row may have fewer cells; remaining space distributed equally
- Instant repack on session enter/exit (no animation in terminal)

### Pane sizing
- All panes in a given grid layout are equal size
- Minimum useful pane size: ~35 cols × 8 rows (for info strip + 5 history lines)
- When terminal is too small to fit all panes at minimum size, reduce history lines first, then switch to a scrollable list fallback
- Single session gets full screen

### Process-info strip
- Position: **top of each pane** (2 lines + dashed separator)
- Line 1: `{agent} ▸ {session_name}    pid:{pid}`
- Line 2: `{cwd_tail}    idle:{age}  {status_indicator}`
- Strip uses slightly differentiated background (terminal reverse or dim)

---

## Pane behavior

### Inactivity eviction
- Default inactivity timeout: **5 minutes** (configurable)
- After timeout with no new history entries → pane removed from grid
- Grid repacks immediately
- If session becomes active again → pane reappears, grid repacks

### Exit grace period
- When a session's pid exits → pane shows EXITED (○) status
- Pane remains visible for **30 seconds** (configurable) as grace period
- After grace period → pane evicted, grid repacks
- This allows the operator to notice the exit and check final output

### Session ordering
- Panes ordered by: most recently active first
- New sessions appear at the end of the current grid
- After repack, order is preserved

---

## Theme/token decisions

### Approach: ANSI-indexed colors only
- No hardcoded RGB values anywhere in the codebase
- All colors use ANSI 16-color or 256-color indices
- The terminal's active theme (Omarchy, Catppuccin, etc.) determines actual rendered colors
- This makes acpx-grid automatically theme-compatible with any terminal color scheme

### Semantic tokens
See `docs/ui-spec.md` section 8 for the full token table. Key decisions:
- Status indicators use semantic ANSI colors (green=live, yellow=idle, red=exited)
- Selected pane uses bold cyan border (visible in both light and dark themes)
- Text hierarchy through bold/dim attributes rather than color alone

---

## Intentionally deferred

These features are **not** in the MVP scope:

| Feature | Reason |
|---------|--------|
| Raw PTY mirroring | Complex; polling history is sufficient for monitoring |
| AO-core integration | acpx-grid is AO-independent by design |
| Mouse support | SSH-first; keyboard navigation is primary |
| True-color (24-bit) themes | Not reliably available over SSH |
| tmux pane orchestration | tmux is a companion, not a dependency |
| Multi-user auth | Single-operator tool |
| Export/screenshot | Terminal-native; no export needed for MVP |
| Configurable grid layouts | Auto-packing is sufficient; manual layouts add complexity |
| Pane resize handles | Equal-size panes simplify the grid; resize is a future enhancement |

---

## Runtime scenarios

### Primary: Ubuntu server over SSH
- acpx-grid runs directly in the SSH terminal session
- Any terminal emulator on the client side works
- Polls acpx CLI locally on the server
- Standard VT100/xterm escape sequences only

### Optional: Local desktop with Ghostty
- Ghostty provides nicer font rendering and window management
- acpx-grid renders identically; no Ghostty APIs used
- Ghostty's native tabs/splits can complement the in-app grid

### Optional: tmux persistence
- `tmux new-session -s acpx-grid 'npx acpx-grid'`
- Survives SSH disconnects
- acpx-grid's internal grid is independent of tmux's pane layout
- Recommended for long-running monitoring on remote servers
