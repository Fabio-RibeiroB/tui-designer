# tui-designer

[![Claude Code Plugin](https://img.shields.io/badge/Claude%20Code-Plugin%20Marketplace-8A2BE2?logo=anthropic&logoColor=white)](https://code.claude.com)
[![Version](https://img.shields.io/badge/version-0.1.0-blue)](https://github.com/Fabio-RibeiroB/tui-designer/releases)
[![License](https://img.shields.io/github/license/Fabio-RibeiroB/tui-designer)](LICENSE)

A Claude Code plugin for designing Terminal UIs.

## Install

Run both commands in Claude Code:

```shell
/plugin marketplace add Fabio-RibeiroB/tui-designer
/plugin install tui-designer@tui-designer
```

## Usage

Once installed, ask Claude to design a TUI:

- "Design a TUI for X"
- "Build a terminal UI for my tool"
- "Make this CLI feel like a modern dev tool"
- "Design a dashboard like k9s"

Claude produces a design covering layout, keyboard interactions, components, colour system, tech stack recommendation, and an implementation plan.

## What's included

### TUI Builder Skill

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

## Structure

```
tui-designer/
  .claude-plugin/
    marketplace.json
  plugins/
    tui-designer/
      .claude-plugin/
        plugin.json
      skills/
        tui-builder/
          SKILL.md
          references/
            deep-research-report.md
            library-comparison.md
            design-system.md
            interaction-patterns.md
            performance-security.md
```

## Troubleshooting

**Plugin fails to install after adding the marketplace**

If you added the marketplace before v0.1.1, run an update first:

```shell
/plugin marketplace update tui-designer
/plugin install tui-designer@tui-designer
```

**Local testing**

```shell
/plugin marketplace add ./
/plugin install tui-designer@tui-designer
```
