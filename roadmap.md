# Roadmap

mach-sdl provides [SDL3](https://www.libsdl.org/) bindings for the
[mach](https://machlang.org/) programming language.

The current public API grew organically and has three problems: it is
inconsistent, incomplete, and does not follow mach's conventions. This
roadmap describes the rebase of the whole public API. The target design is
specified in [`spec.md`](spec.md) (to be written in Phase 1).

## Status

- Builds on the latest mach compiler (tests are named with identifiers).
- Gamepad/joystick support has been removed. It will return as a new module
  designed from scratch (Phase 7).
- The raw layer (`src/c.mach`) works and stays as is.
- The public layer is being redesigned. Expect breaking changes.

## Problems with the current API

### Inconsistent

- `Window` is a typed `rec` in some modules and a raw `ptr` in others
  (e.g. `mouse.set_relative_mode(window: ptr)`).
- `Renderer` and `Texture` are bare `ptr`s, so the compiler cannot tell
  them apart.
- Naming varies: `create_window` / `destroy_window` but `audio.destroy`,
  `window_is_valid` / `stream_is_valid`, `rand.rand_int`, `new_clock`,
  `clear_stream`.
- Everything lives in one flat namespace: `sdl.A`, `sdl.F`, `sdl.UP` sit
  next to `sdl.BLEND_*` and `sdl.EVENT_*`.

### Incomplete

- Events return raw `c.*Event` structs and require a `*c.Event`, so the C
  layer leaks into the public API.
- `audio.get_playback_devices` returns a raw `ptr` the caller must free.
- Missing: key modifiers, audio loading, surfaces, text, and many window
  and renderer functions.

### Not idiomatic mach

- Every fallible function returns `bool` and the cause is fetched with a
  separate `get_error()`. This is C style. mach has canonical
  `res` / `opt` / `err` types in `std.types`.
- `sdl.mach` re-exports every symbol by hand with `fwd`, so each new
  function has to be added in two places.

## Design decisions

1. **Namespaced modules instead of a flat `sdl.*`.**
   Usage becomes `sdl.window.create(...)`, `sdl.key.A`, `sdl.event.poll(...)`.
   This removes the `fwd` list in `sdl.mach` and the name collisions.
   Function names drop their module prefix (`rand.int`, not `rand.rand_int`).
2. **Typed handles.**
   Every resource is its own single-field `rec`: `Window`, `Renderer`,
   `Texture`, `Stream`, `GlContext`. No bare `ptr` in the public API.
3. **Errors via `res` / `opt` / `err`**, not `bool` plus `get_error`. mach-std
   error tags are closed domains and never carry strings, so `sdl.Error` is a
   small tag and the SDL error text is read with `error.message()`.
4. **Typed events.**
   No `c.*` types in the public API. A single `poll` returns an `opt` with
   the event.
5. **`c.mach` stays raw** and is not part of the public contract.
6. **Spec first.**
   `spec.md` describes how the API *should* look, not how it looks today.
   Code is brought in line with the spec, not the other way around.
7. **No over-engineering.**
   Only what SDL3 requires. No abstraction layers on top of SDL.

## Phases

### Phase 0: groundwork (done)

- [x] Fix the build for the latest mach compiler.
- [x] Remove gamepad support.

### Phase 1: spec

- [x] Study how mach-std defines and uses `res` / `opt` / `err`.
- [x] Write `spec.md` (draft): module list, types, signatures, naming
      conventions, error model, usage examples.
- [x] Review and freeze the spec before writing code.

### Phase 2: conventions and skeleton

- [ ] `src/` layout: one file per module, `sdl.mach` without the `fwd` list
      (or a minimal one if the compiler requires an entry point).
- [ ] Shared SDL error type.
- [ ] One handle pattern (`rec` + validity check + destroy) used everywhere.

### Phase 3: core, window, event

- [ ] `sdl.core`: init/quit, init flags, dummy-driver mode for tests.
- [ ] `sdl.window` on a typed `Window`, with the full window API.
- [ ] `sdl.event` with typed events instead of `c.*`.

### Phase 4: input

- [ ] `sdl.key`: scancodes namespaced in the module, key modifiers.
- [ ] `sdl.mouse`: everything takes `Window`, never `ptr`.

### Phase 5: graphics

- [ ] `sdl.renderer` with typed `Renderer` and `Texture`.
- [ ] `sdl.gl` with a typed `GlContext`.
- [ ] Surfaces and image loading (scope to be decided in the spec).

### Phase 6: everything else

- [ ] `sdl.audio`: device enumeration without manual freeing, loading, streams.
- [ ] `sdl.timer`, `sdl.rand`, `sdl.rect`, `sdl.clipboard`, `sdl.msgbox`
      moved to the new conventions.

### Phase 7: gamepad (from scratch)

- [ ] Design it in the spec first, then implement.
- [ ] Events and polling, typed `Gamepad`.

### Phase 8: polish

- [ ] Tests for every module.
- [ ] Port the demo (`main.mach`) to the new API.
- [ ] README with a usage example.
- [ ] Verify Windows and Darwin builds, subject to the compiler's linking support.

## Open questions

- Resolved questions about `spec.md` are tracked in its own "Open questions".
- Whether the compiler accepts a minimal `sdl.mach` as the entry point or
  needs explicit re-exports.
- How far to go with surfaces and image loading in the first release.
- Versioning: the rebase breaks the API, so the next release should be
  `0.2.0` or higher.
