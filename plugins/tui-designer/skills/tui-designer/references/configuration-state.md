# TUI Configuration, State Management, and Operations

Research-grounded guidance on the practical concerns every production TUI must address: configuration persistence, signal handling, async data architecture, and debugging. These topics are absent from most introductory TUI content but are consistently where real projects stall.

---

## Configuration and Persistence

### XDG Base Directory Spec

Follow the XDG Base Directory Specification for config file placement. This is the standard used by k9s, gitui, bottom, and most Linux-native TUIs:

| Directory | Env var | Default | Use for |
|-----------|---------|---------|---------|
| Config | `$XDG_CONFIG_HOME` | `~/.config` | User preferences, keymaps, themes |
| Data | `$XDG_DATA_HOME` | `~/.local/share` | Persistent app data (history, caches) |
| State | `$XDG_STATE_HOME` | `~/.local/state` | Ephemeral state (last scroll pos, open tabs) |
| Runtime | `$XDG_RUNTIME_DIR` | `/run/user/$UID` | Sockets, locks (temp, cleared on logout) |

Standard path for your app: `$XDG_CONFIG_HOME/myapp/config.toml`

```python
import os
from pathlib import Path

def config_dir() -> Path:
    base = os.environ.get("XDG_CONFIG_HOME") or Path.home() / ".config"
    return Path(base) / "myapp"

def config_path() -> Path:
    return config_dir() / "config.toml"
```

### Config Format Trade-offs

| Format | Human editable | Schema validation | Recommended for |
|--------|---------------|-------------------|-----------------|
| **TOML** | Excellent | Yes (toml-rs, tomllib) | User config, keymaps, themes — the default choice |
| **YAML** | Good | Yes (serde_yaml, PyYAML) | Ops tools with complex nested config |
| **JSON** | Poor | Yes | Programmatic/generated config; never for user-facing |
| **INI** | Good | Limited | Simple legacy tools only |

**Recommendation**: TOML for user config (k9s, gitui, bottom all use TOML), JSON for internal cache/state files.

### Config Layering

Merge in this order — later layers win on conflicts, but only override keys they specify:

```
1. Built-in defaults (compiled in)
2. System config   (/etc/myapp/config.toml)
3. User config     ($XDG_CONFIG_HOME/myapp/config.toml)
4. Project-local   (./.myapp.toml)  ← opt-in only, skip if $HOME == cwd
5. CLI flags       (--theme dark, --no-color)
6. Environment     (MYAPP_THEME=dark)
```

Implement this as explicit merge, not override: if user config omits `theme`, use the default, not an empty value.

### Hot Reload

When config changes on disk:

1. Watch config file via `inotify` (Linux), `FSEvents` (macOS), `ReadDirectoryChangesW` (Windows)
2. Debounce events: wait 300ms for writes to settle before reloading
3. **Validate before applying**: parse and validate the new config; on error, keep the current config and show an inline notification
4. Apply only the changed keys — do not reinitialise the full app state

```python
# Textual example: live theme reload
class App(App):
    def watch_theme_file(self) -> None:
        # Called when theme file changes
        try:
            new_theme = Theme.from_file(self.theme_path)
            self.apply_theme(new_theme)
            self.notify("Theme reloaded")
        except ValueError as e:
            self.notify(f"Theme error: {e}", severity="error")
```

---

## Signal Handling

### Required Signals for Production TUIs

| Signal | Default action |
|--------|---------------|
| `SIGTERM` | Flush dirty state → restore terminal → exit 0 |
| `SIGINT` (Ctrl+C) | Same as SIGTERM; respect dirty-state confirmation |
| `SIGWINCH` | Recalculate layout → re-render immediately |
| `SIGPIPE` | Ignore (graceful degraded output) |
| `SIGHUP` | Reload config, or exit if no config reload support |

### SIGWINCH: Proportional Panel Resize

On terminal resize, recalculate panel dimensions from ratios, not stored cell counts:

```
on SIGWINCH:
  new_cols, new_rows = get_terminal_size()          # ioctl TIOCGWINSZ
  for each panel:
    panel.width = max(panel.min_width, int(new_cols * panel.ratio))
  if sum(panel.widths) < MIN_USABLE_WIDTH:
    activate single-panel mode (collapse secondary panels)
  re-render immediately
```

**Key rules:**
- Store panel sizes as ratios (0.35 / 0.65), not absolute cell counts
- Enforce minimum widths (e.g., navigation panel: 20 cols minimum)
- Below ~60 cols: collapse to single-panel mode; `Enter` enters detail full-screen

### Terminal Cleanup on Crash

A TUI that panics and leaves the terminal in raw mode with the alternate screen active is a serious UX failure. Register cleanup handlers before entering raw mode.

**Rust (Ratatui):**
```rust
// Restore terminal even on panic — set this up before enabling raw mode
let original_hook = std::panic::take_hook();
std::panic::set_hook(Box::new(move |panic_info| {
    let _ = crossterm::terminal::disable_raw_mode();
    let _ = crossterm::execute!(
        std::io::stdout(),
        crossterm::terminal::LeaveAlternateScreen
    );
    original_hook(panic_info);
}));
```

**Go (Bubble Tea):**
```go
func main() {
    oldState, _ := term.MakeRaw(int(os.Stdin.Fd()))
    defer term.Restore(int(os.Stdin.Fd()), oldState)
    defer fmt.Print("\033[?1049l") // leave alternate screen
    // ...
}
```

**Python (Textual / prompt_toolkit):**
Both frameworks use context managers that handle cleanup automatically on exit and exception. Always run via `app.run()`, never drive the event loop manually.

---

## Async Data Architecture

### The Core Rule

**Never put I/O in the render loop or event handler.** All network, file, and process I/O must run in background workers and communicate results back to the UI via messages/channels.

### Channel-Based Worker Pattern

```
UI event loop (main thread/goroutine):
  receives ← DataMsg | ErrorMsg | ProgressMsg
  sends    → FetchRequest | CancelRequest

Worker (background goroutine/thread/task):
  receives ← FetchRequest
  sends    → DataMsg | ErrorMsg
```

**Go + Bubble Tea:**
```go
func fetchPodsCmd(namespace string) tea.Cmd {
    return func() tea.Msg {
        pods, err := k8sClient.ListPods(namespace)
        if err != nil {
            return ErrorMsg{err}
        }
        return PodsLoadedMsg{pods}
    }
}
// In Update(): return model, fetchPodsCmd(ns)
```

**Rust + Ratatui:**
```rust
// Spawn tokio task; send result back via mpsc channel
let tx = self.event_tx.clone();
tokio::spawn(async move {
    match fetch_pods(&namespace).await {
        Ok(pods) => tx.send(AppEvent::PodsLoaded(pods)).unwrap(),
        Err(e)   => tx.send(AppEvent::FetchError(e.to_string())).unwrap(),
    }
});
```

### Cache and TTL

Every data source needs a configurable TTL to prevent stale reads:

```python
@dataclass
class CachedData:
    data: Any
    fetched_at: float  # time.monotonic()
    ttl: float = 5.0   # seconds

    @property
    def is_stale(self) -> bool:
        return time.monotonic() - self.fetched_at > self.ttl

    @property
    def staleness_seconds(self) -> float:
        return time.monotonic() - self.fetched_at
```

**UI rules for stale data:**
- Show stale data while background re-fetch is in progress (never show blank)
- If `staleness > ttl * 2`: show `⚠ 30s ago` indicator near the data
- On re-fetch success: update and clear indicator atomically

### Three Async States Every Data Component Must Handle

Never collapse these — each requires distinct UX:

| State | UI behaviour |
|-------|-------------|
| **Loading** | Show spinner in the panel; disable interactive actions |
| **Error** | Show inline error message; provide `r` to retry; keep last-good data visible if available |
| **Empty** | Show placeholder: "No items. Press `n` to create one." Never show a blank panel |

### Optimistic Updates

For write operations (delete, patch, create):
1. Immediately apply the change to local state (optimistic)
2. Show `◌` pending indicator on the modified item
3. On success: confirm and clear indicator
4. On failure: revert state + show error notification

### Request ID Pattern (Prevent Race Conditions)

```python
self.fetch_request_id += 1
request_id = self.fetch_request_id

async def fetch():
    result = await load_data()
    if request_id == self.fetch_request_id:  # only apply if still current
        self.data = result
```

Without this, a slow earlier response can overwrite a fast later response.

---

## Debugging TUIs

### The Problem

Full-screen raw mode takes over stdout and stderr, making conventional print debugging and log output invisible.

### File Logging (Always Implement)

Write logs to the XDG state directory, gated behind an env var:

```python
import logging, os
from pathlib import Path

log_path = Path(os.environ.get("XDG_STATE_HOME", "~/.local/state")).expanduser() / "myapp/debug.log"
log_path.parent.mkdir(parents=True, exist_ok=True)

logging.basicConfig(
    filename=log_path,
    level=logging.DEBUG if os.environ.get("MYAPP_DEBUG") else logging.WARNING,
    format="%(asctime)s %(levelname)s %(name)s: %(message)s",
)
```

```rust
// Rust: tracing with rolling file appender
let file_appender = tracing_appender::rolling::daily(
    dirs::state_dir().unwrap().join("myapp"),
    "debug.log"
);
tracing_subscriber::fmt().with_writer(file_appender).init();
```

Usage during dev:
```bash
MYAPP_DEBUG=1 ./myapp 2>/dev/null &
tail -f ~/.local/state/myapp/debug.log
```

### Debug Overlay Panel

Add a hidden debug panel toggled by `Ctrl+D` or `F12`, gated behind `MYAPP_DEBUG=1`:

Show:
- Last 50 log lines (ring buffer)
- Current focus ring state
- Render FPS / TPS counters
- Active data fetch status per panel
- State summary (selected item, cursor position, filters active)

This is far more productive than printf debugging.

### Key Events to Log

```python
# Focus changes
log.debug("focus: %s → %s", old_panel, new_panel)

# State transitions
log.debug("state: %s → %s (trigger: %s)", old_state, new_state, event)

# Data fetch lifecycle
log.debug("fetch start: %s ns=%s", resource_type, namespace)
log.info("fetch complete: %s items in %.0fms", len(items), elapsed_ms)
log.error("fetch error: %s", exc)

# Render timing (enable with MYAPP_PERF=1)
log.debug("render: %.1fms (frame %d)", render_ms, frame_count)
```

---

## State Design at Scale

### State Shape Principles

**Flat is better than nested:**
```python
# Prefer this
state = {
    "pods": {"pod-abc": Pod(...), "pod-xyz": Pod(...)},  # ID → data map
    "pod_ids": ["pod-abc", "pod-xyz"],                   # ordered list of IDs
    "selected_pod_id": "pod-abc",
    "scroll_offset": 0,
}

# Avoid this
state = {
    "pods": [Pod(id="pod-abc", is_selected=True), ...]   # selection mixed into data
}
```

**Derived state in View, not State:**
- `selected_pod = state.pods[state.selected_pod_id]` — compute in View
- Don't store `selected_pod` separately — it creates sync bugs

### Common State Bugs and Fixes

| Bug | Cause | Fix |
|-----|-------|-----|
| Cursor below last item after reload | Data shrank; cursor not clamped | After data update: `cursor = min(cursor, max(0, len(items)-1))` |
| Focus lost on data refresh | Panel re-initialised on update | Preserve `focused_panel_id` across refreshes; restore after render |
| Stale response overwrites new data | Two concurrent fetches race | Use request IDs (see above) |
| Unbounded memory growth | Log lines / events accumulate forever | Ring buffer with max size: `deque(maxlen=10_000)` |
| Scroll position reset on filter change | Filter applied but offset not reset | Reset `scroll_offset = 0` when filter predicate changes |

### Undo / History (Editor Archetype)

For editor-type TUIs managing structured documents:
- Store edit history as a list of reversible operations (command pattern)
- Maximum undo depth: 100 operations (configurable)
- Persist history to `$XDG_STATE_HOME/myapp/history` across sessions
- `u` to undo, `Ctrl+R` to redo (Vim convention)
