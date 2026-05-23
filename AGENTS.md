# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Critical Non-Obvious Patterns

### Directory-Specific Command Execution
- Commands MUST be run from within the relevant subdirectory (`ansible-playbooks-linux/` or `ansible-playbooks-zos/`)
- Each subdirectory has its own `inventories/inventory.yml` — they are NOT interchangeable
- `ansible-playbooks-zos/` has `ansible.cfg` with `scp_extra_args = -O` (required for z/OS SSH compatibility)

### z/OS File Encoding (Critical)
- `.profile` files MUST preserve IBM-1047 tag — use `cp -p` not `copy` module
- DBB YAML files MUST be tagged UTF-8 after copy: `chtag -tc 1208 <file>`
- Use `zos_copy` with `encoding` params, NOT raw `copy` for z/OS targets
- NEVER use `slurp` on tagged z/OS files — use `command: cat` or `shell` with AUTOCVT

### User Onboarding Sequence (z/OS)
- ALWAYS run `setup_runner_user.yml` BEFORE `setup_dbb_user.yml`
- `setup_dbb_user.yml` will fail fast if runner setup incomplete
- Both require connecting user to have `BPX.SUPERUSER` or UID 0 for chown operations

### Environment Variables
- z/OS tasks reference `environment: "{{ zos_environment }}"` — this is NOT automatic
- The dict name `zos_environment` in `group_vars/all.yml` is conventional, not magic
- Contains 10 env vars including `_BPXK_AUTOCVT: "ON"` and ZOAU paths

### Template Behavior
- `Languages.yaml.j2` sorts `dbb_languages_datasets` alphabetically for stable diffs
- This is intentional — prevents spurious changes when dict order varies

### Async Operations
- `backup_z32a_volumes.yml` uses `async: 7200` with `poll: 0` for 2-hour tar operations
- Status checking happens in subsequent tasks, not inline

### Network Tuning Idiosyncrasies
- `optimize_zade_network.yml` disables "rx" offload (RX checksum, not RX path itself)
- Unsupported offloads are logged and ignored, not failed — this is intentional
- IRQ pinning targets CPUs 2-3 specifically (leaves 0-1 for zADE engine threads)
- All tuning is ephemeral by default — `make_persistent` flag exists but not implemented

### Validation Pattern
- Playbooks use built-in smoke-test assertion tasks, NOT external test frameworks
- No Makefile, tox, or molecule setup exists
- Tags like `verify`, `smoketest` control validation steps

## Commands from CLAUDE.md

```bash
# Standard run (from subdirectory)
ansible-playbook -i inventories/inventory.yml <playbook>.yml

# User onboarding with extra vars
ansible-playbook -i inventories/inventory.yml setup_runner_user.yml -e "target_user=newuser"
ansible-playbook -i inventories/inventory.yml setup_dbb_user.yml -e "target_user=newuser"

# Privilege escalation (optimize_zade_network.yml only)
ansible-playbook -i inventories/inventory.yml optimize_zade_network.yml --ask-become-pass

# Tag-based execution
ansible-playbook -i inventories/inventory.yml backup_z32a_volumes.yml --tags "verify,backup"