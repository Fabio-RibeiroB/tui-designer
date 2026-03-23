# TUI Design System Reference

Token-based visual design for professional terminal interfaces. All decisions traced to the research.

---

## Core Principle: Semantic Tokens Over Raw Colours

Terminal colour capabilities span 16-colour, 256-colour, and 24-bit truecolour, but actual palettes vary by emulator and user theme. Designing with hardcoded RGB creates fragile UIs that break on light terminals or accessibility themes.

**The rule**: define semantic tokens, then map them to terminal representations at render time.

---

## Canonical Token Vocabulary

### Foreground tokens

| Token | Purpose |
|-------|---------|
| `fg.default` | Primary body text |
| `fg.muted` | Secondary labels, metadata, timestamps |
| `fg.subtle` | Placeholder text, disabled items |
| `fg.inverse` | Text on coloured backgrounds (e.g., selected item) |
| `fg.danger` | Errors, destructive actions |
| `fg.warning` | Cautions, deprecations |
| `fg.success` | Confirmations, healthy states |
| `fg.info` | Informational accents |
| `fg.accent` | Primary brand / highlight colour |

### Background tokens

| Token | Purpose |
|-------|---------|
| `bg.base` | Main application background |
| `bg.panel` | Panel / sidebar background (slightly elevated) |
| `bg.overlay` | Modal, dropdown overlay background |
| `bg.selected` | Selected row / item in lists |
| `bg.hovered` | Cursor hover (mouse assist) |
| `bg.input` | Text input fields |

### Border tokens

| Token | Purpose |
|-------|---------|
| `border.focused` | Active panel border (high contrast) |
| `border.inactive` | Inactive panel border (muted) |
| `border.separator` | Internal dividers |
| `border.overlay` | Modal/dialog border |

### Status tokens (use sparingly)

| Token | Colour direction | Non-colour cue |
|-------|-----------------|---------------|
| `status.error` | Red family | `✗` or `[!]` prefix |
| `status.warning` | Yellow family | `⚠` or `[~]` prefix |
| `status.success` | Green family | `✓` or `[✓]` prefix |
| `status.info` | Cyan/blue family | `ℹ` or `[i]` prefix |
| `status.pending` | Muted / spinner | `◌` or `[…]` prefix |

**Non-colour cues are required.** Users with colour vision deficiency rely on shape and label differences. Never encode meaning in hue alone.

---

## Reference Dark Theme Palette (Default)

Expressed as 24-bit truecolour. Map to 256-colour or 16-colour equivalents for fallback modes.

```
fg.default       #e2e8f0   (near-white, primary text)
fg.muted         #94a3b8   (slate-400, secondary)
fg.subtle        #475569   (slate-600, disabled)
fg.inverse       #0f172a   (near-black, on-accent)
fg.danger        #f87171   (red-400)
fg.warning       #fbbf24   (amber-400)
fg.success       #4ade80   (green-400)
fg.info          #38bdf8   (sky-400)
fg.accent        #818cf8   (indigo-400)

bg.base          #0f172a   (slate-950)
bg.panel         #1e293b   (slate-800)
bg.overlay       #1e293b   (slate-800, with border)
bg.selected      #334155   (slate-700)
bg.hovered       #1e3a5f   (blue-tinted)
bg.input         #1e293b

border.focused   #818cf8   (indigo-400, matches accent)
border.inactive  #334155   (slate-700, subtle)
border.separator #1e293b
border.overlay   #475569
```

---

## Light Theme

TUIs run on dark terminals in the overwhelming majority of cases. Implement a light theme only when explicitly required. The mapping principle: invert `fg` and `bg` roles, use darker variants of accent/status colours (higher chroma at lower brightness) to maintain contrast ratios against light backgrounds. All AA contrast requirements still apply — re-verify every token pair.

---

## Accessibility Requirements

### Contrast targets

WCAG 2.2 defines contrast ratios for text legibility:
- **AA (minimum)**: ≥4.5:1 for normal text, ≥3:1 for large text (≥18pt or ≥14pt bold)
- **AAA (enhanced)**: ≥7:1 for normal text

Apply AA as the hard minimum for all body text. Apply AAA for critical status indicators (error messages, confirmation prompts).

**Test all theme tokens** against their paired backgrounds before shipping. The dark theme above achieves:
- `fg.default` on `bg.base`: ~13:1 (exceeds AAA)
- `fg.muted` on `bg.base`: ~5.2:1 (AA pass)
- `fg.danger` on `bg.base`: ~4.9:1 (AA pass)

### Colour-blind safety

Design the full colour set to be distinguishable for the three most common deficiencies:
- **Deuteranopia** (green-blind): avoid green/red alone — pair with `✓`/`✗` glyphs
- **Protanopia** (red-blind): same mitigation
- **Tritanopia** (blue-blind): avoid blue/yellow discrimination as sole signal

The canonical mitigation: **every colour-based status must have a non-colour cue** (glyph prefix, text label, shape).

---

## Colour Depth Fallback Strategy

Implement all three tiers:

### Truecolour (24-bit)
Use the full palette above. Auto-detect via `$COLORTERM=truecolor`.

### 256-colour
Map semantic tokens to the 256-colour cube. Best approximations:
- `fg.accent` (#818cf8) → xterm colour 105 (`#8787ff`)
- `bg.selected` (#334155) → xterm colour 237 (`#3a3a3a`)
- `fg.danger` (#f87171) → xterm colour 210 (`#ff8787`)
- `fg.success` (#4ade80) → xterm colour 120 (`#87ff87`)

### 16-colour / NO_COLOR
When `NO_COLOR` environment variable is set, disable all colour output per the NO_COLOR spec. Fall back to:
- Bold for `fg.accent` and focused borders
- Underline for selected items
- `[ ]` bracketed prefixes for all status indicators
- ASCII box-drawing characters only (no Unicode box-drawing if unsupported)

Detection order:
```
1. NO_COLOR env var → disable all colour
2. COLORTERM=truecolor → 24-bit
3. TERM=xterm-256color → 256-colour
4. fallback → 16-colour
```

---

## Typography and Spacing

### Monospace constraints

TUIs cannot choose the terminal font. Design within these constraints:
- All layout is cell-based: 1 character = 1 cell (except wide characters)
- **Wide characters** (CJK, certain Unicode) occupy 2 cells — use `wcwidth()` to measure
- **Combining characters** (accents, diacritics) occupy 0 additional cells
- Terminals vary in how they handle **ambiguous-width** Unicode (East Asian Ambiguous class)

Implement a width utility:
```python
# Python example
import unicodedata

def display_width(s: str) -> int:
    """Return the number of terminal cells occupied by string s."""
    width = 0
    for ch in s:
        eaw = unicodedata.east_asian_width(ch)
        if eaw in ('W', 'F'):
            width += 2
        elif eaw in ('Na', 'H', 'N'):
            width += 1
        else:  # 'A' (ambiguous) — treat as 1 for Western terminals
            width += 1
    return width

def truncate(s: str, max_cells: int, ellipsis: str = '…') -> str:
    """Truncate s to fit within max_cells, appending ellipsis if needed."""
    if display_width(s) <= max_cells:
        return s
    ellipsis_w = display_width(ellipsis)
    result = ''
    w = 0
    for ch in s:
        ch_w = display_width(ch)
        if w + ch_w + ellipsis_w > max_cells:
            return result + ellipsis
        result += ch
        w += ch_w
    return result
```

### Spacing system

Use consistent cell-count spacing:
- **Padding**: 1 cell horizontal, 0 vertical (compact); 1 cell both (normal); 2 cells horizontal (relaxed)
- **Panel gutter**: 1 cell between panel content and border
- **Label gap**: 1 cell between icon/prefix and text

### Bold and underline semantics

| Usage | Attribute |
|-------|-----------|
| Active panel title | Bold |
| Selected list item label | Bold |
| Keyboard shortcut keys (in help) | Bold |
| Clickable hyperlinks (if OSC 8 enabled) | Underline |
| Focused form field label | Underline |
| Column headers in tables | Bold |

---

## Icon Registry

Use Unicode glyphs for a modern aesthetic, with mandatory ASCII fallbacks.

| Icon | Unicode | ASCII fallback | Meaning |
|------|---------|---------------|---------|
| Error | `✗` U+2717 | `[!]` | Error / failure |
| Success | `✓` U+2713 | `[+]` | Success / healthy |
| Warning | `⚠` U+26A0 | `[~]` | Warning / caution |
| Info | `ℹ` U+2139 | `[i]` | Information |
| Pending | `◌` U+25CC | `...` | Loading / pending |
| Arrow right | `›` U+203A | `>` | Expand / navigate right |
| Arrow down | `▾` U+25BE | `v` | Collapsed → expanded |
| Arrow up | `▴` U+25B4 | `^` | Expanded → collapsed |
| Directory | `▸` U+25B8 | `+` | Tree node with children |
| Search | `⌕` U+2315 | `/` | Search prompt |
| Selected | `●` U+25CF | `*` | Active selection |
| Unselected | `○` U+25CB | ` ` | Inactive item |
| Spinner frames | `⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏` | `\|/-\` | Loading animation |

Detect Nerd Fonts support: attempt to render a known Nerd Font glyph and check if the terminal reports correct width. Fall back to the Unicode tier if undeterminable. Fall back to ASCII if terminal is 7-bit or non-UTF-8.

---

## Border Styles

| Style | Characters | Use case |
|-------|-----------|---------|
| Rounded | `╭ ╮ ╰ ╯ ─ │` | Default panels, modern look |
| Sharp | `┌ ┐ └ ┘ ─ │` | Tables, compact density |
| Double | `╔ ╗ ╚ ╝ ═ ║` | Modal dialogs, emphasis |
| Thick | `┏ ┓ ┗ ┛ ━ ┃` | Focused panel indicator |
| None | (no border) | Inner content areas |

Use `border.focused` token colour on the active panel's border style. Use `border.inactive` on all others. This creates an immediate spatial hierarchy without any text label.

---

## Density Variants

Support three density settings, switchable at runtime or via config:

| Setting | Horizontal padding | Vertical spacing | Row height |
|---------|-------------------|-----------------|-----------|
| `compact` | 1 cell | 0 rows | 1 row per item |
| `normal` (default) | 2 cells | 0 rows | 1 row per item |
| `relaxed` | 2 cells | 1 row | 2 rows per item (label + subtitle) |

htop uses compact density; k9s uses normal; Textual's demo apps use relaxed for readability.
