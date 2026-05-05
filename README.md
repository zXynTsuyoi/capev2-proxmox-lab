# CAPEv2 Malware Sandbox — Lab Deployment

## Overview

CAPEv2 sandbox for automated malware analysis, deployed on Ubuntu 22.04 inside
Proxmox VM 115. Uses KVM/QEMU with a Windows 10 Lite analysis guest on an
isolated host-only network.

## Hardware / Host

| Component | Detail |
|---|---|
| Hypervisor | Proxmox (VM 115) |
| OS | Ubuntu 22.04.5 LTS (Jammy) |
| RAM | 24 GB |
| CPU | 8 cores |
| Disk | 100 GB (65 GB free) |
| Management IP | <HOST_IP> (ens18) |
| Nested KVM | Enabled (`/proc/sys/kernel/nested = Y`) |

## Network Topology

```
Management: <HOST_IP> (ens18)
    │
    ├── virbr0 (libvirt default): 192.168.122.0/24 (unused)
    │
    ├── virbr1 (isolated host-only): 192.168.100.0/24
    │       Host IP:     192.168.100.1
    │       Guest IP:    192.168.100.100 (static)
    │       DHCP range:  192.168.100.10 - 192.168.100.200
    │
    └── localhost services:
          CAPE Web UI:   127.0.0.1:8000
          PostgreSQL:    127.0.0.1:5432
          MongoDB:       127.0.0.1:27017
          VNC:           :5901
```

## Analysis VM

| Setting | Value |
|---|---|
| Name | win10 |
| OS | Windows 10 Pro Lite Edition 19H2 x64 |
| RAM | 4096 MB |
| Disk | 40 GB qcow2 (`/var/lib/libvirt/images/win10.qcow2`) |
| Video | VGA |
| Network | e1000 on virbr1 |
| IP | 192.168.100.100 (static) |
| Windows user | `cape` / `cape` (Administrator, auto-login) |
| CAPE Agent | Python 3.10 x86, auto-start via Startup folder |
| Agent port | 8000 |
| Snapshot | `snapshot1` (running state, taken 2026-05-05) |

## Installed Components

| Component | Path / Details |
|---|---|
| CAPEv2 | `/opt/CAPEv2/` (git clone, commit 781161f2) |
| Python env | Poetry venv at `/home/cape/.cache/pypoetry/virtualenvs/capev2-*/` |
| PostgreSQL 14 | Main CAPE database (user: `cape`, db: `cape`) |
| MongoDB 8.0 | Reporting storage |
| YARA 4.5.5 | Signature matching |
| Suricata | (installed, service disabled) |
| libvirt-python 8.0.0 | KVM machinery bindings |

## Key Configuration Files

| File | Purpose |
|---|---|
| `conf/cuckoo.conf` | Main CAPE config (machinery=kvm, resultserver_ip=192.168.100.1) |
| `conf/kvm.conf` | VM definitions (win10, snapshot1, IP) |
| `conf/auxiliary.conf` | Sniffing interface (virbr1) |
| `conf/reporting.conf` | MongoDB reporting enabled |
| `systemd/cape*.service` | Systemd unit files |

## Services

```bash
# All services (enabled, auto-start on boot):
systemctl status cape         # Scheduler / main process
systemctl status cape-processor  # Report processor
systemctl status cape-rooter     # Network rooter (iptables)
systemctl status cape-web        # Django web UI (127.0.0.1:8000)
systemctl status mongodb         # MongoDB
systemctl status postgresql      # PostgreSQL
```

## Credentials

| Service | Username | Password | Notes |
|---|---|---|---|
| CAPE Web UI | `cape` | `cape123` | http://127.0.0.1:8000 |
| Windows VM | `cape` | `cape` | Auto-login configured |
| VNC | — | `capelab` | Port 5901 |
| PostgreSQL | `cape` | `SuperPuperSecret` | Database: `cape` |

## Usage

### Access Web UI (remote)

The web UI binds to localhost. Tunnel via SSH from your workstation:

```bash
ssh -L 8000:127.0.0.1:8000 cape@<HOST_IP>
```

Then open **http://localhost:8000** in your browser.

### Submit a sample (CLI)

```bash
cd /opt/CAPEv2
sudo -u cape /etc/poetry/bin/poetry run python3 utils/submit.py /path/to/malware.exe
```

### Submit via API

```bash
curl -F file=@/path/to/malware.exe http://127.0.0.1:8000/apiv2/tasks/create/file/
```

### Check task status

```bash
curl http://127.0.0.1:8000/apiv2/tasks/view/<task_id>/
```

### VM management

```bash
virsh start win10      # Start the VM
virsh destroy win10    # Force stop
virsh snapshot-list win10      # List snapshots
virsh snapshot-revert win10 snapshot1  # Revert to clean state
virsh vncdisplay win10          # Show VNC port
```

### Agent check

```bash
curl http://192.168.100.100:8000/
# Expected: {"message": "CAPE Agent!", "version": "0.20", ...}
```

## Troubleshooting

### Cape service crashes with libvirt error
```bash
# Ensure libvirt-python is installed in the Poetry venv:
export PKG_CONFIG_PATH=/usr/lib/x86_64-linux-gnu/pkgconfig
cd /opt/CAPEv2 && sudo -u cape /etc/poetry/bin/poetry run pip install libvirt-python==8.0.0
```

### Snapshot state error
CAPE requires snapshots taken while the VM is **running**:
```bash
virsh start win10
# Wait for agent to respond:
curl http://192.168.100.100:8000/
virsh snapshot-delete win10 snapshot1
virsh snapshot-create-as win10 snapshot1 "description"
```

### Agent not responding
- Ensure Windows auto-login is working
- Check `C:\Users\cape\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\agent.bat` exists
- Agent runs on `C:\Python310\python.exe C:\agent.py 0.0.0.0 8000`

### DPkg lock issues
```bash
sudo killall apt-get dpkg 2>/dev/null
sudo dpkg --configure -a
```

### VNC not accessible
```bash
# VNC binds to 0.0.0.0:5901. Tunnel if needed:
ssh -L 5901:127.0.0.1:5901 cape@<HOST_IP>
```

## Updates

```bash
cd /opt/CAPEv2
git pull
sudo -u cape /etc/poetry/bin/poetry install
sudo -u cape /etc/poetry/bin/poetry run python3 utils/community.py -waf -cr
systemctl restart cape cape-processor cape-rooter cape-web
```

## Security Notes

- Web UI binds to `127.0.0.1` only — not exposed externally
- Windows VM has no internet access (isolated virbr1)
- All services run under the `cape` user except `cape-rooter` (root, for iptables)
- This is a **lab environment** — not hardened for production
