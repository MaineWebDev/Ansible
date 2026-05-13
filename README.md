# homelab-ansible

Ansible automation for my personal homelab environment. This repository contains playbooks, inventory configuration, and supporting resources for managing and documenting the Linux nodes in my lab.

## Overview

This project uses Ansible running in a rootless Podman pod on an Apple Silicon Mac Mini (macOS) to manage and query Linux infrastructure nodes in the homelab. The goal is to automate configuration management, perform infrastructure discovery, and store system state in a PostgreSQL database for historical tracking.

All playbooks are version-controlled and synced to this repository via a self-hosted [Forgejo](https://forgejo.org/) git server.

---

## Infrastructure

| Host | OS | Role |
|---|---|---|
| Lenovo IdeaPad | Fedora Server | Primary Linux server — containers, git, services |
| Raspberry Pi 3B+ | Ubuntu Server 22.04 | Backup node, NFS file sharing |

### Ansible Control Node
- **Platform:** Apple Mac Mini M4 (macOS)
- **Runtime:** Rootless Podman pod (`ansible-stack`) containing the Ansible environment and a PostgreSQL database container
- **Connectivity:** SSH with key-based authentication
- **Remote access:** Tailscale VPN for access outside the local network

---

## Pod Architecture

Both the Ansible control container and the PostgreSQL database run inside the same rootless Podman pod (`ansible-stack`). Sharing a pod means both containers share a network namespace — Postgres is reachable at `127.0.0.1:5432` from within the Ansible container without any external port exposure.

### Starting the Pod

```bash
# Create the pod
podman pod create --name ansible-stack -p 8080:8000

# Start the PostgreSQL container
podman run -d \
  --name ansible-db \
  --pod ansible-stack \
  -e POSTGRES_PASSWORD=<your_password> \
  -e POSTGRES_USER=ansible \
  -e POSTGRES_DB=ansible_inventory \
  -v ~/homelab/ansible/postgres_data:/var/lib/postgresql/data:Z \
  postgres:16 -c listen_addresses='*'

# Start the Ansible container
podman run -d \
  --name ansible-server \
  --pod ansible-stack \
  -v ~/ansible:/ansible:Z \
  -v ~/ansible/ssh_keys:/root/.ssh:Z \
  localhost/local-ansible:latest \
  sleep infinity
```

### Verify Both Containers Are in the Pod

```bash
podman ps --pod
```

Both `ansible-db` and `ansible-server` should show `ansible-stack` in the PODNAME column.

---

## Repository Structure

```
homelab-ansible/
├── ansible.cfg                        # Ansible configuration
├── hosts.ini                          # Inventory file defining managed hosts
├── gather_and_store.yml               # Gathers facts and stores results in PostgreSQL
├── playbooks/
│   ├── gather_info.yml                # Queries each host and outputs individual JSON files
│   └── gather_to_single_file.yml      # Queries all hosts and consolidates into a single JSON file
├── output/
│   ├── <hostname>.json                # Per-host output (gitignored)
│   └── all_lab_configs.json           # Consolidated output (gitignored)
├── vars/
│   └── secrets.yml                    # Ansible Vault encrypted credentials (gitignored)
├── ssh_keys/                          # SSH keys and known_hosts (gitignored)
└── README.md
```

---

## Playbooks

### `gather_and_store.yml` — Gather Facts and Store in PostgreSQL

Connects to all managed hosts, gathers system facts, and stores the results in a PostgreSQL database running in the same Podman pod.

**What it does:**
- Phase 1: Connects to all hosts and gathers full system facts via `gather_facts: yes`
- Phase 2: Creates the `homelab_inventory` database if missing, creates the `host_configs` table if missing, and inserts gathered facts as JSONB rows

**Database schema:**
```sql
CREATE TABLE host_configs (
    id            SERIAL PRIMARY KEY,
    captured_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    host_name     TEXT,
    data          JSONB
);
```

**Usage:**
```bash
podman exec -it ansible-server ansible-playbook /ansible/gather_and_store.yml --ask-vault-pass
```

**Querying results:**
```bash
podman exec -it ansible-db psql -U ansible -d homelab_inventory
```
```sql
SELECT host_name, captured_at FROM host_configs;
```

---

### `gather_info.yml` — Per-Host Configuration Discovery

Queries each managed host individually and writes a separate JSON file for each machine, named after the host.

**Usage:**
```bash
ansible-playbook playbooks/gather_info.yml -i hosts.ini

# Run against a single host
ansible-playbook playbooks/gather_info.yml -i hosts.ini --limit fedora-server
```

**Output:**
```
output/
├── fedora-server.json
└── raspberrypi.json
```

---

### `gather_to_single_file.yml` — Consolidated Configuration Discovery

Queries all managed hosts and consolidates gathered facts into a single JSON array file.

**Usage:**
```bash
ansible-playbook playbooks/gather_to_single_file.yml -i hosts.ini

# Dry run
ansible-playbook playbooks/gather_to_single_file.yml -i hosts.ini --check
```

**Output:**
```
output/
└── all_lab_configs.json
```

---

## Setup & Requirements

### Prerequisites
- Podman installed on the control node (macOS)
- Ansible running inside a rootless Podman pod
- SSH key-based access configured between the control node and all managed hosts
- Tailscale installed on all nodes for remote access

### Vault Setup

Credentials are stored in `vars/secrets.yml` encrypted with Ansible Vault:

```bash
ansible-vault create vars/secrets.yml
```

The file should contain:
```yaml
db_user: ansible
db_password: <your_password>
```

### Inventory Configuration

Edit `hosts.ini` to match your environment:

```ini
[homelab]
fedora-server ansible_host=<IP_OR_HOSTNAME> ansible_user=ansible
raspberrypi   ansible_host=<IP_OR_HOSTNAME> ansible_user=ansible
```

### ansible.cfg

```ini
[defaults]
inventory = /ansible/hosts.ini
interpreter_python = auto_silent
stdout_callback = debug
deprecation_warnings = False

[ssh_connection]
pipelining = True
ssh_args = -o UserKnownHostsFile=/ansible/ssh_keys/known_hosts -i /ansible/ssh_keys/id_ed25519
```

---

## Roadmap

- [x] PostgreSQL integration — store playbook execution results and system state data in PostgreSQL
- [ ] Additional playbooks — package management, service state verification, security compliance checks
- [ ] Scheduled execution — automate regular discovery runs via cron or Forgejo Actions
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
- Ansible and PostgreSQL run inside a rootless Podman pod following principle of least privilege
- Pod networking keeps PostgreSQL off the external network — only accessible via `127.0.0.1` within the pod
- Credentials are encrypted with Ansible Vault and excluded via `.gitignore`
- SSH keys and known_hosts are excluded via `.gitignore`

---

## Author

Personal homelab project. Background in Unix/Linux systems administration, government IT infrastructure, and Certification & Accreditation engineering within the U.S. Intelligence Community.

Currently pursuing Red Hat Certified System Administrator (RHCSA) certification.
