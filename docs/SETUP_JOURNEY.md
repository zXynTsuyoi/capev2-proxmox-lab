# CAPEv2 Setup Journey — AI Agent Narrative

## What We Built

CAPEv2 malware sandbox on Ubuntu 22.04 inside Proxmox VM 115, with a Windows 10
Lite analysis guest on an isolated host-only network. All services verified with
live analysis of /bin/ls completing with status "reported".

---

## Chapter 1: The Installer Script

### Problem: dpkg in a broken state

The very first `apt-get` command in `cape2.sh` failed because a previous
interruption left dpkg locked. Every subsequent package install printed:

    E: dpkg was interrupted, you must manually run 'sudo dpkg --configure -a'

**Fix:** `sudo dpkg --configure -a` cleared the lock.

### Problem: Ubuntu mirrors unreachable

The `cape2.sh` script called `apt-get update` which tried to reach
`id.archive.ubuntu.com` and `security.ubuntu.com`. Both timed out — the lab
network apparently can't reach Canonical's infrastructure reliably.

**Fix:** Switched all apt sources to DigitalOcean mirrors:
```bash
sed -i 's|http://[a-z.]*archive.ubuntu.com|http://mirrors.digitalocean.com|g' /etc/apt/sources.list
sed -i 's|http://security.ubuntu.com|http://mirrors.digitalocean.com|g' /etc/apt/sources.list
```

### Problem: Installer timeout at PostgreSQL

The full `cape2.sh all` command timed out after 30 minutes. It was stuck trying
to add the PostgreSQL apt repository (which tried to reach security.ubuntu.com
again). The dependencies function had mostly completed — apt packages, Poetry,
system tools were installed — but the script never made it past PostgreSQL.

**Fix:** Ran individual script components instead:
```
cape2.sh sandbox    → clone CAPE, install Python deps, yara-python, libvirt-python
cape2.sh mongo      → install MongoDB 8.0
cape2.sh yara       → install YARA 4.5.5
cape2.sh systemd    → install systemd service units
```

Manually installed PostgreSQL via apt and created the cape user/database.

---

## Chapter 2: Database Setup

### Problem: PostgreSQL peer authentication

After installing PostgreSQL, the `cape` user couldn't connect because Ubuntu's
default is `peer` authentication for local connections.

**Fix:** Changed `pg_hba.conf` to use `md5` (password) authentication:
```
local   all   all   md5
```

### Problem: MongoDB user didn't exist

The MongoDB install script created a systemd service referencing user `mongodb`
but the user package was never created because an apt lock blocked the
installation. The first `cape2.sh mongo` run was partially successful.

**Fix:** Manually created the `mongodb` user, killed lingering apt processes,
re-ran `cape2.sh mongo`.

---

## Chapter 3: Python Dependencies

### Problem: Django not found after sandbox install

The `cape2.sh sandbox` command completed but `python3 manage.py migrate` failed
with `ModuleNotFoundError: No module named 'django'`. The `pip install -r
pyproject.toml` step inside the script apparently didn't install all packages.

**Fix:** Ran `poetry install` manually, which resolved all 200+ dependencies.

### Problem: libvirt-python failed to build

The `cape` scheduler service crashed with:
```
ModuleNotFoundError: No module named 'libvirt'
```

The `install_libvirt()` function in cape2.sh needed `libvirt-dev` package to
compile the Python bindings, and it needed the correct `PKG_CONFIG_PATH`.
Neither was available.

**Fix:**
```bash
apt-get install -y libvirt-dev
export PKG_CONFIG_PATH=/usr/lib/x86_64-linux-gnu/pkgconfig
poetry run pip install libvirt-python==8.0.0
```

### Problem: pymongo --break-system-packages flag

The MongoDB install script tried `pip3 install pymongo --break-system-packages`
which failed on this Python version. pymongo was needed in the Poetry venv.

**Fix:** Installed via Poetry: `poetry run pip install pymongo`

---

## Chapter 4: The Windows VM

### Problem: QEMU couldn't access files in /home/cape

The VM disk and ISO were initially placed in `/home/cape/vms/`. The
`libvirt-qemu` user lacked permissions (uid 64055 couldn't traverse `/home/cape`).

**Fix:** Moved everything to `/var/lib/libvirt/images/` and set ownership:
```bash
chown libvirt-qemu:kvm /var/lib/libvirt/images/win10.*
```

### Problem: VNC port 5900 already in use

`gnome-remote-desktop` was already bound to port 5900.

**Fix:** Used port 5901 for the analysis VM VNC.

### Problem: BIOS boot showed solid purple screen

With QXL video, the VNC framebuffer showed uniform RGB(24,0,82) — a dark purple
— for 17+ minutes. The VM was doing CPU work but nothing rendered. This wasn't
a freeze; QXL wasn't updating the framebuffer during Windows PE boot.

**First attempt — VGA:** Switched from QXL to VGA video. Same issue.

**Second attempt — UEFI:** Switched to UEFI boot (OVMF). Screen now showed 3
unique colors (black, gray, white) — the Windows logo was visible. Progress!

But UEFI doesn't support floppy drives, and our `autounattend.vfd` with the
full unattended config was on floppy. Without it, the installer stopped at disk
selection.

**Final approach — BIOS + VGA + manual VNC:** Reverted to BIOS boot with VGA
video. The user connected via VNC (port 5901, password `capelab`) and completed
the disk selection manually. The ISO's built-in autounattend.xml handled
language, EULA, and OOBE automatically.

### Problem: CD-ROM agent ISO unreadable by Windows

The `cape_agent.iso` created with `genisoimage` showed "You need to format the
disk" when opened in Windows. The ISO lacked proper Joliet/UDF extensions.

**Fix:** Served the files via HTTP instead:
```bash
python3 -m http.server 8080 --bind 192.168.100.1
```
The Windows VM downloaded `agent.py` and `python-3.10-x86.exe` via browser.

---

## Chapter 5: Agent & Auto-Start

### Problem: No PowerShell on Windows 10 Lite

The usual approach — `schtasks` via PowerShell — wasn't available because this
Lite Edition stripped PowerShell.

**Fix:** Used the classic Startup folder approach:
```
C:\Users\cape\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\agent.bat
```
Contents:
```bat
@echo off
start /b C:\Python310\python.exe C:\agent.py 0.0.0.0 8000
```

### Problem: No auto-login on Windows

Without auto-login, the Startup folder items wouldn't run until someone
manually logged in — making the VM useless for automated analysis.

**Fix:** Registry entries for auto-login:
```cmd
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AutoAdminLogon  /t REG_SZ /d 1 /f
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName /t REG_SZ /d cape /f
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword /t REG_SZ /d cape /f
reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultDomainName /t REG_SZ /d . /f
```

---

## Chapter 6: The Snapshot That Wasn't

### Problem: Snapshot taken while VM was shut off

The first snapshot was taken with `virsh snapshot-create-as win10 snapshot1`
while the VM was powered off. The snapshot state was `shutoff`. CAPE's startup
check explicitly rejects this:

```
CuckooStartupError: Snapshot 'snapshot1' for VM 'win10' is not in a 'running'
state (current state: 'shutoff'). Please ensure you take snapshots of running VMs.
```

This caused both `cape` and `cape-processor` services to crash-loop with exit
code 1. All submitted tasks stayed stuck at "pending".

**Fix:** Started the VM, waited for the agent to respond, then took the snapshot
while the VM was running. The snapshot now shows `state: running`.
```bash
virsh start win10
# wait for agent...
virsh snapshot-delete win10 snapshot1
virsh snapshot-create-as win10 snapshot1 "Clean state"
```

### Problem: Floppy disk blocked snapshot creation

The VM still had the `autounattend.vfd` floppy attached from the install
attempt. Internal snapshots on raw storage type (floppy) are unsupported.

**Fix:** Destroyed the VM, edited the domain XML to remove the floppy device,
then redefined and snapshotted.

---

## Chapter 7: First Analysis — Success

After fixing the snapshot state, all four CAPE services started cleanly:

```
cape:            active
cape-processor:  active
cape-rooter:     active
cape-web:        active
```

Submitted `/bin/ls` — it took ~4 minutes for the scheduler to pick it up, then:

```
Task #2: Processing task
Task #2: Starting analysis on guest (id=win10, ip=192.168.100.100)
Task #2: Guest is running CAPE Agent 0.20
Task #2: Uploading script files to guest
```

90 seconds later: **status: reported**. No errors. Full analysis with
signatures, behavior log, and dropped files available in the web UI.

---

## Root Cause Summary

| # | Symptom | Root Cause | Fix |
|---|---|---|---|
| 1 | dpkg errors | Previous apt interruption | `dpkg --configure -a` |
| 2 | apt timeouts | Ubuntu mirrors unreachable | Switched to DO mirrors |
| 3 | Installer timeout | Script too large, slow network | Ran components individually |
| 4 | Django missing | Incomplete Poetry install | `poetry install` |
| 5 | libvirt module missing | libvirt-dev not installed | Installed headers + pip |
| 6 | File permission denied | libvirt-qemu can't access /home | Moved to /var/lib/libvirt |
| 7 | VNC port conflict | gnome-remote-desktop on 5900 | Used port 5901 |
| 8 | Solid purple screen | QXL framebuffer not updated | Switched to VGA |
| 9 | Floppy in UEFI | UEFI doesn't support floppy | BIOS boot + manual VNC |
| 10 | ISO unreadable | Missing Joliet/UDF | HTTP file server instead |
| 11 | No PowerShell | Windows 10 Lite stripped it | Startup folder .bat |
| 12 | No auto-login | Manual install overwrote | Registry entries |
| 13 | Snapshot state wrong | Taken while VM powered off | Retook while running |
| 14 | Floppy blocks snapshot | Raw storage no internal snap | Removed floppy from XML |

---

## Final State

```
VM:         running, snapshot1 (running state)
Services:   6/6 active (cape, processor, rooter, web, mongo, postgres)
Agent:      responding at 192.168.100.100:8000
Web UI:     127.0.0.1:8000 — accessible via SSH tunnel
Analyses:   3 completed (tasks 1, 2, 3 — all "reported")
```
