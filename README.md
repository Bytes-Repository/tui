# tui

A [Bubble Tea](https://github.com/charmbracelet/bubbletea) / [Lip Gloss](https://github.com/charmbracelet/lipgloss)
-inspired terminal UI toolkit for **H#**, distributed as a `bytes` package.

Model → Update → View, a synchronous "Cmd" pipeline, ANSI styling with padding/
borders/alignment, and a handful of ready-made components (spinner, progress
bar, text input, scrollable viewport, selectable list, paginator, help bar,
timer). **100% H#** — there is no `extern` FFI block anywhere in this
library.

## Install

```
bytes add tui
```

or copy this directory into your project and add it as a workspace member /
local dependency — see `Bytes.hk`.

## Quick start

```h#
use "bytes -> tui" from "tui"

fn init() -> any is
    return tui::spinner_new("dot")
end

fn update(model: any, msg: any) -> any is
    if tui::is_ctrl_c(msg) || tui::msg_is_key(msg, "q") is
        return (model, tui::cmd_quit())
    end
    if tui::msg_is_tick(msg) is
        return (tui::spinner_tick(model), tui::cmd_none())
    end
    return (model, tui::cmd_none())
end

fn view(model: any) -> string is
    return tui::spinner_view(model) + " doing work... (press q to quit)"
end

tui::run(init, update, view)
```

Run it with the interpreter — **not** the native/LLVM compiler (see
"Why interpreter-only?" below):

```
h# preview main.h#
# or, from inside a bytes project:
bytes run
```

## Why interpreter-only?

Reading a single keystroke without waiting for Enter needs a byte-at-a-time
read from stdin. H#'s interpreter has a real primitive for that
(`__builtin_io_read_char`, wrapped as `std/io.h#`'s `read_char()`), but there
is still no LLVM/native-codegen implementation of it — `h# build` /
`bytes build --release` will fail at compile time with a clear "not
supported by the LLVM backend" error if a program reaches it, rather than
silently misbehaving. This isn't a limitation `tui` invented — it's the
current state of the H# toolchain itself — so build/run any `tui` program
through `h# preview` / `bytes run` (the interpreter/JIT path).

Raw terminal mode (no line-buffering, no local echo) is switched on with the
`stty` utility, shelled out to exactly the way `std/term.h#` already does to
read the terminal's size — this is *not* the `extern` FFI feature, it's a
plain subprocess call through the same process builtins the standard library
itself uses.

## A note on types

Every public function that hands you back one of this library's own values
(a message, a `Style`, a `Spinner`, ...) types it as `any`, and every example
in this README does too — `let sp = tui::spinner_new("dot")`, never
`let sp: Spinner = ...`. This isn't a style nitpick: when H# pulls in a
package with `use "bytes -> tui" from "..."` it mangles that package's type
names with the alias you chose, so there's no single fixed spelling of e.g.
`Spinner` for your code to name safely. Field access (`msg.kind`) and this
library's accessor functions (`tui::msg_key(msg)`, `tui::spinner_view(sp)`,
...) work fine regardless — that's how every example here reads a value back
out, exactly the pattern `std/sync.h#`'s `Mutex` and `Channel` already use.

## API overview

**Terminal control** — `term_width()` `term_height()` `term_size()`
`enable_raw_mode()` `disable_raw_mode(saved)` `enter_alt_screen()`
`exit_alt_screen()` `hide_cursor()` `show_cursor()` `clear_screen()`
`move_home()` `move_to(row, col)` `bell()` `set_title(t)` `draw(frame)`

**Keyboard** — `poll_key()` (non-blocking, ~100ms), `read_key_blocking()`,
`is_special_key(key)`. Key names: `"up" "down" "left" "right" "home" "end"
"pgup" "pgdown" "delete" "backspace" "tab" "enter" "esc" "space" "ctrl+c"
"ctrl+a" "ctrl+e" "ctrl+d" "ctrl+u" "ctrl+k" "ctrl+w"`, or a literal
character. **Verified against a real build of the H# interpreter** (see
"Verified against a real build" below): ASCII keys — letters, digits,
punctuation, arrows, enter, backspace, tab, ctrl+combos — all read back
correctly. Non-ASCII keys (Polish `ą ę ś ć ł ż źn`, emoji, ...) are still
grouped into one `poll_key()` event each rather than splintering into
several confusing fragments, but the character *value* comes through
corrupted — this is a bug in the H# interpreter's own `__builtin_io_read_char`
(it round-trips each raw byte through Rust's `char`, which is lossy for
anything ≥ 128), not something fixable from library code. Practically:
typing plain ASCII text into `textinput` works correctly; typing e.g. "ą"
does not insert "ą".

**Messages** — `key_msg` `tick_msg` `resize_msg` `quit_msg` `none_msg`
`custom_msg`, and readers `msg_kind` `msg_key` `msg_data` `msg_width`
`msg_height`, and predicates `msg_is` `msg_is_key` `msg_is_any_key`
`msg_is_tick` `msg_is_resize` `msg_is_quit` `msg_is_custom` `is_ctrl_c`.

**Commands** — `cmd_none()` `cmd_quit()` `cmd_custom(data)` `cmd_tick(ms)`,
or write your own zero-argument closure returning a message.

**The program loop** — `run(init, update, view)` (idle-tick every ~100ms so
animations keep moving) and `run_no_tick(init, update, view)` (blocks for a
keypress instead, for static screens). Both return the final model when the
program quits, and both bail out on Ctrl+C unconditionally so a broken
`update` can never strand your terminal in raw mode.

**Color & style** — `colorize(text, color)`, and the `Style` box model:
`style_new()` and `style_fg/bg/bold/italic/underline/width/align/padding/
border/border_color(...)` (each returns a modified copy), fluent sugar
`.with_fg(...)` etc. + `.render(text)`, and the plain function
`style_render(style, text)`. Colors: `""`, `"#rrggbb"`, an ANSI 256 index as
a decimal string, or a name (`black red green yellow blue magenta cyan
white`, and `bright_*` variants). Borders: `"none" "normal" "rounded"
"thick" "double"`.

**Layout** — `join_horizontal(blocks, gap)`, `join_vertical(blocks)`,
`center_block(s, width, height)`.

**Components** — `spinner_*` (kinds: `dot line circle arrow bounce pulse
bar`), `progress_*`, `textinput_*`, `viewport_*`, `list_*` (as `ListModel`),
`paginator_*`, `help_view(keys, descriptions)`, `timer_*`
(start/stop/reset/elapsed/`mm:ss` view).

See `examples/` for three complete programs: a spinner, a text-input form,
and a small styled dashboard combining several components.

## Verified against a real build

Earlier drafts of this library were checked only by reading H#'s own
compiler/interpreter source (module resolution, the builtin dispatch table,
string/slice semantics) — no locally built `h#` binary, since the full
toolchain needs LLVM 21. That's since been improved: `hsharp-parser` and
`hsharp-interpreter` (the two crates that don't need LLVM — that's only
`hsharp-compiler`'s native/AOT backend) build fine with plain `rustc`/`cargo`
1.75, so every function in `src/lib.h#` has now actually been **parsed and
executed** against a real build of the interpreter, including feeding raw
bytes through `poll_key()` for arrow keys / enter / backspace / ctrl+c /
tab / space / a plain letter. 44 checks, 0 failures. That process caught
three real bugs, two in this library (now fixed) and one in H# itself:

- **Fixed** — a bare top-level statement like `tui::run(init, update, view)`
  is not valid H#; only `fn`/`struct`/`enum`/`trait`/`impl`/`type`/`const`/
  `mod`/`extern`/`use` are allowed at the top level (confirmed straight from
  the parser). All three `examples/` now correctly wrap that call in
  `fn main() is ... end`, since `run_module` specifically looks for and
  calls a function named `main`.
- **Not a bug, but worth knowing** — H# doesn't support chaining a call
  directly onto another call's result, i.e. `some_fn()()` doesn't do what
  it looks like it does. Store the intermediate result first:
  `let cmd = some_fn()` then `cmd()`. `run`/`run_no_tick` already do this
  correctly; if you write your own `Cmd`-calling code, do the same.
- **An H# interpreter bug, not this library's** — see the UTF-8 caveat
  under "Keyboard" above; `__builtin_io_read_char` mis-encodes any raw byte
  ≥ 128, which corrupts non-ASCII key input at the source.

## Other honest limitations

- `style_render`/the components don't reflow long lines to fit a given
  width — they pad short lines and leave long ones as-is.
- `run`'s redraw does a full-screen clear every frame rather than diffing
  against the previous frame, so fast repaints may flicker on some
  terminals. Good enough for a first version; a future one could track the
  previous frame and only rewrite changed lines.
- `Cmd`s run synchronously on the render thread (H# only has real
  concurrency for shell/HTTP tasks today, not arbitrary user closures) — a
  slow `cmd` will visibly pause the UI. `cmd_tick` documents this; keep
  your own `Cmd`s fast.
- The interactive `run`/`run_no_tick` loop itself (the part that needs a
  real tty: raw mode, the render/redraw cycle, live key polling) could not
  be exercised end-to-end in the sandboxed environment this was built in
  (no real terminal to attach). Every pure function it's built from has
  been verified directly (see above); the loop that wires them together is
  short and was re-read carefully, but hasn't been watched running on a
  real screen. Please try the examples and report back if anything looks
  off.
