# Changelog

All notable changes to this project will be documented here.

Format: `## vX.Y.Z — YYYY-MM-DD`

---

## v2.0.0 — 2026-03-12

Major expansion: alert rules, fleet patching, and full remediation coverage.

### Added

**Alert rules**
- `playbooks/deploy-alert-rules.yml` — deploy 23 Prometheus alert rules
- `templates/alert-rules.yml.j2` — parameterized alert rules across 7 categories
- Alert threshold variables in `group_vars/all/vars.yml`

**Fleet patching**
- `playbooks/fleet-patch.yml` — 5-stage safe patching pipeline (audit, snapshot, patch, smoke, rollback)
- `playbooks/vars/fleet-patch-vars.yml.example` — host VMID mapping and smoke test template

**Remediation playbooks**
- `remediation/host-recovery.yml` — Proxmox VM/LXC auto-recovery for `HostDown` alerts
- `remediation/remount-nfs.yml` — NFS mount recovery for `NFSMountUnavailable` alerts

### Changed

- Remediation bridge map expanded: 4 → 17 alert-to-playbook mappings
- Added `label_vars` and `extra_vars` support to remediation mappings
- Added `strategy: defer_to_timer` for alerts that shouldn't auto-retry (e.g. `ResticBackupFailed`)
- README updated with alert rules, fleet patching, and expanded remediation docs

---

## v1.0.0 — 2026-02-28

Initial public release.

### Added

**Playbooks**
- `playbooks/deploy-restic-fleet.yml` — Restic backup deployment across all backup_servers
- `playbooks/deploy-node-exporter.yml` — Prometheus node_exporter fleet deployment
- `playbooks/deploy-loki.yml` — Grafana Loki log aggregation backend (Docker Compose)
- `playbooks/deploy-promtail-fleet.yml` — Promtail log shipper fleet deployment
- `playbooks/deploy-remediation-bridge.yml` — Alertmanager → Ansible automated remediation

**Remediation playbooks**
- `remediation/restart-container.yml`
- `remediation/service-restart.yml`
- `remediation/cleanup-disk.yml`
- `remediation/caddy-reload.yml`

**Roles**
- `roles/common` — passwordless sudo baseline
- `roles/restic` — Restic backup with service pausing, prune, and integrity check
- `roles/restic_backup` — systemd timer/service units for scheduled backups

**Infrastructure**
- `group_vars/all/vars.yml` — all generalizable variables (zero hardcoded IPs)
- `group_vars/all/vault.yml.example` — all required secrets documented
- `inventory.yml.example` — full inventory template with placeholder IPs
- `ansible.cfg.example` — control node configuration template
- `templates/promtail-config.yml.j2` — Promtail config template

**Documentation**
- `README.md` — quickstart, architecture, variable reference
- `CONTRIBUTING.md` — development setup, conventions, PR guidelines
- `playbooks/REMEDIATION-BRIDGE.md` — architecture, alertmanager config, ops guide
- `roles/restic/README.md` — variables, exit code 3 behavior, NFS recovery
- `roles/restic_backup/README.md` — scheduling, timer staggering, verification
- `roles/common/README.md` — usage, what it writes, extension points

**CI**
- `.github/workflows/ci.yml` — syntax-check + ansible-lint on push/PR

### Design decisions

- New clean repo rather than in-place refactor of live homelab — keeps homelab stable
- All IPs, usernames, and domain names replaced with Jinja2 variables
- All tasks tagged with `deploy`, `configure`, or `validate` for selective runs
- All playbooks safe to run with `--check` mode (check-mode guards throughout)
- `restic_backup` role declares `restic` as a dependency in meta/main.yml
