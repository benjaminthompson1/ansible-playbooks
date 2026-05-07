# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running Playbooks

All commands run from within the relevant subdirectory (`ansible-playbooks-linux/` or `ansible-playbooks-zos/`):

```sh
# Linux playbooks
ansible-playbook -i inventories/inventory.yml <playbook>.yml

# Linux playbooks requiring root
ansible-playbook -i inventories/inventory.yml optimize_zade_network.yml --ask-become-pass

# z/OS playbooks
ansible-playbook -i inventories/inventory.yml <playbook>.yml

# Run only specific tags (e.g. backup_z32a_volumes.yml)
ansible-playbook -i inventories/inventory.yml backup_z32a_volumes.yml --tags backup,verify

# Override variables at runtime
ansible-playbook -i inventories/inventory.yml provision_volumes.yml -e "dest_dir=/custom/path"
```

Install required z/OS collections before running z/OS playbooks:

```sh
ansible-galaxy collection install ibm.ibm_zos_core ibm.ibm_zosmf
```

## Architecture

The repo is split into two independent subtrees — each has its own `inventories/` directory and is run independently:

- **`ansible-playbooks-linux/`** — targets `nuc` (an ASUS NUC running RHEL, `ansible_host: localhost`). Manages zADE disk volumes and host network tuning.
- **`ansible-playbooks-zos/`** — targets z/OS LPARs (`z31c_s0w1`, `z31a_s0w1`). Uses `ibm.ibm_zos_core` and `ibm.ibm_zosmf` collections.

### z/OS environment

`ansible-playbooks-zos/inventories/group_vars/all.yml` sets the Python interpreter, ZOAU paths, and all required z/OS environment variables (`_BPXK_AUTOCVT`, `_CEE_RUNOPTS`, EBCDIC encoding tags, etc.) for every play. Any new z/OS playbook inherits these automatically via group_vars.

## Key Patterns and Constraints

**`changed_when: false` on ethtool commands** — `ethtool` exits 0 whether or not a value was actually changed; `changed_when: false` is intentional and correct, not a gap. Unsupported offload options are logged and skipped (`ignore_errors: true`) rather than failing the play.

**IRQ affinity uses `smp_affinity_list` (decimal CPU numbers), not `smp_affinity` (hex bitmask)** — MSI-X IRQs on the IGC driver are single-CPU constrained; a multi-CPU bitmask is rejected with EINVAL. CPUs 0–1 are intentionally left free for zADE engine threads; NIC IRQs are pinned to CPUs 2–3.

**Idempotency via `args: creates:`** — `provision_volumes.yml` uses `creates: "{{ dest_dir }}/{{ item }}"` on the `alcckd` commands so existing volumes are skipped without a separate stat check.

**Async backup** — `backup_z32a_volumes.yml` runs `tar` with `async`/`poll: 0` then polls with `async_status` to avoid the 2-hour SSH timeout. The backup is skipped if an archive for the current timestamp already exists.

**`make_persistent: false` default** — network tuning in `optimize_zade_network.yml` is ephemeral by default (lost on reboot). Set `-e "make_persistent=true"` to persist via network scripts.
