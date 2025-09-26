# TODO

## Architecture override

### Core idea: Declarative, layered themes with an atomic apply

- Themes are data-first (YAML/JSON manifests), not scripts.
- A build step renders templates into a staging dir, generates a plan (diff), then applies atomically with backups and rollback.
- Themes are layered: base → flavor → user overrides → host profile.

### Directory layout
- `themes/<name>/theme.yaml` — manifest
- `themes/<name>/templates/` — files with placeholders
- `themes/<name>/modules/*.sh` — optional lifecycle logic
- `profiles/<machine>.yaml` — compose themes, variables, toggles
- `schemas/*.json` — validation for manifests and variables
- `bin/arch-theme` — CLI: apply, preview, plan, rollback, list

### Theme manifest (single source of truth)
```yaml
# themes/cottagecore/theme.yaml
apiVersion: v1
name: cottagecore
version: 1.2.0
description: Pastel, clean typography, minimal blur.
requires:
  os: arch
  packages:
    pacman: [lightdm, i3, rofi, zathura]
    aur: [ttf-iosevka-nerd]
assets:
  fonts:
    - source: url:https://example/fonts/iosevka-nerd.ttf
      sha256: <checksum>
  wallpapers:
    - path: assets/wallpapers/meadow.jpg
variables:
  schema: ../../schemas/variables.schema.json
  defaults:
    accent: "#a6c48a"
    font_family: "Iosevka Nerd Font"
templates:
  - from: templates/rofi/config.rasi.tmpl
    to: $XDG_CONFIG_HOME/rofi/config.rasi
  - from: templates/i3/config.tmpl
    to: $XDG_CONFIG_HOME/i3/config
symlinks:
  - from: assets/icons
    to: $XDG_DATA_HOME/icons/cottagecore
services:
  enable: [lightdm]
modules:
  - name: custom
    script: modules/custom.sh
    hooks: [pre_apply, post_apply]
```

### Profiles: compose themes and overrides
```yaml
# profiles/laptop.yaml
include:
  - themes/common/theme.yaml
  - themes/cottagecore/theme.yaml
overrides:
  variables:
    accent: "#88d4a3"
  options:
    lightdm: true
    sway: false
targets:
  - user: alba
    home: /home/alba
    xdg: true
```

### Module API (optional, but standardized)
- Functions: `module::pre_apply`, `module::apply`, `module::post_apply`, `module::verify`, `module::rollback`
- Inputs: resolved variables, paths, dry-run flag, logger
- Contract: pure where possible; no direct `rm -rf` without receiving an allow-listed path from the planner

### Build/apply pipeline
1. Validate manifests and variables against schemas.
2. Resolve layers (base → theme → overrides → profile).
3. Fetch assets (or install via `pacman`/AUR) with checksums.
4. Render templates to a staging dir using `envsubst` or `gomplate`.
5. Generate a plan (file diffs, service changes).
6. Preview (`--dry-run`): print plan with colorized diff.
7. Apply atomically:
   - Backup targets (timestamped).
   - Apply file writes/symlinks.
   - Enable/disable services in a single batch.
   - Run module hooks.
   - Write a rollback manifest.
8. Verify and record state (`~/.cache/arch-theme/state.json`).

### CLI interface
- `arch-theme list` — available themes and versions
- `arch-theme plan --profile profiles/laptop.yaml`
- `arch-theme preview --profile profiles/laptop.yaml`
- `arch-theme apply --profile profiles/laptop.yaml --yes`
- `arch-theme rollback <id>`
- `arch-theme doctor` — environment checks

### Layering and reuse
- Shared assets live under `themes/_shared/` and are referenced via `symlinks` or `assets`.
- Deduplicate fonts/icons; prefer system packages; fall back to `assets` with checksums.
- Variables are strongly typed via JSON Schema; missing/invalid vars fail fast with a helpful error.

### Safety and idempotency
- All paths are resolved and allow-listed before any write.
- Every run is idempotent; unchanged files are skipped.
- Privileged steps are grouped; minimal `sudo` scope.
- Full rollback using recorded backups and service state.

### Migration path from current repo
- Extract repeated fonts/wallpapers to `_shared`.
- For each theme:
  - Write `theme.yaml` with `requires`, `variables.defaults`, `templates`, `services`, `modules`.
  - Move `variables.json` into `variables.defaults` or keep as `variables.json` consumed by the renderer.
  - Convert sed-based substitutions to templates (`${accent}`, `${font_family}`).
- Introduce `profiles/laptop.yaml` and `bin/arch-theme` shim that wraps current `src/main.sh` until fully migrated.

#### Fixes

- [ ] Update polybar theme on gruVbox
- [ ] Spotify (loading theme automatically without having to open spotify)
- [ ] There might be files that are merged because they have the same name (polybar and lock `launch.sh`)
