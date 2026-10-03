# mach-sdl

SDL3 bindings for the [mach](https://github.com/briar-systems/mach) programming language.

## Requirements

* A mach compiler matching the required version range (`mach info` prints your version).
* **Linux:** The SDL3 development files, so that `libSDL3.so` (not just `libSDL3.so.0`) exists in a standard library directory such as `/usr/lib` or `/usr/lib/<arch>-linux-gnu`.
* **Windows:** After building your `.exe`, you need to manually copy `SDL3.dll` into the same folder as the executable.

## Status

`mach-sdl` is still a work in progress and does **not yet implement the entirety of SDL3**. Some APIs and functionality may be missing or incomplete.

If you need an SDL3 feature that isn't currently available in `mach-sdl`, you're encouraged to implement it and open a pull request. Contributions that help expand the bindings and improve SDL3 coverage are very welcome.

macOS support is currently untested, as I don't own a Mac to test it on.
 
## License

GNU Lesser General Public License v3.0. See `COPYING.LESSER` and `LICENSE`.
