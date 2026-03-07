# Homelab Ansible Stack

[![CI](https://github.com/opscairn/homelab-ansible-stack/actions/workflows/ci.yml/badge.svg)](https://github.com/opscairn/homelab-ansible-stack/actions/workflows/ci.yml)

> **Production-grade Ansible automation for self-hosted homelabs.**
> Backups, monitoring, logging, and self-healing infrastructure — battle-tested, running 24/7.

---

## The pitch

Most homelab guides show you how to set something up. This shows you how to **keep it running**.

The centerpiece is the **remediation bridge**: Alertmanager fires → a Python webhook dispatcher
picks it up → the right Ansible playbook runs automatically → ntfy notifies you that your homelab
fixed itself. No equivalent exists in the public open-source ecosystem.

The rest of the stack is the pipeline that makes this possible:

```
Prometheus ──alerts──▶ Alertmanager ──webhook──▶ Remediation Bridge
                                                          │
                                              ansible-playbook runs
                                                          │
                                            ✅ ntfy: "auto-fixed"
```

---

## What's included

| Component | What it does | Playbook |
|-----------|-------------|---------|
| **Restic backups** | Fleet-wide encrypted backups, systemd timers, prune + verify | `deploy-restic-fleet.yml` |
| **node_exporter** | Prometheus metrics agent on every host | `deploy-node-exporter.yml` |
| **Loki** | Log aggregation backend (Docker Compose, NFS storage) | `deploy-loki.yml` |
| **Promtail** | Log shipper on every host → Loki | `deploy-promtail-fleet.yml` |
| **Remediation bridge** | Alertmanager → Ansible auto-remediation | `deploy-remediation-bridge.yml` |

**4 targeted remediation playbooks** — called automatically by the bridge or manually:

| Playbook | When it runs |
|---------|-------------|
| `remediation/restart-container.yml` | `ContainerDown` alert |
| `remediation/service-restart.yml` | `SystemdServiceFailed` alert |
| `remediation/cleanup-disk.yml` | `DiskSpaceHigh` alert |
| `remediation/caddy-reload.yml` | `TLSCertExpiringSoon` alert |

---

## How it compares

| | This stack | ansible-nas | Jeff Geerling roles | ironicbadger/infra |
|--|:----------:|:-----------:|:-------------------:|:-----------------:|
| Restic backups | ✅ | ❌ | ❌ | partial |
| PLG monitoring (Prometheus + Loki + Grafana) | ✅ | ❌ | ❌ | ❌ |
| Fleet-wide Promtail | ✅ | ❌ | ❌ | ❌ |
| Automated self-healing | ✅ | ❌ | ❌ | ❌ |
| Multi-host support | ✅ | ❌ | varies | ❌ |
| Reusable / portable | ✅ | partial | ✅ | ❌ |
| `--check` mode safe | ✅ | varies | varies | varies |

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
│  ├── Loki 3.3.2 (log store, 90-day retention)               │
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

## Quickstart

```bash
# 1. Clone and configure
git clone https://github.com/opscairn/homelab-ansible-stack ansible-homelab && cd ansible-homelab
cp ansible.cfg.example ansible.cfg         # set your SSH key + ansible user paths
cp inventory.yml.example inventory.yml     # fill in your host IPs
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml  # add real secrets

# 2. Set your environment in group_vars/all/vars.yml:
#    homelab_domain, docker_host_ip, dns_server_ip, pbs_host_ip, ntfy_url, etc.

# 3. Validate connectivity
ansible all -m ping

# 4. Deploy — always --check first, then real run
ansible-playbook playbooks/deploy-restic-fleet.yml --check --limit <one-host>
ansible-playbook playbooks/deploy-restic-fleet.yml

# 5. Deploy monitoring stack
ansible-playbook playbooks/deploy-node-exporter.yml
ansible-playbook playbooks/deploy-loki.yml --limit docker-host
ansible-playbook playbooks/deploy-promtail-fleet.yml

# 6. Deploy the remediation bridge
ansible-playbook playbooks/deploy-remediation-bridge.yml --limit ansible-host
```

---

## Prerequisites

- **Ansible 2.14+** on the control node
- **Python 3.9+** on the control node
- SSH key-based access to all managed hosts
- `ansible-vault` for secret management
- An NFS or SFTP-accessible backup target for Restic

---

## Key design decisions

**No hardcoded values.** Every IP address, hostname, username, and domain name is a
variable. `grep -r "192.168" .` returns zero results. Adapt to any network.

**`--check` safe.** Every playbook runs cleanly with `--check` — network-touching
and stateful tasks are guarded with `when: not ansible_check_mode`.

**Tagged for surgical runs.** All tasks are tagged `deploy`, `configure`, or `validate`:
```bash
ansible-playbook playbooks/deploy-loki.yml --tags configure  # re-push config only
ansible-playbook playbooks/deploy-restic-fleet.yml --tags validate  # check status only
```

**Idempotent.** Safe to re-run. State is declared, not imperative.

---

## Variable Reference

All key variables are in `group_vars/all/vars.yml`:

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

Secrets go in `group_vars/all/vault.yml` (see `vault.yml.example` for all required keys).

---

## Roles

| Role | Purpose | README |
|------|---------|--------|
| `roles/common` | Passwordless sudo for ansible_user | [README](roles/common/README.md) |
| `roles/restic` | Restic install, repo init, backup + prune | [README](roles/restic/README.md) |
| `roles/restic_backup` | Systemd timer/service units for scheduled backups | [README](roles/restic_backup/README.md) |

---

## Testing

```bash
# Install test dependencies
pip install -r requirements.txt

# Syntax-check all playbooks
for pb in playbooks/*.yml; do ansible-playbook "$pb" --syntax-check -i inventory.yml.example; done

# Run Molecule tests for any role (requires Docker)
cd roles/common && molecule test
cd roles/restic && molecule test
cd roles/restic_backup && molecule test
```

CI runs syntax-check, ansible-lint, and Molecule tests (all 3 roles) on every push via GitHub Actions.

---

## Remediation bridge deep-dive

See [`playbooks/REMEDIATION-BRIDGE.md`](playbooks/REMEDIATION-BRIDGE.md) for:
- Full architecture diagram
- Alertmanager webhook configuration
- How to add a new remediation mapping
- Cooldown behavior and ntfy notifications
- Monitoring the bridge service

---

## Security Notes

- All secrets encrypted via `ansible-vault` — no plaintext passwords anywhere
- SSH key-only access; `become` for privilege escalation
- No passwords in inventory, playbooks, or git history
- Rotate vault password every 6 months: `ansible-vault rekey group_vars/all/vault.yml`

---

## License

MIT — fork it, adapt it, use it. Attribution appreciated but not required.
