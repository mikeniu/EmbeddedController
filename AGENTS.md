# AGENTS.md — EmbeddedController (EC) Firmware

## Build
- Default board: `hx30` (Framework Laptop 12th/13th Gen Intel). Output: `build/hx30/ec.bin`
- Build any board: `make BOARD=<name>` (see `board/` for ~178 boards)
- ARM toolchain: system `arm-none-eabi-gcc` is auto-detected; override with `CROSS_COMPILE_arm=`
- Framework boards: `hx20` (11th Gen Intel), `hx30` (12th/13th Gen). Both use chip `mchp` (MEC1521H, Cortex-M4)
- Each board selects its chip, core, and baseboard via `board/<name>/build.mk`
- Build artifacts in `build/<board>/`

## Test
- Host (emulator) tests: `make runhosttests` — builds + runs all
- Single host test: `make host-<name>` then `make run-<name>` (e.g. `make host-printf; make run-printf`)
- List host tests: `make print-host-tests`
- Device tests: `make tests BOARD=<name>`
- All tests + fuzz: `make runtests`
- Coverage: `make coverage` (requires lcov/genhtml)
- Fuzz tests: `make runfuzztests`
- Test sources in `test/`, each foo.c + foo.tasklist; `test/build.mk` declares `foo-y=foo.o`

## Code style
- **Linux kernel style**: 8-wide tabs, 80-column limit. `.clang-format` based on kernel config
- Lint: `checkpatch.pl` (Linux kernel), configured in `.checkpatch.conf` and `PRESUBMIT.cfg`
- Presubmit hooks: branch naming, checkpatch, kerneldoc (`ec_commands.h`), signed-off-by

## Architecture
```
board/<name>/build.mk → sets CHIP, BASEBOARD
baseboard/<name>/build.mk → shared board logic
chip/<name>/build.mk → sets CORE, chip drivers
core/<name>/build.mk → CPU core support (cortex-m, host, etc.)
```
- All `build.mk` files accumulate `*-y` (always built), `*-$(CONFIG_FOO)` (conditional), `*-(HAS_TASK_FOO)`
- `include/` has all headers; `common/` has cross-chip code; `driver/` has peripheral drivers
- `power/` — chipset power sequencing
- `builtin/` — freestanding libc replacements (used for firmware builds)

## Key Makefile variables
| Variable | Purpose |
|----------|---------|
| `BOARD=<name>` | Select board |
| `V=1` | Verbose build output |
| `CROSS_COMPILE=<prefix>` | Override toolchain prefix |
| `CROSS_COMPILE_arm=<prefix>` | Override ARM toolchain only |
| `TEST_ASAN=y` / `TEST_MSAN=y` / `TEST_UBSAN=y` | Sanitizer builds |

## CI / presubmit
- No GitHub Actions in this repo; CI runs via Chromium OS infra (`firmware_builder.py` → `make buildall_only` / `make runtests`)
- Presubmit scripts: `util/presubmit_check.sh`, `util/config_option_check.py`, `util/host_command_check.sh`

## Python utilities (installed via `setup.py`)
| Tool | Path | Use |
|------|------|-----|
| `ectool` | `util/ectool.c` | Host ↔ EC communication |
| `ec3po` | `util/` | EC console interpreter |
| `usb_console` | `extra/usb_serial/` | USB console for servo/cr50 |
| `powerlog` | `extra/usb_power/` | Sweetberry power logger |

## Git conventions
- `.gitignore`: `build/`, `private*/`, `local/`, `*.pyc`
- Use `Signed-off-by:` in commits (presubmit enforces)

## Framework-specific notes
- hx30 uses `baseboard/fwk/`, chip `mchp` variant `mec152x_3400`, core `cortex-m`
- Flash layout: RO + RW regions + LFW (loader firmware) packed via `chip/mchp/util/pack_ec.py`
- EC flash via OpenOCD: `make flash BOARD=hx30` (or `flash_ec`, `flash_dfu`)
