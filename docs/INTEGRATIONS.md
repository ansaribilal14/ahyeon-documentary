# Integrations — MoneyPrinterTurbo + ytagent

This repo integrates two external systems into the S0–S8 documentary pipeline.
Both integrations are **optional, env/secrets-gated, and CI-safe**: with no keys
set, the pipeline behaves exactly as v1.0 (license-pure media, zero behavior change —
verified by byte-identical `timeline.json` rebuild).

---

## 1. MoneyPrinterTurbo (harry0703/MoneyPrinterTurbo, MIT) — `pipeline/mpt_bridge.py`

**What was adopted.** MoneyPrinterTurbo's proven *footage matching* stage: for every
script beat's search query, call a stock-footage API, download HD clips, then
normalize them to the project's frame (`frame fitting: crop`), and feed them into the
media pool with per-clip license + author attribution.

| MPT concept | Our implementation |
|---|---|
| Pexels / Pixabay video search per segment | `pexels_videos()` / `pixabay_videos()` per beat query (81 queries in this project) |
| `frame fitting: crop`, clip duration caps | `normalize()`: scale→crop→1920×1080 30fps, ≤12 s, silent |
| Per-material metadata retention | manifest entries `kind:"video"` with license, artist, origin URL |
| Provider-key indirection | env `PEXELS_API_KEY` / `PIXABAY_API_KEY` (or GitHub Secrets of the same name) |
| Batch/no-key degradation | clean skip — base media unchanged |

**Enable:**
```bash
export ENABLE_MPT_STOCK=1
export PEXELS_API_KEY=...      # free: pexels.com/api
export PIXABAY_API_KEY=...     # free: pixabay.api
python pipeline/mpt_bridge.py          # or --dry-run
```
**CI:** set repo Secrets `ENABLE_MPT_STOCK=1`, `PEXELS_API_KEY`, `PIXABAY_API_KEY`.
**Licenses:** Pexels License / Pixabay Content License — commercial use permitted;
attribution not required but we record it anyway (auto-appended to the YouTube
description by `build_timeline.py`).

## 2. ytagent (Bilal140202/ytagent, MIT) — `pipeline/ytagent_bridge.py` (LOCAL ONLY)

**What it does.** Sources *subject footage* (official BABYMONSTER uploads) listed in
`project/subject_sources.json`, downloads them via ytagent's 13-method fallback
chain, cuts short commentary excerpts (≤10 s, silent, 1920×1080 30fps), and adds
them to the media pool tagged for human review.

**HARDENED RULE R8 (2026-10-13): GitHub Actions must NEVER download YouTube videos.**
The former remote download farm workflow (`.github/workflows/yt-download-farm.yml`)
has been REMOVED from this repository and from the engine. All YouTube
acquisition happens locally on our own server via ytagent's direct methods
(android_vr + BGutil POT, SOCKS5 relays, Cobalt, Invidious) — never on
GitHub-hosted runners.


## 3. Timeline + render support for real footage

- `build_timeline.py` now indexes `kind:"video"` assets per query and assigns them to
  beats (reuse cap 2, deterministic order, **drops entries whose files are missing**).
  `USE_VIDEOS=0` disables globally.
- `render.py` Phase 1 renders video slots via `scale→crop→fps→eq→fades` with
  `-stream_loop -1` (loops short clips to fill the slot). Images keep Ken Burns.
- Videos live outside git (`project/media/videos/`, `project/ytdl_cache/` in `.gitignore`);
  in CI the bridges run **before** the deterministic timeline rebuild in the same job.

## 4. Quick matrix

| Want | Do |
|---|---|
| v1.0 behavior, zero changes | set nothing (default) |
| + stock footage (Pexels/Pixabay) | `ENABLE_MPT_STOCK=1` + keys |
| + subject clips via LOCAL ytagent | `ENABLE_YTAGENT=1` (local) |
| render without videos | `USE_VIDEOS=0 python pipeline/build_timeline.py` |
