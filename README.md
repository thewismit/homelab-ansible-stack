# Ops Kernel Stack

[![CI](https://github.com/opskern/ops-kernel-stack/actions/workflows/ci.yml/badge.svg)](https://github.com/opskern/ops-kernel-stack/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Ansible 2.14+](https://img.shields.io/badge/Ansible-2.14%2B-EE0000?logo=ansible)](https://docs.ansible.com/)

> Your homelab breaks at 2am. This stack fixes it before you wake up.

From [OpsKern](https://opskern.io) — the companion stack for the book [*Self-Healing Infrastructure*](https://leanpub.com/self-healing-infrastructure).

**[Live status dashboard](https://status.opskern.io)** (pending DNS)

---

## Why this exists

Most homelab guides show you how to set something up. None of them show you how to keep it running.

The centerpiece is the **remediation bridge**: Alertmanager fires, a Python webhook dispatcher picks it up, the right Ansible playbook runs automatically, and ntfy notifies you that your homelab fixed itself. No equivalent exists in the public open-source ecosystem.

```
Prometheus ──alerts──▶ Alertmanager ──webhook──▶ Remediation Bridge
                                                        │
                                            ansible-playbook runs
                                                        │
                                              ntfy: "auto-fixed"
```

---

## What's inside

### Deployment playbooks

| Playbook | What it does |
|----------|-------------|
| `deploy-restic-fleet.yml` | Fleet-wide encrypted Restic backups with systemd timers, prune, and verify |
| `deploy-node-exporter.yml` | Prometheus metrics agent on every host |
| `deploy-loki.yml` | Loki log aggregation backend (Docker Compose, NFS storage) |
| `deploy-promtail-fleet.yml` | Promtail log shipper on every host, ships to Loki |
| `deploy-remediation-bridge.yml` | Alertmanager webhook receiver that dispatches Ansible playbooks |
| `deploy-alert-rules.yml` | 23 parameterized Prometheus alert rules across 7 categories |
| `fleet-patch.yml` | 5-stage safe patching pipeline: audit, snapshot, patch, smoke test, rollback |

### Remediation playbooks

Called automatically by the bridge when alerts fire. You can also run them manually.

| Playbook | Triggered by |
|----------|-------------|
| `remediation/restart-container.yml` | `ContainerDown`, `ContainerOOM` |
| `remediation/service-restart.yml` | `SystemdServiceFailed`, `HighMemoryUsage`, `DNSResolutionFailed` |
| `remediation/cleanup-disk.yml` | `DiskSpaceWarning`, `DiskSpaceCritical` |
| `remediation/caddy-reload.yml` | `TLSCertExpiringSoon`, `TLSCertExpiringCritical`, `TLSCertExpired`, `TLSHandshakeFailed` |
| `remediation/host-recovery.yml` | `HostDown` — Proxmox VM/LXC auto-recovery |
| `remediation/remount-nfs.yml` | `NFSMountUnavailable` |

### Roles

| Role | Purpose |
|------|---------|
| `roles/common` | Passwordless sudo for the Ansible user |
| `roles/restic` | Restic install, repo init, backup + prune |
| `roles/restic_backup` | Systemd timer/service units for scheduled backups |
| `roles/server_hardening` | SSH hardening, UFW firewall, sysctl tuning |

### Templates

| Template | Used by |
|----------|---------|
| `templates/alert-rules.yml.j2` | `deploy-alert-rules.yml` — all 23 alert rules, parameterized |
| `templates/promtail-config.yml.j2` | `deploy-promtail-fleet.yml` — per-host Promtail config |

---

## Features

**Self-healing** — alerts trigger automated remediation. Container down? Restarted. Disk full? Cleaned. TLS cert expiring? Caddy reloaded. Host unreachable? Proxmox recovers the VM.

**Fleet monitoring** — Prometheus, Loki, Grafana, and Promtail deployed across every host. 23 alert rules cover host health, containers, TLS, backups, infrastructure, network, and systemd services.

**Automated backups** — Restic with systemd timers. Encrypted, deduplicated, SFTP to NAS. Prune and verify built in.

**Safe update pipeline** — 5-stage patching: dry-run audit, Proxmox snapshot, apt safe-upgrade with service restarts, post-patch smoke tests, one-command rollback. Each stage is tag-gated.

**TLS management** — Caddy-based cert lifecycle. Alert rules catch expiring certs at 30, 14, and 0 days. Remediation reloads Caddy automatically.

**Server hardening** — SSH lockdown, UFW firewall, sysctl tuning. One role, applied fleet-wide.

---

## How it compares

| | This stack | ansible-nas | Jeff Geerling roles | ironicbadger/infra |
|--|:----------:|:-----------:|:-------------------:|:-----------------:|
| Restic backups | yes | no | no | partial |
| PLG monitoring (Prometheus + Loki + Grafana) | yes | no | no | no |
| Fleet-wide Promtail | yes | no | no | no |
| Automated self-healing | yes | no | no | no |
| Multi-host support | yes | no | varies | no |
| Safe update pipeline | yes | no | no | no |
| TLS cert lifecycle | yes | no | no | no |
| Reusable / portable | yes | partial | yes | no |
| `--check` mode safe | yes | varies | varies | varies |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Control Node (ansible-host)                                │
│  ├── inventory.yml        ← hosts + groups                  │
│  ├── group_vars/all/      ← shared vars + vault             │
│  ├── playbooks/           ← deployment playbooks            │
│  └── remediation/         ← targeted fix playbooks          │
└─────────────────────────────────────────────────────────────┘
         │ SSH + become (all ops via Ansible — no direct edits)
         ▼
┌─────────────────────────────────────────────────────────────┐
│  backup_servers group (each host gets):                     │
│  ├── Restic systemd timer (daily backup → NAS/PBS)          │
│  └── Promtail (ships logs → Loki on docker-host)            │
├─────────────────────────────────────────────────────────────┤
│  docker-host:                                               │
│  ├── Loki (log store, 90-day retention)                     │
│  ├── Prometheus (scrapes node_exporter fleet-wide)          │
│  ├── Grafana (dashboards + Loki datasource auto-provisioned)│
│  ├── Alertmanager (routes alerts to bridge + ntfy)          │
│  └── Remediation Bridge :9999 (webhook → playbook dispatch) │
├─────────────────────────────────────────────────────────────┤
│  Storage (NAS):                                             │
│  └── Restic repos (encrypted, deduplicated, SFTP/NFS)       │
└─────────────────────────────────────────────────────────────┘
```

---

## Quick start

```bash
# 1. Clone and configure
git clone https://github.com/opskern/ops-kernel-stack ansible-homelab
cd ansible-homelab

cp ansible.cfg.example ansible.cfg
cp inventory.yml.example inventory.yml
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml

# 2. Edit your environment
#    inventory.yml          → your host IPs
#    group_vars/all/vars.yml → homelab_domain, docker_host_ip, ntfy_url, etc.

# 3. Test connectivity
ansible all -m ping

# 4. Deploy backups (always --check first)
ansible-playbook playbooks/deploy-restic-fleet.yml --check --limit <one-host>
ansible-playbook playbooks/deploy-restic-fleet.yml

# 5. Deploy monitoring
ansible-playbook playbooks/deploy-node-exporter.yml
ansible-playbook playbooks/deploy-loki.yml --limit docker-host
ansible-playbook playbooks/deploy-promtail-fleet.yml

# 6. Deploy self-healing
ansible-playbook playbooks/deploy-alert-rules.yml --limit docker-host
ansible-playbook playbooks/deploy-remediation-bridge.yml --limit ansible-host
```

---

## Alert rules

23 Prometheus alert rules across 7 categories, all parameterized via `group_vars`:

| Category | Alerts |
|----------|--------|
| **Host health** | `HostDown`, `DiskSpaceWarning`, `DiskSpaceCritical`, `HighMemoryUsage`, `EnabledServiceInactive` |
| **Container** | `ContainerDown`, `ContainerOOM`, `ContainerRestartLoop` |
| **TLS** | `TLSCertExpiringSoon`, `TLSCertExpiringCritical`, `TLSCertExpired`, `TLSProbeFailure`, `TLSHandshakeFailed` |
| **Backup** | `ResticBackupStale`, `ResticBackupFailed` |
| **Infrastructure** | `NFSMountUnavailable`, `HighSwapUsage`, `HighLoadAverage`, `HighDiskIOWait` |
| **Network** | `HTTPEndpointDown`, `DNSResolutionFailed` |
| **Services** | `SystemdServiceFailed`, `SystemdTimerFailed` |

---

## Fleet patching

Each stage is tag-gated. Nothing runs unless you ask for it.

| Stage | Tag | What it does |
|-------|-----|-------------|
| Audit | `audit` | Dry-run `apt upgrade`, map packages to running services |
| Snapshot | `snapshot` | Proxmox VM/LXC snapshot before patching |
| Patch | `patch` | `apt safe` upgrade + automatic service restarts |
| Smoke | `smoke` | Post-patch service health validation |
| Rollback | `rollback` | Revert to pre-patch Proxmox snapshot |

```bash
# Audit all hosts
ansible-playbook playbooks/fleet-patch.yml --tags audit

# Full patch cycle on one host
ansible-playbook playbooks/fleet-patch.yml --tags snapshot,patch,smoke --limit wiki-host

# Rollback if something broke
ansible-playbook playbooks/fleet-patch.yml --tags rollback -e target_host=wiki-host -e rollback_date=20260312
```

Requires `playbooks/vars/fleet-patch-vars.yml` — see `fleet-patch-vars.yml.example`.

---

## Prerequisites

- **Ansible 2.14+** on the control node
- **Python 3.9+** on the control node
- SSH key-based access to all managed hosts
- `ansible-vault` for secret management
- An NFS or SFTP-accessible backup target for Restic

---

## Variable reference

All key variables live in `group_vars/all/vars.yml`:

| Variable | Description |
|----------|-------------|
| `homelab_domain` | Your domain (e.g. `example.com`) |
| `ansible_user` | Ansible SSH user (set in inventory) |
| `docker_host_ip` | IP of your Docker/container host |
| `dns_server_ip` | Primary DNS server IP |
| `gateway_ip` | Default gateway IP |
| `pbs_host_ip` | Proxmox Backup Server IP |
| `synology_ip` | NAS IP (Restic SFTP target) |
| `loki_url` | Loki URL (auto-derived from `docker_host_ip`) |
| `ntfy_url` | ntfy notification server URL |
| `ntfy_topic` | ntfy topic name for alerts |
| `restic_sftp_repo` | Full Restic SFTP repository path |
| `vault_pass_file` | Path to ansible-vault password file on control node |

Secrets go in `group_vars/all/vault.yml` (see `vault.yml.example`).

---

## Design decisions

**No hardcoded values.** Every IP, hostname, username, and domain is a variable. `grep -r "192.168" .` returns zero results.

**`--check` safe.** Every playbook runs cleanly with `--check`. Network-touching and stateful tasks are guarded with `when: not ansible_check_mode`.

**Tagged for surgical runs.** All tasks are tagged `deploy`, `configure`, or `validate`:
```bash
ansible-playbook playbooks/deploy-loki.yml --tags configure    # re-push config only
ansible-playbook playbooks/deploy-restic-fleet.yml --tags validate  # check status only
```

**Idempotent.** Safe to re-run. State is declared, not imperative.

---

## Testing

```bash
pip install -r requirements.txt

# Syntax-check all playbooks
for pb in playbooks/*.yml; do ansible-playbook "$pb" --syntax-check -i inventory.yml.example; done

# Molecule tests (requires Docker)
cd roles/common && molecule test
cd roles/restic && molecule test
cd roles/restic_backup && molecule test
```

CI runs syntax-check, ansible-lint, and Molecule tests on every push via GitHub Actions.

---

## Remediation bridge deep-dive

See [`playbooks/REMEDIATION-BRIDGE.md`](playbooks/REMEDIATION-BRIDGE.md) for the full architecture, Alertmanager webhook config, how to add new remediation mappings, and cooldown behavior.

---

## Security

See [`SECURITY.md`](SECURITY.md) for the security policy. The short version:

- All secrets encrypted via `ansible-vault`
- SSH key-only access; `become` for privilege escalation
- No passwords in inventory, playbooks, or git history

---

## The book

This stack is the reference implementation for [*Self-Healing Infrastructure*](https://leanpub.com/self-healing-infrastructure). The book walks through building this from scratch — why each piece exists, what broke before it did, and how to adapt it to your own setup.

---

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md).

---

## License

MIT — see [`LICENSE`](LICENSE).
