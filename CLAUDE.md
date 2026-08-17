# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository shape

`nuttxspace` is a thin workspace wrapper, not the NuttX source tree itself. Only `.vscode/`, `.github/`, and `.gitignore` are tracked by this repo's git history. The actual code lives in two directories that are **gitignored** (`/nuttx/`, `/apps/` in `.gitignore`) and managed as their own independent checkouts, pinned to tag `nuttx-12.5.1`:

- `nuttx/` — the NuttX RTOS kernel (Apache NuttX, upstream `apache/nuttx`)
- `apps/` — companion applications/libraries (upstream `apache/nuttx-apps`), referenced by the kernel via `CONFIG_APPS_DIR="../apps"`

`build/`, `Documentation/`, and `staging/` are also gitignored build/output artifacts. Don't assume changes inside `nuttx/` or `apps/` show up in `git log`/`git status` for this repo — check status from inside those directories if needed.

The target board for this workspace is the **STM32F3Discovery** (STM32F303VCT6, Cortex-M4): `nuttx/.config` has `CONFIG_ARCH_CHIP="stm32"` / `CONFIG_ARCH_BOARD="stm32f3discovery"`. Board configs live under `nuttx/boards/arm/stm32/stm32f3discovery/configs/` (`nsh`, `usbnsh`, `alarm_rtc`, `lsm303dlhc`, `nsh_usart1_vcp`, `nsh_usart1_lsm303dlhc`, `usbnshfinal`).

## Commands

The ARM toolchain (GNU 14.2.rel1) is expected at `/opt/arm-gnu-toolchain-14.2.rel1-x86_64-arm-none-eabi/bin/`.

**Configure** (selects board + defconfig, run once or after `distclean`):
```
nuttx/tools/configure.sh -l stm32f3discovery:usbnsh
```
(swap `usbnsh` for another config name under `configs/` above; `-l` = symlink Kconfig tools for Linux host)

**Reconfigure interactively:**
```
make menuconfig -C nuttx
```

**Build (local/VS Code — classic Make, used by the "Build all" task):**
```
make CPP=/opt/arm-gnu-toolchain-14.2.rel1-x86_64-arm-none-eabi/bin/arm-none-eabi-cpp \
     CC=/opt/arm-gnu-toolchain-14.2.rel1-x86_64-arm-none-eabi/bin/arm-none-eabi-gcc \
     CXX=/opt/arm-gnu-toolchain-14.2.rel1-x86_64-arm-none-eabi/bin/arm-none-eabi-g++ \
     LD=/opt/arm-gnu-toolchain-14.2.rel1-x86_64-arm-none-eabi/bin/arm-none-eabi-ld \
     -j4 -C nuttx
cd nuttx && arm-none-eabi-objcopy nuttx nuttx.elf   # produce the .elf used by debuggers
```

**Build (CI — CMake + Ninja, see `.github/workflows/ci-build.yml`):**
```
cmake -S nuttx -B build -DBOARD_CONFIG=stm32f3discovery:nsh -GNinja
cmake --build build -j$(nproc)
```

**Clean:** `make clean -C nuttx` · **Full reset:** `make distclean -C nuttx`

**Flash (ST-Link):**
```
st-flash --connect-under-reset write nuttx/nuttx.bin 0x8000000
```

**Debug:** use the VS Code launch configs in `.vscode/launch.json` (`STlink attach/launch`, `JLink attach/launch`, `ST-util attach/launch`, `Renode launch` for the simulator) — all target `nuttx/nuttx.elf`, device `STM32F303VCT6`. `STlink launch`/`JLink launch`/`ST-util launch` have `preLaunchTask: "Build all"`, so they rebuild and reflash automatically.

**Style check** (what upstream CI runs against patches, see `.github/workflows/check.yml`): `nuttx/tools/checkpatch.sh` and `nuttx/tools/nxstyle.c` (compile and run against changed files) enforce the coding standard below.

## Testing

NuttX has no host-side unit-test suite for kernel/driver code. Verification means building a config, flashing or running it, and exercising the target through the NuttShell (NSH):

- Console is USART1, routed over the ST-LINK virtual COM port: `screen /dev/ttyACM0`.
- The ST-LINK VCOM and VS Code's Serial Monitor extension cannot both hold the port open at once — only one can be connected at a time.
- `CONFIG_SYSLOG_CONSOLE=y` sends `syslog()`/boot messages to the same serial console.
- Renode (`Run Renode` / `Renode launch` VS Code tasks) can simulate the board without hardware.

## Architecture

**Kernel layering** (inside `nuttx/`): `arch/` (CPU/MCU-specific code, e.g. `arch/arm/src/stm32/`), `boards/` (board bring-up + the `configs/*/defconfig` sets, e.g. `boards/arm/stm32/stm32f3discovery/`), `drivers/` (device-class drivers, OS/chip-agnostic), plus `fs/`, `sched/`, `mm/`, `net/`, `binfmt/`, `syscall/`, etc. User-space programs and libraries are a separate tree in `apps/` (selected into the build via `CONFIG_*` options that append to `CONFIGURED_APPS`, see `apps/README.md`).

**Upper-half / lower-half driver pattern**: most device drivers are split in two and bound together through an ops struct:
- *Upper half* — OS-facing, chip-agnostic glue that registers a char device under `/dev` and implements the VFS file operations, e.g. `drivers/timers/rtc.c`.
- *Lower half* — chip-specific implementation of that device class's ops struct, e.g. `arch/arm/src/stm32/stm32_rtc_lowerhalf.c` (wrapping `arch/arm/src/stm32/stm32_rtc.c`), registered into the upper half from board bring-up code in `boards/arm/stm32/stm32f3discovery/src/`.

Understanding or changing a subsystem generally requires reading both halves plus the board bring-up call site that wires them together.

**Return-value convention**: functions return `OK` (0) on success or a negated `errno` on failure — never a positive value. Error checks like `if (ret < 0)` are complete on their own; there is no separate positive-error case to handle.

## Coding standard

This is upstream Apache NuttX code, and patches that don't conform are rejected by `nxstyle`/`checkpatch` in CI. Key rules (full standard: https://nuttx.apache.org/docs/latest/contributing/coding_style.html):

- C89-compatible in common code; C99/C11 only in architecture-specific code.
- 2-space indentation, **no tabs**, 78-column max line width, Unix line endings.
- C-style `/* ... */` comments only, no `//`. Every function has a standard header block comment (Name/Description/Input Parameters/Returned Value).
- Naming: globals `g_*`, structs `*_s`, unions `*_u`, enums `*_e`, typedefs `*_t`, functions `module_verb()` lowercase, pointer `*` binds to the variable name (`char *buffer`).
- Braces always used for control structures, opening brace on its own new line.
- SPDX + Apache-2.0 license header block at the top of every source file.

Also worth knowing (`nuttx/INVIOLABLES.md`, upstream project philosophy): strict POSIX compliance is never sacrificed for expediency, the modular architecture (well-defined internal interfaces) must be preserved, and sometimes code duplication across modules is preferred over introducing coupling.

## Working with Claude Code on this project

- **Verify current state before answering**: don't rely on an earlier turn's cached view of a file — re-read the file (or check `git status`/`git diff`) before making claims about its current contents, especially if the user or a linter may have changed it since you last looked.
- **Validate before presenting**: before showing a proposed change or answer, check that it matches the codebase's actual conventions and that every claim is grounded in what you just read, not assumed.
- **Delegate implementation to subagents**: when implementing changes (not just answering questions), use the Agent tool. Give the subagent precise context (relevant files, the exact requirement) and explicit success criteria for tests/verification, rather than a vague instruction. For writing new functions in a source file from a template, use the three-stage pipeline defined in `.claude/agents/`: `function-planner` (Opus, plans the functions from the template — no code) → `function-implementer` (Sonnet, writes the code and builds to confirm it compiles) → `function-reviewer` (Opus, checks for memory leaks, logic errors, and optimizations). Run them in that order, passing each stage's output to the next.
