# RHEL 9 — L1 System Administrator Command Cheat Sheet

A practical, primarily read-only command reference for first-line Linux incident triage. Commands may require `sudo` depending on your access. Always follow change control and escalation procedures on production systems.

## 1. Files and directories

| Command | Purpose |
|---|---|
| `ls` | List directory contents. |
| `ls -l` | Show permissions, ownership, size and modification time. |
| `ls -la` | Include hidden files. |
| `ls -lh` | Show human-readable file sizes. |
| `ls -ltr` | Sort by modification time, oldest first. |
| `ls -ld /var/log` | Inspect the directory itself rather than its contents. |
| `pwd` | Show the current working directory. |
| `cd /var/log` | Change directory. |
| `cd ..` | Move to the parent directory. |
| `file filename` | Identify a file's type. |
| `stat filename` | Show detailed file metadata. |
| `find /var/log -name '*.log'` | Find files matching a name pattern. |

## 2. Reading files and logs

| Command | Purpose |
|---|---|
| `cat filename` | Print a file (best for short files). |
| `less filename` | Read a file page by page (`q` to quit). |
| `head filename` | Show first ten lines. |
| `tail filename` | Show last ten lines. |
| `tail -n 50 filename` | Show last fifty lines. |
| `tail -f filename` | Follow appended log lines (`Ctrl+C` to stop). |
| `grep 'error' filename` | Search for matching text. |
| `grep -i 'error' filename` | Search without case sensitivity. |
| `grep -n 'failed' filename` | Include line numbers. |
| `wc -l filename` | Count lines. |

## 3. Users and permissions

| Command | Purpose |
|---|---|
| `whoami` | Show current effective username. |
| `id` | Show current user ID and group membership. |
| `id username` | Inspect another account. |
| `groups` | List groups for current user. |
| `who` | Show logged-in sessions. |
| `w` | Show sessions and activity. |
| `getent passwd username` | Look up an account in configured identity sources. |
| `ls -l filename` | Inspect file permissions and ownership. |
| `ls -ld directory` | Inspect directory permissions. |
| `getfacl filename` | Show filesystem access control lists. |
| `ls -Z filename` | Show SELinux security context. |
| `sudo -l` | Show allowed sudo commands. |

**Permission reminder:** `r` = read, `w` = write, `x` = execute/traverse. `644` = `rw-r--r--`; `755` = `rwxr-xr-x`. A filesystem mounted `rw` can still deny writes through file permissions, ACLs or SELinux.

## 4. Storage and LVM

| Command | Purpose |
|---|---|
| `df -h` | Show mounted filesystem space usage. |
| `df -hT` | Include filesystem type. |
| `du -sh /var/log` | Estimate directory disk usage. |
| `du -h --max-depth=1 /var` | Compare immediate subdirectories. |
| `lsblk` | Show block devices and hierarchy. |
| `lsblk -f` | Show filesystem types and mount points. |
| `sudo pvs` | Show LVM physical volumes. |
| `sudo vgs` | Show volume groups and free extents. |
| `sudo lvs` | Show logical volumes. |
| `findmnt` | Show mounts and mount options. |
| `findmnt -T /var` | Identify filesystem containing `/var`. |
| `swapon --show` | Show active swap. |

**My lab:** `/dev/sda1` contains `/boot` outside LVM; `/dev/sda2` is an LVM physical volume supporting VG `rhel`, with logical volumes for `/`, `/var`, `/home` and swap. Check current free space using `sudo vgs` rather than assuming it has not changed.

## 5. CPU, memory and processes

| Command | Purpose |
|---|---|
| `uptime` | Show uptime and load averages. |
| `free -h` | Show memory and swap statistics. |
| `top` | Interactive process and resource view (`q` to quit). |
| `ps aux` | List processes and resource use. |
| `ps -ef` | List processes with parent IDs. |
| `ps -eo pid,comm,%cpu,%mem --sort=-%cpu` | Sort processes by CPU use. |
| `pgrep -a sshd` | Find matching processes. |
| `vmstat 1 5` | Collect five system-activity samples. |
| `nproc` | Show available processing units. |

## 6. Services and systemd

| Command | Purpose |
|---|---|
| `systemctl status sshd` | Inspect SSH service state. |
| `systemctl is-active sshd` | Check whether service is active. |
| `systemctl is-enabled sshd` | Check boot-time enablement. |
| `systemctl --failed` | List failed units. |
| `systemctl list-units --type=service` | List loaded service units. |
| `journalctl -b` | Read current-boot journal. |
| `journalctl -u sshd` | Read SSH unit logs. |
| `journalctl -u sshd -n 50` | Read latest 50 SSH log entries. |
| `journalctl -p err -b` | Read current-boot error-priority messages. |
| `journalctl -f` | Follow new journal messages. |

## 7. Networking

| Command | Purpose |
|---|---|
| `ip addr` | Show network interfaces and IP addresses. |
| `ip -br addr` | Compact IP address view. |
| `ip route` | Show routes and default gateway. |
| `nmcli device status` | Show NetworkManager device status. |
| `nmcli connection show` | Show configured connections. |
| `ping -c 4 10.0.2.2` | Test reachability of your lab NAT gateway. |
| `ss -tuln` | Show listening TCP and UDP sockets. |
| `sudo ss -tulpn` | Include process information where permitted. |
| `curl -I https://example.com` | Request HTTP headers. |
| `getent hosts example.com` | Test hostname lookup. |

## 8. RHEL security and packages

| Command | Purpose |
|---|---|
| `getenforce` | Show SELinux enforcing/permissive/disabled state. |
| `sestatus` | Show SELinux status details. |
| `ls -Z /var/log` | Show SELinux contexts. |
| `sudo firewall-cmd --state` | Check firewalld state. |
| `sudo firewall-cmd --list-all` | Show active/default-zone firewall configuration. |
| `sudo dnf check-update` | Check for updates; exit code 100 means updates are available. |
| `rpm -q openssh-server` | Check installed RPM package version. |

## 9. First ten commands to practise

Run these on `rhel-lab01` and explain the output in your own words:

```bash
hostnamectl
uptime
free -h
df -hT
lsblk
ls -la
id
systemctl --failed
journalctl -p err -b
ip -br addr
```

For each command, ask:
1. What does it tell me?
2. What would abnormal output look like?
3. What should I check next or escalate?

## 10. Example L1 triage flow: `/var` is 92% full

```bash
df -hT /var
findmnt -T /var
sudo vgs
sudo lvs
du -h --max-depth=1 /var 2>/dev/null
```

Record the timestamp, hostname, alert, observed utilisation, mount type, relevant logs, and whether usage is growing. **Do not delete logs, extend LVM, change permissions, restart services or remount filesystems without authorisation.** Escalate with evidence and the applicable runbook.

## 11. Example L1 triage flow: application cannot write a log

```bash
ls -ld /var/log/myapp
ls -l /var/log/myapp/app.log
id myapp
getfacl /var/log/myapp/app.log
findmnt -T /var/log/myapp/app.log
ls -Z /var/log/myapp/app.log
getenforce
```

Replace `myapp` and the paths with actual values. Investigate ownership, permission bits, directory traversal, ACLs, mount mode and SELinux. Avoid `chmod 777` as a shortcut.

---

**Scope:** RHEL 9 lab learning and L1 triage. Examples are not a substitute for your organisation's production procedures.
