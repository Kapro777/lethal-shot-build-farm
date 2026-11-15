# HANDOFF — Lethal Shot: native port bring-up

> Complete context for the next engineer/AI taking over. Nothing here is
> game code or copyrighted material: this document, the repositories and
> the logs are tooling, build infrastructure and per-title metadata
> (addresses, section layout, import counts) of the kind public
> recompilation projects carry. The proprietary inputs and outputs live
> only in the private repositories.

Read `PROMPT.md` first — it is the prompt for whoever works next. This
file is the operational detail behind it.

## 1. Mission

Statically recompile the owner's own 2002-era console disc into a native
executable: **Windows x64 bring-up first**, then **Android arm64** as the
deliverable. Product shape is the GDroid/Hades-port model: a distributable
**engine**, the **user supplies their own copy of the game**, it runs
natively. Legal posture is non-negotiable:

* `default.xbe`, the disc contents, generated game C and built `.exe`
  files are derivatives of a retail executable. They live ONLY in the
  private repositories / private artifact stores / the owner's machine.
* Public repositories contain engine code, CI, docs and per-title
  metadata only, with legal notices in every README.

## 2. Repositories (all under GitHub account `Kapro777`)

| Repo | Visibility | Contents |
|---|---|---|
| `Kapro777/lethal-shot-engine` | **public** | host engine `src/main.c`, `src/recomp_manual.c`, `CMakeLists.txt`, `config/seed_functions.json`, `config/trace_functions.json`, `docs/` (bring-up log, targets), `tools/bringup/` (disassembler, generated-C analyser, safe runner) |
| `Kapro777/lethal-shot-build-farm` | **public** | CI: `.github/workflows/build.yml`, docs (this file + `PROMPT.md`) |
| _private input store_ | private | the title executable for CI -- referenced only by repository secret, never named |
| _private artifact store_ | private | built `.exe` per CI run, as releases tagged `run-<run_id>` -- referenced only by repository secret |

Toolkit: `github.com/sp00nznet/xboxrecomp`, pinned at **v0.17.0** in CI.
Everything is on branch `main` in every repository — no feature branches;
the CI, the fingerprints and the cache keys all assume `main`.

## 3. How to work here (GitHub-only, one repository at a time)

The next agent works in a single repository at a time; the owner relays
between them. The split is:

* **`lethal-shot-engine`** — where changes are made (the real work:
  `src/main.c`, `config/*.json`, `docs/bringup-log.md`).
* **`lethal-shot-build-farm`** — where builds are triggered and results
  are read (`gh workflow run build`, the Actions tab, the smoke-run
  artifact), and where CI behaviour is changed (probe env vars, cache
  keys).

The relay for one iteration: (engine repo) fix → (build-farm repo)
`gh workflow run build -R Kapro777/lethal-shot-build-farm --ref main` →
(build-farm repo) read the `SMOKE_SUMMARY` line in the job log and, if
more detail is needed, the `smoke-run-<run-id>` artifact → repeat. The
engine repo does not trigger builds on push, so a dispatch (or a
docs-only push to the farm repo) is what starts a run; the CI checks the
engine repo out fresh, so uncommitted engine changes are invisible.

The workflow's **smoke run** executes the freshly built exe on the
windows runner for up to two minutes with the runtime watchdog
(`RECOMP_WATCHDOG_SECS=25`) and publishes the stderr as a build artifact.
That is the primary feedback channel — no local machine is involved at
any point. A crash is data, not a failure; the smoke step never fails the
build on the game's exit code.

The pipeline touches two private stores: an input store (the title
executable for CI) and an artifact store (built binaries). Both are
referenced only by repository secret from inside the workflow — neither
their names nor their contents are named anywhere in this documentation.
An engineer who needs the title executable locally (for the
disassembler) asks the owner for it.

## 4. Auth / secrets

* The workflow authenticates to the two private stores with a repository
  secret (a PAT), also referenced only by name inside the workflow.
  Repository renames did not break it (verified after the 2026-11-15
  renames).
* `gh` CLI is logged in as the repository owner.

## 5. Current state (this handoff)

**Working** (each was a diagnosed root cause — the full trail with
disassembly and probe evidence is in the engine repo's
`docs/bringup-log.md`; read it before touching anything):

* Full pipeline in CI: `xbe_parser` → `disasm` → `func_id` →
  `abi_analysis` → `recomp` → CMake configure → MSVC Release build →
  exe published to the private store. ~2.5 min pipeline, ~14 min compile
  (cached).
* The recompiled game boots through its real entry point (`0x000DAB5D`):
  CRT startup, 103/103 kernel imports resolved, heap, save subsystem
  (TDATA/UDATA creation), `XeLoadSection`-by-name — **201 kernel calls**
  before the D3D work starts.
* D3D device create completes its setup: the 512 KB push-buffer ring is
  allocated (`sub_000ED7E0`), `MmClaimGpuInstanceMemory` + the GPU ISR
  (`KeConnectInterrupt`, vector 3) + the SMC/PCI probes run, and the
  96-byte device context is allocated. Two recompiler defects on the way
  are fixed: 808 seed functions (converged locally) plus the recomp flag
  `--entry-stubs`, which is what makes the tail-jump entry
  `0x000EE943` exist at all.
* The push-buffer wait at `0x000EECD6` **exits** (4 busy-loop iterations,
  was 12,922). The library keeps its PFIFO user-DMA channel at guest VA
  `0x00800000` (PUT +0x40, GET +0x44) and kicks a doorbell at VA
  `0x80000000`; `engine_pfifo_ack_thread` (engine, step 5c) reports
  submitted commands consumed — GET follows PUT — which is the same
  handshake the runtime models for its own aperture USER window. All
  constants are proven against the XBE disassembly and documented in
  `src/main.c`.
* The NV2A register model is installed (engine, step 5b: aperture
  `0xFD000000` trapped through a VEH into the toolkit's handlers), so
  the library's PFB flush spin at `0x000EDAB4` now clears and aperture
  reads answer hardware values.
* The VRAM window fault was routed to the toolkit's placeholder handler
  (`nv2a_hook_handle_vram` for `0xF0000000`–`0xFCFFFFFF`): the
  post-create read at `0xF71182A3`.

**Unverified at handoff time**: build `run-43760804327` contains the VRAM
fix, the repository renames and the smoke-run workflow; its smoke run had
not been read yet. Read it first (or trigger a fresh run) — it says how
much further the boot gets.

**Endpoint to expect**: after device create the library continues
(`0x000EED14` → `sub_000EE3C0` → surface/framebuffer setup → the first
`Present`). New blockers there will look like a hardware wait
(vblank/PCRTC counters), a pointer built from a kernel result that comes
back zero (the same shape as the PFIFO channel), or surface allocation
through the VRAM window.

**Not yet brought up** (expected next blockers): menus after D3D succeeds,
FMV playback (`Media/movie/*.wmv`, WMVDEC — the runtime plays `.wmv` via
Media Foundation when the game opens the file), audio (DirectSound/AC97),
saves to memory unit, then the whole graphics path (nothing renders yet:
the title submits an NV2A push buffer and no executor runs it).

## 6. The bring-up loop

1. Change the engine (`src/main.c` for modelling, `config/*.json` for
   pipeline inputs — both the seed list and the trace list are
   fingerprinted into the CI cache key, so changing them re-runs the
   pipeline).
2. From the build-farm repo: `gh workflow run build -R
   Kapro777/lethal-shot-build-farm --ref main` (or push a change to the
   farm repo).
3. Read the smoke run (job log tail + `smoke-run-<run-id>` artifact) and
   classify the endpoint:
   * `[CRASH] ... Xbox VA of fault: 0x...` — the new frontier.
     `py -3 tools/bringup/xbe_tools.py disasm <xbe> <va-32> <va+64>`
     shows the faulting instruction; `py -3 tools/bringup/analyze_gen.py
     whom <gen-dir> <addr>` finds the owning function in the generated C
     (the `generated-c-<run-id>` artifact).
   * An unresolved-call storm (`RECOMP_ICALL` failures) — seed
     convergence: the toolkit's `tools.seed_from_log` against the smoke
     run's err.txt and the XBE, then add entries to
     `config/seed_functions.json`.
   * A silent gap (kernel calls stop, no crash) — extend
     `config/trace_functions.json` around the last traced address.
4. Fix at the right level: engine modelling in `main.c` first, probes
   (`RECOMP_PEEK`, `RECOMP_POKE`, `RECOMP_WATCH`, `RECOMP_WATCHDOG_SECS`,
   all settable in the workflow's `env:` block) second, seeds for
   recompiler gaps third.
5. Update `engine/docs/bringup-log.md` with every root cause proven.
6. Bump the CI cache key prefix in `build.yml` (`lethal-build-vN-`)
   whenever pipeline arguments or the seed/trace sets change.

Owner-side shortcut for quick local checks (the owner's notebook, not
needed by the next agent): `tools/bringup/safe-run.ps1` runs a downloaded
exe constrained (idle priority, one core, watchdog, hard kill, game-dir
write snapshot).

## 7. Per-title facts (metadata only)

* XBE: title `0x54430004`, version 30724, XDK 1.0.4721, base `0x00010000`,
  entry `0x000DAB5D` (labeled `xbe_entry_point`), image `0x00395DC0`.
* Sections (11): `.text`(0x11000, 874 KB), D3D, D3DX, XGRPH, DSOUND,
  WMVDEC, XPP, `.rdata`(0x155FA0 = kernel thunk table), `.data`, DOLBY,
  `$$XTIMAGE`.
* Kernel imports: **103** (all bridged; `validate_ordinals.py` passes).
* Disk: `Media/3ddata/*.pkx`, `Media/**/*.xpr`, `Media/movie/scene*.wmv`,
  proprietary formats parsed by the game's own code.
* Boot flow mapped statically: `xbe_entry_point` → main thread
  (`sub_000DAAEE`, via `PsCreateSystemThreadEx` routine `0x000D8F30`) →
  save/storage init (`sub_000DD86B`…, `sub_000DD717` with the TLS-sanity
  check `[TLS+8] == 0xfffe0000` → `HalReturnToFirmware(4)`) → `sub_000D7FA0`
  → `sub_00011DD0` (platform/device init: `sub_000EBFE0` →
  `sub_000E8070` → `sub_000EE900` D3D device create) → `sub_00011F00`
  (the game's main loop object) → title exit (`sub_000D97B1` →
  `HalReturnToFirmware(2)`).
* The title uses the XDK sample framework (strings `OXBApp: Unable to
  create Direct3D!`, `XBUtil_FindMediaFile`), single-player, no network.

## 8. Reference docs

* `xboxrecomp/docs/GETTING_STARTED.md` — the canonical pipeline + build.
* `xboxrecomp/docs/technical/` — `candidate-games.md`,
  `lessons-learned.md`, `indirect-calls.md`, `d3d-translation.md` (the
  D3D8LTCG device-context layout and multi-entry render-flush pattern).
* `xboxrecomp/docs/runtime/environment-variables.md` — every
  `RECOMP_*` probe knob.
* `xboxrecomp/src/nv2a/README.md` — the NV2A model + MMIO hook.
* `lethal-shot-engine/docs/targets.md` — the Windows→Android plan
  (SDL2/GLES path, NDK, Gradle module, touch overlay, user-supplied game
  folder). Start that scaffolding in parallel with the boot work.
* `lethal-shot-engine/docs/bringup-log.md` — the full root-cause trail.
* `lethal-shot-engine/tools/bringup/README.md` — the diagnostic tools.

## 9. Rules for whoever continues

1. Nothing derived from the game ever goes public: no XBE, no generated
   C, no `.exe`, no release assets in public repos.
2. Every pipeline-argument change needs a cache-key bump
   (`lethal-build-vN-`).
3. Keep the bring-up log current — one entry per finding.
4. The engine repo has no CI; the farm repo triggers CI on push. Batch
   engine changes, then one dispatch.
5. The project is deliberately anonymous: no console or title names in
   code, docs, commit messages, or repository metadata. It is "the
   owner's own game disc" and nothing more specific.
6. Android is the destination: do not design anything Windows-only in
   `src/main.c` (the toolkit's POSIX/SDL2+GLES path is the Android
   backend; the MMIO hook's POSIX path is unimplemented and becomes a
   SIGSEGV handler there).