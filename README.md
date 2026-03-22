# terminalui-design

[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-Plugin%20Marketplace-8A2BE2?logo=anthropic&logoColor=white)](https://code.claude.com)
[![Version](https://img.shields.io/badge/version-0.1.0-blue)](https://github.com/Fabio-RibeiroB/terminalui-design/releases)
[![License](https://img.shields.io/github/license/Fabio-RibeiroB/terminalui-design)](LICENSE)

A Claude Code plugin marketplace for designing and building professional Terminal UIs.

## Overview

This marketplace distributes the **TUI Builder** skill — a research-grounded Claude Code plugin that transforms Claude into a senior TUI engineer. Every design decision is grounded in a comprehensive research document covering the full TUI ecosystem, bundled directly with the skill.

The skill produces designs that feel like:
- **lazygit** — panel-driven, keyboard-first, fast
- **k9s** — table-first data exploration, dense and navigable
- **htop** — process-level density, mouse-assisted
- **neovim** — modal, composable, deeply configurable

## Install

```shell
# Add this marketplace
/plugin marketplace add Fabio-RibeiroB/terminalui-design

# Install the plugin
/plugin install terminalui-design@terminalui-design
```

## What's included

### TUI Builder Skill

Triggered when you ask to:
- "Design a TUI for X"
- "Build a terminal UI for my tool"
- "Make this CLI feel like a modern dev tool"
- "Design a dashboard like k9s"

Produces a complete, research-backed design in 8 sections:

| # | Section | What you get |
|---|---------|-------------|
| 1 | UX Philosophy | Archetype, mental model, guiding principles |
| 2 | Layout System | ASCII diagram, panel hierarchy, focus flow |
| 3 | Interaction Model | Keyboard-first design, full keymap table |
| 4 | Visual Design System | Semantic colour tokens, accessibility (AA contrast) |
| 5 | Component System | Reusable primitives with states and performance notes |
| 6 | Performance Strategy | Rendering model, SSH/local/low-CPU presets |
| 7 | Technology Recommendation | Justified stack (Rust/Go/Python/Node) |
| 8 | Implementation Plan | Step-by-step build sequence and module structure |

## Research Foundation

The skill bundles a comprehensive research report covering:

- **Ecosystem** — Ratatui, Bubble Tea, Textual, Ink, ncurses trade-offs
- **Interaction design** — component catalogue, layout archetypes, keyboard patterns
- **Visual design** — semantic token system, colour, typography, accessibility (WCAG AA)
- **Performance** — diff-based rendering, TPS/FPS decoupling, SSH-safe presets
- **Compatibility** — SSH, Windows/ConPTY, web terminals (xterm.js), CI
- **Security** — terminal escape injection (CWE-150), sanitisation patterns

## Marketplace Structure

```
terminalui-design/
  .claude-plugin/
    marketplace.json              # Marketplace catalog
  plugins/
    terminalui-design/
      .claude-plugin/
        plugin.json               # Plugin manifest
      skills/
        tui-builder/
          SKILL.md                # Core skill — lean trigger + 8-section output spec
          references/
            deep-research-report.md       # Full bundled research (primary source)
            library-comparison.md         # Framework trade-offs and selection guide
            design-system.md              # Tokens, colour palettes, accessibility
            interaction-patterns.md       # Components, layouts, keyboard patterns
            performance-security.md       # Rendering, compatibility, CWE-150
  README.md
```

## Local Testing

```shell
/plugin marketplace add ./
/plugin install terminalui-design@terminalui-design
```

Then ask: *"Design a TUI for a Kubernetes resource explorer"*
