# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Git & Authorship

- All commits MUST be authored as the project owner. Never include `Co-Authored-By: Claude` or any trace of Claude in commits, code, comments, or documentation.
- Author: `Jcapehart2 <jcapehart2@hotmail.com>`
- Never mention Claude, AI assistance, or Anthropic anywhere in the repository.
- Commit subjects are short imperative sentences ("Fix crash on app close and game quit cleanup bug"), not Conventional Commits.

## Project

RuneKit Reforged is an Alt1-compatible widget host: it runs Alt1 toolkit web apps (clue solvers, XP meters) in QtWebEngine windows next to the official RuneScape 3 client, feeding them screenshots of the game window. It never touches the game client.

Fork of the abandoned `whs/runekit`, migrated from PySide2/Qt5/Python 3.9 to PySide6/Qt6/Python 3.11+. `MODERNIZATION_REPORT.md` records that migration; `DESIGN.md` is the upstream package design and still accurate.

- Python 3.11–3.14, PySide6/Qt6, Poetry
- **Linux (X11) is the primary target.** Wayland is unsupported. macOS support is experimental and untested — the maintainer has no Mac
- No test suite exists in this repo. CI only builds artifacts

## Commands

```sh
poetry install                       # deps
poetry run make dev                  # generate runekit/_resources.py — REQUIRED before first run
poetry run python main.py            # run (start RuneScape 3 first)
poetry run python main.py settings   # settings dialog only
poetry run python main.py <appconfig.json URL>   # launch a single Alt1 app, quit when it closes
poetry run python main.py --remote-debugging-port=9222   # Chromium devtools at chrome://inspect
poetry run black runekit             # format
```

Builds:

```sh
poetry run make dist/RuneKit.AppImage   # Linux (needs linuxdeploy-$(uname -m).AppImage on PATH, or LINUXDEPLOY=)
poetry run make dist/RuneKit.app.zip    # macOS, via PyInstaller + RuneKit.spec
```

Gotchas:

- `runekit/_resources.py` is a gitignored, generated Qt resource module (`pyside6-rcc` over `resources.qrc`). `runekit/main.py` imports it unconditionally, so a fresh checkout crashes on import until `make dev` runs. Re-run it after changing `alt1.js`, tray/skin PNGs, or `resources.qrc`.
- The macOS zip must be made with `ditto` (in the Makefile) — plain `zip` drops PyInstaller's framework symlinks and the bundle balloons from ~300MB to 3GB+.
- Logs: `~/.config/cupco.de/RuneKit/logs/runekit.log` (rotating, 3 × 1MB). Console gets DEBUG too.
- Settings live in `QSettings` INI format under the same `cupco.de/RuneKit` org/app; see the settings layout at the bottom of `DESIGN.md`.
- `RK_WM_APP_NAME` overrides the X11 window name to match (default `RuneScape`). The Quartz backend instead matches the window owner `rs2client`.

## Architecture

Flow: `main.py` → `runekit.main.main()` → `browser.init()` (register custom URL schemes before `QApplication` exists) → `get_platform_manager()` → `Host`.

- **`runekit/host`** — `Host` is the application root. Owns the tray icon, settings dialog, `AppStore`, and the list of open `App`s. `launch_app()` binds an app to a `GameInstance`; apps are closed when their instance disappears.
- **`runekit/game`** — the **only** package allowed platform-dependent code (`DESIGN.md` rule). `GameManager`/`GameInstance` are Qt-signal-based ABCs; `get_platform_manager()` picks `x11/` (xcffib + XComposite + XShm + sysv_ipc shared memory, background `QThread` watching for window create/destroy) or `quartz/` (Quartz CGWindowList + a CGEventTap; blocks on `AXIsProcessTrusted()` until Accessibility is granted). Porting = reimplementing this package.
  - `game/overlay.py` `DesktopWideOverlay` is a frameless, click-through, always-on-top `QGraphicsView` spanning the virtual desktop, created by the manager. Alt1 overlay drawing renders into `QGraphicsItem`s inside it.
- **`runekit/app`** — `App` wraps one Alt1 app: manifest, its `GameInstance`, its `Alt1Api`, its `WebProfile`, its `AppWindow`. `App.close()` checks `shiboken6.isValid` before `deleteLater` because Qt may already have destroyed the C++ side. `app/view/` holds the window stack (`BrowserWindow` → `AppWindow`, plus the custom-skinned frame in `ui/windowframe.py`).
- **`runekit/browser`** — the Alt1 API surface exposed to app JavaScript, in two halves:
  1. `alt1.js` (a Qt resource, injected at document creation into the main world alongside `qwebchannel.js`) implements `window.alt1`. `%%RPC_TOKEN%%` in it is replaced per-profile in `profile.py` with a 64-byte secret.
  2. Small/async calls go over `QWebChannel` to `Alt1Api` (Qt Slots/Properties). Bulk pixel transfers go over the custom **`rk:`** URL scheme, handled by `RuneKitSchemeHandler`, which compares the `token` request header against the profile secret and runs the work on a `QThreadPool`. `alt1:` is also registered but `alt1://addapp/` is not implemented.
  - Each app gets its own `WebProfile` (`QWebEngineProfile`), so the secret and storage are per-app. `App.has_permission()` gates features against the manifest's comma-separated `permissions`.
- **`runekit/image`, `game/qt.py`** — screen capture formats. Internal standard is `numpy.ndarray` of shape (h, w, 4) in **BGRA32**; PIL `Image` (RGBA) is a deprecated second path still present in signatures (`ImageType = Union[np.ndarray, Image.Image]`). Alt1 wants BGRA, so prefer the ndarray path and avoid adding channel swaps.
- **`runekit/alt1`** — Alt1 manifest schema (`AppManifest` TypedDict) and `fetch_bom_json` (runeapps.org serves BOM-prefixed JSON).
- **`runekit/ui`** — tray icon, notifications, tooltip, game-snap, and the skinned window frame (`ui/skins/default/`, disabled on macOS — see `AppWindow.framed`).

`compatibility.md` tracks which Alt1 API calls and events are implemented; update it when adding one.

## Conventions

- Qt6 requires fully-qualified scoped enums (`Qt.WindowType.Widget`, not `Qt.Widget`). Unqualified Qt5-style enums are a migration bug.
- Cross-package type-only imports use `if TYPE_CHECKING:` with string annotations — the package graph is circular (`host` ↔ `app` ↔ `browser` ↔ `game`).
- Per-object loggers: `logging.getLogger(__name__ + "." + self.__class__.__name__)`, with the app name appended for per-app objects.
