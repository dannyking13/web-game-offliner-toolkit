# web-game-offliner 🎮

An agent skill for **Codebuff / any .agents-compatible agent**: download any
web game from a portal (GameSnacks, Famobi, Softgames, Poki, Unity WebGL and
Godot exports…), strip its platform SDK, patch it into a **100% offline
standalone build**, validate it with Playwright + an application firewall,
and deploy it to GitHub Pages **under the game's original name** — with the
in-game branding completely removed (the shipped game is unnamed) and a
clean ZIP release. Only original games qualify: classic games built on an
already-known concept and clones of famous titles are rejected; Unity and
Godot builds must stay under 20 MB.

> Proven end-to-end on 5 shipped games (Construct 3, Phaser 2, Phaser 3,
> PixiJS 5, custom canvas engines) — including the traps: poisoned 404 assets,
> frozen ad callbacks, 2048px mobile GPU texture limits, minified-code surgery.
> Unity WebGL exports are eligible since v1.3.0 and Godot exports since
> v1.4.0 — both are the **priority candidates** during game selection, with
> a hard 20 MB size cap, and only original games (no classic/known-concept
> titles) are processed.

## What the skill does

```
Phase 0  IDENTIFY    catalog scan → engine fingerprint → originality +
                     size screening → game selection
Phase 1  DOWNLOAD    full asset pull + integrity audit (magic bytes!)
Phase 2  PATCH       SDK extraction → neutral game-driver.js
Phase 3  VALIDATE    Playwright + network firewall + mobile emulation
Phase 4  DEPLOY      GitHub repo + Pages + README + game-files-only ZIP,
                     all under the game's original name
Phase 5  DE-BRAND    sweep EVERY screen for in-game branding, remove it
                     everywhere — neutral artwork at most, NO name is
                     inserted in-game — then metadata, republish
```

Every phase encodes the hard-won gotchas that break naive attempts:

- ✅ Ad callbacks (`adBreakDone`, `beforeReward`) **must fire** or the game freezes
- ✅ Sync vs async storage semantics matched to each wrapper (GameSnacks / Softgames / Famobi)
- ✅ 404-HTML-disguised-as-PNG detection via magic bytes
- ✅ Atlas resizing for mobile GPU 2048px texture limits
- ✅ Crash-dialog auto-recovery for transient WebGL/audio glitches
- ✅ Old-title removal on **every screen** (splash, menu, level select,
  settings, pause, game over, credits…) — grep the whole build for the old
  name, assert 0 hits, verify screen by screen in Playwright
- ✅ **No in-game name, no external references** — since v1.4.0 the game
  ships unnamed and the delivery keeps the game's ORIGINAL name: repo,
  Pages URL, ZIP name and README heading all reflect it; where the old logo
  was, neutral artwork or nothing. Portal credits, "powered by" links and
  sitelocks are removed entirely
- ✅ **Original games only** — classic games built on an already-known
  concept (2048, snake, solitaire, memory, Tetris-like, flappy/wordle
  clones…) and famous-title clones are rejected during selection; Unity and
  Godot WebGL builds must stay under a hard 20 MB cap
- ✅ **Game-files-only ZIP** — no README, no docs, no tooling inside the archive
- ✅ **Clean deliverable file names** — no `poki_nettoyé.js`-style names; the
  repo looks like a professional game project, capture/test scripts stay in
  the workspace
- ✅ **Strict two-section README (in English)** — "About the game" (a vivid,
  flattering description with zero tech talk) + "Controls" (one careful
  subsection per platform)
- ✅ Unity WebGL playbook — capture, compression handling, SendMessage/jslib
  shims, bundle-safe de-branding
- ✅ Live-URL verification after Pages build, not just local testing

## Install

```bash
npx skills add dannyking6/web-game-offliner --yes
```

Or manually: copy `SKILL.md` into your project at
`.agents/skills/web-game-offliner/SKILL.md`.

## Usage

Just ask your agent:

> "Download Smarty Bubbles from GameSnacks and make it playable offline,
> then deploy it"

The agent loads the skill and follows the phased pipeline with all validation
gates.

## Shipped proof

| Game | Engine | Live |
|---|---|---|
| Guardians of Gold | Construct 3 | [play](https://dannyking6.github.io/guardians-of-gold/) |
| Geometry Rush | Phaser CE 2.15 | [play](https://dannyking6.github.io/geometry-rush/) |
| Jewels Blitz 5 | Phaser + Softgames wrapper | [play](https://dannyking6.github.io/jewels-blitz-5/) |
| Daily Word Search | PixiJS 5 + GSAP | [play](https://dannyking6.github.io/daily-word-search/) |
| Smarty Bubbles | Custom canvas + Famobi wrapper | [play](https://dannyking6.github.io/smarty-bubbles/) |

## License

MIT
