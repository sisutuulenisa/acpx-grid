# acpx-grid

Terminal-first live grid monitor for active ACPX sessions.

## Idea

`acpx-grid` is a standalone TypeScript/OpenTUI project for watching active ACPX sessions as a flexible full-screen pane grid:

- one pane per active session
- panes appear when sessions become active
- panes disappear after inactivity timeout
- grid repacks automatically
- thin process-info strip per pane
- designed for Ubuntu over SSH
- Ghostty/Omarchy-friendly, but not dependent on Ghostty
- tmux-friendly as an optional companion layer

## Status

Early planning/design phase.

Current docs:
- `docs/plans/2026-03-25-acpx-session-monitor.md`

## Goals

- ACPX session discovery across scopes
- live active-only dynamic grid
- process liveness enrichment
- terminal palette/theme token support
- optional Figma design assets for layout exploration
