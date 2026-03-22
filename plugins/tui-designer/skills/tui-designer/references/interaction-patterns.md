# TUI Interaction Patterns and Component Catalogue

Research-grounded patterns for keyboard-first terminal interfaces. Reference when designing interaction models and component systems.

---

## Layout Archetypes

### Two-Pane Workbench (Explorer)

The dominant pattern for navigational TUIs. Used by lazygit, ranger, yazi, Midnight Commander.

```
┌─────────────────────────────────────────────────┐
│  Header: title + mode + status                  │
├──────────────┬──────────────────────────────────┤
│              │                                  │
│  Navigation  │  Detail / Preview                │
│  (primary)   │  (secondary)                     │
│              │                                  │
├──────────────┴──────────────────────────────────┤
│  Footer: context-sensitive key hints            │
└─────────────────────────────────────────────────┘
```

Rules:
- Navigation panel owns default focus
- Selection in navigation drives content in detail panel
- Detail panel is read-only by default; requires explicit focus transfer
- Ratio: 30–40% navigation / 60–70% detail is common
- At narrow widths (<80 cols): collapse detail panel; `Enter` enters detail full-screen

### Three-Pane Orthodox (Deep Explorer)

For hierarchical data with three levels. Used by Midnight Commander, some file managers.

```
┌─────────────────────────────────────────────────┐
│  Header                                         │
├──────────┬──────────────┬───────────────────────┤
│          │              │                       │
│  Root    │  Items       │  Preview              │
│  Tree    │  List        │  / Details            │
│          │              │                       │
├──────────┴──────────────┴───────────────────────┤
│  Footer                                         │
└─────────────────────────────────────────────────┘
```

### Dashboard (Multi-Panel Monitor)

For live data streams. Used by k9s, htop, bottom, grafterm.

```
┌─────────────────────────────────────────────────┐
│  Header: title + cluster + time + health        │
├─────────────────────────────────────────────────┤
│                                                 │
│  Primary Table (full width, scrollable)         │
│  Columns: sortable, filterable                  │
│                                                 │
├──────────────────┬──────────────────────────────┤
│  Metrics Panel   │  Log Tail                    │
│  (charts/stats)  │  (streaming)                 │
└──────────────────┴──────────────────────────────┘
```

Rules:
- Primary table dominates most of the screen
- Status bar shows aggregate health (counts, timestamps, active filters)
- Column customisation is a product-grade requirement (k9s documents this explicitly)

### Editor / Workspace

For editing structured content. Used by neovim, helix, lazysql.

```
┌─────────────────────────────────────────────────┐
│  Tab bar: open buffers                          │
├──────────────┬──────────────────────────────────┤
│              │                                  │
│  File Tree   │  Editor / Form                  │
│  / Outline   │  (primary focus)                │
│              │                                  │
├──────────────┴──────────────────────────────────┤
│  Status: mode + position + linter status        │
└─────────────────────────────────────────────────┘
```

---

## Component Catalogue

### List

The most fundamental TUI component. Used for navigation, selection, item browsing.

**States**: default, focused, selected, loading (spinner), empty (placeholder), error

**Behaviour**:
- Single selection: `j/k` or `↑/↓` to navigate, `Enter` to select
- Multi-selection: `Space` to toggle, `a` to select all
- Filter: `/` opens inline search; filters items in real-time
- Always show selected item count in footer when multi-select active

**Performance**: Virtualise rendering for >500 items. Only render visible rows + 10 row buffer.

**Implementation note (Bubble Tea)**: Use `list.Model` from Bubbles. It provides built-in filtering, help text, title, and status bar. Override `ItemDelegate` for custom row rendering.

### Table

For structured data with columns. The primary component in dashboards (k9s, htop, ps-style tools).

**States**: same as List, plus: sorted-by-column indicator, column-resize

**Behaviour**:
- `←/→` to scroll columns when wider than viewport
- `s` or click column header to sort ascending; again for descending
- Column widths: auto-size to content with min/max constraints
- **Width engine required**: use `display_width()` for each cell before rendering to handle CJK/Unicode correctly
- Overflow: truncate with `…` at column boundary, never overflow

**Performance**: Same virtualisation rules as List. For >1000 rows, paginate or apply server-side filtering.

**Security**: If any column displays user-generated content or remote data, apply sanitisation on the data layer before reaching the component.

### Tree

For hierarchical data (file systems, process trees, resource hierarchies).

**States**: node collapsed, node expanded, node selected, node loading

**Behaviour**:
- `Enter` or `→` to expand; `←` to collapse; `←` at root navigates up
- Lazily load children on expand (avoid deep upfront fetching)
- Indent: 2 cells per level
- Show expand/collapse glyph: `▸` (collapsed) / `▾` (expanded)

**Performance**: Virtualise; only render visible nodes. For deep trees (>3 levels), provide a "flatten" mode.

### LogView

For streaming or historical log display. Required in any ops/monitoring TUI.

**States**: tailing (auto-scroll to bottom), paused (user scrolled up), searching, filtering

**Behaviour**:
- Auto-tail: scroll to bottom on new lines when in tailing mode
- Pause tail: any `↑`/`k`/`PgUp` gesture freezes tail; resume with `G` (jump to bottom)
- Search: `/` opens search bar; `n`/`N` cycle matches; highlight match in log line
- Filter by level or pattern
- `f` to follow/unfollow tail

**Critical security**: ALL log content is untrusted. Strip ESC (`\x1b`) and all C0 control characters before rendering. Render control characters as visible glyphs (e.g., `^[` for ESC) when explicitly in debug mode. This is a hard requirement — terminal escape injection in log viewers is a real attack vector (CWE-150).

### Form / Input

For structured data entry (config editing, create/update workflows).

**States**: default, focused, valid, invalid (with inline error), disabled

**Behaviour**:
- `Tab` / `Shift+Tab` to move between fields
- `Enter` on submit button; `Esc` to cancel (with unsaved-changes prompt)
- Inline validation: validate on field blur and on submit
- Show validation errors adjacent to the field, not in a modal

**Implementation note (Textual)**: Use `Input`, `Select`, `RadioButton`, `Checkbox` widgets from the built-in widget set. For Bubble Tea, use the `huh` library from Charm.

### Modal

For confirmations, dangerous operations, and prompts. Never skip confirmations for destructive actions.

**States**: open (focused), closed

**Behaviour**:
- `Enter` to confirm; `Esc`/`q`/`n` to cancel
- `Tab` between buttons
- Modal must visually dim the background (if terminal supports it, use `bg.overlay`)
- Do not allow background panel interaction while modal is open
- Title should state the action, not a question: "Delete 3 items" not "Are you sure?"

### HelpBar (Footer)

Always visible. Context-sensitive: update content based on which panel has focus.

**Format**: `[key] action  [key] action  [key] action  ...`

**Rules**:
- Show the 4–6 most common actions for the currently focused component
- Always include `?` to open full help overlay
- Always include quit binding
- Left-align action hints; right-align status info (mode, counts, filter state)

**Implementation**: derive help content from the keymap DSL. Never hardcode strings.

### Help Overlay

Full-screen or modal overlay showing all keybindings. Triggered by `?`.

**Layout**: two-column table — left: keybinding, right: description. Grouped by context (Global / Navigation / Actions / etc.).

### Command Palette

Power navigation/action layer. Triggered by `Ctrl+P` or a dedicated key.

**Behaviour**:
- Fuzzy search over all available actions and navigation targets
- Score by: recency, frequency, string match quality
- `Enter` to execute; `Esc` to dismiss
- Entries include: command name, current keybinding, category

**Implementation**: can be built on top of the List component with a filter mode.

### Progress / Spinner

For async operation feedback. Never block the UI while data loads.

**Types**:
- **Spinner** (indeterminate): use animated frames `⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏` at 100ms intervals
- **Progress bar** (determinate): show percentage and ETA when measurable

**Performance rules**:
- Cap spinner refresh at 10 Hz on SSH, 20 Hz local
- Use `low_cpu` preset to replace animations with discrete steps if CPU-constrained
- Never spin in the main UI thread — use background tick

### Tabs

For multiple views within a panel.

**Behaviour**:
- `[` / `]` or numeric keys `1–9` to switch tabs
- Tabs label shows count badge when relevant (e.g., `Errors (3)`)
- Active tab: bold label, `border.focused` underline
- Inactive tabs: `fg.muted` label

---

## Keyboard Design Principles

### Keyboard-first is non-negotiable

Every action must be reachable without a mouse. Mouse support is a welcome enhancement, not a substitute.

### Action DSL pattern

Define actions as named symbols, then bind keys to actions. This enables:
- Remapping without code changes
- Auto-generated help panels
- Context-sensitive binding activation

```python
# Example action definition
ACTIONS = {
    "action.quit":       {"keys": ["q", "ctrl+c"], "help": "Quit"},
    "action.help":       {"keys": ["?"],            "help": "Toggle help"},
    "action.search":     {"keys": ["/"],            "help": "Search/filter"},
    "action.focus_next": {"keys": ["tab"],          "help": "Next panel"},
    "action.focus_prev": {"keys": ["shift+tab"],    "help": "Previous panel"},
    "action.nav_down":   {"keys": ["j", "down"],    "help": "Move down"},
    "action.nav_up":     {"keys": ["k", "up"],      "help": "Move up"},
    "action.nav_top":    {"keys": ["g"],            "help": "Jump to top"},
    "action.nav_bottom": {"keys": ["G"],            "help": "Jump to bottom"},
    "action.select":     {"keys": ["enter"],        "help": "Select/open"},
    "action.back":       {"keys": ["esc"],          "help": "Cancel/back"},
    "action.refresh":    {"keys": ["r", "ctrl+r"],  "help": "Refresh"},
    "action.copy":       {"keys": ["y"],            "help": "Copy to clipboard"},
    "action.delete":     {"keys": ["d", "delete"],  "help": "Delete (confirmation)"},
}
```

### Ergonomic defaults (from research)

| Pattern | Implementation |
|---------|---------------|
| Vim-like navigation | `j/k` for down/up, `g/G` for top/bottom, `/` for search |
| Quit safety | `q` in navigational context, `Ctrl+C` always, no accidental exit |
| Modal close | `Esc` always closes the topmost modal |
| Help access | `?` always opens help overlay |
| Tab flow | `Tab` next panel, `Shift+Tab` previous |
| Confirmation | `Enter` to confirm, `Esc`/`n` to cancel |
| Multi-select | `Space` to toggle, `a` for all |

### Focus management

Maintain an explicit focus ring across panels:
- Track active panel ID in application state
- Border style changes to `thick` + `border.focused` colour on active panel
- Route all keyboard events to the focused panel's handler
- `Tab` / `Shift+Tab` advances/reverses the focus ring regardless of current panel

### Mouse as "assist"

Enable mouse support but ensure all actions are available via keyboard:
- Click to focus a panel
- Click to select a list item
- Scroll wheel in lists/tables
- Double-click as shortcut for `Enter`

Never require mouse for any core workflow.

---

## Reference TUIs to Study

| Tool | What to study | Language |
|------|--------------|---------|
| **lazygit** | Panel layout, key-driven multi-step workflows, custom keymaps | Go (gocui) |
| **k9s** | Table-first data exploration, column customisation, command palette, fast navigation | Go |
| **htop** | Dense process table, tree mode, mouse support, config UI | C (ncurses) |
| **yazi** | Async I/O for responsiveness, modern visuals, preview system | Rust (Ratatui) |
| **gitui** | Ratatui usage patterns, event-driven rendering | Rust (Ratatui) |
| **bottom** | Modern charts/graphs, heavy customisation, widget composition | Rust (Ratatui) |
| **Textual demo** | CSS-like styling, reactive attributes, widget model | Python (Textual) |
| **ranger** | Vim-like navigation, preview system, minimal visuals | Python |
| **Zellij** | Plugin architecture, layout management, UX defaults | Rust |
| **tmux** | Pane management, key table/prefix model, composability | C |

Study these to calibrate "professional feel" across density, navigation, and visual polish.
