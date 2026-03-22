# TUI Performance, Security, and Compatibility Reference

Research-grounded guidance on rendering, compatibility, and security for professional TUIs.

---

## Rendering Models

### Why some TUIs feel snappy and others don't

Terminal output is slow relative to screen pixels. The difference between a snappy and sluggish TUI is almost always:
1. How much is written to the terminal per frame
2. Whether the event loop is blocked during data fetching
3. Whether rendering is decoupled from state update frequency

### Diff-based buffer rendering

The standard approach in professional TUI libraries:
- Maintain two buffers: **current frame** and **previous frame**
- On each draw call, compute the diff between them
- Emit only the changed cells as ANSI sequences

This is how ncurses has worked since the 1980s ("optimising output by computing a minimal volume of operations to mutate the screen between refreshes"). Ratatui implements the same pattern with its double-buffer technique.

Result: a frame with minor changes (e.g., cursor moved one row) emits a few bytes, not a full-screen redraw.

### Immediate-mode (Ratatui)

```
loop:
  events = poll_events(timeout=tick_rate)
  state = handle_events(events, state)
  terminal.draw(|frame| render(frame, &state))
```

Draw the entire logical frame from state on every tick. The library diffs and emits only changes. Straightforward mental model but requires keeping `render()` cheap — no blocking I/O, no heavy computation.

### MVU / Declarative (Bubble Tea)

```
msg = wait_for_event()
model, cmd = update(model, msg)
output = view(model)
```

State is updated in `update()`, rendered in `view()`. Background work is represented as `Cmd` (a function that runs concurrently and returns a `Msg`). The event loop is never blocked.

### Retained widget tree (Textual)

Widgets maintain their own state and redraw independently. The framework manages dirty-marking and partial re-renders. Background work uses asyncio tasks. The UI remains responsive as long as handlers are non-blocking.

---

## Performance Presets

Apply these as named configurations. Emit the appropriate rendering policy in generated code.

### `local_default`
- Max animation FPS: 30–60
- Background tick TPS: 10
- Colour: truecolour (24-bit)
- Spinner: full Unicode, animated
- List virtualisation: enabled for >500 rows

### `ssh_safe`
- Max animation FPS: 10 (reduce ANSI write volume over network)
- Background tick TPS: 4
- Colour: 256-colour fallback (no full-screen gradients)
- Spinner: low-FPS (100ms → 500ms frame interval)
- Batch writes: aggregate multiple cell changes before flushing
- Avoid: full-screen redraws, high-frequency progress bars

### `low_cpu`
- Render on events only (no fixed-rate tick)
- Animations: disabled, replaced with discrete state labels ("Loading..." text, no spinner)
- Colour: 16-colour
- Background tick: only for critical updates (e.g., elapsed time display)

### Detection heuristic

```python
import os

def detect_preset() -> str:
    ssh = bool(os.environ.get("SSH_CLIENT") or os.environ.get("SSH_TTY"))
    ci = bool(os.environ.get("CI"))
    if ci:
        return "low_cpu"
    if ssh:
        return "ssh_safe"
    return "local_default"
```

---

## Event Loop Architecture

The conceptual backbone all TUI templates should follow:

```
User input → Terminal/PTY → Input Decoder
                                   ↓
                          Normalised Action (event)
                                   ↓
                          App State (Model) → Update(action) → new Model
                                   ↓
                          View Renderer → Layout + Frame
                                   ↓
                          Output Writer → Diff → ANSI sequences → Terminal
```

Rules:
- **Never block the event loop** — all I/O must be async or run in a background thread/goroutine
- **Decouple TPS from FPS** — Ratatui templates explicitly separate tick rate (state update frequency) from frame rate (render frequency)
- **Backpressure**: if the render queue is falling behind, drop intermediate frames rather than buffering indefinitely

---

## Virtualisation

For lists and tables with large datasets:

- **Render only visible rows** plus a buffer of ~10 rows above/below the viewport
- Track `scroll_offset` and `viewport_height` in state
- Compute `visible_range = scroll_offset..scroll_offset + viewport_height`
- Render items in that range only

```rust
// Ratatui example
let visible: Vec<_> = items
    .iter()
    .skip(scroll_offset)
    .take(viewport_height + 10)
    .collect();
```

This makes 100,000-row tables perform identically to 100-row tables.

---

## Terminal Compatibility

### The compatibility matrix

| Feature | Wide support | Limited support | Avoid if compatibility matters |
|---------|-------------|----------------|-------------------------------|
| 256-colour | ✓ Most terminals | ✗ Old terminals | — |
| Truecolour | ✓ Modern terminals | ✗ Some SSH setups | Don't hardcode, check `$COLORTERM` |
| Alternate screen | ✓ All major terminals | — | Required for full-screen TUIs |
| Mouse reporting | ✓ xterm/most | ✗ Some SSH clients | Enable opt-in only |
| Bracketed paste | ✓ Modern terminals | ✗ Old terminals | Enable opt-in |
| Unicode box drawing | ✓ UTF-8 terminals | ✗ ASCII terminals | Provide ASCII fallback |
| OSC 8 (hyperlinks) | Partial | Many don't support | Off by default |
| OSC 52 (clipboard) | Partial | Security-sensitive | Off by default |
| Sixel / kitty images | Limited | Many don't support | Off by default |

### $TERM and terminfo

The `$TERM` variable tells the application what terminal capabilities are available. Always:
1. Read `$TERM` to detect terminal type
2. Query terminfo for specific capabilities (colour support, alternate screen, mouse)
3. Never assume truecolour without `$COLORTERM=truecolor`
4. Test in `TERM=dumb` to verify graceful degradation

### SSH considerations

Full-screen TUIs require a PTY. When running over SSH:
- Users must use `ssh -t host command` or the server must allocate a PTY
- Detect no-PTY context: `os.isatty(sys.stdin.fileno())` → False
- Provide a degraded mode: non-interactive output or "press Enter to launch UI"
- Apply `ssh_safe` performance preset automatically when `SSH_TTY` is set

### Windows / ConPTY

Windows 10+ provides ConPTY (CreatePseudoConsole) which streams UTF-8 text with virtual terminal sequences. For Windows support:
- Prefer libraries with explicit ConPTY support: crossterm (Rust), Textual (Python), Ink (Node)
- Test CRLF handling (`\r\n` vs `\n`)
- Test Unicode rendering in Windows Terminal

### Web terminals (xterm.js)

xterm.js supports most features including mouse events, curses apps, and accessibility (screen reader mode, minimum contrast ratio). Relevant for browser-based dev environments (Gitpod, GitHub Codespaces, VS Code web).

---

## Security: Terminal Escape Injection (CWE-150)

### The threat

Terminal escape/control sequences are an injection surface. MITRE CWE-150 defines "Improper Neutralisation of Escape, Meta, or Control Sequences."

TUIs that display untrusted content — logs, git commit messages, Kubernetes resource names, remote command output, file contents — are vulnerable if they render that content without sanitisation.

Injection can:
- Clear the screen and rewrite prompts (deceptive UI)
- Rewrite terminal title bars
- Exfiltrate clipboard contents via OSC 52
- Trigger URL opens via OSC 8
- Steal keystrokes via bracketed paste abuse

In agentic/AI contexts, ANSI escape injection has been demonstrated as a method to deceive users about tool outputs.

### Required mitigations

**Apply sanitisation at the data ingestion layer** — not at the rendering layer. Sanitise when data enters your application, before storing in state.

```python
def sanitize_untrusted(s: str) -> str:
    """
    Remove ESC and C0 control characters except safe whitespace.
    Mitigates CWE-150 terminal escape injection.
    Safe: \t (0x09), \n (0x0A), \r (0x0D)
    Unsafe: \x1b (ESC), \x00-\x08, \x0b-\x0c, \x0e-\x1f, \x7f
    """
    result = []
    for ch in s:
        cp = ord(ch)
        if cp in (0x09, 0x0A, 0x0D):  # tab, newline, carriage return
            result.append(ch)
        elif cp < 0x20 or cp == 0x7F:  # control characters
            continue
        else:
            result.append(ch)
    return ''.join(result)
```

```rust
fn sanitize_untrusted(s: &str) -> String {
    s.chars()
        .filter(|&ch| ch == '\t' || ch == '\n' || ch == '\r'
                || (ch >= ' ' && ch != '\x7f'))
        .collect()
}
```

### Separation of trusted vs untrusted content

Architecture rule: **styling sequences come from your code; content data never contains raw terminal sequences**.

- Your theme system emits ANSI codes
- Your layout engine emits ANSI codes
- Data from external sources is always plain text after sanitisation
- The two streams never mix in the same string

### Optional advanced features: explicit allowlist

OSC features must be explicitly enabled:

| Feature | Default | Enable when |
|---------|---------|------------|
| OSC 52 clipboard | **Off** | User explicitly enables and terminal is confirmed to support it |
| OSC 8 hyperlinks | **Off** | User explicitly enables; use for trusted internal links only |
| Bracketed paste | **On** (safety feature) | Always enable — it prevents pasted text from being interpreted as typed commands |
| Sixel / kitty images | **Off** | Experimental mode only |

---

## Testing TUIs

### PTY-based testing

For robust regression testing, run your TUI under a pseudo-terminal and record frame output.

```python
import pty, os, subprocess

# Launch TUI under PTY
master, slave = pty.openpty()
proc = subprocess.Popen(['./my_tui'], stdin=slave, stdout=slave, stderr=slave)
# Send key events via master fd
os.write(master, b'j')  # nav down
# Read rendered output
output = os.read(master, 4096)
```

### Snapshot/golden testing

Record terminal state as a text snapshot. Diff against expected output in CI.

Tools:
- **VHS**: script terminal interactions; renders to GIF, MP4, or terminal recording
- **asciinema**: records to structured JSON (newline-delimited); replay in terminal or web

VHS script example:
```vhs
Output demo.gif
Set FontSize 14
Set Width 120
Set Height 30

Type "j"
Sleep 200ms
Type "j"
Sleep 200ms
Type "Enter"
Sleep 500ms
Screenshot result.png
```

### Snapshot test approach

1. Run TUI under PTY with scripted inputs
2. Capture the rendered terminal buffer as text (strip ANSI codes for comparison)
3. Store as `.snap` file
4. CI: re-run and diff against stored snapshot
5. Update snapshots intentionally, not accidentally

This catches regressions in layout, column widths, truncation, and focus behaviour.

---

## Checklist: Before Shipping a TUI

### Performance
- [ ] Event loop never blocks on I/O
- [ ] Lists/tables virtualise at >500 rows
- [ ] Rendering preset matches target environment
- [ ] TPS and FPS are decoupled

### Compatibility
- [ ] Tested with `TERM=dumb` (graceful degradation)
- [ ] `NO_COLOR` environment variable respected
- [ ] Alternate screen entered and exited cleanly
- [ ] Handles SIGWINCH (terminal resize) correctly
- [ ] Runs in SSH session without crashing

### Security
- [ ] All untrusted content passes through sanitise function
- [ ] OSC 52, OSC 8 are disabled by default
- [ ] Bracketed paste mode enabled
- [ ] No string concatenation of trusted ANSI codes with untrusted data

### Accessibility
- [ ] All text meets AA contrast minimum (≥4.5:1)
- [ ] Colour-blind-safe: every status has a non-colour cue
- [ ] `NO_COLOR` fallback produces usable interface
- [ ] Full keyboard access to all features

### Cleanup
- [ ] Raw mode disabled on exit
- [ ] Alternate screen exited on exit
- [ ] Exit handlers registered for signals (SIGTERM, SIGINT, panics)
