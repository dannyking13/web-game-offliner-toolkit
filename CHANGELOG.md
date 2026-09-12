# Changelog

All notable changes to the **web-game-offliner** skill.
Format based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

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
