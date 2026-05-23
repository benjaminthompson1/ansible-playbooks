# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

Ansible automation for IBM zADE (z Application Development Environment) infrastructure. Two independent subdirectories with separate inventories:

- `ansible-playbooks-linux/` — manages an ASUS NUC (localhost) running Linux as the zADE host
- `ansible-playbooks-zos/` — manages a z/OS system (ADCD z32a) for GitLab Runner and DBB user onboarding

## Running Playbooks

Commands must be run from within the relevant subdirectory:

```bash
# Standard run
ansible-playbook -i inventories/inventory.yml <playbook>.yml

# With extra variables (e.g., user onboarding)
ansible-playbook -i inventories/inventory.yml setup_runner_user.yml -e "target_user=newuser"
ansible-playbook -i inventories/inventory.yml setup_dbb_user.yml -e "target_user=newuser"

# Privilege escalation (required for optimize_zade_network.yml)
ansible-playbook -i inventories/inventory.yml optimize_zade_network.yml --ask-become-pass

# Run only specific tags
ansible-playbook -i inventories/inventory.yml backup_z32a_volumes.yml --tags "verify,backup"
```

There is no Makefile, tox, or molecule setup. Correctness is validated by smoke-test assertion tasks built into each playbook.

## Architecture

### Linux Playbooks (`ansible-playbooks-linux/`)

Target host: `nuc` (localhost, user `ibmsys1`, SSH key `~/.ssh/nuc_ed25519`)

| Playbook | Purpose |
|---|---|
| `backup_z32a_volumes.yml` | Compressed tar.xz backups of z32a volumes with 4-backup retention rotation |
| `optimize_zade_network.yml` | NIC tuning — disables offloads, sets ring buffers/coalescing, pins IRQs to CPUs 2-3, applies TCP sysctl |
| `provision_volumes.yml` | Creates CKD disk volumes via `alcckd` (idempotent; skips existing volumes) |

### z/OS Playbooks (`ansible-playbooks-zos/`)

Target host: `z32a` (10.1.1.2, port 65522, user `ibmuser`)

| Playbook | Purpose |
|---|---|
| `zos_ping.yml` | Connectivity test using `ibm.ibm_zos_core.zos_ping` |
| `post_uuid.yml` | POSTs z/OS UUID to z/OSMF (port 10443) |
| `setup_runner_user.yml` | **Step 1**: copies `.profile`, creates `~/gitlab-runner/{builds,cache}` dirs |
| `setup_dbb_user.yml` | **Step 2**: adds DBB v3 working files; asserts step 1 was run first |

**User onboarding** always runs `setup_runner_user.yml` before `setup_dbb_user.yml`. The `setup_dbb_user.yml` playbook will fail fast if the runner-user setup is incomplete.

### Key Variable Files

`ansible-playbooks-zos/inventories/group_vars/all.yml` defines:
- `zos_environment` dict — 10 env vars (AUTOCVT, ZOAU paths, PYTHONPATH, LIBPATH, etc.) applied to all z/OS tasks
- `gitlab_runner` dict — build/cache subdirectory names, zopen config path
- `dbb` dict — DBB home, template dirs, workspace paths
- `dbb_languages_datasets` — 15 dataset names mapping to COBOL/LE/CICS/HLASM compiler datasets

### Jinja2 Template

`ansible-playbooks-zos/Languages.yaml.j2` — templates `dbb_languages_datasets` variables into `/u/<user>/zBuilder/build/Languages.yaml`. Keys are sorted alphabetically for stable diffs. Tagged UTF-8 after deployment so DBB's Java reader can parse it correctly.

### z/OS File Encoding

z/OS tasks frequently deal with EBCDIC/ASCII encoding. Key conventions:
- `.profile` is copied with IBM-1047 file tag preserved
- YAML files for DBB are tagged UTF-8 after copy (`chtag -tc 1208`)
- Use `zos_copy` with appropriate `encoding` parameters rather than raw `copy` for z/OS targets
- Avoid `slurp` on tagged z/OS files; use `command: cat` or `shell` with appropriate AUTOCVT settings

### Collections Used

- `ibm.ibm_zos_core` — `zos_ping`, `zos_copy`, `zos_file_copy`
- `ibm.ibm_zosmf` — `zmf_swmgmt_zos_system_uuid` role (included dynamically in `post_uuid.yml`)

No custom roles are defined; all logic lives in playbook task files.

## Common Variables

| Variable | Scope | Default | Description |
|---|---|---|---|
| `target_user` | z/OS onboarding | required | USS username to onboard |
| `boolean_debug` | z/OS | `false` | Enable verbose debug output |
| `network_interface` | Linux | `enp86s0` | NIC to tune |
| `keep_backups` | Linux | `4` | Backup retention count |
