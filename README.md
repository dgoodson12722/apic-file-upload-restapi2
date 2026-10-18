# apic-file-upload-restapi2

Ansible toolkit for pre-upgrade checks and image staging on Cisco ACI
(APIC) fabrics. Three interactive playbooks, no inventory of hosts to
maintain — everything needed is either prompted for at runtime or read
from `inventory.yml`.

## Requirements

- `ansible-core` 2.20.1 (Python 3.14 compatible; also validated on
  ansible-core 2.20.1 / Python 3.13)
- Network/SSH reachability to the APIC(s) you're targeting
- For `00_check_apic_health.yml`: HTTPS reachability to the APIC's REST
  API (no SSH needed for that script)

## Repository layout

```
.
├── inventory.yml                      # image filenames + reference MD5s (shared by all scripts)
├── playbooks/
│   ├── build_inventory.yml            # generates inventory.yml from local files
│   ├── 00_check_apic_health.yml       # pre-flight health check (REST API, report-only)
│   └── 01_copy_and_verify_images.yml  # copies images to the APIC and verifies them (SSH)
└── tasks/
    ├── hash_one_image.yml             # helper for build_inventory.yml
    ├── check_one_image.yml            # helper for 00_check_apic_health.yml
    └── copy_and_verify_one_image.yml  # helper for 01_copy_and_verify_images.yml
```

## inventory.yml

All three playbooks share one inventory of images:

```yaml
images:
  - label: "APIC ISO"
    filename: "aci-apic-dk9.6.1.6g.iso"
    reference_md5: "..."
  - label: "NX-OS 64-bit"
    filename: "aci-n9000-dk9.16.1.6g-cs_64.bin"
    reference_md5: "..."
  - label: "NX-OS 32-bit"
    filename: "aci-n9000-dk9.16.1.6g.bin"
    reference_md5: "..."
```

`filename` is the image's filename as it exists locally (and as Cisco
named it). `reference_md5` is the known-good MD5 for that file — leave it
blank (`""`) to skip the checksum check for that image. You can hand-edit
this file, or generate/regenerate it automatically — see below.

## Usage

Run all playbooks from the repo root (so the relative paths to
`inventory.yml` and `tasks/` resolve correctly):

```bash
ansible-playbook playbooks/build_inventory.yml
ansible-playbook playbooks/00_check_apic_health.yml
ansible-playbook playbooks/01_copy_and_verify_images.yml
```

### 1. `build_inventory.yml` — generate inventory.yml

Run this first (or whenever your image set changes). It prompts for the
local directory containing your image files, then for each of the three
expected images' filenames. It finds each file in that directory and
computes its MD5 automatically — no hand-typed hashes. If a file isn't
found, that image's `reference_md5` is left blank with a console warning
rather than failing the run.

**This overwrites `inventory.yml`** — it regenerates the file from scratch
each run, so re-run it any time the image set changes rather than
hand-editing around it.

```
ansible-playbook playbooks/build_inventory.yml
```
Prompts: local image directory, APIC ISO filename, NX-OS 64-bit filename,
NX-OS 32-bit filename.

### 2. `00_check_apic_health.yml` — pre-flight health check (report-only)

Talks to the APIC over its REST API (HTTPS `aaaLogin` + the same
`GET /api/node/class/...` calls the APIC GUI itself makes) — no SSH. Run
this before staging any images to confirm the fabric is in good shape:

- **Cluster health** — all controllers must report `health: fully-fit`.
- **Free space on `/firmware`** — warns if any controller is at or above
  a threshold you set (default 80%).
- **Images already present** — checks each image in `inventory.yml`
  against the APIC's firmware repository. Matching is done by filename
  *stem* (ignoring the extension), since the repository doesn't always
  list an image under the same extension you have it as locally. When a
  `reference_md5` is set, it's compared against the repository's checksum,
  giving three possible results per image: **PRESENT**, **CHECKSUM
  MISMATCH** (same name, different file), or **NOT FOUND**.

Only cluster health and free space cause a non-zero exit; image status is
reported but doesn't fail the run (this phase is report-only — it never
copies, uploads, or changes anything on the APIC).

```
ansible-playbook playbooks/00_check_apic_health.yml
```
Prompts: APIC host, username, password, whether to validate its TLS cert
(default no — APICs commonly use a self-signed cert), and the free-space
warning threshold (default 80%).

Writes a JSON summary to `logs/apic_health_check_<host>_<timestamp>.json`.

### 3. `01_copy_and_verify_images.yml` — copy and verify images

Copies each image in `inventory.yml` to the APIC over SSH/SFTP and
verifies it landed intact:

1. Checks the local file's MD5 against `reference_md5` (skipped if blank)
   — a mismatch stops that image before anything is copied.
2. Copies the file to the APIC.
3. Computes the MD5 on both ends after the copy and compares them.

Logs a PASS/FAIL per image to the console, writes a JSON summary to
`logs/image_copy_verify_<host>_<timestamp>.json`, and exits non-zero if
any image failed.

```
ansible-playbook playbooks/01_copy_and_verify_images.yml
```
Prompts: APIC host, SSH username, SSH password (leave blank to use SSH
key/agent auth instead), local image directory, and the destination
directory on the APIC (default `/firmware/fwrepos/fwrepo/`).

## Typical workflow

```bash
ansible-playbook playbooks/build_inventory.yml        # generate inventory.yml with real hashes
ansible-playbook playbooks/00_check_apic_health.yml    # confirm cluster health / free space / what's already present
ansible-playbook playbooks/01_copy_and_verify_images.yml  # copy + verify whatever's missing
```

## Not yet included

- No support yet for checking/upgrading multiple devices in one run — each
  invocation targets one APIC.
- No REST-API-based image upload (as opposed to the SSH/SFTP copy in
  Script 1) — the documented APIC REST mechanism only supports pulling an
  image from a remote HTTP/SCP server, not a direct-upload equivalent of
  the GUI's "Local" option.
