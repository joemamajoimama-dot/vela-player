# Making the Vela fixes permanent

## Why the fixes seemed to disappear

The fixes were applied to files on disk with `sed` and `cat`, and then built
into `build/vela`. They are lost when:

- you unzip a fresh copy of the source (the original zip doesn't have them),
- you launch a different copy of Vela (an installed one, or another folder),
- you delete `build/` and rebuild from files that were never edited.

The fix has to live in the source files **and** be rebuilt.

## 1. Use the zip that already contains every fix

```sh
cd ~/Downloads
rm -rf vela-final && mkdir vela-final
unzip -q vela-fixed.zip -d vela-final
cd vela-final/vela
```

Check the fixes are present (each line should print a match):

```sh
grep -n 'maximumWidth: 200'        qml/views/SettingsView.qml
grep -n 'allheaders'               CMakeLists.txt
grep -n 'QSize size{'              src/vlcplayer.cpp
grep -n '#include "vlcplayer.h"'   src/videoitem.h
grep -n 'property color color'     qml/components/Icon.qml
```

## 2. Rebuild and always run this build

```sh
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -Wno-dev
ninja -C build
./build/vela
```

After editing any `.qml` or `.cpp` file, run `ninja -C build` again. QML is
compiled into the binary, so changes don't show until it is rebuilt.

## 3. Install it so the app menu launches the fixed build

```sh
sudo cmake --install build
```

Without this, the app menu may still start an older copy.

## 4. Keep the fixes in git

```sh
git init
git add -A
git commit -m "Fix Customize layout and Qt 6.11 / GCC 16 build"
```

## Settings that persist

Customize choices (theme, accent, sliders, toggles) are saved by Qt in
`~/.config/Vela/`. They survive relaunches. **Reset to theme defaults** clears
the overrides but keeps the selected theme.

## If it is still broken after relaunch

Run from the build folder and send the output:

```sh
QT_LOGGING_RULES="qml*=true" ./build/vela 2>&1 | grep -v portal
```
