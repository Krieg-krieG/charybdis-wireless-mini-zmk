# AGENTS.md

ZMK firmware repo for the Charybdis wireless split keyboard (nice!nano, PMW3610 trackball). There is no host-side application code: everything is Kconfig confs, devicetree overlays, keymaps, and build orchestration.

## Building

- `build.yaml` is the single source of truth for what gets built; both CI (`.github/workflows/build.yml`) and the local Docker build parse it. Add/remove firmware variants there.
- Local build (no toolchain needed, requires Docker):
  ```
  docker-compose -f local-build/docker-compose.yml run --rm builder
  ```
  - `ENABLE_USB_LOGGING=true` enables USB-CDC logging (conflicts with the ZMK Studio snippet; the build script auto-skips Studio in that case).
  - `SKIP_WEST_UPDATE=true` builds against current local module checkouts instead of `west update` (only for deliberate local module testing).
  - Debug shell: `docker-compose run --rm --entrypoint bash builder`, then `bash ./local-build/build_setup.sh`.
  - Artifacts land in `firmwares/` at repo root (fully gitignored; distribution happens via CI release bundles).
- CI (`build.yml`) runs on push to `main` and on `v*` tags; tags produce zipped GitHub Releases.
- West workspace: `west init -l config` (manifest is `config/west.yml`, `self.path: config`). `zmk` is pinned to `main` (README's "v0.4.1" is stale — trust west.yml), so upstream ZMK changes can break builds and may need compatibility fixes. `zmk-pmw3610-driver` and `prospector-zmk-module` are 280Zo forks, also pinned to `main`.
- Build.yaml gotchas: every entry needs its own `board:` (the top-level `board:` key is decorative; an entry without a board is silently skipped by CI and falls back to a bogus default locally).

## ZMK config gotchas

- ZMK only looks for `<shield>.conf` / `<shield>.keymap` at the `ZMK_CONFIG` root — it does not descend into subdirectories. Shared confs live in `config/charybdis/` and are copied up to `config/` by both build pipelines; do the same if you add new shared confs.
- Keymaps live in `config/keymaps/<name>.keymap`; builds copy the selected one to `config/<target>.keymap` so ZMK's cmake finds it by shield name. Keymaps include `keymap_features/*.dtsi` by relative path (shared behaviors, combos, macros).
- Shield strings in `build.yaml` may be space-separated stacks (e.g. `"charybdis_dongle prospector_adapter"`): the first token is the local shield used for overlay/target discovery, the full string is passed to `-DSHIELD`. The second token comes from extra modules — `boards/`, `zmk-pmw3610-driver`, `prospector-zmk-module` are all registered via `ZMK_EXTRA_MODULES`.
- Prospector APDS9960 builds deliberately set `CONFIG_APDS9960=n` + `CONFIG_PROSPECTOR_APDS9960=y` (`config/dongle_prospector/dongle_prospector_sensor.conf`) because the module ships a replacement driver — do not "fix" this back to the stock driver.
- ZMK Studio (`studio-rpc-usb-uart` snippet) is applied only to the central shields: `charybdis_right_bt` (BT split) and `charybdis_dongle` (dongle). Prospector screen builds have no Studio at all (RAM); USB logging and Studio both use USB CDC and are mutually exclusive.
- `settings_reset` is the upstream ZMK shield (no custom overlay); the local build script sed-patches its overlay to silence zero-length array warnings.
- Trackball/pointer behavior (speed, accel, scroll, thumb-scroll, precision mode) is all in `config/trackball/charybdis_pointer.dtsi`.

## Keymap drawings (keymap-drawer)

- `.github/workflows/draw_keymaps.yml` regenerates the `keymap-drawer/` SVGs/PNGs from `config/keymaps/qwerty.keymap` and **auto-commits** them to `main` (auto-commit is skipped when run under `act`).
- Run locally: `act -W .github/workflows/draw_keymaps.yml --bind --reuse`. Needs: python venv with `keymap-drawer` + pyyaml, `librsvg2-bin`, Node with Playwright (Chromium), and JetBrainsMono Nerd Font installed.
- The parse step symlinks `config/keymap_features` into `config/keymaps/` so `#include` resolves; replicate that if you run `keymap parse` yourself.
- After editing a keymap or anything under `keymap-drawer/configs/`, re-run this pipeline (or push to main and let it auto-commit). Only the final composites (`stacked-combos-{dark,light}.png`, `all_layers.svg`, `legend-1x1.yaml`) are committed; intermediate YAMLs and per-layer SVGs are gitignored.

## Verification & flashing notes

- There are no unit tests, lint, or typecheck. Verification for config changes = a successful `west build` (Docker or CI) and, for keymap edits, a successful keymap-drawer parse.
- Flashing gotchas (from README): when switching between BT and dongle modes, flash the `settings_reset` firmware to all devices first; for Prospector dongle builds power on the dongle before the left side, then the right (battery widget side sync).
