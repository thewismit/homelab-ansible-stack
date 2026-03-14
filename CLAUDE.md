# Ansible Public Collection

Public Ansible collection for homelab infrastructure automation.

## File Structure

- roles/ — 4 roles: common, restic, restic_backup, server_hardening (each with Molecule tests)
- playbooks/ — 7 deployment playbooks (deploy-restic-fleet, deploy-node-exporter, deploy-loki, etc.)
- remediation/ — 6 auto-fix playbooks (restart-container, cleanup-disk, caddy-reload, etc.)
- templates/ — Jinja2 templates for alert rules and promtail config
- .github/workflows/ci.yml — CI: syntax check + ansible-lint + Molecule matrix

## Commands

- Syntax check: `ansible-playbook playbooks/<name>.yml --syntax-check -i inventory.yml`
- Lint: `ansible-lint playbooks/ remediation/ roles/`
- Test role: `cd roles/<role> && molecule test`
- Deploy: `ansible-playbook playbooks/<name>.yml --check --limit <host>` then without --check

## Conventions

- kebab-case for playbooks/roles, snake_case for tasks/vars
- All tasks idempotent, safe for --check mode
- Variables in defaults/main.yml per role, shared in group_vars/all
- Handlers for service restarts via notify:
- Tags on all tasks (audit, snapshot, patch, rollback)
- Molecule + Docker for role testing
