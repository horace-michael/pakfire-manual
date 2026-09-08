<!--
MIT License
Copyright (c) 2026 H&M
-->

# Changelog - pakfire-manual

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

---

## [1.6.5] - 2026-08-26

### Fixed

- Phase 4 (fetch-verify): now downloads only the single next core package instead of the entire
  chain up to the latest available core. Previously fetch-verify looped from `current_core + 1`
  to `latest_core`, downloading every intermediate package regardless of how many steps were
  actually needed for the current operation.

---

## [1.6.4] - 2026-08-24

### Fixed

- Phase 4 (fetch-verify): all IPFire package signing keys (`/opt/pakfire/pakfire-*.key`)
  are now imported automatically at install time and again at fetch time if absent from
  the GPG keyring. Previously Phase 4 failed immediately on a fresh install, requiring a
  manual `gpg --import` step.

---

## [1.6.3] - 2026-08-24

### Security

- Control binary: input validation hardening. Update recommended.

---

## [1.6.2] - 2026-08-24

### Fixed

- Control Panel: SSH deploy instructions now show the correct commands. Previously the
  WUI instructed users to run the internal control binary (`pakfire-manual-ctrl`) directly
  from SSH, which is wrong — that binary exists only for the web server's privilege
  escalation path and must not be called by hand. The `--phase` flag shown was also
  invalid. Correct SSH commands are:
  `pakfire-planner.sh fetch-verify` then `pakfire-planner.sh deploy`.

---

## [1.6.1] - 2026-08-23

### Fixed

- Mirror Survey: syslog now shows the actual probe outcome for every mirror. Previously
  "geo-policy: PERMITTED" looked like a final pass verdict in syslog; the HTTP probe result
  (FOUND or MISS) was only visible with `-D`. Both steps are now logged unconditionally so
  `/var/log/messages` tells the complete story without debug mode.

---

## [1.6.0] - 2026-08-23

### Fixed

- Mirror Survey: no longer requires `/var/ipfire/fwhosts/customlocationgrp` to exist or
  contain any entries. When the file is absent, empty, or defines no country groups,
  the survey treats all mirrors as permitted and proceeds normally. Previously any of
  these conditions caused an immediate abort, preventing the survey from running on
  boxes that do not use the Outgoing Location Block egress policy.

---

## [1.5.7] - 2026-08-23

### Fixed

- Mirror Survey: no longer requires `/var/ipfire/fwhosts/customlocationgrp` to exist.
  When the file is absent or empty (Outgoing Location Block not configured), the survey
  now treats all mirrors as permitted and proceeds normally. Previously the survey aborted
  immediately with an error on boxes that do not use the Location Block egress policy.

---

## [1.5.6] - 2026-08-23

### Fixed

- Fresh install: `pakfire-manual-ctrl` now has correct ownership (`root:nobody`) and setuid
  mode (`4750`) applied immediately after payload extraction. Previously the binary was left
  as `root:root 4750` which prevented Apache (running as `nobody`) from executing it, causing
  "Control binary not found or not executable" in the WUI Control Panel.

---

## [1.5.5] - 2026-08-22

### Added

- README: signing key table with fingerprint, subkey ID, expiry, and keyserver link.
- README and installation guide: GPG signature verification instructions for manual installs.

---

## [1.5.4] - 2026-08-22

### Fixed

- Control Panel: the "Last fetch" line in the Deploy section now shows "Done" or "Failed"
  instead of always showing "DONE". Previously the raw internal status word was printed
  regardless of exit code, making a failed fetch look like a success to a quick read.
- WUI: Known Breakage table Status column no longer truncates "ASSIGNED" to "ASSI". The
  column now has `white-space:nowrap` so the browser cannot collapse it.

---

## [1.5.3] - 2026-08-22

### Fixed

- Phase 4 (fetch-verify): GPG keyring check now searches for `pakfire@ipfire.org` - the
  actual IPFire package signing key UID. The old check used `info@ipfire.org` which is not
  in the keyring, causing Phase 4 to abort before downloading anything.
- Phase 4 (fetch-verify): Core upgrade packages (`.ipfire`) carry an embedded PGP signature
  (PGP compressed data, ZIP algorithm) - not a separate `.gpg` detached signature file.
  Download and verification now use `gpg --decrypt` on the file itself instead of looking
  for a nonexistent `.gpg` sidecar file.
- Phase 5 (deploy): Core packages now extracted with `gpg --decrypt | tar -x`. Previously
  `tar -xf` was called directly on the PGP-wrapped `.ipfire` file and failed immediately.
- Phase 5 (deploy): `core/mine` is now updated to the deployed core number after `update.sh`
  exits. IPFire's own pakfire infrastructure normally writes `mine`, but since we invoke
  `update.sh` directly (bypassing pakfire), the planner now writes it explicitly.
- Phase 5 (deploy): Post-update grub check no longer emits a false-positive warning. The
  previous check searched for the core number inside the kernel filename (`vmlinuz-.*202`);
  IPFire kernels are named by kernel version string (`6.18.32-ipfire`), not core update
  number - so the check always failed. Now checks whether grub.cfg was modified recently.

---

## [1.5.1] - 2026-08-22

### Fixed

- WUI control panel: "Local State", "Per-Step Plan", and "Survey" buttons now launch
  the correct planner phase. The control binary was passing a `--phase` flag that the
  planner does not accept, causing every WUI-triggered job to fail immediately with
  exit code 1.

## [1.5.0] - 2026-08-22

### Fixed

- WUI control panel: jobs no longer fail with "cannot open lock file" after an IPFire
  reboot. The runtime directory (`/var/run/pakfire-manual/`) is recreated automatically
  on the first button press after a reboot.

---

## [1.4.0] - 2026-08-22

### Added

- **Control Panel** (new WUI section): Run buttons for Local State, Upgrade Plan, and Mirror
  Survey launch operations in the background without blocking the browser. Each row shows live
  job status (Running / Done / Error / Not yet run), the last start time, and the exit code.

- Version footer in the WUI: the bottom-right corner of the page now shows `pakfire-manual v<VERSION>`.
- Version disclosure in syslog: install, update, and uninstall operations now log the package
  version (`Installing/Updating/Uninstalling pakfire-manual vX.Y.Z`) so the version is visible
  in `/var/log/messages` without opening any file.
- `--version` flag on all package scripts (`pakfire-planner.sh`, `mirror-survey.sh`,
  `pakfire-common.sh`, `paks/install.sh`, `paks/update.sh`, `paks/uninstall.sh`):
  prints the script name and version and exits.

### Changed

- Package version is now a single source of truth: `VERSION=` in `pakfire-planner.sh` is the
  authoritative value; the build tool propagates it to all other package scripts and the CGI
  at build time so every installed file reports the same version.
- `update.sh`: re-creates `/var/run/pakfire-manual/` (tmpfs, wiped on every IPFire reboot)
  after payload extraction, and restores correct ownership and setuid mode on the control
  binary (Linux `chown` strips the setuid bit; `chmod 4750` is now applied after `chown`).

### Fixed

- CGI: Perl syntax error (double statement modifier `for LIST unless COND` is illegal); fixed
  to `unless (COND) { EXPR for LIST; }`.
- CGI: em dash (`—`) rendered as `â€"` under IPFire's Latin-1 page encoding; replaced with
  `&mdash;` HTML entity in all rendered strings.

---

## [1.3.0] - 2026-08-13

### Changed

- Phase 5 (deploy): now creates a full pre-deploy backup (main OS + all registered addons)
  before applying any core upgrade, matching official pakfire behaviour.

---

## [1.1.0] - 2026-08-08

### Added

- `load_addon_matrix()`: merges the package-provided addon version table with the
  machine-specific local override file. Local entries win on key collision; local-only
  additions are appended. Called by the planner and CGI for all matrix reads.

### Changed

- `core_addon_matrix.tsv` is now package-managed and updated on every package upgrade;
  it is no longer included in the backup manifest.
- `core_addon_matrix.local.tsv` (new file): machine-specific release numbers and user
  additions; included in the backup manifest and preserved across package upgrades.
  Populated automatically by `enrich_addon_matrix()` after Phase 1.
- WUI matrix section shows merged view. Package rows display a lock icon (read-only).
  User additions and local overrides have full edit/delete. Schema mismatch shows a
  recovery banner instead of crashing the page.

### Fixed

- Package upgrades no longer overwrite machine-enriched release numbers: enrichments
  are now stored in `core_addon_matrix.local.tsv` which survives updates.
- Upgrading from v1.0.x to v1.1.0 automatically migrates any existing enriched release
  numbers from the old single file to the new local file.

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

- **`mirror_is_permitted()` - geo-check returned UNKNOWN for every mirror** (`pakfire-common.sh`):
  `location lookup` outputs the full country name (`Country : United Kingdom of Great Britain…`),
  not an ISO 2-letter code. The original `awk` pattern matched `^country:` (lowercase, no indent)
  and compared the result directly against the 2-letter blocklist - so every mirror resolved as
  UNKNOWN and was skipped. Fix: added `_load_location_name_map()` to build a lazy reverse map
  from `location list-countries --show-name` (full name → ISO CC); updated `awk` pattern to match
  the actual indented, capitalised `Country :` line.

- **Phase 1 - core state capture failed** (`pakfire-common.sh`):
  `pakfire list update` is not a valid subcommand in IPFire 2.29. The original code called it to
  discover pending updates; the command exited 1 and the captured output was empty. Fix: rewrote
  `parse_pakfire_update()` to use `pakfire status` for current and available core numbers, and
  `pakfire list upgrade --no-colors` (block format) for pending add-ons.

- **`parse_pakfire_installed()` - block format not handled** (`pakfire-common.sh`):
  Original code assumed tabular output from `pakfire list installed` and used `awk '{print $1}'`
  etc. per line. Actual output is key-value block format (one field per line, blank-line delimited).
  Fix: replaced with a state-machine block parser using `pakfire list installed --no-colors`.

- **`mirror-survey.sh` aborted after first MISS under `set -euo pipefail`** (`mirror-survey.sh`):
  `result="$(probe_mirror "${hostname}")"` - `probe_mirror` returns exit code 1 on a MISS; under
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
