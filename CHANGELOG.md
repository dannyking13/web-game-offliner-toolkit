# Changelog

All notable changes to the **web-game-offliner** skill.
Format based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.5.0] — 2026-09-15

### Changed
- **Selection is originality-driven, NOTHING else**: the engine-priority
  rule is REMOVED — no more Unity/Godot preference, 2D web-native games no
  longer "come next"; every engine and every game type (2D, 3D…) is treated
  equally, and no portal/platform is preferred either. Only the originality
  filter decides which games are processed.
- **3D is fully acceptable**: `BABYLON` no longer disqualifies a candidate —
  3D engines follow the generic WebGL pipeline like any other engine.
- **Hard 20 MB size cap extended to EVERY engine**: previously explicit for
  Unity/Godot with a softer "~20 MB" note for others, the cap is now one
  single rule for all engines alike (Unity, Godot, Construct, Phaser,
  PixiJS, custom…), HEAD-measured before download, no exception.

### Added
- **CrazyGames exclusion — hard rule**: the chosen game must NEVER already
  be published on CrazyGames. Before any download, search
  `https://www.crazygames.com/search?q=<title keywords>`; same title or
  unmistakably the same gameplay already there = candidate disqualified.

## [1.4.0] — 2026-09-15

### Changed
- **Delivery naming restored to the ORIGINAL game name (supersedes
  v1.2.0/v1.3.0)**: the repo name, Pages URL, ZIP file name and README
  heading must all reflect the game's original title exactly — the invented
  delivery-title mechanism is removed entirely. In-game branding is still
  fully removed: the shipped game carries no name (0 in-game hits for the
  original name), with neutral artwork or nothing where the old logo was.
- **Originality filter in Phase 0**: classic games built on an
  already-known concept (2048, snake, solitaire, memory, Tetris-like,
  flappy/wordle clones, bubble-shooter "classics"…) and clones/reskins of
  famous titles are rejected before download; familiar genres with a real
  original twist remain eligible.
- **Hard 20 MB size cap made explicit for Unity** (previously an
  "audit-larger-builds" soft rule): HEAD-measure every `Build/` file before
  committing; over the cap = disqualified.

### Added
- **Godot WebGL support**: recognition (`engine.startEngine` / `.pck` +
  `.wasm` fingerprint), full-export capture including side worklet/worker
  files, uncompressed-variant delivery, MIME requirements, SDK-wrapper
  neutralization, no-binary-surgery de-branding guidance — under the same
  hard 20 MB size cap as Unity.

## [1.3.1] — 2026-09-12

### Changed
- **Game selection now prioritizes Unity WebGL games**: actively seek them
  out and prefer them over other engines when choosing what to process
  (2D web-native games come next).

## [1.3.0] — 2026-09-12

### Added
- **Unity WebGL support**: dedicated playbook — export recognition
  (`createUnityInstance` / `UnityLoader`), `Build/` capture + manifest-based
  integrity audit, `br`/`gz` compression handling for GitHub Pages,
  `SendMessage`/`jslib` SDK shims, sitelock patching inside `*.data`,
  bundle-safe de-branding, `application/wasm` MIME + mobile-emulation testing.

### Changed
- **No new in-game title anymore** (supersedes the v1.2.0 retitle rule): the
  invented title is a **delivery label only** (repo name, Pages URL, ZIP file
  name, README heading, `<title>`/PWA metadata). Old branding is removed from
  EVERY screen and replaced with **neutral artwork (no text) or nothing** —
  the shipped game carries no name and no external reference.
- **Game README is a strict English template** with exactly two sections:
  "About the game" (vivid, flattering, tech-free description) + "Controls"
  (one carefully written subsection per platform: smartphone & tablet,
  desktop, trackpad note). No tech stack, no engine names, no modification
  lists, no run instructions.
- **ZIP contains game files only** — no README, no docs, no license files,
  no tooling; verified with `unzip -l` before every release.
- **Deliverable hygiene**: clean, conventional, English-only file names in
  the repo and the ZIP (no portal names, no "cleaned"/"patched"/version
  suffixes, no non-ASCII or spaces). Capture/test scripts stay in the
  workspace and are never pushed to a game repo.
- Phase 5 renamed RETITLE → DE-BRAND; verification now also asserts the new
  title has 0 hits inside the game itself.

## [1.2.0] — 2026-09-05

### Changed
- **Complete old-title removal enforced**: the original portal title must not
  survive anywhere in the delivery — repo name, Pages URL, ZIP name, README
  heading, metadata or in-game branding. Every screen (splash, menu, level
  select, settings, pause, game over, credits) is swept and verified
  individually; a case-insensitive grep of the shipped build + repo for the
  original game id must return 0 hits.

## [1.1.0] — 2026-09-04

### Changed
- Phase 4: game repos are named after the game's own delivery title — never
  the skill's own repo name (`web-game-offliner`), which must never be
  created, forked or overwritten when publishing a game.

## [1.0.x] — 2026-09-03

### Added
- Initial skill: phased pipeline (Identify → Download → Patch → Validate →
  Deploy) for fully offline web-game localization — GameSnacks, Famobi,
  Softgames, Poki capture playbooks, neutral `game-driver.js` pattern,
  Playwright + application-firewall validation, GitHub Pages + ZIP release.
- Phase 5: AI-invented catchy delivery title + styled title-image generation
  (PIL), with a name-uniqueness check across both publishing accounts.
