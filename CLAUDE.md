# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A fork of Omarchy's built-in Quickshell status bar (`omarchy.bar`), loaded by
omarchy-shell as a third-party **bar** plugin (manifest id `leonardom011.bar`,
selected by `bar.id` in `~/.config/omarchy/shell.json`). The only intended
difference from upstream is the "island" look: one pill-shaped slab per section
instead of one full-width bar. `ISLAND-PATCH.md` is the design doc for it.
`README.md` is upstream's bar README copied verbatim, so its relative links point
into the omarchy source tree and are broken here. Its widget API section is still
accurate.

The history is kept deliberately small: `ad64748` is the untouched
`omarchy plugin clone` output (4.0.2-1), `562e8d1` is the island patch, and
`island.patch` is exactly that commit's diff. Keep fork code separable from
upstream code. Mark every edit site in `Bar.qml` with a `leonardom011.bar fork`
comment (there are 14 today), and regenerate the patch after changing island code:
`git diff ad64748 HEAD -- Bar.qml > island.patch`.

## Where it runs

- `~/.config/omarchy/plugins/io.github.leonardom011.better-bar` is a **symlink to
  this repo**, so edits here are live once the shell restarts. The plugin registry
  keys plugins by manifest `id`, not by directory name, so the name mismatch is fine.
- Host source (read-only reference) is `$OMARCHY_PATH/shell` = `/usr/share/omarchy/shell`:
  `shell.qml` (bar loader, `configureBar()`), `services/PluginRegistry.qml`
  (plugin scan), and `plugins/bar/`, the **current upstream bar** to diff against.
  The `qs.Commons` / `qs.Ui` imports resolve from there.
- `centerAnchor` is `io.github.leonardom011.workspaces`, the user's own widget
  from the sibling repo `~/Dev/io.github.leonardom011.workspaces`.

### `widgets/` and `indicators/` are inert

For third-party plugin directories the registry scans only the root `manifest.json`
(`scan_thirdparty` in `PluginRegistry.qml`). Sibling `*.manifest.json` files are
only honored in first-party dirs. So layout ids like `omarchy.microphone` resolve
to `/usr/share/omarchy/shell/plugins/bar/widgets/`, and upstream's `Indicators.qml`
loads `../indicators/` relative to *itself*. The copies here are clone leftovers,
and editing them changes nothing. Only `Bar.qml`, `BarModel.js`, and `manifest.json`
are live.

## Commands

There is no build, no test suite, and no working linter (`qmllint` exits 255 with
no output here, so don't treat it as a check).

```bash
omarchy-restart-shell                    # required after ANY .qml/.js change
qs log -p /usr/share/omarchy/shell | rg better-bar   # this bar's warnings (file URLs contain the dir name)
qs ipc -p /usr/share/omarchy/shell show  # IPC targets; this bar registers `omarchy.bar` (syncHidden)
node -e 'const m=require("./BarModel.js"); console.log(m.normalizePosition("left"))'  # BarModel is node-requireable
omarchy plugin disable leonardom011.bar  # rollback to stock bar (or set "island": false in shell.json)
```

`shell.json` edits hot-reload through the `barConfig` binding, but QML code
changes don't, and a component that already failed to load is never retried
without a restart. If this bar fails to load, 4.0.3 logs `bar option
leonardom011.bar failed to load, falling back to omarchy.bar` and shows the stock
bar, so check the log before assuming a change took effect.

## Architecture (`Bar.qml`)

One large root `Item` with inline `component`s. Helper logic with no QML
dependencies lives in `BarModel.js`.

- **Host injection happens after construction.** `configureBar()` assigns
  `omarchyPath`, `shell`, `manifest`, `barWidgetRegistry`, `pluginRegistry`, and
  `barConfig` in the Loader's `onLoaded`, guarded by `"x" in target`. That means
  none of them may be `required`, everything reading them must tolerate null at
  first, and an undeclared property is silently never injected. This fork does not
  declare `pluginRegistry`.
- **Config flow:** `barConfig` → `applyBarConfig()` → `normalizeLayout()`
  (`Util.normalizeLayout` plus `BarModel.pinTrayToInner`). Then
  `BarModel.inlineSettingsDelta()` decides whether the change is settings-only
  (patched into the running widgets in place) or structural (reassigns
  `layoutConfig` and bumps `barConfigSerial`, which rebuilds every widget on
  every monitor).
- **One surface per monitor:** three `Variants` over `Quickshell.screens` build
  `BarPanel` (the real `PanelWindow`), `DragGhostPanel`, and `BarMoveGhostPanel`
  (drag ghosts for widget reordering and moving the bar to another edge). State
  that must be global lives on root (`barHoverCount`, `activePopout`, drag state).
  `BarModel.pickPanelSlot` chooses which monitor's copy of a widget answers a
  panel hotkey.
- **Sections:** `LeftModules`/`RightModules` are a `ModuleList` (a Loader
  wrapping a Row or Column around a Repeater of `ModuleSlot`). `CenterModules`
  implements `centerAnchor` by pinning the anchor at the exact center and flanking
  the others around it. The anchor is mounted twice (drawn copy plus a zero-size
  placeholder); see `pickDrawnSlot`. `ModuleSlot` resolves an entry to a registry
  widget (`barWidgetRegistry.widgets[id].component`), a custom `qml` module
  (`~/.config/omarchy/bar/modules/<id>.qml` or `source`), or a
  `CustomCommandModule` (`type: command`).
- **Widget API:** widgets receive this root as `bar` and use the properties and
  functions listed in README.md ("Bar properties available to widgets"). Renaming
  them breaks every widget, third-party ones included.

## Island implementation: must-knows

- The config keys `island`, `islandGap`, `islandPadding`, and `islandRadius` have
  defaults in **three** places: the root property defaults, `fallbackBarConfig`,
  and the fallbacks in `applyBarConfig()`. Keep all three in sync. Values go
  through `Style.space()`.
- `BarPanel` grows by `islandGapPx` (`barThickness`) so the exclusion zone
  reserves the gap, and its window is transparent in island mode. Pills are
  `z: -1` siblings bound to live section geometry and animated.
- `CenterModules` must keep `anchors.fill`, because `CenterGestureArea` is the
  drag-to-move target (double-click does nothing). The center pill is sized from
  computed `contentLeft/contentRight` (`contentTop/contentBottom` when vertical)
  instead.
- `islandEdgeMargin = islandGapPx + islandPadPx`, because the pill is drawn
  outside the module list.
- Horizontal and vertical bars have separate code paths (`horizontalBar`/`verticalBar`,
  `horizontalCenterModules`/`verticalCenterModules`). Change both, and check
  left/right positions as well as top.

## Layout lock and pinned transparency

A second fork feature on top of the island patch (not part of `island.patch`).

- `bar.locked` in shell.json (root property `layoutLocked`) blocks widget
  drag-reorder (`ModuleSlot`'s `canReorder`), dragging the bar to another edge
  (`CenterGestureArea.startDrag`), and `dropBarModule`. Clicks, tooltips, and
  popouts are unaffected. It defaults to **locked**: an absent `bar.locked`
  means locked, in the same three places as the island keys.
- `toggleLayoutLock()` writes `bar.locked` through `mutateShellConfig` and sends a
  `notify-send`. It has exactly two triggers: the IPC call
  `omarchy-shell omarchy.bar toggleLock`, and `SUPER + ALT + B` in
  `~/.config/hypr/bindings.lua`, which calls that IPC. There is deliberately no
  mouse trigger -- `CenterGestureArea` has no `onDoubleClicked` at all (upstream
  toggled transparency there), so the bar can't be unlocked by a stray
  double-click.
- The lock does not cover the tray plugin (`io.github.tyrichards.tray`). Its
  drawer has its own drag MouseArea and writes shell.json itself. It also does not
  cover `omarchy bar move/position` or hand edits to shell.json.
- Transparency is pinned off. `applyBarConfig()` calls
  `setRequestedTransparency(false)`, so `bar.transparent` is ignored. The look
  comes from the theme's `bar.background` plus the `[bar] background-alpha` in
  `~/.config/omarchy/shell.toml`. `toggleTransparency()` is kept for diff
  minimality, but nothing calls it.

## Rebasing onto a newer upstream

The fork is based on 4.0.2-1. The installed omarchy is 4.0.3-1, and upstream
`Bar.qml` has changed since then (`manifest.json` always differs because of the
fork's id). Check how far upstream has drifted:

```bash
for f in $(git ls-tree -r --name-only ad64748); do git show ad64748:"$f" | cmp -s - /usr/share/omarchy/shell/plugins/bar/"$f" || echo "$f"; done
```

**Don't follow the "Re-applying" steps in ISLAND-PATCH.md.** They predate moving
the plugin into this repo behind a symlink (`plugins/leonardom011.bar` no longer
exists), and `git apply --3way island.patch` on a fresh clone of 4.0.3 fails
because the pre-image blobs are missing. Rebase inside this repo instead: create
an `upstream` branch at `ad64748`, copy in the changed upstream files (never
`manifest.json`), commit, and run `git rebase upstream` from `master`. Tested
against 4.0.3, the only conflicts are the two property-declaration hunks
(~lines 15–46). Upstream dropped `required` itself (it now defaults to
`Quickshell.env("OMARCHY_PATH")`, `fallbackBarWidgetRegistry`, and `({})`, and
adds `pluginRegistry`), so take upstream's side there and drop the fork's
"NOT `required`" comment. 4.0.3 also fixed the `shell.qml` `errorString` fallback
bug that ISLAND-PATCH.md describes.
