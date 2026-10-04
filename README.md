# mach-sdl

[SDL3](https://github.com/libsdl-org/SDL) (and [SDL3_ttf](https://github.com/libsdl-org/SDL_ttf), [SDL3_image](https://github.com/libsdl-org/SDL_image) and [SDL3_mixer](https://github.com/libsdl-org/SDL_mixer)) bindings for the [mach](https://github.com/briar-systems/mach) programming language.

## Requirements

* A mach compiler matching the required version range (`mach info` prints your version).
* **Linux:** The SDL3 development files, so that `libSDL3.so`, `libSDL3_ttf.so`, `libSDL3_image.so` and `libSDL3_mixer.so` (not just the `.so.0` symlinks) are available in a standard library directory such as `/usr/lib` or `/usr/lib/<arch>-linux-gnu`.
* **Windows:** After building your `.exe`, you need to manually copy `SDL3.dll`, `SDL3_ttf.dll`, `SDL3_image.dll` and `SDL3_mixer.dll` into the same directory as the executable.

## Usage

Add this repo as a dependency in your `mach.toml`:

```toml
[dep.sdl]
git = "https://github.com/Wmakes/mach-sdl"
ref = "branch/master"
```

Then import the modules you need, e.g. `use sdl.window;`.

## Status

`mach-sdl` is still a work in progress and does **not yet implement the entirety of SDL3**. Some APIs and functionality may be missing or incomplete.

If you need an SDL3 feature that isn't currently available in `mach-sdl`, you're encouraged to implement it and open a pull request. Contributions that help expand the bindings and improve SDL3 coverage are very welcome.

macOS support is currently untested, as I don't own a Mac to test it on.
 
## License

GNU Lesser General Public License v3.0. See `COPYING.LESSER` and `LICENSE`.
