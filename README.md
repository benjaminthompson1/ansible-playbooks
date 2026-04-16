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
    ├── post_uuid.yml
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
- Python 3.12 at `/usr/lpp/IBM/cyp/v3r12/pyz` on the z/OS target
- ZOAU (z/OS Ansible Utility) at `/usr/lpp/IBM/zoautil`
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

Tunes network settings on the zADE host for maximum throughput. Disables NIC offloads, increases ring buffer sizes, sets RX interrupt coalescing, and applies TCP socket buffer tuning via sysctl.

**Target host:** `nuc`
**Requires:** root (`become: yes`)

**Key behaviour:**
- Installs `ethtool` and `tuned` if not present
- Disables offloads: `rx`, `tso`, `gso`, `gro`, `lro` — unsupported offloads are skipped and logged rather than failing the play
- Sets NIC ring buffers to 4096 (RX and TX)
- Sets RX interrupt coalescing to 32 µs (IGC-compatible)
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

**Target host:** `z31c_s0w1`

**Usage:**

```sh
ansible-playbook -i inventories/inventory.yml zos_ping.yml
# Enable debug output:
ansible-playbook -i inventories/inventory.yml zos_ping.yml -e "boolean_debug=true"
```

---

### post_uuid.yml

POSTs z/OS UUID information to a z/OSMF endpoint using the `ibm.ibm_zosmf` collection. Verifies z/OSMF connectivity before executing the `zmf_swmgmt_zos_system_uuid` role.

**Target host:** `z31a_s0w1`
**Requires:** `zmf_swmgmt_zos_system_uuid` role from the `ibm.ibm_zosmf` collection

**Usage:**

```sh
ansible-playbook -i inventories/inventory.yml post_uuid.yml
```
