# Tiptoe Treats — project guide

A single-file HTML5 canvas game for Kelly's young son (age ~5). Kid-friendly: drag
anywhere and a stretchy arm follows your finger to grab treats without touching the
guards. No ads, no lives, no timers — a fail is a gentle sound + instant retry.

This is a **standalone side project**. It is intentionally isolated from Kelly's other
work (DeepThought, scanners, etc.) and does not need to be reviewed in those sessions.

## Repo & deployment
- The **entire game is one file**: `index.html` at the repo root. No build step, no deps.
- `tiptoe-treats.html` is a tiny redirect stub to `index.html` (preserves old iPad
  home-screen shortcuts). `apple-touch-icon.png` is the whale-shark home-screen icon.
- Canonical git remote: `github.com/kellyb145/tiptoe-treats`. Local working copy lives
  in this folder (`C:\AI projects\TiptoeTreats`).
- Hosted on **GitHub Pages** (legacy build, `main` branch, root). Any push to `main`
  redeploys. Live at https://kellyb145.github.io/tiptoe-treats/
- **Only commit/push when Kelly asks.** A CLAUDE.md/doc change never needs to be pushed
  to make the game work — Pages only serves `index.html` + assets.

## Deploy gotchas (learned the hard way)
- **Never hand-commit a `CNAME` file.** Kelly does not own tiptoe-treats.com. Doing so
  once wedged Pages: every deploy failed with an opaque "Deployment failed, try again
  later," surviving even a Pages disable/re-enable.
- GitHub Pages intermittently returns **"Deployment failed, try again later"** with no
  real cause. The reliable fix is to push an **empty commit** (`git commit --allow-empty`)
  to retrigger a fresh deploy.
- The Pages build API (`/pages/builds/latest`) can report a **stale "building"** status
  while the underlying workflow run has already failed. Trust `gh run list` for the truth.
- Git Bash `curl` on Kelly's machine **fails TLS to github.io (exit 35)**. Use PowerShell
  `Invoke-WebRequest` for any live-URL checks.
- After a push, verify the live site actually serves the new bytes (fetch with a cache-bust
  query and grep for a new string) — don't trust "it deployed."

## Code map (all inside `index.html`)
- **`LEVELS[]`** (classic, currently 30) and **`MAZES[]`** (currently 20) — plain data
  arrays near the top. A level = `{scene, emoji, treat{type,x,y,stand}, haz[], blockers[],
  armMax?}`. `stand` ∈ table|shelf|basket|none. Mazes raise `armMax` (default 1600; mazes
  3000–4000) for the longer reach.
- **Hazards** (`haz[]` entries, each `{type,...}`):
  - `dog` / `parent` — cycle safe→warn(?)→active(!) using per-level `safe`/`warn`/`active`
    seconds and an optional `off` phase offset. `who`: dog=rusty|ruby, parent=mom|dad.
  - `bug` — patrols `x1,y1 ↔ x2,y2` at `speed`; **always active** (no safe phase). Drawn as
    a green **grasshopper** (`drawBug()`), but the internal type is still `'bug'` so patrol
    and collision logic are untouched.
  - `pot` / `cactus` — static, always-active awareness ring.
- **Two-tier collision**: `hazBodies()` = body circles, **always touch-sensitive** (poking a
  sleeping dog wakes him); `hazCircle()` = big awareness ring, hot only while `active`. Every
  arm point is checked each frame (game pads: body +6, active ring +10, blockers +10).
- **Avatars**: `drawKid()` dispatches on the global `avatar` ('boy' | 'girl'). `drawKidBoy`
  (original) and `drawKidGirl` (blonde pigtails, pink dress, tiara) share `drawKidFace()`.
  Menu picker (`#avPick`) persists the choice in localStorage key `tiptoe-avatar`.
- **Progress**: localStorage key `tiptoe-v2` = `{c:{unlocked,stars}, m:{...}}` (v1 migrates).
- Logical canvas 1000×700, floor y=615, kid x=145, shoulder (167,515). All art is
  canvas-drawn; all audio is WebAudio-synthesized.

## Level-design rules — DO NOT BREAK (impossible levels have shipped before)
1. **Every bug/grasshopper patrol must leave a PERMANENT safe lane** the arm's resting path
   can use. Never let a patrol sweep the full cross-section of a corridor the arm must lie
   across — that makes the level mathematically impossible.
2. **Corridors ≥ ~90px** for little fingers.
3. **Guards must open in their safe phase** so a level never starts hot (the cycle starts at
   phase 0 = safe, so this is automatic unless you fight it).
4. **A treat's grab point must clear every guard BODY circle.** Bodies are always
   touch-sensitive, so a treat sitting inside one is ungrabbable = impossible. Low-centre
   treats (whale TC 45, ray 48, candy 26) placed next to a floor dog are the classic trap;
   tall treats (rainbow 140, trex 80) clear the body. Keep the dog's body off the treat
   centre while its **awareness ring** still overlaps the treat (that's the intended
   "wait for the nap" gate).
5. **Multi-guard timing**: if two guards' awareness rings both overlap the arm's reach at
   once, staggered `off` offsets mean they're *never* both quiet → unwinnable. **Synchronize
   them** (same safe/warn/active, drop `off`). A comfortable band is `safe 3.2 / warn .7 /
   active 2.0`, which gives a clean "everyone looks away together" window.
6. Target difficulty: **hard but solo-solvable** by a patient 5-year-old. Fails are
   penalty-free, so "hard" is fine; "impossible" and "frustrating-tight" are not. When
   tuning, adjust per-level `safe`/`warn`/`active` seconds and bug `speed`; don't
   speculatively tweak — finish, then verify with the harness below.

## QA harness — validate EVERY new/changed level before pushing
The game's functions are global (classic, non-module script), so a headless browser can
drive the real game directly. Serve the folder (`python -m http.server`) and load it in
Playwright; then, in `browser_evaluate`, run an in-page solver:

- **Path**: BFS on a ~12px grid from `SHOULDER` to `treatCenter(L)`. Block: level `blockers`
  padded **+16** (the thin centreline must clear the arm's +10 collision pad or it snags on
  walls), parent/dog **body** circles from `hazBodies` padded +8 (use the *asleep* dog
  footprint — that's when you grab), and static pot/cactus rings padded +12. **Do NOT block**
  bugs or dog/parent **awareness rings** — those are timed; the sim handles them.
- **Simulate**: `start(mode,i)`; set global `t = phase`; `resetArm(); arm.mode='extend'`;
  each frame march a lead point along the path at ~800–1350 px/s, `addTowards(lead)`, then
  `update(1/60)`. Win iff `win===true`; fail iff `arm.mode==='fail'`.
- **Sweep ~48 phase offsets** per level (levels with moving/cycling hazards); require ≥1 win
  = solvable, and read the win-count as a difficulty gauge (a healthy "hard but fair" band is
  roughly 10–35 of 48). `0` = broken/impossible → fix. Also confirm path length ≤ `armMax`.
- Finally, drive one **real pointer-event drag** on the canvas and assert `win` / `#winov`
  to confirm the input wiring, not just the synthetic loop.

Visual QA: screenshot the menu (picker + grids), the boy, the girl, and a grasshopper level.

## Working notes
- Match the file's existing terse style (compact canvas helpers `rr`/`ell`/`line`, dense one
  hazard/level per line). Keep everything kid-gentle.
- When Kelly asks for difficulty changes, tune the per-level seconds/speeds — then re-run the
  harness across all touched levels (3×+ at different phases for anything with motion).
