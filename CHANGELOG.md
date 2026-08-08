<!--
MIT License
Copyright (c) 2026 H&M
-->

# Changelog — pakfire-manual

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [1.0.2] - 2026-08-08

### Added

- `enrich_addon_matrix()` in `pakfire-common.sh`: at Phase 1, after reading installed
  packages, back-fills `UNKNOWN` release numbers in `core_addon_matrix.tsv` for the
  current core update using live `pakfire list installed` data. Blog posts never state
  release numbers; this is the only source for them. The file is preserved across
  `update.sh` runs via the IPFire backup manifest.

### Fixed

- `docs/wui-overview-v1.1.0.png` renamed to `wui-overview-v1.0.1.png`: screenshot
  filename must reflect the installed package version at capture time (v1.0.1).

---

## [1.0.1] - 2026-08-08

### Fixed

- **`mirror_is_permitted()` — geo-check returned UNKNOWN for every mirror** (`pakfire-common.sh`):
  `location lookup` outputs the full country name (`Country : United Kingdom of Great Britain…`),
  not an ISO 2-letter code. The original `awk` pattern matched `^country:` (lowercase, no indent)
  and compared the result directly against the 2-letter blocklist — so every mirror resolved as
  UNKNOWN and was skipped. Fix: added `_load_location_name_map()` to build a lazy reverse map
  from `location list-countries --show-name` (full name → ISO CC); updated `awk` pattern to match
  the actual indented, capitalised `Country :` line.

- **Phase 1 — core state capture failed** (`pakfire-common.sh`):
  `pakfire list update` is not a valid subcommand in IPFire 2.29. The original code called it to
  discover pending updates; the command exited 1 and the captured output was empty. Fix: rewrote
  `parse_pakfire_update()` to use `pakfire status` for current and available core numbers, and
  `pakfire list upgrade --no-colors` (block format) for pending add-ons.

- **`parse_pakfire_installed()` — block format not handled** (`pakfire-common.sh`):
  Original code assumed tabular output from `pakfire list installed` and used `awk '{print $1}'`
  etc. per line. Actual output is key-value block format (one field per line, blank-line delimited).
  Fix: replaced with a state-machine block parser using `pakfire list installed --no-colors`.

- **`mirror-survey.sh` aborted after first MISS under `set -euo pipefail`** (`mirror-survey.sh`):
  `result="$(probe_mirror "${hostname}")"` — `probe_mirror` returns exit code 1 on a MISS; under
  `set -e`, a non-zero exit from a command substitution on the right-hand side of an assignment
  terminates the script. In dry-run mode (which never calls `probe_mirror`) the bug was silent.
  Fix: `result="$(probe_mirror "${hostname}")" || true`.

### Changed

- **Expanded `mirrors.allow` and `mirrors.tsv` from 2 entries to 12**: all 12 mirrors were
  probed and confirmed to serve the full pakfire2 2.29-x86_64 tree with old package retention.
  Mirrors grouped by region: Europe (8), Americas (3), Asia-Pacific (1).

---

## [1.0.0] - 2026-08-08

### Added

- Six-phase sequential upgrade planner for IPFire 2.29 x86_64 (`pakfire-planner.sh`):
  capture system state, review per-step plan with known-breakage overlay, survey
  mirrors, fetch and GPG-verify intermediate core paks, deploy one step, write
  post-deploy handoff notes
- Mirror survey tool (`mirror-survey.sh`): probes all IPFire-configured mirrors for
  the pakfire2 2.29-x86_64 path; validates each against the active Location Block
  egress policy; records results to `mirrors.tsv`
- Shared function library (`pakfire-common.sh`): GPG verification (hard stop on
  failure), geo-policy mirror check, state snapshot, post-update guard, structured
  dual logging
- Pakfire addon deployers (`install.sh`, `update.sh`, `uninstall.sh`) with full
  syslog logging and debug flag support
- CLI flags on all scripts: `-h|--help`, `-D|--debug`, `--dry-run`, `-v|--version`
- Data files: `mirrors.allow` (egress-allowed mirror hostnames), `mirrors.tsv`
  (survey database), `core_addon_matrix.tsv` (CU200-203 add-on version table
  sourced from IPFire blog), `known_breakage.tsv` (open Bugzilla entries with
  per-step severity ratings)
- WUI menu entry under IPFire Extras
- GPG verification against IPFire project signing key for every downloaded core
  pak; hard stop on any failure
