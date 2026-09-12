---
name: web-game-offliner
description: Download any web game (GameSnacks, Famobi, Softgames, Poki, Unity WebGL exports, etc.), strip its platform SDK, patch it into a fully self-contained offline build with a neutral driver, validate it with Playwright plus an application-level firewall, then deploy it to GitHub Pages under a NEW delivery title with the original branding fully removed (the game itself ships unnamed) and a clean ZIP release. Use when the user asks to localize, offline-ify, self-host, mirror, de-SDK, or republish a browser game.
version: 1.3.0
author: buffy
tags: [games, offline, download, playwright, github-pages, game-snacks]
---

# Web Game Offliner

Take any web game from a portal (GameSnacks, Famobi, Softgames, CrazyGames…,
Unity WebGL exports) and turn it into a **100% offline-playable standalone
build** with zero external dependencies, then deploy it. This skill encodes a
pipeline proven end-to-end on 5 games (Construct 3, Phaser 2/3, PixiJS 5,
custom canvas engines; Unity WebGL supported since v1.3.0).

## Pipeline overview

```
Phase 0  IDENTIFY    → catalog scan, engine fingerprint, game selection
Phase 1  DOWNLOAD    → full asset pull + integrity audit
Phase 2  PATCH       → SDK extraction → neutral game-driver.js
Phase 3  VALIDATE    → Playwright + application firewall
Phase 4  DEPLOY      → GitHub repo + Pages + ZIP release v1.0.0, ALL named
                       with the delivery title (invented in Phase 5 step 0
                       first) — game files only in the ZIP
Phase 5  DE-BRAND    → sweep EVERY screen for the old title, remove it
                       everywhere (neutral artwork at most — no new name is
                       inserted in-game), update metadata, republish
```

Never skip Phase 3: "it loads locally" is not "it works offline".

**Delivery naming is non-negotiable (v1.3.0)**: every game is delivered with
its original name completely removed — the portal's original title must not
appear anywhere in the delivery. The NEW invented title is used ONLY for
delivery naming (repo name, Pages URL, ZIP file name, metadata); it is NOT
integrated inside the game anymore — the shipped game itself carries no name
or external reference at all. See Phase 4 step 2 and Phase 5.

---

## Phase 0 — Identify the game and its engine

1. **Catalog scan** — GameSnacks' catalog is one big JSON payload. Fetch the
   homepage and pull every `/games/<id>` link (≈423 IDs). Each game page yields
   the **real CDN URL** hidden behind `h5games.usercontent.goog` plus its title.
2. **Engine fingerprint** — fetch each candidate's `index.html` and grep the
   runtime JS for:
   - `c3runtime` / `Scirra` → Construct 3
   - `Phaser` (check version, CE vs 3) → Phaser
   - `pixi.js` / `PIXI` → PixiJS
   - `cocos` → Cocos Creator
   - `BABYLON` → **disqualify (3D)**
   - `UNITY` / `UnityLoader` / `createUnityInstance` → Unity WebGL build →
     **eligible** (see the Unity WebGL playbook below)
   - `godot` → **disqualify**
3. **Watch for false positives**: matching the substring `constructor` is NOT
   Construct; "Phaser" inside a comment is NOT Phaser. Confirm on the runtime
   file, not the index.
4. **Verify 2D-ness before committing**: probe for 3D markers (`BABYLON`,
   `webgl` 3D pipelines, camera.z). One candidate (Retro Drift) turned out to
   be Babylon.js despite an initial "Construct" match — it was disqualified.
5. **Unity WebGL builds are eligible (v1.3.0)**: `UNITY`/`UnityLoader`
   markers are NOT a disqualification anymore. Recognize the export
   (`createUnityInstance` + `Build/*.wasm`/`*.data`/`*.framework.js`, or the
   older `UnityLoader` + `Build/*.json`), download the whole build and follow
   the Unity WebGL playbook below.

Selection criteria: 2D web-native games preferred; Unity WebGL exports
eligible regardless of rendering style; total size under ~20 MB (audit larger
Unity builds before committing); clean asset manifest; no aggressive DRM.

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
  └── GameSnacks.* / famobi / sg calls rewritten to GameDriver.*
```

1. **Inventory the SDK surface actually used** — grep the whole codebase for
   every call shape: `GameSnacks.`, `window.GameSnacks`, `famobi.`, `sg.`
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
   `GameSnacks.` → `GameDriver.` (watch for `window.GameSnacks` too). If the
   game has a fallback stub like `window.GameSnacks = window.GameSnacks || {}`
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
   `unzip -l <zip>` before shipping. Name it `<new-title>-v1.0.0.zip` (delivery
   title only — see step 2); verify zero `.git` entries inside.
2. **GitHub repo** (one per game), push, enable **Pages via API**
   (`gh api repos/<owner>/<repo>/pages -X POST`), wait for the Pages build
   (poll the API until `status: built`).
   **Repo naming rule — MANDATORY (v1.2.0, supersedes the old rule)**: the
   repo name MUST be the game's **NEW invented title** (Phase 5 step 0,
   kebab-case: `frosty-rush`, `pocket-golf`…), NEVER the original portal
   game id. **Every game is delivered with its original name completely
   removed**: the original name must not survive anywhere in the delivery —
   not in the repo name, not in the Pages URL (`<owner>.github.io/<new-title>/`),
   not in the ZIP file name, not in the README heading. No prefix, no suffix,
   no `-offline`. Invent the title BEFORE creating the repo. Since v1.3.0 the
   invented title is a **delivery label only**: it is NOT written inside the
   game (no splash, no logo, no `<title>` text beyond the delivery title,
   nothing). ⚠️ Do NOT confuse this repo with this skill's own repo
   (`web-game-offliner`): never create, fork over, or overwrite a repo named
   `web-game-offliner` when publishing a game.
3. **Live verification**: Playwright against the GitHub Pages URL — 0 errors,
   0 third-party requests, game advances past menus.
4. **README.md — STRICT template (v1.3.0, in ENGLISH)**. Exactly TWO sections,
   nothing else — no tech stack, no engine names, no modification lists, no
   run instructions, no badges, no screenshots section:

   ```markdown
   # <New Title>

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

Every deployed game gets its **original branding completely removed**: never
ship the portal's original title, logos or external references. Since
**v1.3.0 there is NO new in-game title anymore**: no replacement name, no
generated title image, no rebranding. The game ships **unnamed** — where the
old logo or title used to be, put **neutral artwork** (a clean abstract
shape, decorative frame or emblem with NO text) or nothing at all. The
invented title exists ONLY as the delivery label: repo name, Pages URL, ZIP
file name and README heading. The AI agent must invent it itself — pick a
short (2 words max), catchy, genre-fitting name (e.g. "Frosty Rush" for a
snowman puzzle game, "Pocket Golf" for a mini-golf game). Do not ask the user
to name it; propose it in the final report.

**Step 0 — invent the title BEFORE deploying**: Phase 4 names the repo, the
Pages URL and the ZIP after this title, so pick it (and run the uniqueness
check below) *before* creating anything public.

**ALL-OR-NOTHING RULE (zero tolerance)**: the old branding must be removed
from **every screen of the game**, not just the first one. A build where the
old name/logo survives anywhere (menu, level select, settings, pause, game
over, credits…) is a **FAILED build**. When this skill ships a game, the old
title must be completely gone: no asset, no string, no metadata, on any
screen — and since v1.3.0, **no new name is inserted in its place**: the
game simply has no name.

> **Scope note (v1.3.0 — supersedes v1.2.0)**: the invented title is a
> **delivery label only** — repo name, Pages URL, ZIP file name, README
> heading, `<title>`/PWA metadata. It is NOT integrated inside the game: no
> in-game logo, no splash text, no title-screen wording. The original portal
> name must not appear in any delivered artifact; a case-insensitive grep of
> the shipped build + repo for the original game id must return **0 hits**.

**MANDATORY — Uniqueness check before committing to a name**: before picking
the name, list the repos on BOTH publishing accounts and make sure the game
hasn't already been cleaned/published under that name (or a name too similar
to it):

- `dannyking6`  → `gh api users/dannyking6/repos?per_page=100 --jq '.[].name'`
- `d2658182-hub` → `gh api users/d2658182-hub/repos?per_page=100 --jq '.[].name'`

If a repo (or a previous build folder in the workspace) already matches the
candidate name or the same original game, reuse that existing work instead of
deploying a duplicate — and pick a different candidate name if the collision
is on the *name* itself. The name must be unique across both accounts.

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
   - i18n/locale strings, HTML overlays, CSS background images, PWA manifest,
     `<title>`/meta tags
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
   index.html and PWA manifest `name`/`short_name` carry the **delivery
   title** (Phase 5 step 0) — these are delivery surfaces, not in-game
   screens; i18n strings that embed the old name are removed or neutralized;
   `GameData.BuildTitle`-style constants that expose the old name are emptied
   or removed.
5. **Verify the de-branding is COMPLETE — screen by screen**:
   - for EVERY screen listed in step 1: navigate to it in Playwright,
     screenshot it, and confirm no old-title pixels remain AND that no name,
     logo or wording is rendered where the old branding was (neutral artwork
     contains no letters);
   - grep the ENTIRE shipped build (HTML/JS/JSON/CSS, case-insensitive) for
     the old game name AND the portal name — assert **0 hits**;
   - also assert the NEW title does NOT appear inside the game itself
     (0 in-game hits — it may live only on delivery surfaces: repo, Pages
     URL, ZIP name, README, `<title>`/PWA metadata);
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
- **SDK surface**: portal wrappers (Poki/GameSnacks/Crazy Unity plugins)
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
- **The original name must be gone from the delivery (v1.3.0)**: repo name,
  Pages URL path, ZIP name, README title, `<title>`, PWA manifest — grep the
  delivered repo for the original game id and assert 0 hits. And the game
  itself ships unnamed: no new title inside the game, no external references
  (portal credits, "powered by", portal links) anywhere.
- **ZIP contents (v1.3.0)**: game files ONLY — no README, no docs, no
  capture/test scripts. `unzip -l` before every release.
- **File names must be clean (v1.3.0)**: no `poki_nettoyé.js`, no
  `sdk-stripped-final2.js`, no portal names, version suffixes, non-ASCII or
  spaces in shipped names — rename during Phase 1/2 and fix all references.

## Reference implementation

A complete working example of every artifact this skill describes (driver
files, firewall tests, atlas resizing, deploy scripts) lives in
`gamesnacks-batch/*/game-driver.js` and `gamesnacks-local/` in the agent
workspace that produced it. Reproduce the same artifact names so future agents
can navigate quickly: `game-driver.js`, `serve.sh`, `README.md`,
`resize_atlas.js`, `test_*.js` — the `test_*`/capture scripts stay in the
workspace and are never pushed to a game repo or added to a ZIP.
