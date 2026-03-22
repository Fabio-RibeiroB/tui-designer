# TUI Library Comparison and Selection Guide

Research-grounded analysis of the major TUI stacks. Use this to justify technology recommendations.

---

## The Four Layers of a TUI (Architecture Context)

Every TUI sits on layered abstractions. Design decisions at each layer affect performance and portability:

1. **Terminal capability discovery** — terminfo database, `$TERM` variable, ANSI/VT escape sequences
2. **Screen and input abstraction** — libraries like ncurses, crossterm, tcell that normalise terminal differences
3. **Application architecture** — event loops, MVU vs immediate-mode vs retained widget trees
4. **Widgets vs primitives** — rich toolkits (tview, Textual) vs low-level cell APIs (crossterm, termbox)

---

## Full Comparison Table

| Library / Stack | Language | Level | Architecture | Strengths | Pitfalls |
|----------------|----------|-------|-------------|-----------|---------|
| **Ratatui + crossterm** | Rust | Widget library | Immediate-mode + double-buffer diff | Peak performance; safe text handling; strong ecosystem | Must manage full draw cycle; steeper learning curve |
| **Bubble Tea + Bubbles** | Go | App framework | MVU (Init/Update/View) | Predictable state; testable; Charm ecosystem (Bubbles, Lip Gloss) | Requires disciplined state design |
| **tcell + tview** | Go | Widget toolkit | Cell grid + widget tree | Rich built-in widgets (list, form, table); good for ops tools | Focus/composition complexity; less structured theming |
| **Textual** | Python | App framework | Widget tree + asyncio + reactive attrs | Modern CSS-like styling; fastest iteration; excellent docs | Heavier runtime; async correctness required |
| **prompt_toolkit** | Python | Framework | Layout + keybindings + styles | Excellent input editing; composable keybinding system | Complex for large apps |
| **Rich** | Python | Renderables toolkit | Auto-refresh renderables | Beautiful tables/progress quickly | Not designed as a full-screen app framework |
| **Ink** | Node | Framework | React renderer + Yoga/Flexbox | Familiar React model; Flexbox layout | Node runtime footprint; terminal edge cases |
| **blessed + contrib** | Node | Widget toolkit | DOM-like elements | Fast dashboard prototyping | blessed is unmaintained; forks vary in quality |
| **ncurses** | C | Screen abstraction | Terminal-independent + terminfo | Maximum portability; minimal redraws | Complex API; C safety burden |
| **Urwid** | Python | Widget toolkit | Widget → canvas + main loop | Proven; fluid resizing | Older default aesthetic |

---

## Selection Guide

### Choose Rust + Ratatui when:
- Performance is the primary constraint (real-time metrics, high-frequency updates)
- Binary distribution matters (ops tooling, DevOps CLIs distributed to servers)
- The team is comfortable with Rust
- You need the most reliable terminal compatibility via crossterm
- **Reference TUIs**: gitui, bottom, lazysql, similar to k9s in density

**Rendering model**: Immediate-mode with double-buffer diffing. Ratatui draws the entire frame on each tick, diffs it against the previous buffer, and emits only changed cells. Decouple TPS (state tick) from FPS (render) explicitly.

### Choose Go + Bubble Tea when:
- You want MVU architecture with clean testability
- Static binary distribution is valuable (go build produces a single binary)
- The Charm ecosystem (Bubbles, Lip Gloss, Huh, Wish) covers your component needs
- SSH-first tooling (Wish provides an SSH server for TUIs)
- **Reference TUIs**: lazygit uses gocui; k9s; many DevOps tools

**Rendering model**: MVU — every event calls Update(msg) → returns new Model + optional Cmd. View(model) renders the model as a string. Pure-ish, predictable, testable.

### Choose Python + Textual when:
- Iteration speed matters more than raw performance
- Internal tooling, data exploration, team familiarity with Python
- CSS-like styling and a high-level widget model are valuable
- The audience is developers comfortable with Python packaging
- **Reference TUIs**: Textual's own demo apps; posting (HTTP client)

**Rendering model**: Retained widget tree with reactive attributes. Widgets respond to events via `on_*` handlers. Async rendering via asyncio tasks keeps the UI responsive.

### Choose Python + prompt_toolkit when:
- Building a REPL-enhancement or an interactive shell
- Fine-grained keybinding composition is required
- Mixing full-screen with line-editing modes
- **Reference TUIs**: ptpython, pgcli, litecli

### Choose Node + Ink when:
- The team has strong React experience and wants to reuse that mental model
- Prototyping speed is paramount
- The TUI is a companion to a web frontend sharing component logic
- **Reference TUIs**: Ink's own example apps, some Vercel/npm CLI tools

---

## Key Trade-off Dimensions

### Performance envelope
- **Rust > Go ≈ C > Python > Node** for sustained rendering throughput
- Python + Textual is fast enough for most business tools; avoid for real-time high-Hz dashboards

### Distribution
- **Rust, Go** → single static binary, no runtime required
- **Python** → requires Python environment; use PyInstaller/Nuitka for standalone
- **Node** → requires Node runtime; use pkg/nexe for standalone

### Windows / ConPTY support
- **crossterm (Rust), Textual (Python), Ink (Node)** have explicit Windows/ConPTY support
- **Go (tcell/Bubble Tea)** has reasonable Windows support but SSH/PTY edge cases exist
- Prefer crossterm-backed stacks for Windows-first deployments

### SSH deployment
- **Go + Wish** provides a first-class SSH-served TUI story
- Any PTY-requiring TUI needs `ssh -t` for full-screen; plan for no-PTY degraded mode

### Ecosystem maturity
- **Ratatui** (Rust): thriving; 10k+ GitHub stars; active releases
- **Bubble Tea + Charm** (Go): thriving; commercial backing from Charm
- **Textual** (Python): thriving; commercial backing from Textualize
- **Ink** (Node): stable but lower activity
- **blessed** (Node): unmaintained; avoid for new projects

---

## Architecture Patterns to Generate

### MVU (Model–Update–View) — Bubble Tea
```
Init() → initial Model + optional Cmd
Update(msg) → new Model + optional Cmd
View(model) → string (the rendered frame)
```
Best for: clean state management, testable logic, predictable rendering.

### Immediate-Mode — Ratatui
```
loop:
  handle_events() → update state
  terminal.draw(|f| render_frame(f, &state))
  sleep(tick)
```
Best for: performance-critical apps, full control over every pixel.

### Reactive Widget Tree — Textual
```
class App(App):
    def compose() → yield widgets
    def on_event(event) → update reactive attributes → auto-rerender
```
Best for: component reuse, CSS-like styling, rapid development.

---

## Keymap Patterns by Framework

| Framework | Keymap pattern |
|-----------|---------------|
| Bubble Tea | Pattern-match on `tea.KeyMsg` in `Update()`; use `key.Binding` from Bubbles |
| Ratatui | Match on `KeyEvent` in event handler; use `crossterm::event::KeyCode` |
| Textual | `BINDINGS` class var with `Binding()` entries; auto-generates help |
| prompt_toolkit | `KeyBindings()` with `@kb.add("c-q")` decorators; filterable via `Condition` |
| Ink | `useInput()` hook with key inspection |

---

## Recommended Starter Templates

When generating project scaffolds, always include:
1. Terminal lifecycle (enter/leave alternate screen, raw mode, cleanup on panic/signal)
2. Resize handler
3. Help bar component
4. Quit confirmation if state is dirty
5. `NO_COLOR` environment variable check
6. Sanitisation function for untrusted text (see `performance-security.md`)
