# MacNativeGames — code evaluation (2026-10-19)

Evaluated at upstream commit `f5cc151` (ly05010419/MacNativeGames, "Merge pull request #16 …
totk-precompiled-shaders"). Read-only review by three parallel Claude reviewers (build pipeline,
native runtime, graphics layer), consolidated here. File:line citations refer to that commit.

Archival copies of the upstream repo at this commit live on the `archive` branch of this fork
(see "Archive" at the bottom).

---

## TL;DR

- **Clever, coherent, correctness-first engineering.** The game's own ARM64 code runs natively after a
  build-time rewrite; a small in-process Horizon host answers system services; NVN is replayed onto
  Metal 4. Everything fails closed: hash-verified inputs, guard-word-checked patches,
  `runtime_fail`/`die()` on anything unknown.
- **As code, it's hard to maintain.** Code-golf-dense C/ObjC, one 13,165-line graphics file
  (`nvn_mtl.m`, 1.23 MB), ~250 environment-variable toggles, debug/probe code compiled into the
  shipping binary, and D2R (Diablo II: Resurrected) leftovers throughout.
- **Build pipeline is the cleanest part.** List-argv subprocesses only, keys kept out of argv/logs,
  atomic staging, SHA manifests for every stage. 38 tests, all passing (3 Mac-only skips).
- **Main risks.** Two plausible IPC lifetime races. Shader-translation failure kills the game.
  Synchronous, serialized shader translation (the cause of first-use stutter). Deliberately
  unbounded caches (the cause of long-session memory growth). A checked-in prebuilt binary is
  trusted by an adjacent hash only.
- **Reuse for another title (e.g. BotW).** Plausible in principle, but the "generic" layers are
  hard-wired to TOTK (names, offsets, module set, single RomFS, title hooks inside the graphics
  god-file). Expect to generalise/split before porting.

---

## How it differs from a Switch emulator

| | Emulator (Ryujinx/yuzu) | This project |
|---|---|---|
| CPU | JIT-recompiles guest ARM64 | Runs it natively; only forbidden instructions (x18, tpidr*, `svc`, `cntpct`) are rewritten at build time |
| OS | Broad Horizon reimplementation | Only the services TOTK calls, in-process |
| GPU | Low-level Maxwell GPU / command-stream emulation | High-level: NVN API calls reimplemented on Metal 4 |
| Shaders | Runtime translation | Same Ryujinx translator, NativeAOT-compiled, seeded + prewarmed |
| Scope | Hundreds of games | One title, per-version patch tables (1.0.0 / 1.1.2 / 1.4.3) |

Closer to "Wine for one game" than to an emulator. That gives speed and native Mac feel, at the
cost of fragility and per-title effort.

---

## 1. Build & packaging pipeline (`build_app.py`, `games/totk/scripts`, `tools`, tests, CI)

**Verdict:** careful and defensive. Every stage is gated by the SHA-256 manifests
(`games/totk/assets/expected-v*.json`). Title keys stay out of argv and logs, subprocesses use list
argv (no `shell=True`), and staging uses atomic `.partial` files followed by a rename.

**Tests:** `python3 -m unittest discover -s tests` (Linux, Python 3.13) gave **38 tests, OK,
3 skipped** (2 Mac-only Foundation importer, 1 needs rsync). The C path parser under
`cc -fsanitize=address,undefined tests/test_romfs_paths.c` printed `PASS: 27 RomFS path cases`.

Findings (ranked):

1. **Medium (confirmed): `-o` can delete any existing `.app`.** `step_app` does
   `shutil.rmtree(out)` (`build_app.py:669-670`); `main` (`:496-499`) only checks the `.app`
   suffix, so `-o /Applications/Foo.app` silently wipes Foo. `install_desktop_entry.py:28-29`
   already checks `CFBundleIdentifier`, and build_app should do the same.
2. **Medium (supply chain): prebuilt `games/totk/prebuilt/macos-arm64/totk_xlate`** (5.1 MB
   Mach-O) is checked only against an adjacent `.sha256` (`build_app.py:512-516`). That proves
   integrity, not provenance; it isn't reproducible, and it runs inside the shipped app.
3. **Medium (supply chain): pip deps are version-pinned but not hash-pinned.** There is no
   `--require-hashes` and transitive deps are unpinned. `nsz` receives the prod.keys path
   (`:207`).
4. **Low–Med: offline resume fails.** `pip install` runs every time (`:164`).
5. **Low: raw tracebacks.** A corrupt `index.json` (`:325`) raises a raw error, and so does a
   `--check` that exceeds its 1800 s timeout (`:746`).
6. **Low: temporary title-key file** (0600, in a TemporaryDirectory inside the work folder,
   `extract_update_romfs.py:56-68`) survives SIGKILL or power loss. Otherwise key handling is
   good (`--suppresskeys`, line-number-only errors).
7. **Low: downloads are pinned by git commit (SHA-1), not archive hash.** hactool is patched by
   exact string replacement and fails loudly on drift. A half-patched checkout needs a manual
   `rm src/hactool`.
8. **Low: portability.** `/opt/homebrew` is hardcoded (`:49,106`, Makefile). The Python 3.14
   lookup is Homebrew/`/usr/local` only, so pyenv, uv and python.org installs are missed. The
   free-space check ignores `-o` on another volume (no APFS clone there, so a full 17 GB copy).
9. **Low: code quality.** About 100 lines of hactool C patches live as Python string literals.
   The Makefile link rules are triplicated. `extract_romfs.py` (integrity-critical IVFC code) is
   written as dense one-liners.
10. **Test gaps.** CI runs only `tests/`; the 13 probe suites under `games/totk/tests/` are
    Mac-only and not run. Also untested: hactool/SPIRV-Cross builds, prereq checks, and real
    `-o` handling.

Strengths: the version is identified by ExeFS hash, never by filename. All 6 IVFC levels are
verified. The RomFS index rejects cycles, `..` and NUL. Personalized and conflicting tickets are
rejected. The previous app survives a failed build, and the build waits for a running game to
exit. Desktop migration verifies and backs up saves. GitHub Actions are SHA-pinned with
`contents: read`.

---

## 2. Native runtime (`games/totk/runtime`, `third_party/d2r`, `app/launcher.m`)

**Verdict:** a coherent design that fails closed. The parsing of untrusted data is careful. It
is hard to maintain: 166 lines over 200 characters, 76 `TOTK_*` env vars, and debug code
interleaved with production paths. There are no unit tests for IPC, threads or memory SVCs.

How it works:

- **Rewriter:** `third_party/d2r/patch_nso.py`, invoked from `build_app.py:308` and
  `Makefile:42-47`. It maps x18 and tpidr* to Darwin TSD slots through a key page; the loader
  fills in the real key numbers (`loader.c:173`) and validates the layout at startup
  (`loader.c:288-290`). `svc` and `cntpct` jump through one register-saving gate (`gate.S`).
- **W^X:** modules are mapped RW, patched, relocated and SHA-checked, then `mprotect`ed to RX
  (`loader.c:185,246-253`). This works because the app is ad-hoc signed **without hardened
  runtime** and with **no entitlements** (`build_app.py:609`). Upside: no dangerous
  entitlements. Downside: it can't be notarized and gets no hardened-runtime protection.
- **SVC dispatch:** each call does a linear search of all gates by return address
  (`loader.c:298-306`) — a perf cost, not a bug.
- **HLE functions:** 4096 thunks switch onto the guest pthread's own 8 MiB stack
  (`hle_stack.S`, `threads.c:340,241`). This assumes no host replacement takes stack-passed args
  (`hle_stack.S:7-9`), and nothing enforces that.
- **Threads:** one pthread per guest thread, with one global mutex `G` for all sync
  (`threads.c:12-22`): correct, but it will contend under load.

Findings (ranked):

1. **Medium (plausible UAF): IPC session clone.** `CloneCurrentObject` shares `s->state` with
   `close=NULL` (`ipc.c:150-152`). Closing the original frees the state under the clone
   (`ipc.c:56-60`, `threads.c:524-529`).
2. **Medium (plausible race): session lookup vs close.** `runtime_handle_get` returns the
   session after releasing `G` (`threads.c:586`), and `dispatch` then uses it unlocked while
   CloseHandle may free it (`ipc.c:194-201`). Handle slots are reused immediately with no
   generation counter (`threads.c:77-79`).
3. **Low–Med (confirmed): read-before-check in `ipc.c:parse`.** Descriptors, up to 1023 data
   words and the receive list are read before the `MSG_BYTES` bound is checked
   (`ipc.c:110-124`). With svc 0x22 (user buffer) this can over-read. `ipc_guest` only checks
   null and overflow (`ipc.c:62-66`).
4. **Low (confirmed): memory accounting.** SetHeapSize writes `heap_size` without
   `memory_lock`, and its pool check ignores `alias_bytes` (`services.c:320-323`). QueryMemory
   scans the 64 GiB maximum, not the actual alias size (`services.c:307`).
5. **Low: crash handler** calls `getenv`, `strtoul` and `open` inside a signal handler
   (`loader.c:351-384`), and rewinds PC at a hard-coded game address to dodge a game bug
   (`loader.c:316-333`).
6. **Info: leftovers.** D2R build-ID tables in `patch_nso.py:54-160` (including a Battle.net
   offline bypass; inert for TOTK). iOS code remains (`frontend_ios.m`, `TARGET_OS_IPHONE` ×10).
   Stale frame-size comment at `gate.S:20`.
7. **Code quality:** `threads.c` 48 KB, `audio_renderer.c` 44 KB, `loader.c` 38 KB. A single
   ~2 KB `strcmp` dispatch line (`loader.c:449`). Many raw result codes.
8. **Version fragility.** Patches check the original instruction words first
   (`game_patches.c:370-372`, `game_patches_v143.c:72-76`), so a mismatch fails loudly. But
   v1.0/1.1.2 use `GAME_ADDR()` macros while v1.4.3 uses raw literals in a separate file, and
   the Makefile compile line is copied per version (`Makefile:72,338,348`).

Strengths: fails closed everywhere (`ipc.c:79-84,164-166`). RomFS and save paths reject
traversal (`romfs_path.h`, `fs_host.c:96-115` with `O_NOFOLLOW|O_EXCL`). Audio wire parsing is
strictly bounded (`audio_renderer.c:139-174`, `audio_mixer.c:209,234`). 4 KiB permission
tracking with RELRO on 16 KiB pages (`loader.c:55-114`). Good "why" comments. Vendored-code
provenance is documented.

---

## 3. Graphics layer (`third_party/d2r_gfx`, `gfx_bridge.c`, shader runtime, `tools/totk_xlate`)

**Verdict:** strong engineering underneath, with explicit lifetime/ordering discipline. As code
it is close to unmaintainable: `nvn_mtl.m` is 1.23 MB and 13,165 lines (lines up to 815
characters, implementation `#include`d from `.h`/`.inc`), `hle_nvn.c` is 2,704 lines, there are
250 unique `getenv` toggles (214 call sites in `nvn_mtl.m`), and ~20 `mtl_probe_*` self-tests are
compiled into the ship binary.

Architecture:

- **Pools:** each becomes a page-aligned `newBufferWithBytesNoCopy`, with GPU address = CPU
  address, found through a sorted index (`nvn_mtl.m:653-711`). Pools join a single
  `MTLResidencySet` lazily (`:645`).
- **Command buffers:** recorded as `{op,size}` records (`nvn_obj.h`) and replayed on a separate
  thread into an `MTL4CommandQueue`.
- **Sync:** an `MTLSharedEvent` plus commit-feedback handlers drive `g_done` and deferred
  releases (`:1886-1985`).
- **Hazards:** none tracked. A full `MTLStageAll` barrier wraps every encoder (`:1017-1048`).
  Measured cost: 11.82 ms vs 10.66 ms per frame without barriers. Uploads, readbacks and
  presents still use a legacy queue ordered by `usleep` polling (`:933`).
- **CPU writes to textures** are caught by write-protect faults, with a lock-free page state
  machine (`nvn_dirty.c`).
- **Textures:** block-linear deswizzle on CPU and GPU, bcdec, BC→ASTC.
- **Shader cache:** keyed by SHA-256 of control+code plus a 48-bit hash of the "answers" JSON,
  namespaced by translation-chain version (`totk_shader_rt.m:136-209`). The seed import is
  hash- and regex-validated.
- **Prewarm:** blocks the first launch (~10 min on an M4) on a serial queue
  (`nvn_mtl.m:6151-6200`).

Findings (ranked):

1. **High (maintainability): god-file and toggle sprawl** (see verdict).
2. **High: translation failure kills the game.** `totk_rt_variant` calls `die()` on any
   translator error (`totk_shader_rt.m:186-191`), and `prog_fit` does too (`nvn_mtl.m:5401`).
   There is no skip-draw fallback on macOS; the iOS path has one (`:170-180`).
3. **High (perf): translation is synchronous and serialized.** It does pipe IPC to a single
   service process under the global `g_rt_lock` from the replay thread (`totk_shader_rt.m:148-208`,
   `nvn_mtl.m:5399`), and Metal library/PSO compiles block the same thread (`:5855`). This is
   the first-use stutter.
4. **Medium: unbounded memory by design.** `mtl_dead_forget` is a no-op (`:3458`, admitted at
   `:3423`). `g_shcache` and `g_pso` never evict, and PSO keys use raw `NvShader*` pointers.
   `g_live_mem` keeps malloc'd texture-object copies (`hle_nvn.c:378`). Failed PSO builds aren't
   negative-cached, so they are retried every draw (`:5858`). This is the long-session memory
   growth.
5. **Medium: pool teardown is not fence-deferred.** `mtl_pool_del` releases the pool
   immediately (`:673-686`). It relies on `x_drain()` beforehand (`hle_nvn.c:228`), while
   everything else uses `defer_obj_release`.
6. **Medium: CPU deswizzle has no bounds.** `nvn_deswizzle` takes no source or destination
   length (`tex/nvn_deswizzle.c:63-79`), and `nvnTextureInitialize` never checks
   `off + size <= pool->size` (`hle_nvn.c:357-364`). Only the GPU path checks
   (`nvn_mtl.m:2472`).
7. **Low:**
   - The service reply is read with a fixed `fgets(line, 2048)` (`totk_shader_rt.m:184`).
   - `mtl_wait` uses `CLOCK_REALTIME` and gives up after 1 h (`:1987`).
   - `warm_start` busy-waits with no timeout (`:6191`).
   - A variant is marked "broken" by searching the MSL for strings (`totk_shader_rt.m:158`).
8. **Low: dead and stale code.**
   - D2R leftovers: `mtl_ui_ghost`, `g_rt_ovr` stencil masks, Travincal comments.
   - iOS branches remain.
   - `d2r_gfx/README.md` and `vendor/ryujinx/README.md` are stale.

Ryujinx and licensing:

- The vendored Ryujinx 1.3.2 subset is unmodified. Four small, well-explained AST/IR patches
  are applied at build time (`tools/totk_xlate/patches/0001-0004`), plus a hash-gated
  `VertexCounterLowering.cs`.
- The MIT notices are kept, and `THIRD_PARTY_NOTICES.md` is complete, including the .NET runtime
  inside the NativeAOT binary.

Strengths: completion-only side effects; the `defer_obj_release` vs `defer_obj_unref`
distinction (`:323-334`); a submit-thread ownership assert (`:918`); per-submission autorelease
pools; GPU probes with negative controls (`gpu_path_probe.m`); fix comments that cite their
measurements; atomic cache index writes.

---

## Reuse for Breath of the Wild (notes for later, on the Mac)

**Likely carries over:**
- The NSO rewriter and gate.
- The loader, threads and IPC.
- Most Horizon services: fs, saves, audio, HID.
- The NVN→Metal replay core: pool mapping, dirty-page tracking, block-linear/BC code.
- The shader translation chain and cache.

**Needs new work:**
- BotW patch sites and guard words.
- BotW expected-hash manifest and version detection.
- Any services or NVN features BotW uses that TOTK doesn't.
- Its own shader seed and PSO manifest.
- De-TOTK-ing names (`TOTK_*`, module set, single RomFS mount).
- Moving the 11 TOTK-address sites and MetalFX/occlusion hacks out of `nvn_mtl.m`.

**Suggested order:**
1. Build TOTK on the Mac first to prove the dumps, keys and toolchain.
2. Fork the title-specific bits into a `games/botw/` sibling.
3. Aim for "boots to title screen" as milestone 1.
4. Fix the hardening items above (IPC races, `die()` on translation failure, deswizzle bounds) as
   they're hit.

Prerequisites: an Apple-silicon Mac on macOS 26+, your own BotW/TOTK NSP dumps + `prod.keys` +
firmware (needed for fonts with an NSP base). Original-Switch builds only; the Switch 2 Edition
executables are out of scope.

---

## Archive

The `archive` branch of this fork holds, under `archive/2026-10-19/`:

- `MacNativeGames-upstream-f5cc151.bundle`: a git bundle of upstream `main`'s full history
  (37 commits). Restore with `git clone MacNativeGames-upstream-f5cc151.bundle MacNativeGames`.
- `MacNativeGames-f5cc151-source.tar.xz`: a plain source snapshot at `f5cc151`.
- `SHA256SUMS`: checksums for both.
