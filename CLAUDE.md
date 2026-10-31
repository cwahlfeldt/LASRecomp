# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

LAS Recomp is a static recompilation of *The Legend of Zelda: Link's Awakening* (Switch, 2019) into a native macOS (Metal), Windows and Linux (Vulkan) app. The game's AArch64 executables are translated to C, compiled, and run against a host reimplementation of the Switch OS and GPU API. The app ships with no game code: each player's machine generates it from their own game files.

## Hard rules

- **Never commit or attach Nintendo files**: `extracted/`, `generated/`, `rando/` seed folders, Switch fonts, or any game asset, "just one file for testing" included. Share a randomizer **seed ID** instead of a seed folder. Most of these are gitignored; don't override that.
- **No personal paths or information** in code, comments or commit messages. The Makefile strips the checkout path from what ships (`-ffile-prefix-map`).
- **Branches:** upstream is `peiikai/LASRecomp`. Its `main` holds the last release; development and PRs target the highest-numbered `beta_<n>` branch (after 1.0, the latest `vX.Y.Zb`). In this clone `origin` is a fork, and no upstream remote is configured.

## Commands

No game files are in the repo. Without `extracted/` you can build the app and run the regression tests, but not the game.

```sh
# Build (macOS needs: brew install sdl3 ffmpeg). Windows: MSYS2 UCRT64 shell, docs/BUILD-windows.md. Linux: docs/linux.md.
make -j"$(sysctl -n hw.ncpu)"   # build/lasrecomp, plus build/lasgame.* when generated/ exists
make runtime-objs               # quick compile check: runtime objects only, no link

# Game code, from your own game files in extracted/. `make gen` needs tools/venv on every platform
# (the README's macOS steps omit it; without it the recipe runs an empty interpreter path and fails).
python3.12 -m venv tools/venv && tools/venv/bin/pip install lz4 capstone unicorn
make gen && make -j"$(sysctl -n hw.ncpu)"

# Run
./build/lasrecomp extracted          # start screen
./build/lasrecomp extracted --play   # straight into the game

# Randomizer from a source checkout
python3.12 -m venv third_party/LAS-Randomizer/venv
third_party/LAS-Randomizer/venv/bin/pip install -r third_party/LAS-Randomizer/requirements.txt
```

Packaging prerequisites are in the README: `make app` / `make dmg`, `make dist`, `make appimage` / `make appimages`. `make smoketest` is the x86-64 AppImage release check.

### Tests

There is no `make test`, linter or formatter. Each test is a standalone program run from the repo root. `.github/workflows/ci.yml` is the source of truth for what runs on each platform and for the exact compile lines of the C tests. New tests that need no game files go in as steps there, not as new workflows.

```sh
python3 tools/test_settings.py     # a single Python test (tools/test_*.py): compiles production code with stubs
tools/test_translate_ops.sh        # shader translator instruction semantics

# A single C test, with the compile line from ci.yml (mkdir -p build first on a clean tree)
clang -O2 -pthread tools/test_condvar.c -o build/test-condvar && build/test-condvar

# Heavier, or with extra needs
REF=HEAD tools/test_translate.sh 4000 8 glsl    # translator fuzz; needs shaderc + spirv-tools (REF=HEAD: GLSL half only)
tools/test_spirv_val.sh                         # needs glslc + spirv-val
tools/venv/bin/python tools/test_simd.py --encodings 3000 --cases 4 --jobs 8   # FP/SIMD lowering vs Unicorn (CI: x86-64 Linux)
tools/venv/bin/python tools/test_lift.py extracted/exefs/main 0x7000000000     # lifted C vs Unicorn; needs game files
```

On macOS, `test_vulkan_validation.py` and `test_vulkan_overlay.py` fail for lack of Vulkan headers (CI runs them on Linux and Windows). `test_vulkan_overlap.py` is not in CI and aborts on an assertion on macOS. Neither is a sign of a regression there.

### Testing in the game (needs `extracted/`)

- `tools/shot.sh <frame> [max seconds]` runs the game with the caller's `LARECOMP_*` env and writes `build/window.png` and `build/run_shot.log`. `SHOT_ID=<name>` allows parallel runs (give each its own `LARECOMP_SAVE_DIR`). `tools/frames.sh` captures a series of frames.
- About 400 `LARECOMP_*` hooks are documented where they're defined; grep `getenv("LARECOMP_`. The common ones: `LARECOMP_INPUT` (scripted input, `runtime/hle/input.c`), `LARECOMP_WARP_AT` / `LARECOMP_WARP_ON_LOAD` (`runtime/hle/warp.c`), `LARECOMP_RANDO_SEED`, `LARECOMP_REBOOT_AT`, `LARECOMP_SAVE_DIR`, `LARECOMP_VK_VALIDATION_STOP=1`.
- To script into gameplay from a save, copy the recipe in `tools/linux/smoketest.sh`: L+R at the title, A through the file menu, then `LARECOMP_WARP_ON_LOAD`.
- Randomizer: `tools/rando/seed_sweep.py` bulk-checks that seeds are beatable. `tools/doorjump_test.sh <seed>` walks every mismatched doorway of a room-shuffle seed.
- Guest frames in crash logs are `f_<addr>`; symbolize with `cd tools && ./venv/bin/python sym.py <addr>...`.

Watch out for:

- **Real player data.** The data root is `LARECOMP_DATA_ROOT`, else the working directory if it has `extracted/` (dev mode), else `data_root.txt`, else the platform default (`runtime/paths.h`). So from a checkout without `extracted/`, the binary uses the real data folder, and the start screen can delete game files. Set `LARECOMP_DATA_ROOT=<scratch>` for such runs.
- **`tools/game_slot.sh`**, sourced by the run scripts, sets `LARECOMP_NO_LAUNCHER=1` and a scratch `LARECOMP_SETTINGS`. It also blocks until 10 GB of memory is free (`LARECOMP_MIN_FREE_GB`), which can look like a hang.
- **`tools/run.sh` and `tools/fresh_save.sh`** copy from `save_backup_20261007/`, which is a local folder not in git. Supply your own save (`run.sh` has `REALSAVE=1`).

## Architecture

### Recompiler to game library

`tools/gen.py` (with `lifter.py`, `analyze.py`, `passes.py`, `nso.py`, `simd_c.py`, `patches.py`) turns `exefs/main` and `exefs/sdk` into `generated/`. Each guest function becomes `void f_<addr>(Cpu *c)`:

- Registers live in `Cpu` (`runtime/cpu.h`).
- **Guest addresses are host addresses**, so loads and stores are raw pointer accesses. A constructor in `runtime/main.c` claims the fixed window `0x70_0000_0000..0x73_2000_0000` before anything else can, and re-execs on a collision. The layout is in `runtime/rt.h`.
- A direct `bl` inside a module is a plain C call. Indirect branches and calls leaving the module go through `rt_dispatch` (`runtime/dispatch.c`): a per-thread cache, then HLE stubs, then the function tables.
- FP/SIMD is inline AArch64 asm on arm64 hosts and portable C (`simd.h`, `simd_fp.h`) elsewhere (`--simd`), so generated code is per-arch.

The result is a separate library, `lasgame.{dylib,so,dll}`, loaded by `runtime/gamelib.c`. It is built by `make` in a checkout, or on the player's machine by `runtime/gamebuild.c` using the frozen generator (`make gen-helper`) and bundled `zig cc` (`tools/fetch_zig.sh`). Its ID hashes the generator inputs and flags plus the game's build ID, and a library with another ID is rebuilt, never loaded.

What a change costs (the IDs are content hashes, so **even a comment counts**):

| Changed | Consequence |
|---|---|
| Anything in `GAMELIB_SRCS` in the Makefile: generator, `patches.py`, `patches_v*.h`, `patch_variants.json`, `hle_symbols.txt`, `cpu.h`, `simd*.h`, `gamelib.h`, ... | New game library ID: run `make gen && make` locally, and every player rebuilds game code on update |
| `GAMELIB_BASE_SRCS` (those headers, `tools/fetch_zig.sh`) or `GENFLAGS` | Every object recompiles, not only the chunks whose C changed |
| `runtime/nvn/maxwell/translate.c`, `shader.h` | New `SHADER_TRANSLATOR_ID`: shader caches rebuild |
| `runtime/hle/hooks.c`, `runtime_v*.h`, other runtime code | App only |

### Changing what the game does

In order of preference:

1. **`HOOKS` in `tools/patches.py`** plus a case in `runtime/hle/hooks.c`. The site calls `rt_hook(c, site)`, which returns 0 (run the instruction), 1 (the hook did it all) or 2+ (jump to a target). Changing a hook's behaviour only rebuilds the app; only a new site needs regeneration. `patches.py` says new game-code changes go here unless they can't.
2. **`PATCHES` in `tools/patches.py`**: a per-instruction C replacement. Every site asserts the original instruction word, so a wrong address fails loudly. Keep that check in new patches.
3. **`HLE_FUNC("symbol", fn)`** (`runtime/hle.h`): a host implementation of a guest import, bound by name during relocation in `runtime/loader.c`. This cannot intercept a direct `bl` within a module. `runtime/hle/hle_symbols.txt` lists the few SDK functions (libc memory and string) compiled to direct `hle_<name>` calls at generation time.

**Addresses** live in `runtime/game_addresses/`, one table per game version, chosen by the main program's build ID (1.0.1 and 1.1.0 share one; unknown IDs are refused). Runtime code reads `GA(NAME)`; `patches.py` reads `A.NAME` through `tools/game_addresses.py`. `patches_v*.h` feed the generator and `runtime_v*.h` don't. To add a name, edit `*_v1_0_0.h` and regenerate the others with `tools/version_map.py`, which needs game files of both versions. Then run `python3 tools/test_game_addresses.py`.

**Per-boot state.** Switching seeds or returning to the start screen reboots the game inside the process (`runtime/main.c`: stop guest threads, reset, wipe guest memory, boot again; window, GPU device and settings survive). Any host state holding guest pointers, kernel handles or seed data must be cleared with `RT_ON_RESET(fn)` (`runtime/hle.h`).

### Host side

- **OS:** `kernel.c` (threads, sync, memory, `rt_svc`), `ipc.c` and `services/` (HIPC/CMIF: fs, hid, audio, am, acc, ...), `loader.c`. Platform code is in `platform/{macos,windows,linux}.c`.
- **Graphics:** the game's NVN calls are bound by name (`hle/nvn_boot.c`, the `NVN()` macro in `nvn/nvn.h`) to `nvn/objects.c` and `nvn/cmdbuf.c`. Commands are recorded as `CmdRec`s in the game's own command memory. On submit, the render thread (`nvn/rqueue.c`) runs them through the backend.
- **Backends:** Metal (`nvn/renderer.m`, `metal.m`) or Vulkan (`nvn/vulkan/`, whose `exec.c` is a port of `renderer.m` with the same structure). Only one is compiled per platform, so a macOS build doesn't compile `vulkan/` at all. A change to shared rendering usually needs a counterpart in the other backend; say which one you tested.
- **Frame rate:** the game steps its world once per frame, so its logic stays at 60 fps; running it faster speeds up the game. Above 60, `nvn/interp.c` runs each frame's command lists again with blended uniforms (Video tab "Frame Interpolation", `frame_interp`, `LARECOMP_INTERP`; Metal only, off by default). Judge it by the `[interp] screen:` log lines.
- **Shaders:** `nvn/maxwell/translate.c` turns Maxwell binaries into MSL or GLSL. `nvn/shader_prep.c` prebuilds variants from `shader_variants.txt`, which holds hashes and pipeline state only, never shader code.
- **UI:** `ui/*.mm` is the ImGui overlay and start screen (`launcher.mm`). Off macOS these files compile as plain C++ (`-x c++`), so Objective-C belongs only in `__APPLE__` blocks. `LARECOMP_VERSION` in `launcher.mm` must equal the release tag.

### Randomizer

`third_party/LAS-Randomizer` is vendored, with local changes marked "LAS Recomp" in comments. `tools/rando/` extends it, driven headlessly by `las_headless.py`. `runtime/ui/rando_tab.mm` spawns the frozen `rando-gen` helper, or the venv's Python from a checkout, to write `rando/<seed>/`: replacement `romfs/`, `exefs/1.0.0.pchtxt`, settings, spoiler log and the seed's own saves. `runtime/rando.c` then applies it: RomFS overrides and data patches directly, code patches indirectly.

Recompiled code can't be patched at run time, so `gen.py` compiles every variant of each patchable instruction in, and `rando_word()` picks the seed's variant. A new patched instruction means regenerating `tools/rando/patch_variants.json` with `tools/rando/collect_patches.py`, which also changes the game library ID.

Seed IDs are encoded in both `rando_tab.mm` and `tools/rando/seed_sweep.py`; keep the two in step. New options must keep older seed IDs loading.

## Conventions

- Comments explain *why*: what the game does, what an address is, which seed or issue showed the problem. Match the surrounding comment density and naming.
- Game-facing text follows the game's style: pickup messages go through the game's message box, not custom pop-ups, and settings get short labels.
- A fix isn't verified until it's been seen working in game. Say what you tested: platform, backend, seeds. If it couldn't be run for lack of game files, say that too.
