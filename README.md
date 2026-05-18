# Ansible Playbooks

A collection of Ansible playbooks for managing and automating tasks on IBM zADE (z Application Development Environment) and associated Linux host systems.

## Directory Structure

```
ansible-playbooks/
├── ansible-playbooks-linux/
│   ├── inventories/
│   │   └── inventory.yml
│   ├── backup_z32a_volumes.yml
│   ├── optimize_zade_network.yml
│   └── provision_volumes.yml
└── ansible-playbooks-zos/
    ├── inventories/
    │   ├── inventory.yml
    │   └── group_vars/
    │       └── all.yml
    ├── templates/
    │   └── Languages.yaml.j2
    ├── ansible.cfg
    ├── post_uuid.yml
    ├── setup_dbb_user.yml
    ├── setup_runner_user.yml
    └── zos_ping.yml
```

## Requirements

### General
- Ansible 2.9+
- SSH access to target hosts

### Linux Playbooks
- `xz` utility (required by `backup_z32a_volumes.yml`)
- `ethtool` and `tuned` packages (installed automatically by `optimize_zade_network.yml`)
- `alcckd` utility — IBM zADE disk creation tool (required by `provision_volumes.yml`)

### z/OS Playbooks
- Ansible collections:
  - `ibm.ibm_zos_core`
  - `ibm.ibm_zosmf`
- Python 3.14 at `/usr/lpp/IBM/cyp/v3r14/pyz` on the z/OS target
- ZOAU (z/OS Ansible Utility) v1.4 at `/usr/lpp/IBM/zoau/v1r4`
- z/OSMF running on port 10443 (required by `post_uuid.yml`)

Install required collections:

```sh
ansible-galaxy collection install ibm.ibm_zos_core ibm.ibm_zosmf
```

## Getting Started

1. Clone the repository:

    ```sh
    git clone https://github.com/yourusername/ansible-playbooks.git
    ```

2. Update the relevant `inventories/inventory.yml` with your target host details.

3. Run the desired playbook:

    ```sh
    ansible-playbook -i inventories/inventory.yml <playbook>.yml
    ```

---

## Linux Playbooks

### backup_z32a_volumes.yml

Creates a full compressed backup of z32a volumes using async tar with multi-threaded XZ compression. Verifies archive integrity after creation and enforces a configurable retention policy.

**Target host:** `nuc`

**Key behaviour:**
- Skips creation if an archive already exists for the current timestamp
- Uses `xz -T0 -1` (all cores, fast compression) with `--checkpoint=10000` progress markers
- Verifies the archive with `xz -t` before reporting success
- Retains the most recent `keep_backups` archives (default: 4) and deletes older ones

**Variables:**

| Variable | Default | Description |
|----------|---------|-------------|
| `src_dir` | `/home/ibmsys1/volumes-z32a` | Directory to back up |
| `dest_dir` | `/home/ibmsys1/volumes-backup` | Backup destination |
| `keep_backups` | `4` | Number of archives to retain |
| `backup_timeout` | `7200` | Max seconds to wait for tar (2 hours) |

**Usage:**

```sh
ansible-playbook -i inventories/inventory.yml backup_z32a_volumes.yml
```

**Restore:**

```sh
tar -xvf volumes-z32a-<timestamp>.tar.xz -C /desired/restore/path
```

**Tags:** `verify`, `prepare`, `backup`, `cleanup`, `report`

---

### optimize_zade_network.yml

Tunes network settings on the zADE host for maximum throughput. Disables NIC offloads, increases ring buffer sizes, sets RX interrupt coalescing, pins NIC IRQs to isolated CPUs, and applies TCP socket buffer tuning via sysctl.

**Target host:** `nuc`
**Requires:** root (`become: yes`)

**Key behaviour:**
- Installs `ethtool` and `tuned` if not present
- Disables offloads: `rx`, `tso`, `gso`, `gro`, `lro` — unsupported offloads are skipped and logged rather than failing the play
- Sets NIC ring buffers to 4096 (RX and TX)
- Sets RX interrupt coalescing to 32 µs (IGC-compatible)
- Pins NIC IRQs to CPUs 2–3 (via `smp_affinity_list`), leaving CPUs 0–1 free for zADE engine threads
- Tunes TCP/IP socket buffers and backlog via sysctl (`rmem_max`, `wmem_max`, `tcp_rmem`, `tcp_wmem`, etc.)
- Applies `network-throughput` tuned profile

**Variables:**

| Variable | Default | Description |
|----------|---------|-------------|
| `network_interface` | `enp86s0` | NIC to tune |
| `make_persistent` | `false` | Persist settings across reboots |

**Usage:**

```sh
ansible-playbook -i inventories/inventory.yml optimize_zade_network.yml --ask-become-pass
# Override interface:
ansible-playbook -i inventories/inventory.yml optimize_zade_network.yml -e "network_interface=eth0"
# Persist across reboots:
ansible-playbook -i inventories/inventory.yml optimize_zade_network.yml -e "make_persistent=true"
```

---

### provision_volumes.yml

Creates zADE CKD disk volumes using the `alcckd` utility. Provisions a set of 3390-54 and 3390-9 volumes and verifies they exist after creation. Idempotent — existing volumes are skipped.

**Target host:** `nuc`

**Volumes created:**

| Type | Volumes |
|------|---------|
| 3390-54 | `a3usr2`, `a3usr3`, `a3usr4`, `a3usr5`, `a3zcx2` |
| 3390-9 | `work0a`, `work0b` |

**Variables:**

| Variable | Default | Description |
|----------|---------|-------------|
| `dest_dir` | `/home/ibmsys1/volumes-z32a` | Directory to create volumes in |

**Usage:**

```sh
ansible-playbook -i inventories/inventory.yml provision_volumes.yml
# Custom destination:
ansible-playbook -i inventories/inventory.yml provision_volumes.yml -e "dest_dir=/custom/path"
```

---

## z/OS Playbooks

### zos_ping.yml

Tests connectivity to a z/OS system using the `ibm.ibm_zos_core.zos_ping` module and asserts a successful `pong` response.

**Target host:** `z32a`

**Usage:**

```sh
ansible-playbook -i inventories/inventory.yml zos_ping.yml
# Enable debug output:
ansible-playbook -i inventories/inventory.yml zos_ping.yml -e "boolean_debug=true"
```

---

### post_uuid.yml

POSTs z/OS UUID information to a z/OSMF endpoint using the `ibm.ibm_zosmf` collection. Verifies z/OSMF connectivity before executing the `zmf_swmgmt_zos_system_uuid` role.

**Target host:** `z32a`
**Requires:** `zmf_swmgmt_zos_system_uuid` role from the `ibm.ibm_zosmf` collection

**Usage:**

```sh
ansible-playbook -i inventories/inventory.yml post_uuid.yml
```

---

## z/OS User Onboarding (run in order)

Onboarding a new z/OS USS user as a GitLab Runner + DBB workload account on ADCD z32a is a **two-step process**. Run the playbooks in this order:

1. **`setup_runner_user.yml`** — runner-platform foundations (`.profile`, runner directories).
2. **`setup_dbb_user.yml`** — DBB-specific working files on top.

```sh
# Step 1 — runner-side foundations
ansible-playbook -i inventories/inventory.yml setup_runner_user.yml -e "target_user=seb"

# Step 2 — DBB working files (requires Step 1)
ansible-playbook -i inventories/inventory.yml setup_dbb_user.yml -e "target_user=seb"
```

### Shared prerequisites

Before running either playbook:

- The target user must already exist with an OMVS segment (HOME and shell), e.g. from RACF:
  ```
  ADDUSER SEB OMVS(HOME(/u/seb) PROGRAM(/bin/sh))
  ```
- `/u/<target_user>/` must exist and be owned by the target user.
- `/u/ibmuser/.profile` is the canonical template — it must already have DBB env vars and the `zopen-config` sourcing block at column 0.
- The connecting user (`IBMUSER` in `inventories/inventory.yml`) must have `BPX.SUPERUSER` access or UID 0 to chown files into another user's home.

---

### setup_runner_user.yml

Onboards a new z/OS USS user as a GitLab Runner workload account by copying `IBMUSER`'s `.profile` as the canonical template, preserving the IBM-1047 file tag. Creates `~/gitlab-runner/{builds,cache}` directories owned by the target user.

Idempotent — safe to re-run. The `.profile` copy is guarded with `creates:` and skipped if the target already has one.

**Target host:** `z32a`
**Run order:** First (before `setup_dbb_user.yml`).

**Variables:**

| Variable | Default | Description |
|----------|---------|-------------|
| `target_user` | _(required)_ | The new user to onboard. Must not be `ibmuser`. |
| `target_group` | `SYS1` | Default group for chown |
| `boolean_debug` | `false` | Verbose smoke-test output |

**Usage:**

```sh
ansible-playbook -i inventories/inventory.yml setup_runner_user.yml -e "target_user=seb"
```

**Smoke test verifies:** `.profile` tagged `IBM-1047`, owned by the target user, contains expected exports (`DBB_HOME`, `DBB_BUILD`, `zopen-config`), and `~/gitlab-runner/{builds,cache}` exist.

**Tags:** `validate`, `profile`, `dirs`, `smoketest`

---

### setup_dbb_user.yml

Adds IBM DBB v3 working files on top of an already-onboarded runner user. Creates `~/zBuilder/build/`, copies `dbb-build.yaml` and the language YAML samples from `$DBB_HOME` (preserving UTF-8 tags via `cp -p`), and templates a site-specific `Languages.yaml` from `dbb_languages_datasets` in `group_vars/all.yml`. Tags `Languages.yaml` as UTF-8 so DBB's Java reader picks it up correctly.

Does **not** modify `.profile` — the DBB env vars are already present in `/u/ibmuser/.profile` (the template copied by `setup_runner_user.yml`).

**Target host:** `z32a`
**Run order:** Second. Asserts the target user's `.profile` already exists and fails fast otherwise.

**Variables:**

| Variable | Default | Description |
|----------|---------|-------------|
| `target_user` | _(required)_ | The user to set up DBB for |
| `target_group` | `SYS1` | Default group for chown |
| `boolean_debug` | `false` | Verbose smoke-test output |

**Usage:**

```sh
ansible-playbook -i inventories/inventory.yml setup_dbb_user.yml -e "target_user=seb"

# Skip Languages.yaml templating (preserve hand-tuned values):
ansible-playbook -i inventories/inventory.yml setup_dbb_user.yml -e "target_user=seb" --skip-tags languages
```

**Customising dataset names:** the COBOL / LE / CICS / HLASM dataset variables that DBB resolves at build time (`SIGYCOMP`, `SCEELKED`, `SDFHLOAD`, etc.) live in `inventories/group_vars/all.yml` under `dbb_languages_datasets`. Edit that dict and re-run the playbook to push changes — values are sorted alphabetically into `Languages.yaml` for stable templating diffs.

**Smoke test verifies:** `DBB_BUILD` directory listing has expected files (`Languages.yaml`, `dbb-build.yaml`), `Languages.yaml` is tagged `UTF-8`, ownership is correct, target `.profile` has `DBB_HOME` exported.

**Tags:** `validate`, `dirs`, `copy`, `languages`, `smoketest`
