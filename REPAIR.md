# REPAIR.md: building Vela with Qt 6.11 and GCC 16

This tree contains the fixes needed to build and start Vela on a current
toolchain. They were found while building on **Nobara Linux (Fedora 44 based)**:

| Component | Version |
|---|---|
| GCC | 16.2.1 |
| Qt | 6.11.2 |
| CMake | 4.3.0 |
| Ninja | 1.13.2 |
| libVLC | 3.0.24 |

Every fix is a small compatibility change. None of them alters features or
behaviour. All code changes are also available as a single patch:
[`patches/qt611-gcc16-fixes.patch`](patches/qt611-gcc16-fixes.patch).

## Quick start

```sh
tools/install-local.sh
```

This installs the dependencies (sudo), builds, installs to `~/.local`, and adds
an app-menu entry and a Desktop shortcut. Use `--no-deps` to skip the sudo step.

To apply the fixes to a pristine checkout instead:

```sh
git apply patches/qt611-gcc16-fixes.patch
```

## Summary

| # | Symptom | File | Fix | Status |
|---|---|---|---|---|
| 0 | `ninja: error: manifest 'build.ninja' still dirty after 100 tries` | (environment) | Reset file timestamps | Confirmed |
| 1 | `static assertion failed: Meta Types must be fully defined` | `src/videoitem.h` | Include `vlcplayer.h` | Confirmed |
| 2 | `error: capture of non-variable 'size'` | `src/vlcplayer.cpp` | Brace-initialise `QSize` | Confirmed |
| 3 | `error: 'HistoryModel' was not declared in this scope` | `CMakeLists.txt` | Add `src/` to include path | Confirmed |
| 4 | `Cannot override FINAL property` (window never opens) | `qml/components/Icon.qml` | Wrap `IconImage` in an `Item` | Confirmed |
| 5 | `Invalid property assignment: int expected` (window never opens) | 6 QML files | `Math.round()` on decimal `pixelSize` | Applied, see note |

"Confirmed" means the error disappeared in the build or run log after the fix.

**Note on #5:** the error was reported at `LibraryView.qml:244`. The same
pattern was fixed in every place it occurs (7 spots in 5 files). It had not yet
been confirmed by a full successful launch when this document was written. If
Vela still fails to start, the console prints the next QML error, so run
`~/.local/bin/vela` in a terminal and open an issue with that output.

## Details

### 0. Ninja loops forever: future-dated files

**Symptom**

```
ninja: error: manifest 'build.ninja' still dirty after 100 tries, perhaps system time is not set
```

**Cause.** The files in the source archive were stamped with a modification
time later than the system clock, so Ninja considered every generated file
permanently out of date.

**Fix.** No source change. Reset the timestamps and rebuild from scratch:

```sh
find . -exec touch {} +
rm -rf build
```

`tools/install-local.sh` does the first step automatically. If it still
happens on a fresh checkout, check that your system clock is correct
(`timedatectl`).

### 1. `VideoItem` needs the full `VlcPlayer` type

**File:** `src/videoitem.h`

**Symptom**

```
qmetatype.h: error: static assertion failed: Pointer Meta Types must either
point to fully-defined types or be declared with Q_DECLARE_OPAQUE_POINTER(T *)
qmetatype.h: error: static assertion failed: Meta Types must be fully defined
```

**Cause.** `VideoItem` exposes `Q_PROPERTY(VlcPlayer *player ...)` but only
forward-declared `class VlcPlayer;`. Qt 6.11's moc-generated code requires the
complete type for a property of pointer-to-QObject type.

**Fix**

```diff
-class VlcPlayer;
+#include "vlcplayer.h"
```

### 2. Vexing parse in `VlcPlayer::videoSetup`

**File:** `src/vlcplayer.cpp`, line 583

**Symptom**

```
warning: parentheses were disambiguated as a function declaration [-Wvexing-parse]
error: capture of non-variable 'size'
error: no match for 'operator=' (operand types are 'QSize' and 'const QSize(int, int)')
```

**Cause.** `const QSize size(int(width), int(height));` is parsed as a
declaration of a function named `size`, not a variable. Older compilers
accepted the code; GCC 16 reports an error when the "variable" is captured by
the lambda.

**Fix**

```diff
-    const QSize size(int(width), int(height));
+    const QSize size{int(width), int(height)};
```

### 3. Generated QML registrations cannot find the project headers

**File:** `CMakeLists.txt`

**Symptom.** Every project class is reported as missing in the generated
`vela_qmltyperegistrations.cpp`:

```
error: 'HistoryModel' was not declared in this scope
error: 'LibraryModel' was not declared in this scope
error: 'VideoItem' was not declared in this scope
...
```

**Fix.** Add `src/` to the target's include path:

```cmake
target_include_directories(vela PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src)
```

The build succeeded after this change. The exact reason the registrations
could not see the headers without it was not investigated further.

### 4. `Icon.qml` overrides a FINAL property

**File:** `qml/components/Icon.qml`

**Symptom.** The window never opens; the console shows:

```
qt.qml.propertyCache.append: Final member name is overridden in class Icon_QMLTYPE_3.
QQmlApplicationEngine failed to load component
qrc:/qt/qml/Vela/qml/Main.qml:185:21: Type SidebarItem unavailable
qrc:/qt/qml/Vela/qml/components/SidebarItem.qml:37:9: Type Icon unavailable
qrc:/qt/qml/Vela/qml/components/Icon.qml:8:5: Cannot override FINAL property
```

**Cause.** `Icon` was based directly on Qt's internal `IconImage` and declared
its own `property string name`. In Qt 6.11, `IconImage::name` is `FINAL`, so a
QML type may not redeclare it.

**Fix.** Make `Icon` a plain `Item` that keeps the same public properties
(`name`, `size`, `color`) and draws an `IconImage` inside. Every existing use
of `Icon { ... }` in the project sets only `name`, `size`, `color`, `visible`, `scale`
and layout or anchor properties, so no other file needed to change.

```qml
Item {
    id: root
    property string name
    property int size: 20
    property color color: Theme.c.text
    implicitWidth: size
    implicitHeight: size
    width: size
    height: size

    Impl.IconImage {
        anchors.fill: parent
        source: root.name ? Theme.icon(root.name) : ""
        sourceSize: Qt.size(root.size, root.size)
        color: root.color
        fillMode: Image.PreserveAspectFit
    }
}
```

### 5. Decimal literals assigned to `font.pixelSize`

**Files:** `qml/views/LibraryView.qml`, `PlayerView.qml`, `QueueView.qml`,
`EqualizerSheet.qml`, `TracksSheet.qml`

**Symptom**

```
qrc:/qt/qml/Vela/qml/Main.qml:244:21: Type LibraryView unavailable
qrc:/qt/qml/Vela/qml/views/LibraryView.qml:244:85: Invalid property assignment: int expected
```

**Cause.** `font.pixelSize` is an integer. Literals such as `13.5` were accepted
by older Qt but are rejected at load time by Qt 6.11.

**Fix.** Wrap each decimal literal in `Math.round(...)`, which keeps the
intended size (13.5 becomes 14):

```diff
-font.pixelSize: 13.5
+font.pixelSize: Math.round(13.5)
```

Values changed: `10.5` (x1), `12.5` (x3), `13.5` (x2), `14.5` (x1). Expressions such as
`12.5 * Theme.opt.fontScale` are computed at run time and were left alone.

## Harmless build warning

```
Qt policy QTP0004 is not set: You need qmldir files for each extra directory
that contains .qml files for your module.
```

This is a developer warning from Qt and does not affect the build.
`tools/install-local.sh` hides it with `-Wno-dev`. To fix it properly, set the
policy in `CMakeLists.txt` with `qt_policy(SET QTP0004 NEW)`.

## Tested on

Nobara Linux (Fedora 44 based), GCC 16.2.1, Qt 6.11.2, CMake 4.3.0.
The `apt` and `pacman` package lists in `tools/install-local.sh` follow the
README and CI workflow but were not run.

## Screenshots

`branding/screenshots/` contains the project's rendered mockups (generated by
`branding/tools/mockups.py`), not captures of a running window. After Vela
starts on your machine, you can replace them with real screenshots of the same
file names.
