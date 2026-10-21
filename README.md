# RHEL 9 — System Administration Lab

Hands-on Red Hat Enterprise Linux 9 system administration training, documenting practical installation, configuration, storage management, user administration and troubleshooting exercises.

## Project Overview

This repository documents my practical development of Linux system administration skills using Red Hat Enterprise Linux 9.8 in an Oracle VirtualBox lab environment.

The objective is to build confidence in enterprise Linux operations through hands-on exercises, command-line administration, troubleshooting and technical documentation.

The lab focuses on skills relevant to L1/L2 infrastructure support and RHEL system administration.

## Lab Environment

| Component | Configuration |
|---|---|
| Operating System | Red Hat Enterprise Linux 9.8 |
| Virtualisation | Oracle VirtualBox |
| Hostname | rhel-lab01 |
| Installation Type | Minimal Install |
| Virtual Disk | 50 GiB |
| Memory | 4 GiB allocated |
| Filesystem | XFS |
| Storage Management | LVM2 |
| Network Interface | enp0s3 |
| Network Configuration | VirtualBox NAT, DHCP |
| Primary User | niall |
| Administrative User | admniall |

## Lab 01 — RHEL Installation

**Status: Completed**

Installed RHEL 9.8 using the Anaconda installer with manual disk partitioning.

Configuration included:
- Minimal installation for a command-line server environment
- Manual XFS filesystem configuration
- Logical Volume Manager (LVM) storage
- Separate root, boot, home and var filesystems
- Dedicated swap logical volume
- Network configuration and hostname assignment
- Local user account creation

### Disk Layout

| Device / Logical Volume | Mount Point | Size | Filesystem |
|---|---|---|---|
| /dev/sda1 | /boot | 1 GiB | XFS |
| rhel-root | / | 20 GiB | XFS |
| rhel-var | /var | 10 GiB | XFS |
| rhel-home | /home | 5 GiB | XFS |
| rhel-swap | swap | 4 GiB | Linux swap |

The LVM logical volumes reside in volume group `rhel`, backed by physical volume `/dev/sda2`.

### Storage Verification

Commands executed:

```bash
lsblk
lsblk -f
df -hT
pvs
vgs
lvs
```

**Key learning:** Understanding the relationship between physical disks, partitions, physical volumes, volume groups, logical volumes and mounted filesystems.

## Lab 02 — LVM Storage Administration

**Status: Initial inspection completed**

Investigated the Linux storage hierarchy.

```text
/dev/sda (50 GiB)
├── sda1 (1 GiB) → /boot
└── sda2 (LVM physical volume)
    └── Volume Group: rhel
        ├── root (20 GiB) → /
        ├── var  (10 GiB) → /var
        ├── home (5 GiB)  → /home
        └── swap (4 GiB)  → swap
```

### Commands and Purpose

| Command | Purpose |
|---|---|
| lsblk | Display block devices and storage hierarchy |
| lsblk -f | Show filesystem types and UUIDs |
| df -hT | Display mounted filesystem usage and types |
| pvs | List LVM physical volumes |
| vgs | List LVM volume groups |
| lvs | List LVM logical volumes |

### Troubleshooting: LVM Permission Denied

**Issue:** Running `pvs`, `vgs` and `lvs` as a standard user produced permission warnings.

**Cause:** LVM administration requires elevated privileges for full access to system storage metadata and lock files.

**Resolution:**

```bash
sudo pvs
sudo vgs
sudo lvs
```

The commands were also verified from a root shell.

**Lesson learned:** Distinguish a permissions issue from a storage failure before making configuration changes.

## Lab 03 — User and Privilege Management

**Status: Initial configuration completed**

Created and tested a separation between standard user access and administrative access.

### Accounts

| Account | Purpose |
|---|---|
| niall | Standard day-to-day user |
| admniall | Dedicated administrative account |
| root | Built-in superuser account |

### Administrative Group Configuration

The administrative account was added to the `wheel` group:

```bash
usermod -aG wheel admniall
```

Verified group membership:

```bash
id admniall
```

The output confirmed that `admniall` belonged to the `wheel` group.

### Testing Administrative Privileges

```bash
su - admniall
sudo whoami
sudo pvs
```

The `sudo whoami` command returned `root`, confirming successful privilege elevation.

### Removing Administrative Access from the Standard Account

```bash
sudo gpasswd -d niall wheel
```

This removes the standard user from the administrative group. New login sessions should be used when verifying updated group membership.

**Key learning:** Separating normal user activities from privileged administration supports least-privilege access and reduces the risk of accidental system changes.

## Lab 04 — System Health and Resource Inspection

**Status: Initial inspection completed**

Executed the following commands:

```bash
whoami
hostnamectl
uptime
free -h
df -hT
lsblk
```

These commands provide a baseline view of:
- Current user identity
- Operating system and hostname
- System uptime and load averages
- Memory and swap usage
- Filesystem utilisation
- Disk and logical volume layout

## Lab 05 — Troubleshooting Journal

### Incident 001 — LVM Permission Warning

**Symptom:** Permission denied while executing LVM inspection commands.

**Diagnosis:** Insufficient user privileges.

**Resolution:** Re-ran commands using elevated privileges.

**Outcome:** Physical volumes, volume groups and logical volumes displayed successfully.

### Incident 002 — Incorrect Command Entry

**Symptom:** Bash reported a command not found error.

**Diagnosis:** Incorrect command syntax or spelling.

**Resolution:** Verified the command name and re-executed the correct command.

**Lesson learned:** Read shell error messages carefully and verify command syntax before investigating deeper system issues.

## Planned Training

- Linux filesystem navigation and permissions
- systemd service management
- NetworkManager and nmcli
- SSH configuration and remote administration
- SELinux fundamentals
- firewalld administration
- DNF package management
- LVM expansion and filesystem growth
- Log inspection using journalctl
- CPU, memory and disk monitoring
- Scheduled tasks and automation
- Enterprise incident troubleshooting

## Documentation Approach

Each future exercise will record:

1. Objective and operational context
2. Commands executed
3. Explanation of command options
4. Expected results
5. Observed output and verification
6. Errors encountered and troubleshooting
7. Lessons learned

## Project Status

**Phase 1: RHEL installation and foundational administration — In progress**

Initial installation, storage inspection and administrative account configuration have been completed. Further practical exercises will be added incrementally.

---

This repository is a personal hands-on training environment and technical learning portfolio. It is not a production infrastructure deployment. 

