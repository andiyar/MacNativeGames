# Plan: SwitchKit core + Breath of the Wild port

Written 2026-10-19 from the read-only code evaluation (`docs/evaluation/2026-10-19-code-evaluation.md`).
This is a plan, not a status page: nothing below has been started or run yet.

**Goal.** Turn this TOTK-only project into a shared core ("SwitchKit") plus thin per-game folders,
so Switch games I own can be built into native macOS apps. BotW is the first new title and
drives the split.

**Ground rules**
- Personal use only. Use my own dumps (NSP), my own `prod.keys` and my own firmware. Nothing from
  the games ever goes in the repo.
- Follow `AGENTS.md`: built apps, extracted data, shader caches and work folders live **outside**
  the repo (e.g. `~/MacNativeGames-artifacts/`, `TOTK_build`, `--work`).
- Original-Switch builds only. Switch 2 Edition executables are out of scope.
- Claude does the building and coding on the Mac. My job: provide dumps, play, report what I see.
- Keep `main` mergeable with upstream (`ly05010419/MacNativeGames`). Do the core split on a
  `switchkit` branch, and merge `upstream/main` into it regularly so upstream fixes keep flowing.

**Prerequisites (on the Mac)**
- An Apple-silicon Mac on macOS 26+, 16 GB+ RAM, ~60 GB free.
- Xcode CLT, Homebrew, `brew install opus libvpx python@3.14`; optionally the Metal toolchain.
- My own dumps: TOTK base 1.0.0 NSP (+ optional 1.1.2/1.4.3 update), BotW base NSP + latest
  update NSP, `prod.keys`, and an unpacked firmware NCA directory (NSP bases need it for fonts).
- Claude Code running locally in this repo.

---

## Phase 0: prove the toolchain on TOTK (one session)
1. `git fetch upstream && git merge upstream/main` on `main`, to pick up anything new.
2. Build TOTK 1.4.3:
   `python3.14 build_app.py <totk.nsp> -k <prod.keys> --firmware <fw dir> --update <1.4.3.nsp> -o ~/MacNativeGames-artifacts/apps/...`
3. Run the test suite (`python3 -m unittest discover -s tests`) and the Mac-only probes under
   `games/totk/tests/`. Record the results.
4. **Honesty gate (me):** play 20–30 min. Check that it boots, saves and reloads, compare
   against the Switch, and try MetalFX/native 2K (key **3**) and interpolation (key **2**).

Exit: a working TOTK app built from my own dumps. If this fails, fix it here before anything else.

## Phase 1: inventory everything TOTK-specific (one session, no behaviour change)
Write `docs/switchkit/title-coupling.md`, listing each item with its file:line:
- **Names:** `TOTK_*` env vars (76 in the runtime, ~250 incl. gfx), bundle IDs, paths, log names.
- **Module set and mounts:** hard-wired `main`/`sdk`/`subsdk0`, the single RomFS mount.
- **Game patches:** `game_patches.c`, `game_patches_v143.c`, `GAME_ADDR()` in `game_version.h`,
  the crash-handler PC rewind (`loader.c:316-333`).
- **Graphics:** the 11 TOTK `main+0x…` sites in `nvn_mtl.m`, the MetalFX scene/UI split, the
  occlusion-query hacks, hash-gated shader lowerings (`VertexCounterLowering.cs`), the NVN 55.15
  object-size assumptions.
- **Build:** `expected-v*.json`, the version detection in `build_app.py:220-245`, triplicated
  Makefile lines.
- **Dead weight to delete:** D2R tables in `patch_nso.py:54-160`, `mtl_ui_ghost`, `g_rt_ovr`,
  iOS branches (only if we're dropping iOS), stale READMEs.

## Phase 2: extract the core (2–4 sessions, behaviour-preserving)
Target layout:
```
core/                 # SwitchKit: generic, title-agnostic
  rewriter/           # patch_nso.py (minus D2R tables)
  runtime/            # loader, gate.S, hle_stack, threads, ipc, services, fs, romfs, audio, input, frontend
  gfx/                # nvn_mtl split into: pools/residency, cmdbuf replay, textures, shaders/cache, present/MetalFX
  shader/             # totk_xlate -> generic xlate service + patches
  build/              # generic build_app pipeline (hash manifests, container/NCA/RomFS, signing)
games/<title>/        # thin per-title folder
  title.json          # title ID, versions, expected hashes, module list, bundle name
  patches_*.c         # guard-word-checked patch tables, one style (GAME_ADDR-like) for all versions
  gfx_hooks.c         # title-specific graphics hooks behind a small hook interface
  assets/             # pso_manifest, smoke points
```
Rules:
- Move code in small steps, and rebuild TOTK after every step.
- Rename `TOTK_*` env vars to generic `SK_*` names, keeping the old names as aliases for now.
- Collapse the debug/probe toggles: move probes out of the shipping build (a separate probe
  target), and drop measurement-only switches.
- Split `nvn_mtl.m` (13k lines) along the lines in the target layout. This is the biggest and
  riskiest step, so do it last within the phase.

**Gate after every step:** TOTK builds, unit tests pass, the probes match Phase 0, and a short
play check (me) shows no regression.

## Phase 3: BotW boots (open-ended, aim: title screen)
1. Create `games/botw/title.json`: title ID, the supported version (start with the latest
   update), expected hashes generated from my own dump.
2. Run the rewriter on BotW's modules and collect the patch sites it can't handle.
3. Boot loop: run → read `~/Library/Logs/MacNativeGames/<title>.log` → find the missing SVC,
   service or NVN feature → implement it in `core/` when it's generic, or `games/botw/` when it's
   BotW-only → repeat.
4. Build BotW's shader seed and PSO manifest the same way TOTK's is built.

Milestone: the BotW title screen renders with audio. **Honesty gate (me).**

## Phase 4: BotW playable
- Load a save, walk the Great Plateau, enter a shrine, save and reload, fast-travel, die and retry.
- Fix rendering bugs as found. Compare against the Switch side by side.
- **Honesty gate (me):** an hour of normal play.

## Hardening backlog (fix when touched, or before Phase 4)
From the evaluation, highest first:
1. Shader translation failure calls `die()` (`totk_shader_rt.m:186-191`, `nvn_mtl.m:5401`).
   Skip the draw and log it instead.
2. IPC clone use-after-free (`ipc.c:150-152`) and session lookup vs close race
   (`threads.c:586`, `ipc.c:194-201`). Use refcounted sessions and generation-tagged handles.
3. IPC read-before-bounds-check (`ipc.c:110-124`).
4. Deswizzle without bounds (`tex/nvn_deswizzle.c:63-79`, `hle_nvn.c:357-364`).
5. Unbounded caches (`mtl_dead_forget` no-op, `g_shcache`, `g_pso`, `g_live_mem`); negative-cache
   failed PSOs.
6. Async/parallel shader translation, off the replay thread (fixes first-use stutter).
7. Pool teardown should be fence-deferred (`nvn_mtl.m:673-686`).
8. `build_app.py -o` should refuse to delete a non-MacNativeGames `.app` (`build_app.py:669-670`).
9. Unit tests for IPC parsing, handles and memory SVCs, which today have none.

## Later titles
Once BotW works, each further game is a new `games/<title>/` folder. Rough cost order:
- **Cheapest:** same-engine sequels.
- **Then:** other first-party NVN games.
- **Then:** Unity/Unreal NVN games.
- **Most expensive:** games that draw through Vulkan or OpenGL, which need a new graphics backend.

---

## Next-session trigger (paste into Claude Code on the Mac, in this repo)

> Read `docs/PLAN-switchkit-botw.md` (whole) and the TL;DR of
> `docs/evaluation/2026-10-19-code-evaluation.md`. Execute **Phase 0** only: merge
> `upstream/main`, build TOTK from my dumps (I'll give you the paths to the NSPs, `prod.keys` and
> firmware), run the tests and probes, and stage the app outside the repo per `AGENTS.md`. Then
> give me a double-click path and what to check while playing. Don't start Phase 1 until I've
> passed the play check.
