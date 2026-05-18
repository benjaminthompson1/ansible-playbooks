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
- **`ansible-playbooks-zos/`** — targets ADCD z32a (`z32a`, z/OS 3.2). Uses `ibm.ibm_zos_core` and `ibm.ibm_zosmf` collections.

### z/OS environment

`ansible-playbooks-zos/inventories/group_vars/all.yml` sets the Python interpreter, ZOAU paths, and all required z/OS environment variables (`_BPXK_AUTOCVT`, `_CEE_RUNOPTS`, EBCDIC encoding tags, etc.) for every play. Any new z/OS playbook inherits these automatically via group_vars.

### z/OS user onboarding playbooks

Two-step onboarding for new z/OS USS users that need to run GitLab Runner workloads with DBB. **Order matters** — run them in this sequence:

1. **`setup_runner_user.yml`** — copies IBMUSER's customised `.profile` as the canonical template (preserves the IBM-1047 file tag), then creates `~/gitlab-runner/{builds,cache}`.
2. **`setup_dbb_user.yml`** — creates writable `~/zBuilder/build/`, copies DBB samples from `$DBB_HOME`, templates `Languages.yaml` from `dbb_languages_datasets` in `group_vars/all.yml`, tags it UTF-8.

`setup_dbb_user.yml` asserts the target user's `.profile` already exists and fails fast otherwise. Both connect as IBMUSER (needs `BPX.SUPERUSER` or UID 0 to chown into another user's home). Both require `-e "target_user=<userid>"` and assume the user has already been created with an OMVS segment via RACF `ADDUSER`.

```sh
ansible-playbook -i inventories/inventory.yml setup_runner_user.yml -e "target_user=seb"
ansible-playbook -i inventories/inventory.yml setup_dbb_user.yml    -e "target_user=seb"
```

## Key Patterns and Constraints

**`changed_when: false` on ethtool commands** — `ethtool` exits 0 whether or not a value was actually changed; `changed_when: false` is intentional and correct, not a gap. Unsupported offload options are logged and skipped (`ignore_errors: true`) rather than failing the play.

**IRQ affinity uses `smp_affinity_list` (decimal CPU numbers), not `smp_affinity` (hex bitmask)** — MSI-X IRQs on the IGC driver are single-CPU constrained; a multi-CPU bitmask is rejected with EINVAL. CPUs 0–1 are intentionally left free for zADE engine threads; NIC IRQs are pinned to CPUs 2–3.

**Idempotency via `args: creates:`** — `provision_volumes.yml` uses `creates: "{{ dest_dir }}/{{ item }}"` on the `alcckd` commands so existing volumes are skipped without a separate stat check.

**Async backup** — `backup_z32a_volumes.yml` runs `tar` with `async`/`poll: 0` then polls with `async_status` to avoid the 2-hour SSH timeout. The backup is skipped if an archive for the current timestamp already exists.

**`make_persistent: false` default** — network tuning in `optimize_zade_network.yml` is ephemeral by default (lost on reboot). Set `-e "make_persistent=true"` to persist via network scripts.

**Template-copy onboarding, not blockinfile patching** — `setup_runner_user.yml` copies `/u/ibmuser/.profile` wholesale via `cp -p` rather than editing the new user's `.profile` line-by-line. IBMUSER's `.profile` is the canonical lab template (DBB exports + zopen-config sourced at column 0). `cp -p` preserves the IBM-1047 file tag; the playbook re-applies `chtag` defensively. Re-running is safe — `creates:` skips the copy if the target user already has a `.profile`.

**z/OS file tag traps** — DBB's Java reads `Languages.yaml` via the file tag, so it must be `UTF-8`. `.profile` must be `IBM-1047` because **bash** reads it natively. Get either wrong and the runner shell silently mangles its own startup files. The onboarding playbooks apply `chtag` defensively after every copy/template.

**Argv encoding on z/OS** — Ansible's Python on z/OS encodes argv strings as IBM-1047 when invoking the shell. This means `grep <pattern>` against a UTF-8-tagged file silently returns no matches (pattern is IBM-1047 bytes, file is ASCII/UTF-8 bytes). For UTF-8 content verification, use `slurp` + Jinja on the controller, not `grep` on the target — or skip the post-hoc check entirely if the templating module's own success status already proves what you need.
