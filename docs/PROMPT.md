# Prompt — bring-up of the Lethal Shot port (for the next engineer/AI)

You are taking over the bring-up of a **native port of the owner's own
game disc** (a 2002-era console title), built by static recompilation.
Everything you need is in GitHub repositories — you do not need, and
should not ask for, a development machine.

## 0. The rules of the project

* **Legal model**: the owner supplies their own copy of the disc. No game
  code or asset ever lands in a public repository. The proprietary bits
  (the title's executable, its derived binaries, the generated C) live
  only in the private repositories and the private artifact store.
* **Public repos**: `lethal-shot-engine` (the engine + docs + bring-up
  tools), `lethal-shot-build-farm` (this repo — CI glue + handoff docs).
* The pipeline additionally reads from a private input store and writes
  to a private artifact store; both are referenced only by repository
  secret inside the workflow, and neither is named in any repository.
* **All compilation runs in GitHub Actions** (windows-latest). Never
  install a local toolchain; never put build outputs anywhere but the
  private artifact store.
* Two agents have already converged the boot to device creation; your job
  is to keep going. The full reasoning is in
  `lethal-shot-engine/docs/bringup-log.md` — read it before changing
  anything, it is the difference between hours and days.

## 1. How to see the state of the world

```
gh workflow run build -R <owner>/lethal-shot-build-farm --ref main   # build + smoke run
gh run list      -R <owner>/lethal-shot-build-farm --limit 1
gh run view <id> -R <owner>/lethal-shot-build-farm --log-failed
```

Each successful run publishes:
* a **private release** `run-<run-id>` (published by CI to the private
  artifact store) containing `lethalshot_native.exe` (the current build
  the owner can test);
* an **artifact `smoke-run-<run-id>`** — the stdout/stderr of a bounded
  on-runner execution of that exe (the game data directory is already in
  the workspace). This is your feedback channel: the job log ends with a
  `SMOKE_SUMMARY kernel_calls=N crashes=M bytes=...` line.

Read the smoke run first, every time: `kernel_calls` is the boot progress
counter (it was 201 at the last handoff), `crashes` counts `[CRASH]`
lines, and the artifact holds the full log including the crash handler's
register dump and guest stack.

## 2. Where the boot stands (state at handoff)

**Working** (each of these was a diagnosed root cause, see the log):
* The whole entry chain runs: entry `0x000DAB5D` → CRT → ~201 kernel
  imports bridged (saves of TDATA/UDATA, `MmAllocateContiguousMemoryEx`,
  `KeConnectInterrupt`, the SMC/PCI probes).
* D3D device create: the 512 KB push-buffer ring is allocated
  (`sub_000ED7E0`), the GPU ISR is connected, the NV2A register model is
  installed (`engine_nv2a_install` — the aperture answers hardware values,
  which is what the library's PFB flush spin at `0x000EDAB4` needs, and
  the runtime's own ack thread clears that bit).
* The push-buffer wait at `0x000EECD6` now exits: the library keeps its
  PFIFO user-DMA channel at guest VA `0x00800000` (PUT +0x40, GET +0x44)
  and kicks a doorbell at VA `0x80000000`; `engine_pfifo_ack_thread`
  reports submitted commands consumed (GET follows PUT), the same
  handshake the runtime models for the aperture's USER window.
* The VRAM window fault was routed to the toolkit's placeholder handler
  (`nv2a_hook_handle_vram` for `0xF0000000`–`0xFCFFFFFF`) — the
  post-create read at `0xF71182A3`.

**Unverified at handoff time**: the build `run-43760804327` contains the
VRAM fix and the repository renames; its smoke run had not been read yet.
Your first action is to read it (or trigger a fresh run) and see how much
further the boot gets.

**The endpoint to expect**: after device create, the library continues
(`0x000EED14` → `sub_000EE3C0` → surface/framebuffer setup → the first
`Present`). New blockers there will look like: another wait on hardware
(vblank/PCRTC counters — the toolkit models some of these; check
`docs/runtime/environment-variables.md` for the RECOMP_* knobs), a
pointer built from a kernel result that comes back zero (the same shape
as the PFIFO channel), or surface allocation through the VRAM window.

## 3. The loop, in detail

1. **Change** either `engine/src/main.c` (engine-level modelling) or
   `engine/config/seed_functions.json` / `trace_functions.json`
   (pipeline inputs — both are fingerprinted into the CI cache key, so
   changing them re-runs the pipeline).
2. **Push** to the engine repo; the build farm's push trigger runs the
   pipeline (docs-only pushes are skipped).
3. **Read the smoke run.** Classify the endpoint:
   * `[CRASH] ... Xbox VA of fault: 0x...` → disassemble around the
     innermost guest return address:
     `py -3 tools/bringup/xbe_tools.py disasm <xbe> <va-32> <va+64>`
(the title executable lives in the private input store -- the CI has it; the owner has it locally)
     function in the generated C:
     `py -3 tools/bringup/analyze_gen.py whom <gen-dir> 0x000EE943`
     (`<gen-dir>` is the `generated-c-<run-id>` artifact, e.g.
     `.../gen`). The answer is usually a wait, a pointer built from a
     zero kernel result, or an unmapped device window.
   * Unresolved-call storm → seed convergence: run the toolkit's
     `tools.seed_from_log` on the smoke-run err.txt plus the XBE, add the
     entries to the seed list. (Locally this is a 4-minute loop; in CI it
     is one dispatch.)
   * A silent gap → extend `trace_functions.json` around the last traced
     address and re-run; the trace shows the values at the boundary.
4. **Fix at the right level.** Prefer, in order: (a) engine modelling in
   `main.c` (documented constants, proven against disassembly/probes);
   (b) probes — the runtime honours `RECOMP_PEEK`, `RECOMP_POKE`,
   `RECOMP_WATCH`, `RECOMP_WATCHDOG_SECS` (documented in the toolkit) —
   you can set these in your own smoke run by editing the workflow's
   `env:` block; (c) seeds for recompiler gaps.
5. **Update `docs/bringup-log.md`** with every root cause you prove. The
   log is the project's memory; the next handoff starts from it.
6. Bump the CI cache key prefix in `build.yml` whenever pipeline
   arguments or the seed/trace sets change (`lethal-build-vN-`).

## 4. The roadmap after the boot converges

1. **Menu on Windows**: push through surface setup, the first Present,
   asset loading (the title parses its own `.pkx`/`.xpr`/`.sgd`), and any
   vblank/save waits. Keep reading the smoke run each round.
2. **Playable on Windows**: the graphics path — the title renders through
   the NV2A push buffer and nothing executes it yet. The toolkit ships
   the pieces (`nv2a_pb_exec`, the D3D11 layer, the PGRAPH register
   handlers) under a reserved name; wiring the title's channel into that
   executor is the largest single task. Then input, then audio
   (the MCPX APU implementation is in the toolkit).
3. **Android arm64**: the final destination. The toolkit's POSIX path for
   MMIO trapping is explicitly unimplemented ("deferred to main.c port")
   — on Android it is a SIGSEGV handler instead of VEH; the D3D11 layer
   needs a GLES/SDL2 backend; CMake needs an Android toolchain and APK
   packaging. Start this scaffolding **in parallel with phase 1** — it is
   infrastructure work that does not need the boot to converge, and it is
   what keeps the endgame from being infra + graphics at once.

## 5. Conventions that exist for a reason

* Every hardcoded guest address in `main.c` carries a comment saying how
  it was proven (disassembly offset, probe value, or log line). Keep that
  invariant — it is how the next engineer trusts the code.
* The project is deliberately anonymous: no console or title names in
  code, docs, commit messages, or repository metadata. The public repo
  says "the owner's own game disc" and nothing more specific.
* The exe is named `lethalshot_native`; the CI smoke run and the private
  release are the only ways a build is exercised.