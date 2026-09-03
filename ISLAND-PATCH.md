# Island bar patch

A fork of Omarchy's built-in bar (`omarchy.bar`) that paints **one pill-shaped
slab per section** — left, center, right — floating clear of the screen edge and
of each other, instead of the stock single full-width background slab.

Created 2026-09-04. Forked from **omarchy 4.0.2-1**.

```
commit 1  Pristine clone of omarchy.bar 4.0.2-1   <- untouched `omarchy plugin clone` output
commit 2  Island bar: per-section pill slabs      <- the patch, also saved as island.patch
```

## Why this is a fork at all

Stock config exposes only `position`, `transparent`, `centerAnchor`, `layout`
and `id` under `bar:`, and `Style.bar` exposes only size/icon tokens. There is
no hook for per-section backgrounds, and a bar *widget* can only paint inside
its own `ModuleSlot` — it cannot draw behind a whole section. Replacing the bar
wholesale is the supported path: `bar.id` in `shell.json` selects any plugin
whose manifest declares `kinds: ["bar"]` (`shell.qml:165-207`).

## Config keys

Added under `bar:` in `~/.config/omarchy/shell.json`. All hot-reload on save.

| key             | default | meaning                                             |
|-----------------|---------|-----------------------------------------------------|
| `island`        | `true`  | `false` restores the stock full-width bar exactly    |
| `islandGap`     | `6`     | gap from the screen edge, and the side inset         |
| `islandPadding` | `10`    | horizontal padding inside each pill                  |
| `islandRadius`  | `-1`    | `-1` = pill (`min(w,h)/2`); `>=0` = fixed radius     |

Values pass through `Style.space()`, so they scale with the theme's spacing
scale like every other Omarchy dimension.

## The upstream bug this had to work around

**`omarchy plugin clone omarchy.bar` produces a bar that cannot load.** This is
not caused by the island work — a pristine clone fails identically.

`Bar.qml` declares three `required` properties:

```qml
required property string omarchyPath
required property var barWidgetRegistry
required property var barConfig
```

- The **stock** bar is built by `shell.qml`'s `defaultBarComponent`, which sets
  all three **inline at construction** — so `required` is satisfiable.
- A bar loaded as a **plugin** goes through `pluginBarLoader`
  (`Loader { source: ... }` + `onLoaded: shell.configureBar(item, ...)`), which
  assigns them **after** construction. QML requires `required` properties to be
  set *at* construction, so the component fails with
  `Required property <x> was not initialized`.

Worse, the fallback that should have caught this is itself broken:

```qml
onStatusChanged: {
  if (status === Loader.Error) {
    var detail = errorString && errorString() ? errorString() : ""   // shell.qml:256
    shell.failedBarId = shell.activeBarId                            // shell.qml:258 — never reached
```

`errorString` is not defined in that scope, so the handler throws
`ReferenceError: errorString is not defined` at line 256 and **never reaches
line 258**. `failedBarId` stays empty, `activeBarId` never reverts to
`omarchy.bar`, and the result is **no bar at all** — not even the stock one.

**The fix in this fork:** drop `required` from those three declarations, give
them defaults (`""` / `null` / `null`), and null-guard the one
`barWidgetRegistry.widgets` read. `configureBar()` then populates them
post-load exactly as intended.

If upstream ever fixes this, the `required` change here becomes redundant but
stays harmless.

## What changed in Bar.qml

Six sites, all marked with a `leonardom011.bar fork` comment:

1. **Property declarations (~line 14)** — drop `required`, add defaults. See above.
2. **Root island properties (~line 40 and ~line 350)** — `island`, `islandGap`,
   `islandPadding`, `islandRadius`, plus derived `islandGapPx`, `islandPadPx`,
   `barThickness`, `islandEdgeMargin`, `islandBackground`, `islandRadiusFor()`.
   Also extends `fallbackBarConfig`.
3. **`applyBarConfig()`** — reads the four keys off `barConfig`.
4. **`BarPanel`** — `implicitWidth`/`implicitHeight` and the hide `margins` use
   `barThickness` (`barSize + islandGapPx`) so the exclusion zone reserves the
   gap; window `color` is transparent in island mode.
5. **`horizontalBar` / `verticalBar`** — sections moved into an inner
   `islandBand` inset by `islandGapPx` on the screen-edge side; the left/right
   pills are `z: -1` siblings bound to `leftModules` / `rightModules` geometry.
6. **`horizontalCenterModules` / `verticalCenterModules`** — ids added to the
   module lists, plus `contentLeft`/`contentRight` (`contentTop`/`contentBottom`)
   and a pill bound to them.

### Two details worth not re-deriving

- **The center pill can't just wrap the content.** `CenterModules` must keep
  `anchors.fill` on its container because `CenterGestureArea` is what you drag
  to move the bar and double-click to toggle transparency. Shrinking it to hug
  the widgets would kill both gestures. Hence the `contentLeft`/`contentRight`
  bounds computed from the child items instead.
- **Empty flanks need no special-casing.** A `ModuleList` with no entries has
  `active: false` so its `width` collapses to 0, and it is anchored to the
  center anchor module's edge — so its `x` lands exactly on that edge and the
  `Math.min`/`Math.max` bounds stay correct.

- `islandEdgeMargin = islandGapPx + islandPadPx`, because the pill is drawn
  `islandPadPx` *outside* the module list. Without the extra term the pill
  overhangs the screen edge (it did, by 2px, on the first pass).

## Re-applying after `omarchy update`

`omarchy update` will not carry upstream `Bar.qml` fixes into this clone, and
`omarchy plugin remove` + re-clone throws this away. To rebase onto a newer
upstream bar:

```bash
cd ~/.config/omarchy/plugins/leonardom011.bar
cp island.patch /tmp/island.patch            # keep a copy outside the dir
cd ~/.config/omarchy/plugins
rm -rf leonardom011.bar
omarchy plugin clone omarchy.bar             # re-sets bar.id in shell.json
cd leonardom011.bar
git init && git add -A && git commit -m "Pristine clone of omarchy.bar <new version>"
git apply --3way /tmp/island.patch           # resolve any rejects by hand
omarchy restart shell
```

**A full `omarchy restart shell` is required after any code change here.**
`shell.json` values are live property bindings and hot-reload, but QML *code*
changes need the component re-instantiated — and a component that already
failed to load will not be retried by hot-reload alone.

## Rollback

```bash
omarchy plugin disable leonardom011.bar   # restores omarchy.bar, keeps the fork on disk
```

Or set `"island": false` in `shell.json` to keep the fork active but render
exactly like the stock bar.
