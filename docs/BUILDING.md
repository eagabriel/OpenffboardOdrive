# Building Odrive-Wheel from source

For most users the pre-built `.hex` from a
[GitHub release](https://github.com/eagabriel/Odrive-Wheel/releases) plus the
in-browser DFU flasher is enough. This document is for when you want to
modify the firmware, keep a private fork in sync with upstream OpenFFBoard,
or investigate a build issue.

## Repository structure

```
.
├── Odrive-Wheel/               ← Main project (CDC + HID composite + FFB)
│   ├── src/                    ← Local sources (USB, FFB bridge, cmd_table, EQ)
│   ├── inc/                    ← Local headers
│   ├── linker/                 ← Custom linker script (S0/S3-9 app, S10-11 EEPROM)
│   ├── tools/                  ← HTML config tool (Web Serial + WebUSB DFU, PT/EN i18n)
│   └── Makefile                ← Build via arm-none-eabi-gcc
├── ODrive-fw-v0.5.6/           ← ODrive firmware (with minimal patches)
├── OpenFFBoard-master/         ← Submodule → upstream Ultrawipf/OpenFFBoard
└── docs/                       ← Getting Started, tuning guides, screenshots
```

## Clone

`OpenFFBoard-master/OpenFFBoard-master/` is a **git submodule** pointing to
upstream [`Ultrawipf/OpenFFBoard`](https://github.com/Ultrawipf/OpenFFBoard)
(currently locked at **v1.17.0**). Clone with submodules:

```bash
git clone --recurse-submodules https://github.com/eagabriel/Odrive-Wheel.git
```

If you already cloned without `--recurse-submodules`:

```bash
git submodule update --init --recursive
```

## Build

Prerequisites:

- `arm-none-eabi-gcc` (tested with 12.2)
- `make`
- `python3` with `pyyaml`, `jinja2`, `jsonschema` (for autogen)
- `dfu-util` (only for the first flash — later updates use the in-browser flasher)

### Option A — GitHub Actions (zero local setup)

Push to `main` or open a PR. The workflow `.github/workflows/build.yml` runs
autogen + `make` on Ubuntu and uploads `odrive-wheel.bin` as a build artifact
(retention 14 days). Also triggers manually via **Actions → Build Firmware →
Run workflow**.

### Option B — Local build, Windows (`build-local.ps1`)

Wraps autogen + MSYS2 + ARM GCC. Requires MSYS2 at `C:\msys64` (override with
`-MsysBash`). The script auto-detects ARM GCC across common install paths.

```powershell
.\build-local.ps1                     # full clean build
.\build-local.ps1 -NoClean            # incremental (skip make clean)
.\build-local.ps1 -SkipSubmoduleInit  # skip git submodule update
```

### Option C — Manual (Linux / WSL / macOS)

```bash
# 1. Generate ODrive autogen headers (once, regenerate if interface YAML changes)
cd ODrive-fw-v0.5.6/ODrive-fw-v0.5.6/Firmware
mkdir -p autogen
python3 ../tools/odrive/version.py --output autogen/version.c
python3 interface_generator_stub.py --definitions odrive-interface.yaml --template fibre-cpp/interfaces_template.j2     --output autogen/interfaces.hpp
python3 interface_generator_stub.py --definitions odrive-interface.yaml --template fibre-cpp/function_stubs_template.j2 --output autogen/function_stubs.hpp
python3 interface_generator_stub.py --definitions odrive-interface.yaml --generate-endpoints ODrive --template fibre-cpp/endpoints_template.j2 --output autogen/endpoints.hpp
python3 interface_generator_stub.py --definitions odrive-interface.yaml --template fibre-cpp/type_info_template.j2 --output autogen/type_info.hpp

# 2. Build
cd ../../../Odrive-Wheel
make -j$(nproc)
```

Output artifacts: `Odrive-Wheel/build/odrive-wheel.{elf,hex,bin}`.

## Updating OpenFFBoard upstream

Several FFB stack files (HidFFB, EffectsCalculator) were **forked and modified**
locally in `Odrive-Wheel/src/` and `inc/`. The originals live in the
`OpenFFBoard-master/` submodule.

When upstream releases relevant updates, workflow:

```bash
# 1. Pull latest upstream commit into the submodule
git submodule update --remote OpenFFBoard-master/OpenFFBoard-master

# 2. See what changed
cd OpenFFBoard-master/OpenFFBoard-master
git log --oneline HEAD@{1}..HEAD          # new commits
git diff HEAD@{1}..HEAD --stat            # changed files
cd ../..

# 3. Compare our forks against the updated upstream
./Odrive-Wheel/tools/check-openffboard-upstream.sh           # summary
./Odrive-Wheel/tools/check-openffboard-upstream.sh --verbose # with diffs

# 4. For each file marked "DIVERGE" with relevant upstream changes,
#    manually integrate into our fork in Odrive-Wheel/

# 5. Compile + test
cd Odrive-Wheel && make -j4

# 6. Commit
git add OpenFFBoard-master/OpenFFBoard-master Odrive-Wheel/...
git commit -m "Bump OpenFFBoard upstream to <hash> + integrate changes"
```

Forked files carry a header at the top indicating:

- Upstream version that was the fork base (commit hash)
- Description of local modifications
- Exact command to diff against upstream

Example: `Odrive-Wheel/src/HidFFB.cpp` documents the `set_effect` modification
for single-axis fallback.

## Flash from source

Once built, put the board into DFU mode (BOOT0 + RESET) and:

```bash
cd Odrive-Wheel
make flash-dfu
```

Equivalent to:

```bash
dfu-util -d 0483:df11 -a 0 -s 0x08000000:leave -D build/odrive-wheel.bin
```

For subsequent updates after the first flash, use the **DFU Flash** tab in
the [HTML config tool](https://eagabriel.github.io/Odrive-Wheel/) instead —
no `dfu-util` needed, and the tool has a 📡 Fetch latest from GitHub button
that pulls the release binary directly.
