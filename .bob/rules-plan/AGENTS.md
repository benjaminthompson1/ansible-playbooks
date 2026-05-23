# Plan Mode Rules (Non-Obvious Only)

## Architectural Constraints

### Dual-Directory Independence
- `ansible-playbooks-linux/` and `ansible-playbooks-zos/` are architecturally isolated
- Separate inventories mean no shared host variables or cross-directory task execution
- This is intentional — different target systems (localhost Linux vs remote z/OS)

### z/OS File Tag Architecture
- File tagging is NOT metadata — it controls runtime encoding interpretation
- IBM-1047 tagged files are read as EBCDIC by shell/utilities
- UTF-8 tagged files are read as UTF-8 by Java/Python parsers
- Mixing tags causes silent data corruption, not errors

### User Onboarding State Machine
- `setup_runner_user.yml` creates foundational state (`.profile`, directories)
- `setup_dbb_user.yml` depends on that state existing and being correct
- No rollback mechanism exists — failed state requires manual cleanup
- This is a deliberate simplification for idempotent re-runs

### Environment Variable Propagation
- `zos_environment` dict must be explicitly passed to each play
- No global environment inheritance exists in Ansible
- This prevents accidental environment pollution across plays

### Async Operation Design
- `async` with `poll: 0` decouples task submission from completion
- Allows playbook to continue while long operations run
- Status checking via job ID is manual — no automatic timeout handling
- Used for operations that exceed Ansible's default SSH timeout (10 minutes)

### Network Tuning Persistence Gap
- All tuning is ephemeral by design (survives until reboot)
- `make_persistent` flag is a placeholder for unimplemented feature
- This allows safe experimentation without permanent system changes
- Production use requires manual persistence via systemd or network scripts

### Validation Strategy
- Smoke tests are embedded in playbooks, not external test suites
- This couples validation with implementation but simplifies execution
- No CI/CD integration exists — validation is manual playbook execution
- Tags provide granular control over which validation steps run