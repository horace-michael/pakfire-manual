<!--
MIT License
Copyright (c) 2026 H&M
-->

# pakfire-manual

Controlled sequential upgrade tool for IPFire 2.29 x86_64.

**[Full usage guide →](docs/USAGE.md)**

Manages the 200 → 201 → 202 → 203 core upgrade chain one step at a time:
per-step planning from the IPFire blog matrix, geo-aware mirror selection,
GPG verification, snapshot-before-deploy, and a structured handoff log.

---

## Why this exists

IPFire's `pakfire upgrade` applies all pending updates in one shot. When
multiple core updates are pending - each with its own add-on set, breaking
changes, and open bugs - a single-shot upgrade gives no opportunity to
validate intermediate state, stop at a known-good point, or recover cleanly
if one step fails.

This tool applies exactly one core step at a time with explicit confirmation
at every gate: geo-policy check, GPG signature check, crash-bug hold, and
post-deploy sanity checks. It does not reboot and it never chains steps.

---

## The mirror problem pakfire can't solve on geo-restricted estates

IPFire's `pakfire` selects a mirror from its built-in server list and trusts
it unconditionally. It has **zero awareness of your outgoing firewall policy**.
On estates with a strict outgoing geo-block list this creates a silent failure
mode: pakfire may quietly fail to reach any mirror, or the connection gets
dropped by the firewall with no useful error surfaced back to the user.

The only native workaround is to hardcode a single mirror in `/etc/pakfire.conf`:

```
# /etc/pakfire.conf — native pakfire workaround for geo-restricted estates
$mirror = "https://mirror1.ipfire.org/pakfire2";
```

This works - until the mirror goes down, changes its path, or the country it
lives in gets added to your block group. There is no fallback, no geo-check,
and no validation. It breaks silently when the world changes.

**pakfire-manual does it differently:**

- Reads your live Location Group directly from
  `/var/ipfire/fwhosts/customlocationgrp` - the same source the firewall
  kernel rules are built from - on every run
- Resolves **every IP address** a mirror hostname returns, including
  round-robin entries and dual-stack (IPv4 + IPv6)
- Calls `location lookup <ip>` against the same libloc database your packet
  filter uses - not a third-party GeoIP service, not a cached lookup
- Skips any mirror with a BLOCKED verdict, logs every decision with country
  code and ASN, and proceeds only with mirrors that pass the check

No static config. No manual overrides. No silent failures.

If you add a country to the block group in the WUI today, pakfire-manual will
refuse any mirror hosted in that country on its very next run - no config
file to update, no human intervention required. Policy lives in one place: the
WUI. Everything else follows.

---

## Integration with firewall-local

This tool is aware of the
[firewall-local](https://github.com/horace-michael/firewall-local) add-on
and its **Outgoing Location Block** module.

### How the integration works

`firewall-local` manages outgoing geo-blocking through a Custom Location
Group maintained in the IPFire WUI (`outgoing-location-block.cgi`). The
module builds `GL_BAD_COUNTRIES` - a kernel `list:set` ipset composed of
one `block_cc_XX` ipset per blocked country - and installs two `REJECT`
rules on `FORWARDFW` and `OUTGOINGFW`. This is the firewall egress policy.

The authoritative source file is:

```
/var/ipfire/fwhosts/customlocationgrp
```

Format: `id,group_name,remark,COUNTRY_CODE,type`

`pakfire-manual` reads this file at runtime through `load_blocked_countries()`
in `/usr/local/bin/pakfire-common.sh`. It never duplicates the country list
into a static config file. Instead, `mirror_is_permitted()` resolves every IP
address a mirror hostname returns, calls `location lookup <ip>` (the same
libloc database the packet filter uses), and checks each result against the
loaded blocklist.

**The consequence:** if you add a country to the Outgoing Location Block
group via the WUI, pakfire-manual will automatically refuse any mirror
hosted in that country on its next run - no config edit required. Policy
lives in one place: the WUI.

### Verified at time of writing (2026-08-03)

- `www.mirrorservice.org` resolves to GB addresses.
- GB is **not** in the configured Location Group (verified 2026-08-03).
- The mirror is therefore permitted and `mirrors.allow` has it uncommented.
- If GB is ever added to the block group, `mirror_is_permitted()` will
  catch it and log `BLOCKED` - the toolkit will then report no usable
  mirrors and stop cleanly.

---

## Repository structure

```
pakfire-manual/
├── src/                             installed layout (pakfire addon payload)
│   ├── usr/local/bin/
│   │   ├── pakfire-planner.sh       main tool (run as root on IPFire box)
│   │   ├── mirror-survey.sh         discover mirrors from server-list.db
│   │   └── pakfire-common.sh        shared function library (source only)
│   ├── srv/web/ipfire/cgi-bin/
│   │   └── pakfire-manual.cgi       IPFire WUI — manages all data files
│   └── var/ipfire/
│       ├── menu.d/
│       │   └── EX-pakfire-manual.menu  IPFire sidebar entry "Pakfire-Manual"
│       ├── backup/addons/includes/
│       │   └── pakfire-manual       IPFire backup manifest for data files
│       └── pakfire-manual/
│           ├── mirrors.allow        mirror hostnames (geo-checked at runtime)
│           ├── mirrors.tsv          known mirrors and their status
│           ├── core_addon_matrix.tsv  per-core add-on version table (from blog)
│           └── known_breakage.tsv   open Bugzilla issues blocking upgrade steps
├── paks/
│   ├── install.sh                   pakfire install handler
│   ├── update.sh                    pakfire update handler (zero-data-loss)
│   └── uninstall.sh                 pakfire uninstall handler
└── docs/
    └── USAGE.md                     step-by-step workflow guide
```

---

## Requirements

- IPFire 2.29 x86_64
- `location` (libloc) tool - standard on IPFire
- `curl` and `wget`
- GPG keyring with the IPFire project signing key - imported automatically by Phase 4 if absent; no manual setup required
- [`firewall-local`](https://github.com/horace-michael/firewall-local) ≥ 1.8.0 recommended (provides customlocationgrp)

---

## Quick start

Download the latest release from the
[GitHub releases page](https://github.com/horace-michael/pakfire-manual/releases).

**Manual download and verify:**

```bash
# On a workstation — download all three release assets:
wget https://github.com/horace-michael/pakfire-manual/releases/latest/download/pakfire-manual-v1.6.4.ipfire
wget https://github.com/horace-michael/pakfire-manual/releases/latest/download/pakfire-manual-v1.6.4.ipfire.sha256
wget https://github.com/horace-michael/pakfire-manual/releases/latest/download/pakfire-manual-v1.6.4.ipfire.asc

# Import signing key from GITHUB (once):
curl -s https://github.com/horace-michael.gpg | gpg --import

# Verify that key was imported:
gpg --list-keys horace-michael@users.noreply.github.com

#It should show the public key:
pub   ed25519 2026-03-27 [SC] [expires: 2028-03-26]
      88B0B4C4E9FF0D8F2292720AC035A0144DEB0A90
uid           [ unknown] Horace Michael (GitHub UID) <32497938+horace-michael@users.noreply.github.com>
sub   cv25519 2026-03-27 [E] [expires: 2028-03-26]
sub   rsa4096 2026-08-22 [S] [expires: 2028-08-21]

# Verify checksum and GPG signature:
sha256sum -c pakfire-manual-v1.6.4.ipfire.sha256
gpg --verify pakfire-manual-v1.6.4.ipfire.asc pakfire-manual-v1.6.4.ipfire

# Copy to IPFire:
scp pakfire-manual-v1.6.4.ipfire root@<ipfire-host>:/opt/pakfire/tmp/

# Switch on the IPFire machine:

# Install the add-on
# Note: since v1.6.4 install.sh imports all /opt/pakfire/pakfire-*.key automatically —
# the manual GPG import step below is no longer required on a fresh install.
# It is kept here for reference and for manual verification on older installs.
cd /opt/pakfire/tmp
tar -xvf pakfire-manual-v1.6.4.ipfire
NAME=pakfire-manual bash install.sh

# Optional: verify pakfire signing keys were imported by install.sh:
gpg --list-keys pakfire@ipfire.org
pub   rsa4096 2018-03-16 [SC]
      3ECA8AA4478208B924BB96206FEF7A8ED713594B
uid           [ unknown] IPFire Pakfire Package Signing Key <pakfire@ipfire.org>

pub   rsa4096 2022-03-21 [SC]
      98448779295007D302A91E32A551AE95C8DCE211
uid           [ unknown] IPFire Pakfire Package Signing Key <pakfire@ipfire.org>
sub   rsa4096 2022-03-21 [E]

# If the above shows no keys, list the available key files and import them manually:
ll /opt/pakfire/*.key # pakfire keys usually are stored in files like pakfire-<YYY>.key where <YYYY i sthe year when key was released.

# Get the names of the keys listed by above command and import them manually  
gpg --import /opt/pakfire/pakfire-<YYYY>.key /opt/pakfire/pakfire-<YYYY>.key
```

See [docs/USAGE.md](docs/USAGE.md) for the full phase-by-phase workflow.

---

## Data files

After installation, data files live at `/var/ipfire/pakfire-manual/`:

- `mirrors.allow` - mirror hostnames permitted for pakfire fetches. Pre-seeded
  with 12 verified mirrors across Europe, Americas, and Asia-Pacific. Manage via the WUI.
- `mirrors.tsv` - survey database populated by `mirror-survey.sh`. Starts
  empty. Run the survey once after installation to discover mirrors - the WUI
  Mirror Database section shows results.
- `core_addon_matrix.tsv` - add-on version table sourced from IPFire blog
  posts for CU200–203. Manage via the WUI.
- `known_breakage.tsv` - open Bugzilla issues blocking upgrade steps.
  Manage via the WUI.

---

## Upgrade context

Production box held at Core 200. Upgrade target: Core 202 (200 → 201 → 202
sequentially). Core 203 is on hold until Bugzilla #14024 (kresd crash at
≥ 16 host definitions per IP) is resolved upstream.

Known open blockers are tracked in `known_breakage.tsv`. Phase 2 of
the planner reports them per step; phase 5 refuses any step carrying a
`severity=crash` bug unless `--override-hold` is passed and the core number
is typed interactively.

---

## Signing key

All release assets (`.ipfire`, `.sha256`, `.asc`) are signed with a dedicated GPG subkey.

| Field | Value |
|-------|-------|
| Master key ID | `C035A0144DEB0A90` |
| Signing subkey | `99BBF17261532022` (rsa4096) |
| Fingerprint | `88B0B4C4E9FF0D8F2292720AC035A0144DEB0A90` |
| Expires | 2028-03-26 |
| Keyserver | [keys.openpgp.org](https://keys.openpgp.org/search?q=C035A0144DEB0A90) |

Import:

```bash
curl https://keys.openpgp.org/vks/v1/by-fingerprint/88B0B4C4E9FF0D8F2292720AC035A0144DEB0A90 | gpg --import
```

Verify a release asset:

```bash
gpg --verify pakfire-manual-vX.Y.Z.ipfire.asc pakfire-manual-vX.Y.Z.ipfire
```

---

## Credits

This tool was inspired by **[pscar13](https://community.ipfire.org/u/pscar13/summary)**,
a retired software engineer from France who has been running IPFire since 2019 and
regularly tests pre-release versions to provide feedback to the project.

The original idea - applying core updates one step at a time, with explicit review at
each gate - came from this post on the IPFire Community Forum:
[Pakfire upgrade to specific version](https://community.ipfire.org/t/pakfire-upgrade-to-specific-version/6907/9)

Without that post this package would not exist.

---

## License

MIT - Copyright (c) 2026 H&M
