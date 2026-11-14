# fatal-frame-xbox-recomp — build & CI

This repository is the **cloud build farm** for the Fatal Frame (Xbox) static
recompilation. It contains **no game code and no game assets** — only the
automation that runs the pipeline and compiles the result in GitHub Actions.

## Why this exists

Static recompilation is CPU-hungry: the Python pipeline (parse → disassemble →
identify → ABI analysis → lift to C) takes several minutes, and the MSVC build
of the generated C takes much longer. Both run here, on GitHub's
`windows-latest` runners, so nothing heavy runs on a low-power machine.

The ported game's source is split across three repositories, deliberately:

| Repo | Visibility | Contents |
|---|---|---|
| `Kapro777/fatal-frame-native-engine` | **public** | host engine, runtime bridge, build glue, docs. Ships no game code. |
| `sp00nznet/xboxrecomp` | public (upstream) | the recompilation toolkit itself (MIT) |
| `Kapro777/fatal-frame-xbox-recomp` (here) | public | CI automation, pipeline notes |
| `Kapro777/fatal-frame-xbe` | **private** | the owner's `default.xbe` — input |
| `Kapro777/fatal-frame-builds` | **private** | built `.exe` — output (derived game code, never distributed) |

Built executables are derivatives of the owner's own legally acquired copy and
are kept in a **private** artifact store. Nothing derived from the game is ever
published from this repository.

## ⚖️ Legal

* This repository provides automation and documentation only. You must own a
  legitimate copy of Fatal Frame for the original Xbox.
* No copyrighted game code or assets are committed here. The XBE comes from a
  private store and is never published; the generated C code lives only in CI
  runners and the private artifact store.
* Fatal Frame is a trademark of its respective owners. This project is not
  affiliated with or endorsed by Tecmo, Microsoft, or any rights holder.

## How a build runs

` .github/workflows/build.yml` — triggered by push to `main` or manually
(`workflow_dispatch`):

1. check out the toolkit (pinned `v0.17.0`) and the engine repo
2. fetch `default.xbe` from the private store (PAT secret `PORT_BUILD_PAT`)
3. install `capstone`
4. `tools.xbe_parser` → `tools.disasm` → `tools.func_id` → `tools.abi_analysis`
   → `tools.recomp --all --split 250`
5. `cmake` configure + `cmake --build Release` (MSVC, D3D11 backend)
6. zip the `.exe` and push it to the private `fatal-frame-builds` repo as a
   release tagged `run-<run_id>`

Public repo → unlimited free Actions minutes. Artifacts stay private.

## Target title (metadata only)

Fatal Frame (Xbox, 2002): title `0x54430004`, entry point `0x000DAB5D`,
`.text` `0x000D5B80` (~874 KB), 11 sections, 103 kernel imports, XDK 4721.
English retail build. Full profile: `docs/title-profile.md` in the engine repo.