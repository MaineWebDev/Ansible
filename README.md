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
│   └── gather_to_single_file.yml  # Queries all hosts and consolidates output into a single JSON file
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

# Dry run either playbook (check mode)
ansible-playbook playbooks/gather_info.yml -i inventory/hosts.ini --check
ansible-playbook playbooks/gather_to_single_file.yml -i inventory/hosts.ini --check

# Run against a single host
ansible-playbook playbooks/gather_info.yml -i inventory/hosts.ini --limit fedora-server
```

---

## Roadmap

- [ ] PostgreSQL integration — store playbook execution results and system state data in a PostgreSQL database for historical tracking and reporting
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
- Ansible runs inside a rootless Podman pod following principle of least privilege
- `.env` files and any files containing secrets or credentials are excluded via `.gitignore`
- Inventory files with specific IP addresses should be reviewed before committing to public repositories

---

## Author

Personal homelab project. Background in Unix/Linux systems administration, government IT infrastructure, and Certification & Accreditation engineering within the U.S. Intelligence Community.

Currently pursuing Red Hat Certified System Administrator (RHCSA) certification.
