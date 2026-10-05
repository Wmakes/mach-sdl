Setup notes for using [mach-sdl](README.md) on macOS.

# TESTED ONLY ON DARWIN-X64 # 

# macOS setup

mach looks up the darwin link inputs as `/usr/local/lib/lib<name>.dylib`. The SDL `.dmg` releases from GitHub (`SDL3-*.dmg`, `SDL3_ttf-*.dmg`, `SDL3_image-*.dmg`, `SDL3_mixer-*.dmg`) contain `.xcframework` bundles instead, which mach can't link directly:

* it only accepts shared libraries with a `.dylib` extension, so the bare framework binary (`SDL3.framework/Versions/A/SDL3`) is rejected;
* a `libSDL3.dylib` symlink to that binary is rejected too (`link.rpath_mismatch`), because the binary's install name is `@rpath/SDL3.framework/Versions/A/SDL3` and mach requires the file path to end in that suffix;
* `source = "framework"` only refers to `/System/Library/Frameworks`, which is reserved for Apple's frameworks.

So each framework binary is copied out as a plain dylib with a matching `@rpath/lib<name>.dylib` install name. When linking, mach finds it in `/usr/local/lib` and records that directory as the executable's runtime search path (`LC_RPATH`), so nothing else needs to be configured.

1. Open the four `.dmg` files (they mount under `/Volumes`).
2. Run:

```sh
LIB=/usr/local/lib
mkdir -p "$LIB"
for n in SDL3 SDL3_ttf SDL3_image SDL3_mixer; do
    fw=$(echo /Volumes/*/$n.xcframework/macos-*/$n.framework)
    cp "$fw/Versions/A/$n" "$LIB/lib$n.dylib"
    install_name_tool -id "@rpath/lib$n.dylib" "$LIB/lib$n.dylib"
    # SDL3_ttf, SDL3_image and SDL3_mixer must load the converted SDL3, not a framework
    [ "$n" != SDL3 ] && install_name_tool -change \
        @rpath/SDL3.framework/Versions/A/SDL3 @rpath/libSDL3.dylib "$LIB/lib$n.dylib"
    # install_name_tool invalidates the signature; re-sign ad hoc
    codesign --force --sign - "$LIB/lib$n.dylib"
done
# optional codecs (avif, jxl, webp, ogg, opus, ...) stay frameworks next to the dylibs
cp -R /Volumes/*/optional/*.xcframework/macos-*/*.framework "$LIB/"
```

3. Check it: `otool -L /usr/local/lib/libSDL3_mixer.dylib` should list `@rpath/libSDL3_mixer.dylib` and `@rpath/libSDL3.dylib`.

On a stock Mac, `/usr/local/lib` may be owned by root; if `mkdir`/`cp` fail with a permission error, run the script with `sudo`.

The built executable loads the libraries from `/usr/local/lib` at run time. To ship it to a machine without them, put the dylibs (and codec frameworks) next to the executable and add an rpath: `install_name_tool -add_rpath @executable_path <exe>`, then `codesign --force --sign - <exe>`.
