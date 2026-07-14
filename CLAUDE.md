# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Мур-Луг" (Mur-Meadow) — a cat-farm idle/merge game for the Yandex Games platform. The entire game (markup, CSS, and JS) lives in one self-contained file, [index.html](index.html) (~1900 lines). There is no build step, no bundler, no package.json, and no test suite — it's a hand-written Canvas2D game meant to be uploaded to Yandex Games as-is.

## Running / testing

- There's nothing to build. Open [index.html](index.html) directly in a browser, or serve the folder with any static server (e.g. `python -m http.server`) if you need `fetch`/SDK-like behavior to work from `http://` instead of `file://`.
- The Yandex Games SDK script tag (`https://yandex.ru/games/sdk/v2`) will fail to load outside the Yandex platform; the game deliberately falls back to `localStorage` in that case (see `init()` at the bottom of the script and the `onerror` on the SDK `<script>` tag). Local development always exercises this fallback path.
- No linter or test runner is configured. Verify changes by loading the page and playing through the affected flow (buying/merging cats, quests, shop, offline-earnings modal, etc.) — see the `web-game-developer` skill below for a heavier Playwright-based loop if you want automated verification.

## Architecture (all in index.html)

The `<style>` block defines the HUD chrome (coin/income chips, FAB buttons for quests/shop/album, bottom action buttons, sliding "sheet" panels, modal overlay, toast). The `<body>` is just the canvas + HUD divs + three sheet panels (`questSheet`, `shopSheet`, `albumSheet`) + `overlay`/`toast`. Everything interesting is in the inline `<script>`.

Key sections, in file order:

- **Data tables** — `BREEDS` (~[index.html:181](index.html#L181), 15 cat levels with palette/marking/accessory per level) and `DECOR` ([index.html:202](index.html#L202), shop items: toy/bed/flowerbed/pond/house/lantern, each with `cost`/`kind`/`cap`). Merging two cats of level N produces level N+1, up to `MAXLVL`.
- **Balance formulas** ([index.html:212](index.html#L212)) — `inc(lvl)` (income curve), `catInc` (shiny cats earn 3x, multiplied by `incMult()` from upgrades), `buyCost`/`buyLvl` (price scales with purchase count and current progress). Tune game economy here.
- **Upgrades** (`UPS` table + `upLvl`/`upCost`/`incMult`/`offRate`/`shinyChance`, right after the state section) — infinite shop upgrades bought with coins: cat income +15%/lvl, offline rate +25%/lvl, shiny chance (asymptotic, never reaches 100%). Rendered as the "Улучшения" tab in the shop sheet (`renderUps`/`buyUp`, `shopTab` switch). Levels persist in `S.up`.
- **Global state `S`** ([index.html:222](index.html#L222)) — the single object that gets serialized. `cats`/`decor` (runtime arrays with live x/y/animation state) are mirrored into `S.cats`/`S.decor` (plain data) on `persist()`; **any new persistent field must be added to both the live object and the `S.*` mirror in `persist()`/`restore()`**, or it won't survive a save/load round-trip.
- **Save system** ([index.html:229](index.html#L229)) — `persist(force)` throttles to one write per 10s unless forced, writes to `localStorage` under key `"murlug"` and, when running inside Yandex Games, to `player.setData({save})`. `restore(raw)` parses and merges into `S` defensively (handles missing/old fields).
- **Canvas world** ([index.html:282](index.html#L282) onward) — `resize()`/`plantDecor()` recompute layout (`horizon`, `fieldTop`/`fieldBot`) and regenerate ambient decoration (flowers, grass, background trees, clouds, birds, foreground grass) whenever the viewport changes; everything is randomly seeded per-resize, not persisted.
- **Cat AI** — `think(c, dt)` ([index.html:379](index.html#L379)) is a per-cat state machine (`idle → walk → play/watch/bedsleep/housesleep/sleep`, plus `waitmate`/`chat` for two cats socializing). `claimSpot`/`releaseSpot` reserve decor slots (`cap` from `DECOR`) so only N cats can use a given bed/pond at once. `matchmaker()` periodically pairs up idle cats to walk toward each other and chat (which pays out coins).
- **Rendering** — layered draw functions called once per frame from `frame()`: sky → hills (with pines) → meadow → entities (cats/decor/hero tree, depth-sorted by `y`) → butterflies/seeds → speech bubbles → particles → merge-hint overlay → foreground grass → vignette. The art style is flat-fill + dark-brown ink outlines (Cats & Soup reference); the outline color is the `INK` constant and most cat/decor shapes are fill+stroke pairs. Cats are drawn procedurally (`drawCat` → `poseSit`/`poseSleep` → `earBig`/`drawTail`/`face`); the sitting cat uses a two-pass union outline (stroke body+head paths, then fill both) so the silhouette reads as one shape. There is no sprite atlas or `textureGenerator` involved despite what the `skills/` folder might suggest (see below).
- **Camera** — `camX/camY` (+ targets `camTX/camTY`) implement a small cosmetic pan: pressing and dragging on empty field shifts the world a few px (clamped ±18/±14) and it springs back on release. The whole world render is wrapped in `ctx.translate(camX,camY)`; pointer hit-testing subtracts the offset. Background fills are overscanned ±30px so pan edges never show gaps.
- **Test hooks** — `window.render_game_to_text()` (JSON snapshot: coins/ips/cats/decor/quests/upgrades) and `window.advanceTime(ms)` (steps `frame()` manually with `manual=true` so it doesn't double-schedule rAF). Used by Playwright scripts for automated verification; `frame` clamps `dt` to ≥0 so manual stepping can't produce negative-dt frames.
- **Input** — `pointerdown`/`pointermove`/`pointerup` on the canvas ([index.html:1391](index.html#L1391)) implement tap-to-pet (`petCat`) vs. drag-to-merge (`mergeTarget`/`doMerge`) vs. drag-to-reposition decor, disambiguated by movement distance.
- **Meta systems**: album (`albumRecord`/`renderAlbum`, tracks discovered breeds + shiny variants), shop (`renderShop`/`buyDecor`), daily quests (`QPOOL`/`ensureQuests`/`qProg`/`renderQuests`, 3 random daily quests + a "chest" bonus for completing all three), daily login reward (`dailyCheck`), offline-earnings modal (`offlineCheck`, capped at `OFFLINE_CAP` seconds and paid at `OFFLINE_RATE`).
- **Yandex SDK integration** ([index.html:1727](index.html#L1727)) — `maybeInterstitial()` (rate-limited fullscreen ads, min 3 min apart) and `showRewarded()` (rewarded video, e.g. for the "free cat" and "boost x2" buttons); both no-op / auto-grant gracefully when `ysdk` is null (local/offline mode).
- **Boot** ([index.html:1869](index.html#L1869)) — `boot(raw)` restores state, rebuilds live `cats`/`decor` from the persisted plain data, ensures at least 2 starter cats, then kicks off quests/HUD/offline-check/daily-check/tutorial and starts `requestAnimationFrame(frame)`. The IIFE at the very end tries `YaGames.init()` first (with a 4s timeout fallback to local save) before falling back to `boot(localRaw)` directly.

All UI text and in-game copy is in Russian; keep new strings consistent with that (this targets Yandex Games' primarily Russian-speaking audience).

## The `skills/` folder

This folder holds general-purpose Claude Code skills for web/HTML5 game development, not something scaffolded specifically for this repo's current state:

- `new-littlejs-game` and `atlas-shape-art` assume a `games/<name>/` multi-project layout built on the LittleJS engine with `templates/*.html` pattern references. **Neither exists in this repo** — `index.html` is a single vanilla-Canvas2D game with no engine dependency. Don't invoke `new-littlejs-game` expecting it to touch `index.html`; it's for scaffolding an unrelated new game under a `games/` directory that doesn't exist yet.
- `iterate-sprite` expects a `textureGenerator`/`drawToTexture` pattern (LittleJS-style) — also not how `index.html` draws its cats (which use direct canvas path-drawing, see "Rendering" above).
- `web-game-developer` (develop-web-game) describes a generic implement/test/screenshot loop using Playwright and hooks like `window.render_game_to_text`/`window.advanceTime` — useful methodology, but **`index.html` does not currently expose those hooks**; add them first if you want to drive this game through that skill's automated loop.
- `graphify` is an unrelated generic repo-analysis tool, not game-specific.
