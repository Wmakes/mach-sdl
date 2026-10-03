# mach-sdl API specification

**Status: draft.** This document describes how the public API *should*
look. It is the target of the rebase in [`roadmap.md`](roadmap.md), not a
description of the current code. Where code and spec disagree, the code is
wrong.

Open points are collected at the end under [Open questions](#open-questions).

## 1. Principles

1. Follow mach and mach-std conventions, not C or SDL conventions.
2. Nothing implicit. Every fallible call says so in its return type.
3. The C layer (`sdl.c`) is internal. No `c.*` type or raw `ptr` appears
   in a public signature.
4. Stay close to SDL3: one SDL concept, one function. No abstraction
   layers on top.
5. Small surface first. Add a function when something needs it.

## 2. Conventions

### 2.1 Modules

The API is split into namespaced modules. There is no flat `sdl.*`
re-export.

```mach
use sdl.window;
use sdl.event;

val win: res[window.Window, sdl.Error] = window.create("demo", 800, 600, 0);
```

| Module            | Contents                                        |
| ----------------- | ----------------------------------------------- |
| `sdl.error`       | `Error` tag, `message()`                        |
| `sdl.core`        | init / quit, subsystem flags, test drivers      |
| `sdl.window`      | `Window`, window management                     |
| `sdl.event`       | `Event` tag, event records, `poll`              |
| `sdl.key`         | scancodes, key modifiers                        |
| `sdl.mouse`       | buttons, cursor, mouse state                    |
| `sdl.gl`          | `GlContext`, buffer swap, proc addresses        |
| `sdl.renderer`    | `Renderer`, drawing, render state               |
| `sdl.texture`     | `Texture`, texture state                        |
| `sdl.rect`        | `Rect`, `FRect`, `Point`, `FPoint`, geometry    |
| `sdl.color`       | `Color`, `FColor`                               |
| `sdl.audio`       | `Stream`, devices                               |
| `sdl.timer`       | ticks, delay, `Clock`                           |
| `sdl.rand`        | SDL random numbers                              |
| `sdl.clipboard`   | clipboard text                                  |
| `sdl.msgbox`      | message boxes                                   |
| `sdl.gamepad`     | *not specified yet* (roadmap Phase 7)           |

`sdl.mach` only exists as the project entry point. It re-exports nothing
except the `Error` tag, so `sdl.Error` is spelled the same everywhere.

### 2.2 Naming

- Functions, variables, fields: `snake_case`.
- Types and tags: `PascalCase`. Tag cases: `snake_case`.
- Constants: `UPPER_SNAKE_CASE`, namespaced by their module
  (`key.ESCAPE`, `window.RESIZABLE`, `mouse.LEFT`).
- A function never repeats its module name: `rand.int`, not
  `rand.rand_int`; `window.create`, not `window.create_window`.
- Standard verbs, used the same way in every module:

  | Verb               | Meaning                                           |
  | ------------------ | ------------------------------------------------- |
  | `create` / `destroy` | allocate / release a resource handle            |
  | `open` / `close`   | acquire / release a device or file-like resource  |
  | `set_x` / `x`      | setter / getter (no `get_` prefix)                |
  | `is_x` / `has_x`   | boolean query, infallible                         |

- Parameters never share a name with an imported module, because the
  parameter would shadow it inside the body. Use `win`, `rend`, `tex`,
  `stream`, not `window`, `renderer`, `texture`.

### 2.3 Return types

SDL3 functions that return `bool` map to mach-std's canonical tags:

| SDL3 shape                                   | mach-sdl return type      |
| -------------------------------------------- | ------------------------- |
| fails, no value                              | `err[sdl.Error]`          |
| fails, produces a value                      | `res[T, sdl.Error]`       |
| genuine absence (nothing pending, no match)  | `opt[T]`                  |
| cannot fail                                  | plain `T`                 |

No fallible function returns a bare `bool`. A plain `bool` is only used
for queries that cannot fail (`is_*`, `has_*`).

Reading follows mach-std (`sel` is the test, the guarded payload is the read):

```mach
val r: err[sdl.Error] = window.set_title(win, "new title");
if (!sel r.ok) { ret r; }
```

### 2.4 Errors

`sdl.Error` is a closed domain tag. The first case is the zero default, so
uninitialized storage never reads as success.

```mach
# why an SDL call did not complete
# ---
# failed:      SDL reported failure; the text is available from error.message()
# invalid:     a nil or invalid handle was rejected before SDL was called
# unsupported: the function or feature is not available on this platform
pub tag Error: u8 {
    failed;
    invalid;
    unsupported;
}
```

The SDL error string is not part of the tag (tags carry closed domains,
never strings). It is read with `error.message() str`. It is valid until the
next SDL call on the same thread.

### 2.5 Handles

Every SDL resource is a single-field `rec` passed by value. The zero value
(nil handle) is always invalid.

```mach
pub rec Window {
    handle: ptr;
}
```

Every handle module provides, where it applies:

- `create(...) res[Handle, sdl.Error]` or `open(...)`
- `destroy(h)` / `close(h)` (infallible, safe on an invalid handle)
- `is_valid(h) bool`

Functions on a resource take the handle as the first parameter.
Destroying a handle does not invalidate copies; the caller owns that
discipline, as with any by-value handle.

### 2.6 Memory

- Strings returned by SDL are plain borrowed `str`. No allocator
  parameters, no `OwnedString`. Each function documents the lifetime:
  - valid until the next SDL call (`error.message`, event text),
  - or released through the module's own `free_*` function
    (`clipboard.text` / `clipboard.free_text`), because SDL hands that
    string out as a heap allocation.
- The caller never calls `SDL_free` directly. Any other SDL allocation is
  copied or released inside the binding.

### 2.7 Closed sets and flags

- A closed set of values (blend mode, flip, scale mode, audio format,
  message box kind) is a `tag`, as in mach-std (`Outcome`, `ParseError`).
  The binding maps tags to SDL's integer values internally.
- Bitmask flags (`window.OPENGL`, `core.VIDEO`, `key.MOD_*`) and large
  numeric tables (scancodes, mouse buttons) stay `val` constants.

### 2.8 Documentation

Every public item carries a doc comment in mach-std style:

```mach
# one-line summary
# ---
# param: description
# ret:   description
```

## 3. Module reference

Signatures only. `T` after `->` is shorthand for the return type.

### 3.1 `sdl.error`

```mach
pub tag Error: u8 { failed; invalid; unsupported; }
pub fun message() str
```

### 3.2 `sdl.core`

```mach
pub val VIDEO:  u32
pub val AUDIO:  u32
pub val EVENTS: u32

pub fun init(flags: u32) err[sdl.Error]
pub fun quit()

# headless drivers, for tests and CI
pub fun use_dummy_video() err[sdl.Error]
pub fun use_dummy_audio() err[sdl.Error]
```

### 3.3 `sdl.window`

```mach
pub rec Window { handle: ptr; }
pub rec Size { w: i32; h: i32; }

pub val OPENGL:    u64
pub val RESIZABLE: u64

pub fun create(title: str, w: i32, h: i32, flags: u64) res[Window, sdl.Error]
pub fun destroy(win: Window)
pub fun is_valid(win: Window) bool

pub fun set_title(win: Window, title: str) err[sdl.Error]
pub fun size(win: Window) res[Size, sdl.Error]
pub fun set_size(win: Window, w: i32, h: i32) err[sdl.Error]
pub fun set_position(win: Window, x: i32, y: i32) err[sdl.Error]
pub fun set_fullscreen(win: Window, on: bool) err[sdl.Error]
pub fun set_min_size(win: Window, w: i32, h: i32) err[sdl.Error]
pub fun set_max_size(win: Window, w: i32, h: i32) err[sdl.Error]
pub fun minimize(win: Window) err[sdl.Error]
pub fun maximize(win: Window) err[sdl.Error]
pub fun restore(win: Window) err[sdl.Error]
pub fun raise(win: Window) err[sdl.Error]
```

`open_gl_window` is dropped: `create(..., window.OPENGL)` covers it.

### 3.4 `sdl.event`

Events are a tag with typed payloads. Nothing from `sdl.c` is exposed.

```mach
pub rec WindowEvent  { window: u32; width: i32; height: i32; }
pub rec KeyEvent     { window: u32; scancode: u32; mods: u16; repeat: bool; }
pub rec MouseMotion  { window: u32; x: f32; y: f32; dx: f32; dy: f32; buttons: u32; }
pub rec MouseButton  { window: u32; x: f32; y: f32; button: u8; clicks: u8; }
pub rec MouseWheel   { window: u32; x: f32; y: f32; dx: f32; dy: f32; }
pub rec TextInput    { window: u32; text: str; }

pub tag Event: u8 {
    unknown: u32;                 # raw SDL event type, not mapped yet
    quit;
    window_close: WindowEvent;
    window_resized: WindowEvent;
    window_focus_gained: WindowEvent;
    window_focus_lost: WindowEvent;
    key_down: KeyEvent;
    key_up: KeyEvent;
    mouse_motion: MouseMotion;
    mouse_button_down: MouseButton;
    mouse_button_up: MouseButton;
    mouse_wheel: MouseWheel;
    text_input: TextInput;
}

pub fun poll() opt[Event]
```

- `poll` returns `none` when the queue is empty.
- `TextInput.text` is borrowed and valid until the next `poll`.
- `key_is_fresh_press` is dropped: a fresh press is
  `key_down` with `repeat == false`.

Usage:

```mach
for (true) {
    val e: opt[event.Event] = event.poll();
    if (!sel e.some) { brk; }
    if (sel e.some.quit) { running = false; }
    if (sel e.some.key_down) {
        if (e.some.key_down.scancode == key.ESCAPE) { running = false; }
    }
}
```

### 3.5 `sdl.key`

```mach
pub val A: u32          # ... Z
pub val NUM_0: u32      # ... NUM_9
pub val RETURN, ESCAPE, BACKSPACE, TAB, SPACE: u32
pub val F1: u32         # ... F12
pub val RIGHT, LEFT, DOWN, UP: u32
pub val LCTRL, LSHIFT, LALT, RCTRL, RSHIFT, RALT: u32

# modifier bits, tested against KeyEvent.mods
pub val MOD_SHIFT: u16
pub val MOD_CTRL:  u16
pub val MOD_ALT:   u16
pub val MOD_GUI:   u16

pub fun is_down(scancode: u32) bool
pub fun mods() u16
```

### 3.6 `sdl.mouse`

```mach
pub val LEFT, MIDDLE, RIGHT, X1, X2: u8

pub rec State { x: f32; y: f32; buttons: u32; }

pub fun state() State
pub fun relative_state() State
pub fun warp(win: Window, x: f32, y: f32)
pub fun show_cursor() err[sdl.Error]
pub fun hide_cursor() err[sdl.Error]
pub fun set_relative(win: Window, on: bool) err[sdl.Error]
pub fun is_relative(win: Window) bool
pub fun capture(on: bool) err[sdl.Error]
```

### 3.7 `sdl.gl`

```mach
pub rec GlContext { handle: ptr; }

pub fun create_context(win: Window) res[GlContext, sdl.Error]
pub fun destroy_context(ctx: GlContext)
pub fun swap(win: Window) err[sdl.Error]
pub fun proc_address(name: str) opt[ptr]
```

### 3.8 `sdl.rect` and `sdl.color`

Plain value records with the exact C layout (checked by tests).

```mach
pub rec Point  { x: i32; y: i32; }
pub rec FPoint { x: f32; y: f32; }
pub rec Rect   { x: i32; y: i32; w: i32; h: i32; }
pub rec FRect  { x: f32; y: f32; w: f32; h: f32; }

pub fun intersects(a: FRect, b: FRect) bool
pub fun intersection(a: FRect, b: FRect) opt[FRect]
pub fun contains(r: FRect, p: FPoint) bool

pub rec Color  { r: u8; g: u8; b: u8; a: u8; }
```

`intersection` returns `opt` instead of an unspecified rect when the
inputs do not overlap. `contains` is inclusive on the near edge and
exclusive on the far edge.

### 3.9 `sdl.renderer`

```mach
pub rec Renderer { handle: ptr; }

pub fun create(win: Window) res[Renderer, sdl.Error]
pub fun destroy(rend: Renderer)
pub fun is_valid(rend: Renderer) bool

pub fun set_color(rend: Renderer, c: Color) err[sdl.Error]
pub fun clear(rend: Renderer) err[sdl.Error]
pub fun present(rend: Renderer) err[sdl.Error]

pub fun fill_rect(rend: Renderer, r: FRect) err[sdl.Error]
pub fun draw_rect(rend: Renderer, r: FRect) err[sdl.Error]
pub fun draw_line(rend: Renderer, a: FPoint, b: FPoint) err[sdl.Error]
pub fun draw_points(rend: Renderer, pts: *FPoint, count: usize) err[sdl.Error]
pub fun draw_lines(rend: Renderer, pts: *FPoint, count: usize) err[sdl.Error]
pub fun fill_rects(rend: Renderer, rects: *FRect, count: usize) err[sdl.Error]

pub fun set_target(rend: Renderer, tex: Texture) err[sdl.Error]
pub fun reset_target(rend: Renderer) err[sdl.Error]
pub fun set_clip(rend: Renderer, r: Rect) err[sdl.Error]
pub fun clear_clip(rend: Renderer) err[sdl.Error]
pub fun clip(rend: Renderer) Rect
pub fun set_scale(rend: Renderer, x: f32, y: f32) err[sdl.Error]
pub fun scale(rend: Renderer) FPoint
pub fun set_blend(rend: Renderer, mode: Blend) err[sdl.Error]

pub fun draw_texture(rend: Renderer, tex: Texture, dst: FRect) err[sdl.Error]
pub fun draw_texture_rotated(rend: Renderer, tex: Texture, dst: FRect,
                             angle: f64, center: FPoint, flip: Flip) err[sdl.Error]
```

Closed sets are tags, not integer constants:

```mach
pub tag Blend: u8 { none; blend; blend_premultiplied; add; add_premultiplied; mod; mul; }
pub tag Flip:  u8 { none; horizontal; vertical; }
```

(The tags map to SDL's bitmask values inside the binding.)

### 3.10 `sdl.texture`

```mach
pub rec Texture { handle: ptr; }
pub rec Size { w: f32; h: f32; }

pub tag ScaleMode: u8 { nearest; linear; }

pub fun load(rend: Renderer, path: str) res[Texture, sdl.Error]
pub fun destroy(tex: Texture)
pub fun is_valid(tex: Texture) bool

pub fun size(tex: Texture) res[Size, sdl.Error]
pub fun set_blend(tex: Texture, mode: Blend) err[sdl.Error]
pub fun blend(tex: Texture) res[Blend, sdl.Error]
pub fun set_color_mod(tex: Texture, c: Color) err[sdl.Error]
pub fun color_mod(tex: Texture) res[Color, sdl.Error]
pub fun set_alpha_mod(tex: Texture, a: u8) err[sdl.Error]
pub fun alpha_mod(tex: Texture) res[u8, sdl.Error]
pub fun set_scale_mode(tex: Texture, mode: ScaleMode) err[sdl.Error]
pub fun scale_mode(tex: Texture) res[ScaleMode, sdl.Error]
```

### 3.11 `sdl.audio`

```mach
pub rec Stream { handle: ptr; }
pub tag Format: u8 { s16; f32; }

pub fun open(format: Format, channels: i32, freq: i32) res[Stream, sdl.Error]
pub fun close(stream: Stream)
pub fun is_valid(stream: Stream) bool

pub fun put(stream: Stream, data: *u8, len: usize) err[sdl.Error]
pub fun resume(stream: Stream) err[sdl.Error]
pub fun pause(stream: Stream) err[sdl.Error]
pub fun clear(stream: Stream) err[sdl.Error]
pub fun queued(stream: Stream) i32
```

Device enumeration is out of scope for v1 (see section 5).

### 3.12 `sdl.timer`

```mach
pub rec Clock { last: u64; }

pub fun ticks_ms() u64
pub fun ticks_ns() u64
pub fun delay(ms: u32)

pub fun clock() Clock
pub fun tick(c: *Clock) f64          # seconds since the previous tick
```

### 3.13 `sdl.rand`

```mach
pub fun seed(s: u64)
pub fun int(n: i32) i32              # [0, n)
pub fun f32() f32                    # [0.0, 1.0)
pub fun range(lo: f32, hi: f32) f32
pub fun bool() bool
```

### 3.14 `sdl.clipboard`

```mach
pub fun has_text() bool
pub fun set_text(text: str) err[sdl.Error]
pub fun text() res[str, sdl.Error]    # release with free_text
pub fun free_text(text: str)
```

### 3.15 `sdl.msgbox`

```mach
pub tag Kind: u8 { info; warn; error; }

pub fun show(kind: Kind, title: str, message: str, win: Window) err[sdl.Error]
```

`info`, `warn` and `error` convenience functions are dropped; `show`
with a `Kind` replaces them. Pass an invalid `Window` for a parentless box.

## 4. Testing

- Every module has `test` blocks (identifier names, per the compiler).
- Struct layouts of every transcribed C record are asserted with
  `$size_of` in tests.
- Functional tests run under the dummy video/audio drivers
  (`core.use_dummy_video`, `core.use_dummy_audio`).

## 5. Out of scope

- Gamepad/joystick (Phase 7, separate design).
- Audio device enumeration.
- Haptics, camera, sensors, 3D GPU API.
- Anything not exposed by SDL3 core.

## Decisions

- Closed sets are tags; flags and scancodes are `val` constants (2.7).
- SDL strings are borrowed `str`, lifetime documented per function (2.6).
- Bulk draw calls take a pointer and a count (`*FPoint`, `*FRect`), not a
  std view.
- Audio device enumeration is deferred past v1.

## Open questions

1. **Error detail.** Is a bare `failed` plus `error.message()` enough, or
   should `Error` carry more structure (e.g. which subsystem)?
2. **`sdl.mach` role.** Does the compiler need it to re-export anything for
   the library artifact? (Answered by trying it in roadmap Phase 2.)
3. **Event coverage.** Keep v1 to the set in 3.4 plus `unknown`, or add
   more of SDL3's events (drop file, display changes, ...)?