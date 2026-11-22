# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Dash to Panel is a GNOME Shell extension (UUID `dash-to-panel@jderose9.github.com`). It merges the dash into the top panel to make a single taskbar. It targets GNOME Shell 46–51 (`metadata.json`) and uses ES modules (`gi://…` and `resource:///org/gnome/shell/…` imports). There is no bundler and no test suite. You check changes by installing the extension and reloading GNOME Shell.

This checkout is a fork that also builds a Debian package (`debian/`, plus a BOA build config in `.env`) and generates `CHANGELOG.md` with auto-changelog (`.auto-changelog`).

## Commands

- `make`: compiles `schemas/gschemas.compiled` and the `po/*.mo` translations.
- `make install`: builds into `_build/` and copies it to `~/.local/share/gnome-shell/extensions/<uuid>`. With `DESTDIR=…` it does a system install, and the schemas and locale go to `/usr/share`.
- `make zip-file`: builds the distributable zip. Pass `VERSION=NN` to stamp `metadata.json`; otherwise the version comes from the latest `v*` git tag plus the commit SHA.
- `make potfile` / `make mergepo`: regenerates `po/dash-to-panel.pot` from `TOLOCALIZE` files and `ui/*.ui`, then merges it into each `.po`. If you add translatable strings in a new source file, add that file to `TOLOCALIZE` in the Makefile.
- `yarn lint` (or `npm run lint`): runs Prettier `--write` and then `eslint --fix` on `src/**/*.js`. Style is no semicolons, single quotes, and parentheses around every arrow-function argument (`.prettierrc`).
- Debian: `debian/rules` `build` calls `make _build`, and `debian/dash-to-panel.install` maps `_build/*` into `/usr/share/...`.

Testing a change: run `make install`, then restart the shell. On Wayland that means logging out and back in, or running a nested session with `dbus-run-session gnome-shell --devkit` (`--nested` before GNOME 49). On X11, press Alt+F2 and type `r`. Then run `gnome-extensions enable dash-to-panel@jderose9.github.com`. Watch logs with `journalctl -f -o cat /usr/bin/gnome-shell`. Open prefs with `gnome-extensions prefs dash-to-panel@jderose9.github.com`; prefs runs in a separate GTK4/libadwaita process. To reset settings, run `dconf reset -f /org/gnome/shell/extensions/dash-to-panel/`.

## Architecture

There are two separate runtimes:

- **Shell side**: everything in `src/` except `prefs.js`. It runs inside gnome-shell with Clutter/St/Meta, and its import entry point is `extension.js`.
- **Prefs side**: `src/prefs.js` together with the GtkBuilder files in `ui/*.ui`. It runs in its own GTK4 process and cannot import shell modules. The two sides communicate only through GSettings (`schemas/org.gnome.shell.extensions.dash-to-panel.gschema.xml`).

The shell side is laid out as follows:

- `extension.js` `enable()` sets up module-level singletons that other modules import directly: `SETTINGS`, `DESKTOPSETTINGS`, `DTP_EXTENSION`, `EXTENSION_PATH`, `tracker`, and `PERSISTENTSTORAGE`, which survives disable/enable. It also exposes `global.dashToPanel`, an EventEmitter for other extensions. It disables Ubuntu Dock if that is present and then creates the `PanelManager`.
- `panelManager.js` owns the panels. It creates one `Panel` per monitor, or one on the primary monitor only. It monkey-patches GNOME Shell internals, such as the overview, workspace views, panel menu box pointers and keybindings, and restores them in `disable()`. When settings change it usually tears everything down and rebuilds (`_reset()`) instead of updating in place.
- `panel.js` holds `Panel`, an `St.Widget`. For the primary monitor it takes over the real `Main.panel` status area. Secondary monitors get their own boxes. Each panel hosts a `Taskbar` and lays out its elements in the order given by `panelPositions.js`, which defines element ids and the TOP/BOTTOM/LEFT/RIGHT anchors. Vertical panels are a first-class case throughout.
- `taskbar.js` is a fork of upstream `ui/dash.js`, and `appIcons.js` is a fork of the upstream app icon with running indicators, context menus and click actions. `windowPreview.js` provides the hover previews. `intellihide.js` handles auto-hide and uses `proximity.js` for window-overlap detection. `transparency.js` handles dynamic opacity, `panelStyle.js` handles theming, `overview.js` contains the overview tweaks, and `notificationsMonitor.js` provides badge and urgency info. `pointerWatcher.js` replaces the shell's pointer watcher, which was removed in GNOME 51.
- `panelSettings.js` handles per-monitor settings. Many GSettings keys, such as panel size, length, anchor, positions and element positions, are JSON strings keyed by monitor id. Always read and write them through the helpers here (`getSettingsJson`/`setSettingsJson`, `getPanelSize`, and similar). The helpers cache values and fall back to legacy keys that used monitor indexes.
- `utils.js` contains `GlobalSignalsHandler`, `InjectionsHandler` and `TimeoutsHandler`. Use these when you connect signals, override shell methods or add GLib timeouts, so that everything gets cleaned up on `destroy()`/`disable()`. A disable that leaves anything behind is a bug for GNOME extensions.

When you add a setting:

1. Add the key to the gschema XML.
2. Wire up a widget in `prefs.js` and the matching `ui/*.ui` file.
3. Have the shell code react, usually by connecting `changed::<key>` through a `GlobalSignalsHandler`.

Running `make` recompiles the schema.
