# terminalui-design

A Claude Code plugin marketplace for designing and building professional Terminal UIs.

## Overview

This marketplace distributes the **TUI Builder** skill — a research-grounded Claude Code plugin that transforms Claude into a senior TUI engineer. Every design decision is grounded in `deep-research-report.md`, a comprehensive research document covering the full TUI ecosystem.

The skill produces designs that feel like:
- **lazygit** — panel-driven, keyboard-first, fast
- **k9s** — table-first data exploration, dense and navigable
- **htop** — process-level density, mouse-assisted
- **neovim** — modal, composable, deeply configurable

## Install

```shell
# Add this marketplace
/plugin marketplace add fabio/terminalui-design

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
1. UX Philosophy (archetype + mental model)
2. Layout System (ASCII diagram + panel hierarchy)
3. Interaction Model (keyboard-first, full keymap table)
4. Visual Design System (semantic tokens, colours, accessibility)
5. Component System (reusable primitives with states + performance notes)
6. Performance Strategy (rendering model + preset selection)
7. Technology Recommendation (justified stack choice)
8. Implementation Plan (step-by-step build plan + module structure)

## Research Foundation

The `deep-research-report.md` at the root of this repository is the primary source of truth. It covers:
- TUI ecosystem and library landscape (C/Rust/Go/Python/Node)
- Interaction design patterns and component catalogue
- Visual design, colour, typography, and accessibility
- Performance, rendering, and responsiveness
- Compatibility (SSH, Windows, web terminals, CI)
- Security (terminal escape injection, CWE-150)

## Marketplace Structure

```
terminalui-design/
  .claude-plugin/
    marketplace.json          # Marketplace catalog
  plugins/
    terminalui-design/
      .claude-plugin/
        plugin.json           # Plugin manifest
      skills/
        tui-builder/
          SKILL.md            # Core skill (1800 words, lean)
          references/
            library-comparison.md     # Framework trade-offs
            design-system.md          # Tokens, colour, accessibility
            interaction-patterns.md   # Components, layouts, keyboard
            performance-security.md   # Rendering, compatibility, CWE-150
  deep-research-report.md     # Primary research source
  README.md
```

## Local Testing

```shell
/plugin marketplace add ./terminalui-design
/plugin install terminalui-design@terminalui-design
```

Then ask: *"Design a TUI for a Kubernetes resource explorer"*
