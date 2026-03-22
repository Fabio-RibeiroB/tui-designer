---
name: tui-designer
description: >
  This skill should be used when the user asks to "design a TUI", "build a terminal UI",
  "create a TUI for X", "improve this CLI into a TUI", "make this interface feel like a modern
  dev tool", "design a dashboard like k9s", "build something like lazygit", "make a TUI like htop",
  or describes wanting a fast, keyboard-first, professional terminal interface. Apply this skill
  whenever the user wants to design, plan, or build a Terminal User Interface at any fidelity —
  from rough layout to full implementation plan.

  <example>
  Context: User wants to build a terminal UI for their tool
  user: "Design a TUI for a Kubernetes resource explorer"
  assistant: "I'll apply the TUI Builder skill to design a production-grade k9s-style TUI."
  <commentary>
  User is asking for a TUI design. Use TUI Builder to produce all 8 sections: UX philosophy,
  layout, interaction model, design system, components, performance, tech recommendation, and
  implementation plan — all grounded in the research report.
  </commentary>
  </example>

  <example>
  Context: User wants to modernise an existing CLI
  user: "Make my log viewer CLI feel like a modern dev tool"
  assistant: "I'll use the TUI Builder skill to redesign it as a professional terminal UI."
  <commentary>
  User wants to upgrade a CLI to a full TUI. Apply TUI Builder to produce a complete design
  with keyboard-first interaction, semantic colour system, and LogView component specification.
  </commentary>
  </example>
version: 0.1.0
---

# TUI Builder

Design and build production-grade Terminal UIs grounded in research-backed principles. This skill operates as a senior TUI engineer + product designer: decisive, opinionated, and optimised for speed, clarity, keyboard efficiency, and low cognitive load.

The primary research source is `references/deep-research-report.md` (bundled with this skill). All design decisions must trace back to its principles. Consult the specialised reference files in `references/` for domain-specific detail.

---

## Strict Knowledge Rule

Extract principles, patterns, and heuristics from the research before designing. Apply them through decisions, not by citing text. When something falls outside the research, use best judgment and mark it clearly as an extension.

Reference files to consult:
- **`references/deep-research-report.md`** — Full primary research (ecosystem, design, security, performance)
- **`references/library-comparison.md`** — Framework trade-offs (Rust/Go/Python/Node), selection criteria, comparison table
- **`references/design-system.md`** — Color tokens, accessibility, typography, icon system
- **`references/interaction-patterns.md`** — Keyboard-first design, keymaps, components, layout patterns
- **`references/performance-security.md`** — Rendering models, performance presets, terminal compatibility, security

---

## Clarifying Questions (Optional)

If the request is underspecified, ask 1–2 sharp questions before proceeding:

- What is the **primary data** the user will explore? (processes, logs, files, metrics, git objects…)
- What is the **target environment**? (local terminal, SSH, Windows/WSL, CI, web terminal)

Skip questions if the request is already clear enough to produce a useful design.

---

## Output Structure

Always produce all 8 sections below. Be decisive. Avoid hedging.

---

### 1. UX Philosophy

State the TUI archetype and the mental model it imposes on the user.

Archetypes (choose one):
- **Explorer** — Navigate and inspect hierarchical data (lazygit, ranger, yazi)
- **Dashboard** — Monitor live data streams (k9s, htop, bottom)
- **Editor** — Edit structured documents or configuration (neovim, helix)
- **Pipeline** — Execute multi-step workflows with status tracking (CI runners, deploy tools)
- **REPL+** — Enhanced interactive shell (ptpython, posting)

State which research principles are driving the design. Examples:
- "Two-pane workbench layout because navigational TUIs benefit from stable list + detail separation"
- "MVU architecture because it enforces predictable state flow and enables testability"
- "Event-driven rendering at 10 TPS because this targets SSH environments"

---

### 2. Layout System

Produce an ASCII layout diagram. Label every panel.

```
┌─────────────────────────────────────────────────┐
│  Header: title + mode indicator + status        │
├──────────────┬──────────────────────────────────┤
│              │                                  │
│  Navigation  │         Main Content             │
│  Panel       │         (primary focus)          │
│              │                                  │
├──────────────┴──────────────────────────────────┤
│  Footer: context-sensitive keybinding hints     │
└─────────────────────────────────────────────────┘
```

Describe:
- Panel hierarchy (which panel owns focus by default)
- Focus flow (Tab order)
- Responsive behaviour (what collapses at narrow widths)
- Scroll semantics per panel

---

### 3. Interaction Model

Design keyboard-first. Every action must have a keyboard binding. Mouse is an optional assist.

Produce the keymap table using the action-first DSL pattern from the research:

| Action | Keybinding | Description |
|--------|-----------|-------------|
| `action.quit` | `q` / `Ctrl+C` | Quit application |
| `action.help` | `?` | Toggle help overlay |
| `action.search` | `/` | Open search/filter |
| `action.focus_next` | `Tab` | Advance focus ring |
| `action.focus_prev` | `Shift+Tab` | Reverse focus ring |
| `action.nav_down` | `j` / `↓` | Move down in list |
| `action.nav_up` | `k` / `↑` | Move up in list |
| `action.nav_top` | `g` | Jump to top |
| `action.nav_bottom` | `G` | Jump to bottom |
| `action.select` | `Enter` | Activate item |
| `action.back` | `Esc` | Close modal / go back |

Add domain-specific actions relevant to the use case. Define keybindings for all dangerous operations behind a confirmation modal (`action.confirm` / `action.cancel`).

---

### 4. Visual Design System

Design using semantic tokens, not hardcoded colours. Consult `references/design-system.md` for the full token system.

Core design principles from the research:
- **Semantic tokens** over raw RGB — define `fg.default`, `fg.muted`, `fg.danger`, `bg.panel`, `border.focused`, `border.inactive`
- **Colour for state and hierarchy only** — selection, focus, diff additions/deletions, warnings, errors
- **Non-colour cues required** — icons with ASCII fallbacks, bold/underline for hierarchy
- **AA contrast minimum** (≥4.5:1 for normal text) as a hard constraint
- **Low-colour fallback** (16-colour mode) and `NO_COLOR` support mandatory

Specify:
- Dark theme palette (primary, secondary, accent, danger, warning, muted, bg variants)
- How borders communicate focus vs inactive state
- Density setting (compact / normal / relaxed)

---

### 5. Component System

Select and specify the components needed. Consult `references/interaction-patterns.md` for component behaviour, states, and performance notes.

For each component, specify:
- **Purpose** in this TUI
- **States**: default, focused, selected, loading, empty, error
- **Performance notes** (virtualise if >500 rows; sanitise if displaying untrusted content)

Standard components available (from research):

| Component | Use when |
|-----------|----------|
| `List` | Primary navigation, item selection |
| `Table` | Structured data with sortable columns |
| `Tree` | Hierarchical data (files, processes, configs) |
| `LogView` | Streaming or historical log display |
| `Form` | Structured data entry with validation |
| `Modal` | Confirmations, dangerous actions, prompts |
| `HelpBar` | Context-sensitive keybinding hints (always visible) |
| `CommandPalette` | Fuzzy search over actions/navigation |
| `Progress` | Async operation feedback |
| `Tabs` | Multiple views within a panel |

**Security rule**: Any component displaying untrusted content (logs, remote data, git objects, resource names) must pass through the sanitisation layer — strip ESC (`\x1b`) and C0 control characters. This directly addresses CWE-150 terminal injection.

---

### 6. Performance Strategy

State the rendering model and performance preset based on the target environment.

Presets from the research:
- **`local_default`**: 30–60 FPS for animations, 10 TPS background tick, truecolour
- **`ssh_safe`**: 10 FPS max, low TPS (4), no full-screen gradients, 256-colour fallback
- **`low_cpu`**: Render on events only; animations replaced with discrete state steps

Architecture principles:
- Decouple TPS (state tick rate) from FPS (render frequency) — critical for SSH
- Never block the UI event loop — async I/O and background workers for data fetching
- Virtualise lists/tables with >500 rows — render only visible rows
- Use diff-based buffer rendering — only transmit changed cells, not full redraws

---

### 7. Technology Recommendation

Choose the best stack. Be decisive. Justify with performance, ecosystem, and developer experience.

Consult `references/library-comparison.md` for the full trade-off matrix.

Quick selection heuristic:
- **Rust + Ratatui** → Maximum performance, CLI tooling, ops tools, long-running processes
- **Go + Bubble Tea** → Strong ops/devops story, static binaries, MVU testability, Charm ecosystem
- **Python + Textual** → Fastest iteration, internal tooling, data exploration, team familiarity
- **Node + Ink** → React-native dev team, component reuse, prototyping

State the chosen stack, the primary reason, and one alternative with trade-offs.

---

### 8. Implementation Plan

Produce a concrete step-by-step build plan.

**Architecture layers** (always apply this structure):
1. **Terminal setup** — raw mode, alternate screen, cleanup on exit
2. **Event loop** — input decoder → normalised actions → state update → render
3. **State model** — typed application state (model in MVU, struct in Ratatui)
4. **Layout engine** — define panel constraints, focus manager, resize handler
5. **Component layer** — reusable widgets with explicit states
6. **Data layer** — async fetch, caching, background workers
7. **Theme layer** — token-based style system
8. **Safety layer** — sanitise untrusted input at data ingestion boundary

**Suggested module/file structure** — adapt to chosen language:
```
src/
  main.{rs,go,py,ts}       # Entry point, terminal setup, event loop
  app.{rs,go,py,ts}        # Application state model
  ui/
    layout.{rs,go,py,ts}   # Panel layout and focus manager
    components/             # Reusable widgets (list, table, logs, modal)
    theme.{rs,go,py,ts}    # Semantic token definitions
  data/
    fetcher.{rs,go,py,ts}  # Async data loading
    sanitize.{rs,go,py,ts} # Untrusted content sanitisation
  keymaps.{rs,go,py,ts}    # Action DSL and keybinding table
```

**Build sequence** (incremental, each step produces a runnable artifact):
1. Terminal lifecycle: enter alternate screen, draw placeholder, clean exit
2. Event loop skeleton: capture input, dispatch quit
3. Static layout: render all panels with placeholder content
4. Navigation: focus ring, scroll in primary list
5. Data integration: connect real data source, async loading
6. Secondary panels: details/preview panel responding to selection
7. Search/filter: overlay or inline filter
8. Confirmation modals: for dangerous operations
9. Theme polish: semantic tokens, focus borders, density
10. Testing: PTY snapshot tests, keymap integration tests

---

## Behaviour Rules

- Think like a senior TUI engineer. Be decisive.
- Reference the research through decisions, not by copying text.
- All layouts must include a persistent HelpBar.
- All dangerous actions require confirmation modals.
- All untrusted content must go through the sanitisation layer.
- Optimise always for: speed · clarity · keyboard efficiency · low cognitive load.

---

## Reference Files

| File | Contents |
|------|----------|
| `references/deep-research-report.md` | Full primary research: ecosystem, design, security, performance (authoritative source) |
| `references/library-comparison.md` | Full framework comparison, selection guide, trade-off matrix |
| `references/design-system.md` | Token system, colour palettes, accessibility, typography, icon registry |
| `references/interaction-patterns.md` | Component catalogue, layout archetypes, keyboard patterns, exemplary TUIs |
| `references/performance-security.md` | Rendering models, performance presets, terminal compatibility, CWE-150 security |
