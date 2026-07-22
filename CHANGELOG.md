# Changelog

All notable changes to **Bunny Tea Parade** (Phaser rhythm game + Beat Mapping Lab) are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project aims to follow [Semantic Versioning](https://semver.org/).

---

## [v1.0-ux] — 2026-07-22

### Highlights

- **Settings/Pause unified** — Space/Esc/P/gear icon all open the same Settings overlay. The legacy "Paused" overlay is removed; blur/visibility still calls `togglePause()` which delegates to Settings.
- **Settings overlay rebuilt with Phaser-native widgets** — `±50/±10/⟲ reset` and `±20%/±5%` stepper buttons replace the old `<input type="range">` sliders that would render off-screen or crash on devices without the Phaser DOM plugin.
- **Press-feedback animations** on every interactive button (`scale 0.97 / 0.92`, 90 ms ease-out, custom `Cubic.easeOut` curve).
- **Accessibility gates** — `@media (prefers-reduced-motion)` and `@media (hover: none), (pointer: coarse)`.
- **Undo/Redo in the Beat Mapping Lab** — full 50-step history snapshot covering notes, markers, and selection, wired to `Cmd/Ctrl+Z` and `Shift+Cmd/Ctrl+Z` (plus `Cmd/Ctrl+Y`).
- **DAW-style keyboard shortcuts** — Space toggles playback; arrows nudge ±10 ms (Shift = ±50 ms), Up/Down change lane; `N`/`P` select next/prev note by time; `Delete` removes selection.

### Gameplay (`index.html`)

#### Added
- `EASE_OUT`, `EASE_IN_OUT`, `EASE_DRAWER` Phaser easing constants wired into tweens (banner, lane flash, judgment, SFX, start-tap text).
- `toggleSettings()` method that opens *or* closes the Settings overlay and handles auto-pause / resume symmetrically.
- `togglePause()` delegates to `toggleSettings()` for blur/visibility events so there is only one pause surface.
- `_settingsBtn(label, x, y, fn)` factory that returns both a wrapped container (for press-feedback scale tween) **and** a hit zone ≥ 44 px tall (Apple/Google HIG minimum).
- Press-feedback on: restart button, gear icon, settings overlay buttons, +/- button rows, reset, ปิด (close).
- Holding the tween reference on each press animation prevents rapid double-fires from competing tweens.

#### Changed
- Settings panel enlarged from 310×200 to **320×260** so Offset/SFX rows fully fit, with a "Space / Esc ปิด" hint and `ปิด` close button.
- Progress bar y-positions updated (`offBar` at `-33`, `sfxBar` at `25`) so they sit centered under their row labels in the new panel size.
- Tweens that previously used `'Power2'`/`'Power3'` now use `EASE_OUT` (custom strong ease-out).
- `setColor()` calls on settings button text run **before** the scale tween (and **after** in the up handler) — Phaser's internal `updateText → setSize → this.cut` path crashes if the layout cache is mid-tween with the wrong parent scale.

#### Removed
- Orphan `pauseOverlay`, `showPauseOverlay()`, `hidePauseOverlay()` — replaced by the unified Settings surface.

#### Fixed (runtime bugs found via browser verification)
- **Null-pointer crash on settings button click** — `TypeError: Cannot read properties of null (reading 'cut')` at `updateText → setSize`. Was caused by combining `setColor` with an active scale tween on the text's parent container in the same frame. Now `setColor` always runs in stable parent state.
- **Settings button hit zones out of panel** — `this.add.zone(x, y, w, h)` is *scene-level* (like the long-known `add.dom` issue). Hit zones were placed at world `(–120, –45)` instead of inheriting the overlay's `(200, 360)` translation. Now wrapped + zone are both added to `settingsOverlay`, inheriting the panel transform.

### Beat Mapping Lab (`beatmap-editor.html`)

#### Added
- **Undo/Redo system** — `snapshot()` / `restore()` / `commit()` / `undo()` / `redo()`, rolling 50-deep history with redo cleared on each new edit. Wrapped at every mutation site:
  - Arrow nudges (`updateSelected`)
  - Drag-to-move notes (`dragStart`)
  - Drag-to-move markers (`dragMarker`)
  - Delete button (notes and guide markers)
  - `Generate guide lines` and `Generate notes from guides`
  - Add note via lane click
  - Add marker via waveform click
- **Undo/Redo buttons** in the Inspector with live counts (`↶ Undo (3)`).
- **Keyboard shortcuts**:
  | Key | Action |
  | --- | --- |
  | `Space` | Play / Stop preview |
  | `Cmd/Ctrl+Z` | Undo |
  | `Shift+Cmd/Ctrl+Z` or `Cmd/Ctrl+Y` | Redo |
  | `←` / `→` | Nudge time ±10 ms (Shift = ±50 ms) |
  | `↑` / `↓` | Change lane −/+ 1 |
  | `N` / `P` | Next / previous note (sorted by time) |
  | `Delete` / `Backspace` | Remove selected note or guide marker |
- **Keyboard hint pills** (`button.kbd-hint` `data-kbd` attribute) on toolbar buttons — `⌘O`, `⌘F`, `⌘E`, `Space`.
- **Inspector visual hierarchy**:
  - History row at the top.
  - Nudge pair (`← −10ms` / `+10ms →`) as a 2-column grid.
  - Delete button styled with red-tinted background to differentiate from neutral nudge buttons.

#### Changed
- `button` global transition now `transform 160ms cubic-bezier(0.23,1,0.32,1), background-color 120ms ease, border-color 120ms ease` (was: no transition).
- `button:active` now `transform: scale(0.97)` — Emil's "Buttons must feel responsive".
- `.note.selected` styling — `scale(1.25)`, `box-shadow: 0 0 18px var(--gold), 0 0 4px #fff`, `z-index: 9`.
- `.note:hover` adds `filter: brightness(1.25)` (gated by `(hover: hover), (pointer: fine)`).
- Inline `<kbd>` tags in help text list the new shortcuts inline.

### Verification (this release)

- Ad-hoc verifier: **20/20 pass** on the new fixes.
- Browser runtime test: **9/9 pass** (149 notes initialised, nudge/undo/redo round-trips, every keyboard shortcut fires, delete+undo round-trip, zero JS errors in console).
- Gameplay runtime test: settings overlay opens via gear, opens/closes via Space and Esc without crash, every +/- stepper button mutates `SETTINGS.offset` / `SETTINGS.sfx` correctly, hit-test via `Zone.emit('pointerdown', ...)` lives at world `(80, 315)`.
- Visual screenshot inspection: settings panel layout correct, gear/restart/settings all carry their press indicators, editor toolbar pills visible, Inspector shows Undo/Redo with counts and red Delete button.

### Known limitations

- `?debug` query string required for browser-driven Phaser-internal hit testing (gated by query — production users unaffected).
- The Undo stack captures `state.notes`, `state.markers`, `state.selected`, and `state.selectedMarker`. It does **not** capture zoom, snap, or waveform state (deliberate — those are view prefs).
- Cross-tab persistence is intentionally not added — a refresh clears Undo. (Tracked separately for v1.1.)

---

## Prior history (pre-`v1.0-ux`)

Commits before this tag focus on:
- Initial game + beatmap scaffold.
- Texture leak fix (`_cleanupAutoTextures` in `SHUTDOWN`).
- Tabbed UI between Game and Beat Mapping Lab panels.
- Settings button hang fix (`dom.createContainer` + try-catch).
- Settings overlay positioning fix (absolute game coords).
- Unified Space/Esc settings/pause toggle (`toggleSettings` method, `Phaser-native` overlay, `auto-pause`/`resume` lifecycle).

Refer to `git log --oneline` and per-commit messages for the canonical history.
