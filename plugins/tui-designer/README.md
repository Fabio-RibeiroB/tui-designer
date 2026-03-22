# tui-designer plugin

Research-grounded TUI design skill for Claude Code. Transforms Claude into a senior TUI engineer that produces production-grade terminal interface designs, grounded in a comprehensive research report covering the full TUI ecosystem.

## Install

```shell
/plugin marketplace add Fabio-RibeiroB/tui-designer
/plugin install tui-designer@tui-designer
```

## Usage

Ask Claude to:
- "Design a TUI for a Kubernetes resource explorer"
- "Build a terminal UI for my log viewer"
- "Make this CLI feel like k9s"
- "Design a dashboard TUI in Go with Bubble Tea"

Claude produces a complete 8-section design document:

1. **UX Philosophy** — archetype, mental model, guiding principles
2. **Layout System** — ASCII diagram, panel hierarchy, focus flow
3. **Interaction Model** — keyboard-first design, full keymap table
4. **Visual Design System** — semantic colour tokens, accessibility (AA contrast)
5. **Component System** — reusable primitives with states and performance notes
6. **Performance Strategy** — rendering model, SSH/local/low-CPU presets
7. **Technology Recommendation** — justified stack (Rust/Go/Python/Node)
8. **Implementation Plan** — step-by-step build sequence and module structure

## What's bundled

```
skills/tui-designer/
  SKILL.md
  references/
    deep-research-report.md    # Full primary research
    library-comparison.md      # Framework trade-offs
    design-system.md           # Tokens, colour, accessibility
    interaction-patterns.md    # Components, layouts, keyboard
    performance-security.md    # Rendering, compatibility, CWE-150
```

## Reference TUIs

Designs are calibrated to match the quality and feel of:
- **lazygit** — panel-driven, keyboard-first, fast
- **k9s** — table-first data exploration, dense and navigable
- **htop** — process-level density, mouse-assisted
- **yazi** — async-first, modern visuals, preview system
- **neovim** — modal, composable, deeply configurable
