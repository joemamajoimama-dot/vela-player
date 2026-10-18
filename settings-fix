# Fix: Customize page empty / controls missing, and build fixes for Qt 6.11 + GCC 16

Tested on Nobara (Fedora 44 base), Qt 6.11.2, GCC 16.2.1, libVLC 3.0.24.

## Symptom

Opening **Customize** (sidebar or `Ctrl+,`) shows only the section list
(Look & feel, Player, Playback, …) stretched across the whole window.
The settings cards (theme gallery, sliders, toggles, font field) never
appear, so nothing can be changed.

## Cause

In `qml/views/SettingsView.qml` the section rail is a `ColumnLayout` with
`Layout.preferredWidth: 200` inside a `RowLayout`. Its children
(`SidebarItem`) use `Layout.fillWidth: true`, and the rail grows to take
all the free width, leaving the settings `Flickable` with none.

## Fix

Pin the rail to 200 px:

```qml
ColumnLayout {
    Layout.alignment: Qt.AlignTop
    Layout.preferredWidth: 200
    Layout.minimumWidth: 200
    Layout.maximumWidth: 200
    Layout.fillWidth: false
    spacing: 4
    ...
}
```

One-liner:

```sh
sed -i 's|^\(\s*\)Layout.preferredWidth: 200$|&\n\1Layout.minimumWidth: 200\n\1Layout.maximumWidth: 200\n\1Layout.fillWidth: false|' qml/views/SettingsView.qml
```

## Related customization fixes

- **Font setting only affected buttons.** Plain QML `Text` does not inherit the
  window font. `src/main.cpp` now applies the chosen `fontFamily` to the
  application font and updates it live when the theme changes.
- **Text size slider ignored by most text.** Fixed pixel sizes in `qml/`
  now multiply by `Theme.opt.fontScale`:

  ```sh
  sed -i -E '/fontScale/! s/(pixelSize: )([0-9]+(\.[0-9]+)?)( *[;}]|[[:space:]]*$)/\1\2 * Theme.opt.fontScale\4/' $(grep -rl pixelSize qml)
  ```

## Build fixes needed on Qt 6.11 / GCC 16

1. **`Meta Types must be fully defined`** – `src/videoitem.h` forward-declared
   `VlcPlayer` but uses it as a `Q_PROPERTY`. Replace `class VlcPlayer;` with
   `#include "vlcplayer.h"`.
2. **`capture of non-variable 'size'`** – vexing parse in `src/vlcplayer.cpp`:
   `const QSize size(int(width), int(height));` becomes
   `const QSize size{int(width), int(height)};`.
3. **`'HistoryModel' was not declared`** in `vela_qmltyperegistrations.cpp` –
   the generated file lacked includes. Force-include the model headers: add
   `src/allheaders.h` including `vlcplayer.h`, `videoitem.h`,
   `thememanager.h`, `librarymodel.h`, `playlistmodel.h`, `historymodel.h`, and
   in `CMakeLists.txt`:

   ```cmake
   target_compile_options(vela PRIVATE "SHELL:-include ${CMAKE_CURRENT_SOURCE_DIR}/src/allheaders.h")
   ```
4. **`Cannot override FINAL property`** in `Icon.qml` – Qt 6.11's `IconImage`
   has a final `name`. Wrap it in an `Item` that owns `name`, `size` and
   `color` and holds an `Impl.IconImage` filling it.

## Build

```sh
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -Wno-dev
ninja -C build
./build/vela
```

The `qt.qpa.services ... portal` warning on start is harmless.

## Known gap

Density (Compact / Cozy / Roomy) only changes spacing in a few places
(`Theme.opt.space` is used in about 15 spots).
