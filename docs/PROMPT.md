# Paste-ready prompt for the other AI (bring-up phase)

Copy everything below the line into the other coding agent (it needs repo
access to `Kapro777` — the token/gh login is already configured on this
machine; if it runs elsewhere, give it a GitHub PAT with `repo` scope).

---

You are taking over the **bring-up of a static recompilation port**: Fatal
Frame (Xbox, 2002, English retail) → native Windows x64 first, then Android
arm64. Product model: a distributable engine; the user supplies their own copy
of the game; nothing derived from the game is ever published.

**Read first** (in this order):
1. `https://github.com/Kapro777/lethal-shot-build-farm/blob/main/docs/HANDOFF.md`
   — full context, architecture, current state, the bring-up loop, per-title
   metadata, and the rules (legal + workflow).
2. `https://github.com/sp00nznet/xboxrecomp/blob/main/docs/GETTING_STARTED.md`
   — the canonical pipeline.

**Where things stand**: the pipeline (parse → disasm → func_id → abi_analysis
→ recomp → MSVC build) works end-to-end in GitHub Actions; the game boots
through its real entry point and runs ~196 real kernel calls; it currently
dies inside its own D3D8 device creation because ~200 mid-function entry
points were emitted as empty stubs. The fix (seeding + `--entry-stubs`) is
converging locally (808 seeds, 202 unresolved targets left) and a CI build is
running.

**Your job, in order**:
1. Check the in-flight build (`gh run list -R Kapro777/lethal-shot-build-farm`),
   download it (`C:\Users\USER\fatal\run-latest.ps1`) and diagnose the new
   `run\err.txt` against the symptom table in HANDOFF.md §5b.
2. If the D3D device create still fails, continue seed convergence locally
   (`bringup-iterate.ps1`, §5a) and rebuild; do not waste CI cycles on
   seed discovery.
3. Once past device creation: walk the boot chain to the menus
   (TRACE list in `lethal-shot-engine/config/trace_functions.json` can
   be extended), fixing ICALL gaps via `tools.seed_from_log` and silent
   direct-call gaps via seeds + `--entry-stubs`.
4. Log every finding in `lethal-shot-engine/docs/bringup-log.md`.
5. Keep the legal boundary: engine/CI/docs public; XBE, generated code and
   built executables private only.

**Constraints**: heavy compiles stay in CI (the dev machine is a Celeron with
no toolchain); never publish game-derived bytes; bump the CI cache key
(`fatal-build-vN-`) whenever the pipeline arguments or seed set change.

---