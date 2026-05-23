# Ask Mode Rules (Non-Obvious Only)

## Project Structure Counterintuitive Aspects
- Two independent subdirectories with separate inventories — NOT a unified Ansible project
- `ansible-playbooks-linux/` and `ansible-playbooks-zos/` cannot share inventory files
- Each subdirectory is its own execution context with different target systems

## z/OS File Encoding Context
- IBM-1047 (EBCDIC) vs UTF-8 tagging is critical for z/OS file operations
- `.profile` must remain IBM-1047 tagged or shell won't parse it correctly
- DBB YAML files must be UTF-8 tagged or Java parser fails silently
- This is NOT standard Unix behavior — z/OS file tags control encoding interpretation

## User Onboarding Two-Step Process
- `setup_runner_user.yml` and `setup_dbb_user.yml` are NOT independent
- Running them out of order or skipping step 1 causes cryptic failures
- The dependency is enforced by assertion checks, not Ansible role dependencies

## Environment Variable Misconception
- `zos_environment` dict in `group_vars/all.yml` looks like it auto-applies
- It does NOT — must be explicitly referenced as `environment: "{{ zos_environment }}"`
- This is a naming convention, not Ansible magic variable behavior

## Async Operations Pattern
- `async` with `poll: 0` means "start and forget" — NOT "run in parallel"
- Status checking happens in later tasks via job ID tracking
- This pattern is used for 2-hour tar operations that would timeout otherwise

## Network Tuning Ephemeral Nature
- All `optimize_zade_network.yml` changes are lost on reboot by design
- `make_persistent` flag exists but does nothing (reserved for future implementation)
- This is intentional — allows testing without permanent system changes

## Validation Approach
- No external test frameworks (pytest, molecule, etc.) are used
- Smoke tests are embedded assertion tasks within playbooks themselves
- Tags control which validation steps run, not separate test files