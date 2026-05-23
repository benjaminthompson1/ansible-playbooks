# Advanced Mode Rules (Non-Obvious Only)

## File Encoding Requirements
- z/OS `.profile` copies MUST use `cp -p` to preserve IBM-1047 tag (NOT Ansible `copy` module)
- DBB YAML files require explicit UTF-8 tagging after deployment: `chtag -tc 1208 <file>`
- Use `zos_copy` module with `encoding` parameters for z/OS targets, never raw `copy`
- Avoid `slurp` module on tagged z/OS files — use `command: cat` or `shell` with AUTOCVT

## Playbook Dependencies
- `setup_dbb_user.yml` has hard dependency on `setup_runner_user.yml` being run first
- Fail-fast assertion checks for `.profile` existence before proceeding
- Both playbooks require `BPX.SUPERUSER` or UID 0 for chown operations

## Template Patterns
- `Languages.yaml.j2` uses `dictsort` filter to alphabetize keys for stable diffs
- This prevents spurious changes when dict iteration order varies
- Template output is tagged UTF-8 after generation for DBB Java parser compatibility

## Async Task Patterns
- Long-running operations use `async` with `poll: 0` (fire-and-forget)
- Status checking happens in separate subsequent tasks, not inline
- Example: `backup_z32a_volumes.yml` uses `async: 7200` for 2-hour tar operations

## Environment Variable Application
- `environment: "{{ zos_environment }}"` is NOT automatically applied
- Must be explicitly declared at play level for z/OS tasks
- Dict name `zos_environment` is conventional, not magic — defined in `group_vars/all.yml`

## Error Handling Conventions
- Network offload disabling uses `ignore_errors: true` — unsupported offloads are logged, not failed
- This is intentional behavior for hardware compatibility variations
- IRQ pinning targets specific CPUs (2-3) to avoid zADE engine thread interference

## Browser/MCP Tool Access
- Advanced mode has access to browser_action and MCP tools
- Use for testing z/OSMF web interfaces or external documentation lookups
- Browser can verify z/OS system UUID posting to z/OSMF (port 10443)