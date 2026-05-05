# AI Agent Skills — CAPEv2 Sandbox Environment

## Environment Summary

CAPEv2 malware analysis sandbox running on Ubuntu 22.04 LTS inside Proxmox VM 115.
Single Windows 10 Lite analysis guest on isolated host-only network (virbr1).
No internet access from the analysis VM — lab environment only.

## Quick Facts

- **CAPE root**: `/opt/CAPEv2`
- **Python**: Poetry virtualenv at `/home/cape/.cache/pypoetry/virtualenvs/capev2-*/`
- **Poetry binary**: `/etc/poetry/bin/poetry`
- **Run user**: `cape` (uid 1000)
- **PostgreSQL**: user `cape`, db `cape`, password `SuperPuperSecret`, port 5432
- **MongoDB**: port 27017, service `mongodb`
- **KVM connection**: `qemu:///system`
- **Web UI**: `http://127.0.0.1:8000` (localhost only)
- **VM images**: `/var/lib/libvirt/images/`
- **VM disks**: `/var/lib/libvirt/images/win10.qcow2`
- **Windows ISO**: `/home/cape/Windows 10 Lite Edition 19H2 x64.iso` (also at `/var/lib/libvirt/images/win10.iso`)
- **Agent files**: `/opt/CAPEv2/agent/agent.py`

## Network

| Interface | IP | Purpose |
|---|---|---|
| ens18 | <HOST_IP>/24 | Management (Proxmox bridge) |
| virbr0 | 192.168.122.1/24 | libvirt default (unused) |
| virbr1 | 192.168.100.1/24 | Isolated analysis network |
| win10 guest | 192.168.100.100 | Static IP on virbr1 |

## KVM Analysis VM

| Property | Value |
|---|---|
| Domain name | `win10` |
| OS | Windows 10 Pro x64 (Lite Edition 19H2) |
| RAM | 4096 MB |
| vCPUs | 2 |
| Disk | 40 GB qcow2, bus=sata |
| Video | VGA |
| Network | e1000, virbr1 |
| Boot | BIOS (not UEFI) |
| Snapshot | `snapshot1` (running state) |
| Windows user | `cape` / `cape` (admin, auto-login) |
| Agent | Python 3.10 x86, `C:\agent.py`, port 8000, auto-start via Startup folder |
| VNC | Port 5901, password `capelab`, listen `0.0.0.0` |

## CAPE Config Reference

| Config File | Key Settings |
|---|---|
| `cuckoo.conf` | `machinery = kvm`, `connection = postgresql://...`, `resultserver ip = 192.168.100.1` |
| `kvm.conf` | `machines = win10`, `interface = virbr1`, `[win10] ip = 192.168.100.100`, `snapshot = snapshot1` |
| `auxiliary.conf` | `interface = virbr1` |
| `reporting.conf` | `[mongodb] enabled = yes`, host `127.0.0.1:27017` |

## Service Management

```bash
# All CAPE services restart:
systemctl restart cape cape-processor cape-rooter cape-web

# Check status:
systemctl is-active cape cape-processor cape-rooter cape-web mongodb postgresql

# View logs:
journalctl -u cape --since "10 min ago" --no-pager
journalctl -u cape-processor -f  # follow

# Service user:
#   cape-rooter  -> root (needs iptables)
#   cape         -> cape
#   cape-processor -> cape
#   cape-web     -> cape
```

## Running CAPE Commands

**Always run as user `cape` via Poetry:**

```bash
cd /opt/CAPEv2
sudo -u cape /etc/poetry/bin/poetry run <command>
```

### Submit samples
```bash
sudo -u cape /etc/poetry/bin/poetry run python3 utils/submit.py <file>
sudo -u cape /etc/poetry/bin/poetry run python3 utils/submit.py --url <url>
```

### Query tasks via API
```bash
curl http://127.0.0.1:8000/apiv2/tasks/view/<id>/
curl http://127.0.0.1:8000/apiv2/tasks/list/
curl http://127.0.0.1:8000/apiv2/tasks/report/<id>/json/
```

### Run Django management
```bash
cd /opt/CAPEv2/web
sudo -u cape /etc/poetry/bin/poetry run python3 manage.py <subcommand>
```

### Update CAPE rules
```bash
cd /opt/CAPEv2
sudo -u cape /etc/poetry/bin/poetry run python3 utils/community.py -waf -cr
```

## Verification Commands

```bash
# Agent health:
curl http://192.168.100.100:8000/
# Expected: {"message": "CAPE Agent!", "version": "0.20", ...}

# Ping VM:
ping -c 2 192.168.100.100

# Web UI health:
curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8000/
# Expected: 200

# VM state:
virsh domstate win10

# Snapshot state:
virsh snapshot-list win10  # Must show "running" state

# Service health:
systemctl is-active cape cape-processor cape-rooter cape-web
# Expected: all "active"
```

## Common Fixes

### libvirt-python missing after Poetry install
```bash
apt-get install -y libvirt-dev
PKG_CONFIG_PATH=/usr/lib/x86_64-linux-gnu/pkgconfig
cd /opt/CAPEv2
sudo -u cape bash -c "export PKG_CONFIG_PATH=$PKG_CONFIG_PATH; /etc/poetry/bin/poetry run pip install libvirt-python==8.0.0"
```

### Snapshot wrong state (shutoff vs running)
```bash
# CAPE requires running-state snapshots
virsh start win10
# Wait for agent:
while ! curl -s --connect-timeout 3 http://192.168.100.100:8000/ | grep -q "CAPE Agent"; do sleep 10; done
virsh snapshot-delete win10 snapshot1
virsh snapshot-create-as win10 snapshot1 "Clean state"
```

### Agent not responding (VM up but no agent)
1. Check Windows auto-login: `reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AutoAdminLogon`
2. Check Startup folder: `dir "C:\Users\cape\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\agent.bat"`
3. Check Python: `dir C:\Python310\python.exe`
4. Start agent manually: `C:\Python310\python.exe C:\agent.py 0.0.0.0 8000`

### Apt mirror issues
The default Ubuntu mirrors (`id.archive.ubuntu.com`, `security.ubuntu.com`) may be unreachable.
```bash
sed -i 's|http://[a-z.]*archive.ubuntu.com|http://mirrors.digitalocean.com|g' /etc/apt/sources.list
sed -i 's|http://security.ubuntu.com|http://mirrors.digitalocean.com|g' /etc/apt/sources.list
apt-get update
```

### DPkg lock
```bash
killall apt-get dpkg 2>/dev/null
dpkg --configure -a
```

## VM Management Quick Reference

```bash
# Start / stop
virsh start win10
virsh destroy win10      # force stop
virsh shutdown win10     # graceful (needs QEMU guest agent)

# Snapshots
virsh snapshot-list win10
virsh snapshot-create-as win10 <name> "<description>"
virsh snapshot-delete win10 <name>
virsh snapshot-revert win10 snapshot1

# VNC
virsh vncdisplay win10   # returns e.g. ":1" (port 5901)

# Edit domain XML
virsh dumpxml win10 > /tmp/win10.xml
# edit file, then:
virsh define /tmp/win10.xml
```

## File Paths Cheat Sheet

| Item | Path |
|---|---|
| CAPE root | `/opt/CAPEv2` |
| Configs | `/opt/CAPEv2/conf/*.conf` |
| Systemd units | `/opt/CAPEv2/systemd/*.service` |
| Installed units | `/lib/systemd/system/cape*.service` |
| Agent script | `/opt/CAPEv2/agent/agent.py` |
| Submission tool | `/opt/CAPEv2/utils/submit.py` |
| Django web | `/opt/CAPEv2/web/` |
| Storage | `/opt/CAPEv2/storage/` |
| Analyses | `/opt/CAPEv2/storage/analyses/<id>/` |
| Binaries | `/opt/CAPEv2/storage/binaries/<hash>` |
| QEMU disk | `/var/lib/libvirt/images/win10.qcow2` |
| ISO | `/var/lib/libvirt/images/win10.iso` |
| Agent ISO | `/var/lib/libvirt/images/cape_agent.iso` |
| Autounattend floppy | `/var/lib/libvirt/images/autounattend.vfd` |
| Python installer | `/var/lib/libvirt/images/python-3.10-x86.exe` |
| Installer script | `/tmp/cape2.sh` |
| Installer log | `/tmp/cape2_install.log` |
| VM console log | `/var/log/libvirt/qemu/win10.log` |

## Testing a Full Analysis Cycle

```bash
# 1. Ensure services are up
systemctl restart cape cape-processor cape-rooter cape-web

# 2. Verify agent
curl http://192.168.100.100:8000/

# 3. Submit test
cd /opt/CAPEv2
sudo -u cape /etc/poetry/bin/poetry run python3 utils/submit.py /bin/ls

# 4. Monitor progress (repeat until "reported")
curl -s http://127.0.0.1:8000/apiv2/tasks/view/<ID>/ | python3 -c "import sys,json;print(json.load(sys.stdin)['data']['status'])"

# 5. View report
curl -s http://127.0.0.1:8000/analysis/<ID>/
```
