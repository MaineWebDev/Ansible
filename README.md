# homelab-ansible

Ansible automation for my personal homelab environment. This repository contains playbooks, inventory configuration, and supporting resources for managing and documenting the Linux nodes in my lab.

## Overview

This project uses Ansible running in a rootless Podman pod on an Apple Silicon Mac Mini (macOS) to manage and query Linux infrastructure nodes in the homelab. The goal is to automate configuration management, perform infrastructure discovery, and build toward a PostgreSQL-backed system state tracking solution.

All playbooks are version-controlled and synced to this repository via a self-hosted [Forgejo](https://forgejo.org/) git server.

---

## Infrastructure

| Host | OS | Role |
|---|---|---|
| Lenovo IdeaPad | Fedora Server | Primary Linux server — containers, git, services |
| Raspberry Pi 3B+ | Ubuntu Server 22.04 | Backup node, NFS file sharing |

### Ansible Control Node
- **Platform:** Apple Mac Mini M4 (macOS)
- **Runtime:** Rootless Podman pod containing the Ansible environment
- **Connectivity:** SSH with key-based authentication
- **Remote access:** Tailscale VPN for access outside the local network

---

## Repository Structure

```
homelab-ansible/
├── inventory/
│   └── hosts.ini                  # Inventory file defining managed hosts
├── playbooks/
│   ├── gather_info.yml            # Queries each host and outputs an individual JSON file per machine
│   ├── gather_to_single_file.yml  # Queries all hosts and consolidates output into a single JSON file
│   ├── gather_and_store.yml       # Gathers facts from all hosts and stores to PostgreSQL
│   └── update_systems.yml         # Runs package updates on all hosts and logs results to PostgreSQL
├── vars/
│   └── secrets.yml                # ansible-vault encrypted credentials (gitignored)
├── output/
│   ├── <hostname>.json            # Per-host output from gather_info.yml (gitignored)
│   └── all_lab_configs.json       # Consolidated output from gather_to_single_file.yml (gitignored)
└── README.md
```

---

## Playbooks

### `gather_info.yml` — Per-Host Configuration Discovery

Queries each managed host individually and writes a separate JSON file for each machine, named after the host.

**What it does:**
- Connects to all hosts in inventory
- Gathers full system facts via `gather_facts: yes` — OS, distribution, version, hardware, network interfaces, and more
- Displays a summary of hostname and OS for each host
- Writes an individual JSON facts file per host to the output directory on the control node

**Usage:**
```bash
ansible-playbook playbooks/gather_info.yml -i inventory/hosts.ini

# Run against a single host
ansible-playbook playbooks/gather_info.yml -i inventory/hosts.ini --limit fedora-server
```

**Output:**
```
output/
├── fedora-server.json
└── raspberrypi.json
```

---

### `gather_to_single_file.yml` — Consolidated Configuration Discovery

Queries all managed hosts and consolidates the gathered facts from every machine into a single JSON array file for review and future database ingestion.

**What it does:**
- Connects to all hosts in inventory
- Gathers full system facts via `gather_facts: yes`
- Displays a summary of hostname and OS for each host
- Consolidates all gathered facts across all hosts into a single JSON array using a Jinja2 loop
- Writes the consolidated output to `output/all_lab_configs.json` on the control node via `delegate_to: localhost` and `run_once: yes`

**Usage:**
```bash
ansible-playbook playbooks/gather_to_single_file.yml -i inventory/hosts.ini

# Dry run (check mode)
ansible-playbook playbooks/gather_to_single_file.yml -i inventory/hosts.ini --check
```

**Output:**
```
output/
└── all_lab_configs.json
```

**Example task output:**
```
TASK [Display basic system info]
ok: [fedora-server] => {
    "msg": [
        "Host: fedora-server",
        "OS: Fedora 44"
    ]
}
ok: [raspberrypi] => {
    "msg": [
        "Host: raspberrypi",
        "OS: Ubuntu 22.04"
    ]
}
```

---

### `gather_and_store.yml` — Configuration Discovery to PostgreSQL

Queries all managed hosts, gathers full system facts, and stores timestamped JSONB snapshots per host into a PostgreSQL database for infrastructure state tracking and drift detection.

**What it does:**
- Phase 1: Connects to all hosts and gathers full system facts
- Phase 2: Creates the `host_configs` table if missing, then inserts a timestamped JSONB snapshot per host into the `homelab_inventory` PostgreSQL database

**Requirements:**
- `community.postgresql` collection installed
- `psycopg2` installed in the Ansible environment
- `../vars/secrets.yml` encrypted with ansible-vault containing `db_user` and `db_password`
- PostgreSQL container running inside the same Podman pod

**Usage:**
```bash
ansible-playbook playbooks/gather_and_store.yml -i inventory/hosts.ini --ask-vault-pass
```

**Database:**
- Database: `homelab_inventory`
- Table: `host_configs`
- Schema: `id SERIAL PRIMARY KEY`, `captured_at TIMESTAMP`, `host_name TEXT`, `data JSONB`

---

### `update_systems.yml` — System Updates with PostgreSQL Logging

Runs distro-appropriate package updates on all managed hosts and logs results — including updated package details, update status, and reboot requirements — to PostgreSQL for change management tracking and audit history.

**What it does:**
- Phase 1: Connects to each host, runs `dnf` (Fedora) or `apt` (Ubuntu) updates, captures updated package names and versions, checks reboot status, and normalizes results across distros
- Phase 2: Creates the `system_updates` table if missing, then inserts a timestamped record per host

**Requirements:**
- `ansible` user has passwordless sudo on managed hosts
- `python3-libdnf5` installed on Fedora hosts
- `dnf-utils` installed on Fedora hosts (for `needs-restarting`)
- `community.postgresql` collection installed
- `../vars/secrets.yml` encrypted with ansible-vault containing `db_user` and `db_password`
- PostgreSQL container running inside the same Podman pod

**Usage:**
```bash
ansible-playbook playbooks/update_systems.yml -i inventory/hosts.ini --ask-vault-pass
```

**Database:**
- Database: `homelab_inventory`
- Table: `system_updates`
- Schema: `id SERIAL PRIMARY KEY`, `updated_at TIMESTAMP`, `host_name TEXT`, `distribution TEXT`, `packages_updated JSONB`, `update_status TEXT`, `reboot_required BOOLEAN`

**Update status values:**
- `updated` — packages were updated on this run
- `nothing_to_update` — system was already up to date
- `failed` — host failed during Phase 1 (fallback default)

**Verify results:**
```sql
SELECT host_name, distribution, update_status,
       reboot_required, updated_at,
       jsonb_array_length(packages_updated) AS pkg_count
FROM system_updates
ORDER BY updated_at DESC;
```

---

## Setup & Requirements

### Prerequisites
- Podman installed on the control node (macOS)
- Ansible running inside a rootless Podman pod
- SSH key-based access configured between the control node and all managed hosts
- Tailscale installed on all nodes for remote access

### Inventory Configuration
Edit `inventory/hosts.ini` to match your environment:

```ini
[homelab]
fedora-server ansible_host=<IP_OR_HOSTNAME> ansible_user=<YOUR_USER>
raspberrypi   ansible_host=<IP_OR_HOSTNAME> ansible_user=<YOUR_USER>

[homelab:vars]
ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

### Running Playbooks
```bash
# Run per-host discovery (individual JSON files per machine)
ansible-playbook playbooks/gather_info.yml -i inventory/hosts.ini

# Run consolidated discovery (single all_lab_configs.json)
ansible-playbook playbooks/gather_to_single_file.yml -i inventory/hosts.ini

# Run fact gathering and store to PostgreSQL
ansible-playbook playbooks/gather_and_store.yml -i inventory/hosts.ini --ask-vault-pass

# Run system updates and log results to PostgreSQL
ansible-playbook playbooks/update_systems.yml -i inventory/hosts.ini --ask-vault-pass

# Run against a single host
ansible-playbook playbooks/gather_info.yml -i inventory/hosts.ini --limit fedora-server
```

> **Note:** Playbooks that connect to PostgreSQL require ansible-vault credentials. Always pass `--ask-vault-pass` or configure a vault password file for automated runs.

---

## Automated Execution

`update_systems.yml` runs automatically on a weekly schedule via cron on the Mac Mini control node, executing inside the Ansible Podman container without manual intervention.

### Vault Password File Setup

For automated runs, store the vault password in a file on the control node rather than entering it interactively:

```bash
echo "your_vault_password" > ~/ansible/.vault_pass
chmod 600 ~/ansible/.vault_pass
```

> **Important:** The vault password file must never be committed to version control. Confirm `.vault_pass` is listed in `.gitignore` before committing.

Test the vault password file works correctly before configuring cron:
```bash
/opt/homebrew/bin/podman exec <ansible-container-name> \
  ansible-playbook playbooks/update_systems.yml \
  -i inventory/hosts.ini \
  --vault-password-file /ansible/.vault_pass
```

### Cron Configuration

The cron job runs on the Mac Mini host and calls `podman exec` to execute the playbook inside the Ansible container every Sunday at 3AM. Output is logged to `logs/update_systems.log` for auditability.

Open crontab:
```bash
crontab -e
```

Add the following — replace `<ansible-container-name>` and `<your-username>` with your actual values:
```bash
PATH=/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin
0 3 * * 0 podman exec <ansible-container-name> ansible-playbook playbooks/update_systems.yml -i inventory/hosts.ini --vault-password-file /ansible/.vault_pass >> /Users/<your-username>/ansible/logs/update_systems.log 2>&1
```

> **macOS note:** cron on macOS runs with a minimal PATH that does not include Homebrew binaries. Setting `PATH` explicitly at the top of the crontab is required for `podman` to be found. Alternatively use the full path to the `podman` binary (typically `/opt/homebrew/bin/podman` on Apple Silicon).

Verify the cron entry saved correctly:
```bash
crontab -l
```

### Log Files

Cron output is written to `logs/update_systems.log`. The `logs/` directory is gitignored. To monitor recent runs:
```bash
tail -50 ~/ansible/logs/update_systems.log
```

---

## Roadmap

- [x] PostgreSQL integration — store fact gathering results as timestamped JSONB snapshots per host
- [x] System update playbook — run package updates and log results to PostgreSQL with reboot flagging
- [x] Scheduled execution — weekly cron job on Mac Mini control node runs update_systems.yml automatically
- [ ] Migrate scheduled execution to Forgejo Actions using stored secrets
- [ ] Configuration drift detection — query PostgreSQL to compare snapshots across runs and surface changes
- [ ] Django query interface — web application to visualize host state, update history, and reboot status
- [ ] Service state verification — playbook to check and log running services across all hosts
- [ ] Alerting — notify on configuration drift or unexpected state changes

---

## Related Projects

This repository is part of a broader homelab infrastructure setup. Other components include:

- **Podman container stack** (Fedora Server) — Forgejo, Hugo blog, PostgreSQL, Stirling-PDF running as rootless containers
- **NFS file sharing** — Raspberry Pi serving as NFS host, automounted on Fedora Server
- **Automated backups** — nightly cron job using Forgejo dump + rsync to Raspberry Pi
- **Django applications** — multi-tenant web applications developed and tested in the homelab environment

---

## Security Notes

- SSH key-based authentication only — password authentication disabled on managed hosts
- Ansible runs inside a rootless Podman pod following principle of least privilege
- Database credentials stored in `vars/secrets.yml` encrypted with ansible-vault — never committed in plaintext
- Vault password file (`.vault_pass`) stored with `600` permissions on control node — never committed to version control
- `vars/secrets.yml`, `.vault_pass`, `.env` files, `logs/`, and output JSON files are excluded via `.gitignore`
- Inventory files with specific IP addresses should be reviewed before committing to public repositories
- Ansible user on managed hosts has passwordless sudo scoped to package manager commands only

---

## Author

Personal homelab project. Background in Unix/Linux systems administration, government IT infrastructure, and Certification & Accreditation engineering within the U.S. Intelligence Community.

Currently pursuing Red Hat Certified System Administrator (RHCSA) certification.

