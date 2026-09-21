---
name: web-game-offliner
description: Download any web game from any portal and any engine (Softgames, Poki, Construct, Phaser, PixiJS, Unity WebGL, Godot WebGL, etc.), strip its platform SDK, patch it into a fully self-contained offline build with a neutral driver, validate it with Playwright plus an application-level firewall, then deploy it to GitHub Pages under the game's ORIGINAL name (repo, Pages URL and ZIP all reflect the original game title — no invented name) with the in-game branding fully removed (the game itself ships unnamed) and a clean ZIP release. Game selection is originality-driven and NOTHING else: only games with an original concept are eligible — no classic games built on an already-known concept, no clones of famous titles; any engine and any game type (2D, 3D…) is acceptable, with NO engine priority and NO platform preference. The chosen game must NOT already be distributed on GamePix or GameMonetize (verify before committing), Famobi-licensed games are excluded even when they surface on other portals through the Famobi wrapper, and the whole build must stay under a 20 MB size cap for EVERY engine. Use when the user asks to localize, offline-ify, self-host, mirror, de-SDK, or republish a browser game.
version: 1.7.0
author: buffy
tags: [games, offline, download, playwright, github-pages, gamepix, gamemonetize, famobi, godot]
---

# Web Game Offliner

Take any web game from any portal (Softgames, Poki…, any
engine from Construct to Unity/Godot WebGL exports) and turn it into a
**100% offline-playable standalone build** with zero external dependencies,
then deploy it. This skill encodes a pipeline proven end-to-end on 5 games
(Construct 3, Phaser 2/3, PixiJS 5, custom canvas engines; Unity WebGL
supported since v1.3.0, Godot since v1.4.0). Any engine and any game type is
eligible — the only hard filters are ORIGINALITY, the 20 MB size cap and the
distribution-exclusion rules (a game already on GamePix or GameMonetize is
out, and so is any Famobi-licensed game — even via a third-party portal).

## Pipeline overview

```
Phase 0  IDENTIFY    → catalog scan, engine fingerprint, originality +
                       GamePix/GameMonetize/Famobi-exclusion checks + size
                       cap, game selection (NO engine priority, NO platform
                       preference)
Phase 1  DOWNLOAD    → full asset pull + integrity audit
Phase 2  PATCH       → SDK extraction → neutral game-driver.js
Phase 3  VALIDATE    → Playwright + application firewall
Phase 4  DEPLOY      → GitHub repo + Pages + ZIP release v1.0.0, ALL named
                       with the game's ORIGINAL name — game files only in
                       the ZIP
Phase 5  DE-BRAND    → sweep EVERY screen for old in-game branding, remove
                       it everywhere (neutral artwork at most — the game
                       ships unnamed), update metadata, republish
```

Never skip Phase 3: "it loads locally" is not "it works offline".

**Delivery naming is non-negotiable (v1.4.0)**: the repo name, Pages URL,
ZIP file name and README heading MUST reflect the game's ORIGINAL title —
the exact name the portal uses for the game. No invented title exists
anymore, anywhere in the delivery. Inside the game itself, in-game branding
is still fully removed: the shipped build carries no name, no logo text and
no external reference at all (Phase 5). See Phase 4 step 2 and Phase 5.

---

## Phase 0 — Identify the game and its engine

1. **Catalog scan** — scan ANY portal's catalog or public game feed: no
   platform is preferred (v1.5.0). Known entry points, for reference only:
   Poki's list is the sitemap
   `https://poki.com/en/sitemaps/games.xml`. Pick candidates wherever
   ORIGINAL games are found.
2. **Engine fingerprint** — fetch each candidate's `index.html` and grep the
   runtime JS for:
   - `c3runtime` / `Scirra` → Construct 3
   - `Phaser` (check version, CE vs 3) → Phaser
   - `pixi.js` / `PIXI` → PixiJS
   - `cocos` → Cocos Creator
   - `BABYLON` → Babylon.js (3D) → **eligible** (follow the generic WebGL
     playbook: capture, driver, de-brand — 3D is acceptable since v1.5.0)
   - `UNITY` / `UnityLoader` / `createUnityInstance` → Unity WebGL build →
     **eligible** (see the Unity WebGL playbook below)
   - `godot` / `@godotengine/godot` / `engine.startEngine` / a `.pck` +
     `.wasm` pair → Godot WebGL build → **eligible** (see the Godot WebGL
     playbook below)
3. **Watch for false positives**: matching the substring `constructor` is NOT
   Construct; "Phaser" inside a comment is NOT Phaser. Confirm on the runtime
   file, not the index.
4. **All game types are acceptable (v1.5.0)**: 2D AND 3D games alike — the
   engine type alone is never a disqualification. Still fingerprint the
   engine correctly (one candidate (Retro Drift) turned out to be Babylon.js
   despite an initial "Construct" match) so you follow the right playbook,
   but a 3D engine is eligible like any other.
5. **Unity WebGL builds are eligible (since v1.3.0)**: `UNITY`/`UnityLoader`
   markers are not a disqualification. Recognize the export
   (`createUnityInstance` + `Build/*.wasm`/`*.data`/`*.framework.js`, or the
   older `UnityLoader` + `Build/*.json`), download the whole build and follow
   the Unity WebGL playbook below.

Selection criteria, in order:

1. **Originality FIRST (v1.4.0)**: process only games with an original
   concept or a genuinely fresh twist. **Reject classic games** built on a
   well-known concept — memory/match-pairs, solitaire, 2048, snake, Tetris,
   brick breakers, sudoku, minesweeper, mahjong, chess/checkers, pinball,
   flappy-style clones, wordle-style clones, bubble shooters presented as
   "the classic"… Also reject transparent clones or reskins of famous
   titles. Judge from the game's own screenshots/gameplay footage: if the
   whole pitch can be summed up as "classic <known game>", it is out. A
   familiar genre with real added mechanics (novel physics, unusual goal,
   original twist) is fine — "known genre" is not the same as "known
   concept".
2. **NO engine priority, NO platform preference (v1.5.0)**: every engine
   (Construct, Phaser, PixiJS, Cocos, Unity WebGL, Godot WebGL, custom…)
   and every game type (2D, 3D…) is treated EQUALLY — the fingerprint only
   selects the right playbook, it never ranks candidates. No portal is
   preferred either.
3. **NOT already on GamePix or GameMonetize — hard exclusion (v1.6.0)**: a
   chosen game must NEVER already be distributed on GamePix or
   GameMonetize. BEFORE any download, search both catalogs and browse the
   result titles:
   - GamePix: `https://www.gamepix.com/search?q=<title keywords>` (bot-walled;
     use a real browser if curl gets 403)
   - GameMonetize: `https://gamemonetize.com/?s=<title keywords>` (or
     `/games?search=<keywords>`)
   If the same game (same title, or unmistakably the same gameplay) already
   exists on either platform, the candidate is DISQUALIFIED.
4. **NO Famobi games — hard exclusion (v1.7.0)**: Famobi-licensed games are
   out EVEN when they surface on another portal, through Famobi's embed
   wrapper. At fingerprint time, BEFORE any download, check the game page
   and its `index.html`/runtime for the Famobi signatures:
   - hosts/iframe URLs on `famobi.com`, `play.famobi.com`,
     `games.cdn.famobi.io`, `cdn.famobi.com`
   - the Famobi loader script (`famobi.js`, `famobi_noa.js`)
   - `window.famobi*` / `Famobi.*` globals and `data-famobi*` attributes
   - "Famobi" / "Powered by Famobi" credits in the page or the game
   Any hit = the game is Famobi-licensed = DISQUALIFIED, no matter which
   portal lists it.
5. **Size — hard cap for EVERY engine (v1.5.0)**: the total download must
   stay **under 20 MB for ALL engines alike** (Unity `.wasm`/`.data`, Godot
   `.wasm`/`.pck`, Construct/Phaser/PixiJS assets, everything). Measure the
   sizes BEFORE committing (HEAD requests on the build files); a build over
   the cap is disqualified — no exception, no "audit later".
6. Clean asset manifest; no aggressive DRM.

## Phase 1 — Download everything (and audit it)

1. **Enumerate assets from the manifest, not from guesses**:
   - Construct 3 → `data.json` (media list) + `appmanifest`
   - Phaser/PixiJS games → grep the loader calls in the game code
     (`load.image`, `load.atlas`, `load.audio`, preload manifests, `game.json`)
2. **Map the real path prefix** — assets often live under a subfolder
   (`assets/img_480/`, `assets/hd/`, `media/`). Sounding 404s with curl on the
   CDN *before* mass download saves an hour.
3. **Retina suffixes**: PixiJS loads may append `@2x` to names
   (`logo@2x.png`, `ui@2x.json`). Check for a resolution-suffix variable.
4. **Download all referenced files** (curl loop with a proper Referer header).
5. **MANDATORY integrity audit** — many traps produce corrupt files:
   - `file <each>` — anything reporting "HTML document" instead of
     PNG/JPEG/JSON/Audio is a poisoned file. This happens constantly (Google
     404 shells saved as `.png`, `.jpg`, even `.json`).
   - Every image must open with PIL/ImageMagick; every JSON must `json.load`.
   - Cross-check count and sizes against the manifest.
   - Delete fake files, retry with corrected paths, or replace with a clean
     transparent PNG / valid stub if the CDN itself 404s them (dead assets the
     original site also failed to load).
   - Fonts referenced by CSS must exist too (`gamefont.ttf` etc.).

## Phase 2 — Patch for offline play

### The core pattern: two-tier bridge + driver

Replace the platform SDK with a **neutral local driver**, never with dead
stubs:

```
index.html
  └── game-driver.js      ← NEW: neutral "driver" = offline SDK replacement
        └── exposes the exact surface the game calls
game code (js/game.js, main.js…)
  └── pokiSDK / sg calls rewritten to GameDriver.*
```

1. **Inventory the SDK surface actually used** — grep the whole codebase for
   every call shape: `PokiSDK.`, `window.PokiSDK`, `sg.`
   (Softgames), `pSDK`. Typical methods: `audio.*`, `ad.*`, `storage.*`,
   `gameStart()`, `levelStart/levelEnd`, `happytime()`, `gameover()`,
   `setProgress`, `firstPlay`, etc.
2. **Write `game-driver.js`** implementing that surface faithfully:
   - `audio`: no-op or WebAudio-based stub (`sfx()`/`bgSound()` no-ops are
     fine, but keep the method names).
   - `ad`: the #1 source of frozen games. If the game passes callbacks
     (`beforeReward`, `adBreakDone`, `adViewed`…), **you MUST call them**
     (immediately with a benign status like `'dismissed'`). A no-op here
     freezes the game the moment it fires a "gameover" interstitial — this
     exact bug froze Daily Word Search at puzzle completion.
   - `storage`: replicate the game's expected semantics. Some wrappers expect
     synchronous `getItem/setItem` (JSON), others (Softgames) expect
     base64-encoded JSON values and **asynchronous** reads. Read the SDK
     client code to know which.
3. **Rewrite call sites** — in minified code, plain-text replacement works:
   `PokiSDK.` → `GameDriver.` (watch for `window.PokiSDK` too). If the
   game has a fallback stub like `window.PokiSDK = window.PokiSDK || {}`
   it's harmless — leave it.
4. **Neutralize analytics/telemetry**: Sentry, DDNA, Google Fonts, A/B test
   fetches, `img` social share paths. Either remove the script tags, or
   redirect to `about:blank`/local data URLs. **Careful with string surgery in
   minified code** — one broken `concat(...)` parenthesis cost an hour of
   `Unexpected token ')'`. Always `node --check` (or `node -e "import()"`)
   after editing JS.
5. **Neutralize sitelocks/domain checks** (grep `location.host`, `hostname`,
   whitelist arrays).
6. **index.html** — remove external `<script>`/`<link>` tags; only local
   files; add `game-driver.js` **first** in `<head>` (it must exist before any
   game script runs, and it's the right place for a global recovery hook).
7. **Keep a game fallback safe**: some games show "reload the page" on any
   uncaught error. Add an auto-recovery in the driver: watch for the crash
   dialog; if it appears, `location.reload()` once, guarded by a cooldown
   (30 s) so a persistent error doesn't loop.

## Phase 3 — Validate (this is what makes it trustworthy)

Run **all** of these; each has caught real bugs:

1. **Playwright pass** (Chromium, `--use-gl=swiftshader`):
   - canvas present and sized,
   - **0 page errors**, **0 failed requests**, **0 external requests**.
2. **Application firewall** — route-block every non-localhost request, then
   play the game (click through menus, start a level). The game must run with
   the network cut. This proves "no blocking external dependency".
3. **Real-browser conditions**, not just happy path:
   - mobile emulation (touch, mobile viewport, **no forced autoplay**) —
     audio unlock flows differ from headless defaults,
   - gameplay interaction: click where the core mechanic lives, then
     screenshot-diff across time to confirm real scene progression (static
     background + nothing = broken).
4. **Visual check**: dump screenshots to ASCII/color histograms; a grid of
   gem colors or a letter grid means the game truly rendered.
5. **Long-run probe**: let it idle 30–60 s; confirm the render loop steps
   (`game.loop.frame` / RAF counting) rather than a frozen first frame.
6. **Headless ≠ real GPU**: a 3646×3583 atlas rendered fine under SwiftShader
   but exceeded many mobile GPUs' 2048 texture limit (Jewels Blitz 5 showed
   only its background on a real phone). Audit texture sizes; if any atlas is
   > 2048 px, resize it and rescale the frame coords in its JSON atlas.
7. **Exercise the SDK paths you shimmed**: from the page, call the driver's
   ad-break flow (reward + interstitial) and assert the callbacks resolve in
   ms. This catches the frozen-callback class of bugs before deploy.

## Phase 4 — Deploy

1. **ZIP release — game files ONLY (v1.3.0)**: the archive must contain
   nothing but what the game needs to run. **No README, no docs, no license
   files, no `.git`, no tooling, no leftover capture scripts.** Verify with
   `unzip -l <zip>` before shipping. Name it `<original-game-name>-v1.0.0.zip`
   — the ORIGINAL game title (kebab-case), see step 2; verify zero `.git`
   entries inside.
2. **GitHub repo** (one per game), push, enable **Pages via API**
   (`gh api repos/<owner>/<repo>/pages -X POST`), wait for the Pages build
   (poll the API until `status: built`).
   **Repo naming rule — MANDATORY (v1.4.0, supersedes v1.2.0/v1.3.0)**: the
   repo name MUST be the game's **ORIGINAL title** — exactly the name the
   portal gives the game (kebab-case: `drive-mad`, `jewels-blitz-5`…), so
   the repo name, the Pages URL (`<owner>.github.io/<original-name>/`) and
   the ZIP file name all **reflect the original game name**. Never use an
   invented title — that mechanism no longer exists (v1.4.0). Never use the
   original game id either if it differs from the human-readable title
   (prefer `jewels-blitz-5` over `jb5_1080`), and never a `-offline`/`-game`
   suffix. ⚠️ Do NOT confuse this game repo with this skill's own repo
   (`web-game-offliner`): never create, fork over, or overwrite a repo named
   `web-game-offliner` when publishing a game.
3. **Live verification**: Playwright against the GitHub Pages URL — 0 errors,
   0 third-party requests, game advances past menus.
4. **README.md — STRICT template (v1.4.0, in ENGLISH)**. The heading is the
   game's ORIGINAL title (same as the repo name). Exactly TWO sections,
   nothing else — no tech stack, no engine names, no modification lists, no
   run instructions, no badges, no screenshots section:

   ```markdown
   # <Original Game Title>

   ## About the game
   <A genuinely flattering, vivid description of what playing the game feels
   like: what the player does, why it is fun and addictive, what makes it
   stand out — polish, juice, satisfying feedback, pick-up-and-play design.
   Talk about the EXPERIENCE only. Never mention technologies, engines,
   frameworks or file formats. 2 short paragraphs max.>

   ## Controls
   <One subsection per platform, written with care — never an afterthought:

   ### Smartphone & tablet
   <exact gestures: tap, swipe, drag…>

   ### Desktop
   <exact mouse input: left-click, click-and-drag… and keyboard if it truly
   helps>
   > Trackpad: single click = same as left-click.
   ```

5. **Deliverable hygiene — clean file names (v1.3.0, MANDATORY)**: the repo
   must look like a professional game project, not a capture dump. Every file
   shipped in the repo and in the ZIP gets a **clean, conventional,
   English-only name**: `index.html`, `game-driver.js`, `serve.sh`,
   `assets/…`, `media/…`. **NEVER** ship names like `poki_nettoyé.js`,
   `sdk-stripped-final2.js`, `capture_dump/`, `test_v3_FIXED.js` or any name
   that betrays the pipeline (portal name, "cleaned", "patched", "copy",
   version suffixes, non-ASCII characters, spaces). If the downloaded sources
   came with such names, rename them during Phase 1/2 and fix every reference
   (`index.html`, loader code, manifests) — then re-run the Phase 3 gate.
   Capture scripts, probe tools and test scripts STAY in the workspace: they
   are never pushed to the game repo.

6. **serve.sh**: tiny script `python3 -m http.server "$PORT" --bind 0.0.0.0`
   (games need HTTP; `file://` won't work).
7. **Release v1.0.0** with the ZIP as asset via `gh release create`.

## Phase 5 — De-brand the game (MANDATORY for every published game)

Every deployed game gets its **in-game branding completely removed**: the
shipped build carries no title, no logos, no external references. The game
ships **unnamed** — where the old logo or title used to be, put **neutral
artwork** (a clean abstract shape, decorative frame or emblem with NO text)
or nothing at all. **The ORIGINAL game name is NOT erased from the delivery
itself**: it is the repo name, the Pages URL, the ZIP file name and the
README heading (Phase 4). There is NO invented title anywhere — that
mechanism was removed in v1.4.0.

**ALL-OR-NOTHING RULE (zero tolerance)**: the in-game branding must be
removed from **every screen of the game**, not just the first one. A build
where a name/logo survives anywhere (menu, level select, settings, pause,
game over, credits…) is a **FAILED build**. When this skill ships a game,
no name may appear inside the game: no asset, no string, no metadata, on
any screen — and **no new name is inserted in its place**: the game simply
has no name.

> **Scope note (v1.4.0 — supersedes v1.2.0/v1.3.0)**: the original game
> title lives ONLY on delivery surfaces — repo name, Pages URL, ZIP file
> name, README heading, `<title>`/PWA metadata. It is NOT integrated inside
> the game: no in-game logo, no splash text, no title-screen wording. Inside
> the shipped build a case-insensitive grep for the original game name must
> return **0 hits** (0 in-game hits), while the delivery surfaces carry it
> as-is.

**MANDATORY — Collision check before creating the repo**: list the repos on
BOTH publishing accounts and make sure this game hasn't already been
cleaned/published (or the original-name repo doesn't already exist for a
different game):

- `dannyking6`  → `gh api users/dannyking6/repos?per_page=100 --jq '.[].name'`
- `d2658182-hub` → `gh api users/d2658182-hub/repos?per_page=100 --jq '.[].name'`

If a repo (or a previous build folder in the workspace) already matches the
original game name or the same game, reuse that existing work instead of
deploying a duplicate. If the name itself is taken by an unrelated repo,
resolve the collision minimally (e.g. append the portal game id) — do NOT
invent a brand-new title.

1. **Sweep ALL screens for the old title (do this FIRST)** — never assume the
   title lives in one place: portals routinely brand several screens. Enumerate
   every screen/state the game has — boot/preloader, splash, main menu, level
   select, settings, pause, game over, credits/about — and for each one hunt
   the old title in:
   - dedicated logo files (`logo_main`, `game-logo.png`, `title.png`…)
   - frames inside texture atlases (grep the atlas JSON for
     `title`/`logo`/`splash`)
   - font-rendered Text objects in scene data (grep the code + scene JSON for
     the old name and its misspellings), then remove the string — or replace
     it with neutral wording ("Options", "Back"…) that carries no name
   - i18n/locale strings, HTML overlays, CSS background images, PWA manifest
     `name`/`short_name`, in-game `<title>`/meta strings
   - portal credits/references ("A game from X", "Powered by Y", portal
     links and branding): remove them entirely — the game ships with **no
     external reference**
   - credits, "About", settings and game-over screens: statistically the most
     forgotten ones — always open them and look
   **Decision rule**: if the old title appears on MORE THAN ONE screen, it
   must be completely removed on EVERY one of those screens. Write down the
   list of screens where it appears — step 5 verifies each one individually.
   Removing the asset but leaving the string (or vice versa) does not count
   as removed.
2. **Replace old branding with neutral artwork (NOT a new title)** — the
   build must not carry any name where the old one was:
   - Design the neutral artwork with PIL: a clean abstract emblem, geometric
     shape, decorative frame or gradient badge — **zero letters, zero
     words**. Compose layers with `Image.alpha_composite` (`ImageDraw` on
     RGBA REPLACES pixels instead of blending — a fill with alpha 0 erased a
     whole gradient once); render at 2× then LANCZOS-downscale to the exact
     original sprite dims.
   - If a screen works better with empty space, remove the element entirely
     and let the layout reflow cleanly — no torn layouts, no dangling
     "undefined"/placeholder text, no orphaned logo containers.
3. **Patch the engine — on EVERY screen found in step 1, not just the menu**:
   - Construct 3: replace the logo PNG frames with same-dims neutral
     artwork, keep `data.js`/`data.json` untouched (dims must match).
   - Phaser/PixiJS atlas: either redraw the frame region in the atlas PNG
     with neutral artwork, or (cleaner) add a standalone neutral image,
     `load.image()` it in the loader, and repoint the old title object
     (`this.titleImg = ...`) to the new texture key.
   - Unity WebGL: swap the logo/title sprites or textures for same-size
     neutral images, or neutralize the UI element that renders them.
4. **Update all metadata**: `<title>` + `meta[name=application-name]` in
   index.html and PWA manifest `name`/`short_name` carry the **original
   game title** (delivery surfaces, not in-game screens); i18n strings that
   embed a name are removed or neutralized;
   `GameData.BuildTitle`-style constants that expose the old name are emptied
   or removed.
5. **Verify the de-branding is COMPLETE — screen by screen**:
   - for EVERY screen listed in step 1: navigate to it in Playwright,
     screenshot it, and confirm no old-title pixels remain AND that no name,
     logo or wording is rendered where the old branding was (neutral artwork
     contains no letters);
   - grep the ENTIRE shipped build (HTML/JS/JSON/CSS, case-insensitive) for
     the original game name AND the portal name — assert **0 in-game hits**
     (the original name lives only on delivery surfaces: repo name, Pages
     URL, ZIP name, README heading, `<title>`/PWA metadata — nothing inside
     the game);
   - assert NO invented title exists anywhere (that mechanism is gone since
     v1.4.0) — every delivery surface carries the original name;
   - open the screens players rarely see (credits, about, settings, game
     over): a sweep that misses one of them is not done;
   - 0 console errors throughout.
6. **Republish EVERYTHING**: push to the repo (Pages rebuilds), wait for the
   CDN to serve the new files, AND **create a new release** (v1.0.1, v1.0.2…)
   with a fresh ZIP — existing GitHub releases are immutable, so a de-brand
   is never done until a new release carries the new build.

## Poki-specific playbook

- **Catalog**: the game list is a sitemap — `https://poki.com/en/sitemaps/games.xml`
  (≈1500 slugs, one `<loc>` per line: `poki.com/en/g/<slug>`). `robots.txt` lists it.
- **Engine discovery**: each game page boots an iframe on
  `https://<uuid>.gdn.poki.com/<uuid>/`. Listen to requests to capture the base,
  then fetch `index.html` + its JS from inside the page and grep for engine markers.
- **The gdn CDN rejects direct curl (403)** — it wants the Poki page context
  (referer/cookies). Download by driving the real `poki.com/en/g/<slug>` page and
  intercepting responses.
- **Capture with route interception + `serviceWorkers: 'block'`**: C3 exports
  register a service worker whose responses have unreadable bodies; plain
  `response.body()` then yields 0-byte files. `page.route('**/*')` +
  `route.fetch()` + `route.fulfill()` guarantees every body, and blocking SWs
  avoids the whole problem.
- **C3 audio/images are not in the network capture**: parse `data.json` for the
  audio entries (`["name", [["audio/webm; codecs=opus",".webm",bytes],…]]` →
  `media/<name>.<ext>`) and fetch them from the CDN with a Poki referer; then
  magic-byte filter — the CDN answers 404-HTML for codec variants it doesn't
  have (e.g. only `.webm` exists for one sound, only `.ogg`/`.m4a` for others).
- **Two C3 Poki-plugin shapes** (patch accordingly):
  1. SDK calls in `scripts/project/scriptsInEvents.js` + dynamic script-tag
     injection → neutralize the obfuscated sitelock event, drop the injection,
     rewrite `PokiSDK.` → `GameDriver.`.
  2. `Avix_PokiSDK_ForC3` plugin inside `c3runtime.js` posting DOM messages
     (`InitPoki`, `RequestCommercialBreak`…) and guarding on
     `typeof PokiSDK !== "undefined"` → do NOT rewrite the runtime; ship a
     global `window.PokiSDK` shim in `game-driver.js` loaded before main.js and
     remove the `<script src=//game-cdn.poki.com/...>` tag.
- In both shapes the plugin resolves ads via callbacks/promises — the shim MUST
  resolve (`rewardedBreak()` → `true`) or the game freezes at the next break.

## Unity WebGL playbook (eligible since v1.3.0)

- **Size cap — HARD, global rule (v1.5.0)**: the whole build
  (`.wasm`/`.br`/`.gz` code, `.data`, `StreamingAssets/`, assets) must total
  **under 20 MB** — the same cap applies to EVERY engine, not just Unity —
  measure with HEAD requests on every `Build/` file BEFORE downloading; over
  the cap = disqualified, no exception.
- **Recognize the export**: `createUnityInstance(canvas, {dataUrl:
  'Build/<name>.data', frameworkUrl: …, codeUrl: 'Build/<name>.wasm'})`
  (2019+) or the older `UnityLoader.instantiate('Build/<name>.json')`. Download
  the whole `Build/` folder (+ `StreamingAssets/` and the loading template).
- **Compression**: exports ship either uncompressed or as `*.br`/`*.gz`
  variants that need matching `Content-Encoding` headers GitHub Pages cannot
  set. Ship the **uncompressed** variants (or decompress locally) and make the
  loader config match what you ship; verify with `file` that each binary is
  real (WebAssembly / data), not an error page.
- **Capture**: Unity loads big binaries — capture with route interception
  like any other build, and also grab the `Build/*.json` manifest that lists
  every expected file so the integrity audit is complete.
- **SDK surface**: portal wrappers (Poki/Crazy Unity plugins)
  bridge JS↔Unity via `SendMessage('GameObject', 'Method', …)` and `jslib`.
  Keep `SendMessage` working and implement the JS side in `game-driver.js`;
  for `jslib`-generated glue, provide the matching global functions or patch
  the generated `*.framework.js`. Ad callbacks/promises MUST resolve (Phase 2
  rule applies unchanged).
- **Sitelock**: hostname checks live as readable strings inside `*.data` —
  patch bytes only with identical-length replacements, or neutralize from the
  driver; grep both `*.framework.js` and `*.data`.
- **De-branding a Unity build**: logo/title sprites live inside
  `.assets`/`.data` bundles where binary surgery is risky. In order of
  preference: neutralize/hide the UI element that renders the logo from the
  driver or a small runtime patch, swap the texture at runtime if the game
  exposes it, or replace the sprite inside the bundle with byte-length-
  preserving patching. Verify every screen in Playwright like any other build.
- **Testing**: serve with the right MIME types (`application/wasm`), watch for
  WebGL context loss, and test mobile emulation for touch input (many Unity
  portal games are mobile-first).

## Godot WebGL playbook (eligible since v1.4.0)

- **Size cap — HARD, global rule (v1.5.0)**: the whole export (`.wasm`,
  `.pck`, assets) must total **under 20 MB** before committing — the same
  20 MB cap applies to every engine alike.
- **Recognize the export**: Godot 4 exports use
  `engine.startEngine({mainPack: '…', canvas: …})` from
  `godot.wasm.js`/`godot.js`, loading `Build/<game>.pck` + `godot.wasm`;
  Godot 3 uses `Engine.load("wasm.js")` + `engine.start_game(...)`. The
  `.pck` + `.wasm` pair is the reliable fingerprint.
- **Download the whole export**: `godot.wasm`, `godot.js`/`godot.wasm.js`,
  `*.pck`, plus the side files (`*.audio.worklet.js`, `*.worker.js`,
  `*.renderer.js` in Godot 4.x) — missing side files = boot failure.
- **Compression**: same rule as Unity — GitHub Pages cannot set
  `Content-Encoding`; ship uncompressed variants and make the loader config
  match.
- **Sitelock / SDK wrappers**: portal wrappers (Poki SDK, Crazy)
  call into the game via JS before/after engine start — neutralize them in
  `game-driver.js` exactly like any other build (grep `pokiSDK`, `sg.`).
- **Engine-start failure mode**: if the game never renders, check the
  browser console for `pck` load errors (wrong path or `Content-Type`) —
  Godot aborts silently otherwise. Serve `.pck` as `application/octet-stream`
  and `.wasm` as `application/wasm`.
- **De-branding**: Godot scene/asset branding lives inside the `.pck`; do
  NOT attempt binary surgery — neutralize in-game name sprites/UI from the
  project side only when a rebuild is possible, otherwise hide the branding
  element via runtime patching (DOM overlay or engine API) and verify every
  screen in Playwright like any other build.

## Hard-won gotchas (read before debugging)

- **404-HTML-as-asset** is everywhere: a "successful" download can be a Google
  error page. Verify file magic bytes, not just HTTP 200.
- **Ad callbacks must fire** or the game freezes at the next interstitial.
- **Match the storage semantics exactly** (sync vs async, base64 vs plain).
- **Texture size limits** (2048) on real mobile GPUs; headless SwiftShader
  hides the bug.
- **Dead CDN assets** (404 on origin too): replace with valid stubs instead of
  shipping error-HTML files.
- **Minified-code edits**: one wrong character kills the whole bundle. Prefer
  whole-string replacements, verify with `node --check` after every patch.
- **node resolves modules from the script's directory, not cwd** — keep test
  scripts in the folder where playwright is installed.
- **CDN/Pages caching** after push: poll until the live content hash changes
  before re-testing.
- **Autoplay policy**: always launch tests with
  `--autoplay-policy=no-user-gesture-required` but ALSO test without it under
  mobile emulation.
- **The old title hides on several screens**: splash, menu, level select,
  settings, pause, game over, credits… Removing it only from the first screen
  ships the old branding to every player who opens the credits. Sweep every
  screen, then verify screen by screen (Phase 5, steps 1 and 5).
- **The delivery name IS the original name (v1.4.0)**: repo name, Pages URL
  path, ZIP name, README title, `<title>`, PWA manifest all carry the game's
  ORIGINAL title — no invented name anywhere. And the game itself ships
  unnamed: the original name must have 0 hits INSIDE the shipped build (grep
  HTML/JS/JSON/CSS case-insensitively), and no external references (portal
  credits, "powered by", portal links) anywhere.
- **Originality screening (v1.4.0)**: reject classic/known-concept games
  (2048, snake, solitaire, Tetris-like, memory, flappy clones…) and famous-
  title clones BEFORE downloading — read screenshots/gameplay, not just the
  title.
- **Famobi games are out even via third-party portals (v1.7.0)**: a portal
  can license and embed Famobi games under its own skin — the wrapper still
  betrays it. Before downloading, grep the candidate's page/index/runtime
  for `famobi` case-insensitively (hosts `play.famobi.com`,
  `games.cdn.famobi.io`, loader `famobi.js`/`famobi_noa.js`, globals
  `window.famobi*`/`Famobi`, "Powered by Famobi" credits). Any hit =
  disqualified.
- **ZIP contents (v1.3.0)**: game files ONLY — no README, no docs, no
  capture/test scripts. `unzip -l` before every release.
- **File names must be clean (v1.3.0)**: no `poki_nettoyé.js`, no
  `sdk-stripped-final2.js`, no portal names, version suffixes, non-ASCII or
  spaces in shipped names — rename during Phase 1/2 and fix all references.

## Reference implementation

A complete working example of every artifact this skill describes (driver
files, firewall tests, atlas resizing, deploy scripts) lives in the
`poki-run2/` per-game builds (`game-driver.js` each) in the agent
workspace that produced it. Reproduce the same artifact names so future agents
can navigate quickly: `game-driver.js`, `serve.sh`, `README.md`,
`resize_atlas.js`, `test_*.js` — the `test_*`/capture scripts stay in the
workspace and are never pushed to a game repo or added to a ZIP.
