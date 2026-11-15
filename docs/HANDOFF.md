# HANDOFF — Fatal Frame (Xbox) static recompilation, bring-up phase

> Complete context for another engineer/AI taking over. No game code or
> copyrighted material is in this repository or in this document; everything
> here is tooling, build infrastructure and per-title *metadata* (addresses,
> section layout, import counts) of the kind every public Xbox-recomp
> repository carries.

## 1. Mission

Statically recompile **Fatal Frame (Xbox, 2002, Tecmo)** — retail English
build — into a **native executable**: bring it up on **Windows x64** first,
then target **Android arm64** as the deliverable. Product shape is the
GDroid/Hades-port model: a distributable **engine**, the **user supplies their
own copy of the game**, it runs natively. Legal posture is non-negotiable:

* `default.xbe`, the disc contents, generated game C and built `.exe` files
  are derivatives of a retail executable. They live ONLY in private
  repositories / private artifact stores / the owner's machine.
* Public repositories contain engine code, CI, docs and per-title metadata
  only, with legal notices in every README.

## 2. Repositories (all under GitHub account `Kapro777`)

| Repo | Visibility | Contents |
|---|---|---|
| `Kapro777/fatal-frame-native-engine` | **public** | host engine: `src/main.c`, `src/recomp_manual.c`, `CMakeLists.txt`, `config/seed_functions.json`, `config/trace_functions.json`, `docs/` (legal notices, targets, bring-up log) |
| `Kapro777/fatal-frame-xbox-recomp` | **public** | CI build farm: `.github/workflows/build.yml`, docs (this file) |
| `Kapro777/fatal-frame-xbe` | **private** | the owner's `default.xbe` (input for CI) |
| `Kapro777/fatal-frame-builds` | **private** | built `.exe` per CI run, as releases tagged `run-<run_id>` |

Toolkit: `github.com/sp00nznet/xboxrecomp`, pinned at **v0.17.0** in CI
(local clone: `C:\Users\USER\fatal\xboxrecomp`).

## 3. Environment

* **Dev machine**: Windows, Celeron — light work only (Python pipeline steps,
  analysis). Never install VS/MSVC locally; compilations run in GitHub
  Actions. Local `cmake` is NOT installed and should not be.
* **CI**: `windows-latest` runners (VS, MSVC, Python preinstalled). Workflow
  `build.yml` in `fatal-frame-xbox-recomp` runs on push to `main` or via
  `gh workflow run build -R Kapro777/fatal-frame-xbox-recomp --ref main`.
* **Auth**: `gh` CLI is logged in as `Kapro777` (token scopes include `repo`).
  The workflow fetches the XBE from the private store using secret
  `PORT_BUILD_PAT` (a PAT stored as a repo secret in `fatal-frame-xbox-recomp`).
* **Local game files**: disc extracted to `C:\Users\USER\fatal\game\`
  (`default.xbe` + `Media/`, 1,856 files). `C:\Users\USER\fatal\run\` holds the
  downloaded exe and a junction `run\game -> game` so the exe finds its data.
  Run with `C:\Users\USER\fatal\run-latest.ps1` (downloads the newest private
  build, runs it, writes `err.txt`/`out.txt`).

## 4. Current state (this handoff)

**Working**:
* Full pipeline in CI: `xbe_parser` → `disasm` → `func_id` → `abi_analysis` →
  `recomp` → CMake configure → MSVC Release build → exe published to the
  private store. ~2.5 min pipeline, ~14 min compile (cached).
* The recompiled game **boots through its real entry point** (`0x000DAB5D`):
  CRT startup, 103/103 kernel imports resolved, heap, save subsystem
  (TDATA/UDATA creation), XeLoadSection-by-name, XAPI layer — ~196 real
  kernel calls before it exits.

**The blocker being fixed**: the game's own D3D8 library rejects device
creation (`sub_000E8070` → `sub_000EE900` returns a negative HRESULT), the
game logs its XDK-sample-framework error path (`"OXBApp: Unable to create
Direct3D!"` string at `0x001563F4`), the main thread returns, the title exits
via `HalReturnToFirmware(2)`. Root cause chain, fully diagnosed:

1. `sub_000EE900` (D3D8 device create) allocates its 96-byte device context
   via `MmAllocateContiguousMemoryEx(0x60, 0, 0x7FFFFFF, 0, 4)` — the bridge
   honours this correctly (contiguous window, `0x80001000`).
2. Its success path is a **tail-jump to `0x000EE943`** — an entry point *in
   the middle of its own body*. The static detector never emitted a function
   starting at `0xEE943`, so the recomp emitted a call to a non-existent
   symbol, which `recomp_stubs_unresolved.c` absorbs as an **empty stub**
   (`g_esp += 4; return`) — the entire rest of device creation never runs.
3. The call returns to the caller with `eax` still holding the alloc pointer
   `0x80001000`, which a negative HRESULT check reads as failure.
4. **FIXED** (2026-11-15): 808 seed functions (converged via the local loop:
   called-vs-defined diff over the generated C) + the recomp flag
   `--entry-stubs`. The device create now runs through: 512 KB alloc
   (`sub_000ED7E0`), `MmClaimGpuInstanceMemory`, the GPU interrupt setup
   (KeInitializeDpc / KeConnectInterrupt / HalRegisterShutdownNotification)
   and HalReadWritePCISpace.

**Current blocker (fully diagnosed, see the engine repo's bring-up log)**: the
XDK D3D8 lib hangs in its pushbuffer DMA wait at `0x000EECD6` (inside
`sub_000EE943`): it polls `DMA_GET == DMA_PUT` where GET comes from
`dev->channel_ptr` (`[0x000F91FC]`) — and that pointer is **garbage
(0xE8500674)** because the NV2A register aperture is plain zeroed RAM with no
register semantics (the toolkit's documented stance: *"bridge the D3D8 entry
point that owns the loop, not to start emulating NV2A"*). Runtime probes
(`RECOMP_PEEK`, `RECOMP_POKE`) confirm the values; forcing PUT=0 passes the
wait and the game runs further, then faults on the inconsistent probe state.
Next options, in order: (1) bridge the D3D entry points to the toolkit's
`xbox_d3d8` host layer via `--manual-functions` overrides; (2) install the
NV2A register model (`nv2a_hook_init` + protect the aperture); (3) xemu as
ground truth.

**Not yet brought up** (expected next blockers): menus after D3D succeeds,
FMV playback (`Media/movie/*.wmv`, WMVDEC — runtime plays `.wmv` via Media
Foundation when the game opens the file), audio (DirectSound/AC97 codec-ready
bit is handled by the runtime), saves to memory unit.

## 5. The bring-up loop

### 5a. Fast local seed convergence (no build, ~4 min/round)

```powershell
# from C:\Users\USER\fatal
powershell -ExecutionPolicy Bypass -File bringup-iterate.ps1 -Rounds 4
```

(Equivalently: run `tools.disasm --seed-functions` + `func_id` +
`abi_analysis` + `recomp --entry-stubs` by hand, then diff called-vs-defined
`sub_XXXXXXXX` symbols in the generated `recomp_00*.c`; add the missing
addresses to the seed JSON. The script automates exactly that.)

Stop when the unresolved count converges. Then **one** CI build:
```bash
cd fatal-frame-native-engine && git add -A && git commit -m "seeds round N" && git push
gh workflow run build -R Kapro777/fatal-frame-xbox-recomp --ref main
```

### 5b. Full CI + run + diagnose loop (~18 min/iteration)

1. Wait for the run: `gh run list -R Kapro777/fatal-frame-xbox-recomp`.
2. `C:\Users\USER\fatal\run-latest.ps1` → download + run.
3. Read `C:\Users\USER\fatal\run\err.txt` in this order:
   * `[ICALL] Failed to resolve VA 0x...` — indirect targets the dispatch
     table lacks. Fix with `tools.seed_from_log` (see below).
   * `[TRACE] ->/<- sub_XXXX` — entry/exit of the traced boot chain
     (`config/trace_functions.json`) with register snapshots; find where the
     chain stops.
   * `[KERNEL] #N: ordinal X (slot M) esp=... ret=0x...` — the kernel call
     trace with **guest return addresses**. Map ordinals to names with
     `xboxrecomp/game_files/../xbe_report.txt` (the `#NNN Name` table).
   * `[CONTIG] ...` / `[KERNEL] → returned 0x...` — allocation and call
     results.
   * `[UNIMPL]`, `[CRASH]`, `HalReturnToFirmware` — the exit path.
4. Seed from the log (measured indirect targets + thread routines):
   ```bash
   cd C:\Users\USER\fatal\xboxrecomp
   py -3 -m tools.seed_from_log C:\Users\USER\fatal\run\err.txt game_files/default.xbe \
       --functions tools/disasm/output/functions.json \
       --seeds ../fatal-frame-native-engine/config/seed_functions.json
   ```
   Then re-run the local 5a loop (the CI pipeline picks up the seed file via
   `--seed-functions`), or push + CI.
5. For direct-call/mid-function gaps, use the 5a analysis instead (the stub
   absorber is silent by design) and `--entry-stubs`.
6. Change the **cache key prefix** (`fatal-build-vN-`) in `build.yml`
   whenever the pipeline arguments or seed set change — otherwise the
   fingerprint check (`engine/src/recomp/gen/.source-sha`) skips the
   pipeline and reuses stale generated code.

### 5c. Useful env knobs when running (already set in `run-latest.ps1`)

`RECOMP_KERNEL_LOG_BUDGET=2000` (default 200 hides calls), `RECOMP_WATCHDOG_SECS=30`,
`RECOMP_UNIMPL_TRAP=1` (abort at first untranslated instruction),
`RECOMP_NV2A_TRACE=1` (framebuffer probe).

## 6. Per-title facts (metadata only)

* XBE: title `0x54430004`, version 30724, XDK 1.0.4721, base `0x00010000`,
  entry `0x000DAB5D` (labeled `xbe_entry_point`), image `0x00395DC0`.
* Sections (11): `.text`(0x11000, 874 KB), D3D, D3DX, XGRPH, DSOUND, WMVDEC,
  XPP, `.rdata`(0x155FA0 = kernel thunk table), `.data`, DOLBY, `$$XTIMAGE`.
* Kernel imports: **103** (all bridged; `validate_ordinals.py` passes).
* Disk: `Media/3ddata/*.pkx`, `Media/**/*.xpr`, `Media/movie/scene*.wmv`,
  proprietary formats parsed by the game's own code.
* Boot flow mapped statically: `xbe_entry_point` → creates main thread
  (`sub_000DAAEE`, via `PsCreateSystemThreadEx` routine `0x000D8F30`,
  ctx1 `0x000DAAEE`) → wait. Main thread: save/storage init
  (`sub_000DD86B`…, `sub_000DD717` with the TLS-sanity check
  `[TLS+8] == 0xfffe0000` → `HalReturnToFirmware(4)`), then
  `sub_000D7FA0` → `sub_00011DD0` (platform/device init:
  `sub_000EBFE0` → `sub_000E8070` → `sub_000EE900` D3D device create) →
  `sub_00011F00` (the game's main loop object) → title exit
  (`sub_000D97B1` → `HalReturnToFirmware(2)`).
* The game uses the XDK sample framework (strings `OXBApp: Unable to create
  Direct3D!`, `XBUtil_FindMediaFile`), single-player, no network.

## 7. Reference docs to read (toolkit clone)

* `xboxrecomp/docs/GETTING_STARTED.md` — the canonical pipeline + build steps.
* `xboxrecomp/docs/technical/candidate-games.md`, `technical/lessons-learned.md`,
  `technical/indirect-calls.md` — per-title pitfalls.
* `xboxrecomp/src/nv2a/README.md` — NV2A model + MMIO hook (aperture is plain
  zeroed RAM today; `nv2a_hook_init` is NOT installed by default).
* `xboxrecomp/tools/seed_from_log/` — runtime-measured seeding.
* `engine repo docs/targets.md` — Windows→Android plan (SDK2/GLES path, NDK,
  Gradle module, touch overlay, user-supplied game folder).

## 8. Rules for whoever continues

1. Nothing derived from the game ever goes public: no XBE, no generated C,
   no `.exe`, no release assets in public repos. Public repos carry engine,
   CI, docs, per-title metadata.
2. Every pipeline-argument change needs a cache-key bump (`fatal-build-vN-`).
3. Keep the bring-up log (`engine/docs/bringup-log.md`) current — one entry
   per finding.
4. The engine repo has no CI; the build repo triggers CI on push. Prefer
   batching engine changes, then one `workflow run`.
5. Android is the destination: do not design anything Windows-only in the
   engine (`src/main.c` must stay portable; the toolkit's POSIX/SDL2+GL path
   is the Android backend).