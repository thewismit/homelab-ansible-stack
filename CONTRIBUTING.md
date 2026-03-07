# Contributing

## Development Setup

```bash
git clone https://github.com/opscairn/homelab-ansible-stack && cd homelab-ansible-stack
cp ansible.cfg.example ansible.cfg
cp inventory.yml.example inventory.yml   # use test hosts
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml
```

## Testing a Playbook

Always run `--check` before applying:

```bash
ansible-playbook playbooks/<playbook>.yml --check --limit <one-host>
ansible-playbook playbooks/<playbook>.yml --limit <one-host>
```

## Playbook Conventions

- Task names follow `"SECTION | What this does"` pattern
- All playbooks must be idempotent (safe to re-run)
- Every playbook ends with a VALIDATE task
- Use `--check` guards for destructive operations: `when: not ansible_check_mode`
- Tags: `deploy`, `configure`, `validate`

## Adding a New Playbook

1. Add to `playbooks/` (general) or `remediation/` (targeted incident response)
2. Replace all hardcoded IPs/usernames with variables from `group_vars/all/vars.yml`
3. Add the playbook to the README table
4. Verify zero leaks: `grep -r "your-ip\|your-username" .`

## Secrets

- Never commit secrets. All sensitive values belong in `vault.yml`.
- Add new vault vars with `vault_` prefix and document them in `vault.yml.example`.

## Pull Requests

- One feature/fix per PR
- Include `--check` output in PR description if modifying existing playbooks
- All task names must follow the naming convention
