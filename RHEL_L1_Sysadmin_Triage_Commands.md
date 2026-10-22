# RHEL 9 — L1 System Administrator Triage Command Reference

**Purpose:** Read-only, first-response troubleshooting for RHEL 9 and similar enterprise Linux systems.  
**Audience:** L1 IT Operations / Systems Support.  
**Principle:** Confirm the alert, establish scope and impact, collect evidence, and escalate with a concise handover. Do **not** restart services, delete files, change permissions, or modify configuration without approval and an applicable runbook.

> Commands marked `sudo` may require approved privileged access. Replace placeholders such as `<service>` and `<host>` before running commands. Use timestamps and capture output in your ticket. `journalctl` output and application logs may contain sensitive data; redact before sharing.

## 1. Initial health check (first 2–5 minutes)

| Command | What it tells you | What to look for |
|---|---|---|
| `hostnamectl` | Hostname, OS, kernel, architecture | Correct server; expected OS |
| `date -Is` | Timestamp with timezone | Correct clock/timezone for correlation |
| `uptime` | Uptime and load averages (1/5/15 min) | Unusual load; recent reboot |
| `whoami` | Current account | Expected identity |
| `id` | UID and group memberships | Expected access/privileges |
| `w` | Logged-in users and activity | Active sessions; unexpected activity |
| `free -h` | RAM and swap overview | Low *available* memory; heavy swap |
| `df -hT` | Mounted filesystem usage/types | Filesystems near capacity |
| `df -i` | Inode usage | 100% inode exhaustion even with free space |
| `systemctl --failed` | Failed systemd units | Failed services or mounts |
| `ip -br addr` | Interface addresses and state | Down interface; unexpected IP |

**Interpretation:** Load average is not CPU percentage; compare it with CPU count (`nproc`) and process states. Linux uses RAM for cache, so `free` alone is not a memory fault: check the `available` column and signs of swapping. Disk thresholds are environment-specific; follow monitoring policy rather than assuming a universal cutoff.

## 2. CPU, load and process triage

```bash
uptime                        # 1, 5, 15-minute load averages
nproc                         # Available logical CPUs
lscpu                         # CPU architecture and core details
top                           # Live CPU, memory, process view (press q to quit)
ps aux --sort=-%cpu | head -15  # Top CPU-consuming processes
ps -eo pid,ppid,user,stat,%cpu,%mem,etime,cmd --sort=-%cpu | head -20
vmstat 1 5                    # Five samples of CPU, memory, run queue, I/O waits
```

**Look for:** Sustained high CPU; high `wa` (I/O wait); many runnable tasks; process repeatedly consuming resources. `top` is interactive; `ps` is a snapshot. Do not kill processes during L1 triage without authorization.

## 3. Memory and swap

```bash
free -h
cat /proc/meminfo | head -25
swapon --show
vmstat 1 5
ps aux --sort=-%mem | head -15
journalctl -k --since '2 hours ago' --no-pager | grep -Ei 'out of memory|oom|killed process'
```

**Look for:** Very low available memory, sustained swap-in/out (`si`/`so` in `vmstat`), OOM-killer entries, and a growing memory-intensive process. Swap *allocated* does not necessarily mean active swapping.

## 4. Disk, filesystems and LVM

```bash
df -hT                        # Space and filesystem types
df -i                         # Inodes
lsblk -f                      # Disks, partitions, filesystems, mounts
findmnt                       # Mounted filesystem tree
sudo pvs                      # LVM physical volumes
sudo vgs                      # LVM volume groups and free extents
sudo lvs -a -o +devices       # LVM logical volumes and backing devices
sudo du -xhd1 /var 2>/dev/null | sort -h  # Space by immediate child of /var
sudo journalctl --disk-usage  # Journal disk footprint
```

**Look for:** Full `/`, `/var`, `/home` or application mounts; inode exhaustion; missing mounts; little VG free space. `du` can be slow on large trees; avoid scanning huge filesystems during an incident unless needed. **Do not** run `rm`, `lvextend`, `xfs_growfs`, or filesystem repair as part of unapproved L1 triage.

## 5. Services and systemd

```bash
systemctl --failed
systemctl status <service> --no-pager -l
systemctl is-active <service>
systemctl is-enabled <service>
systemctl list-units --type=service --state=running --no-pager
journalctl -u <service> --since '1 hour ago' --no-pager
journalctl -u <service> -n 100 --no-pager
systemctl show <service> -p ActiveState -p SubState -p MainPID -p ExecMainStatus
```

**Look for:** Failed/inactive status, recent restarts, exit codes, dependency failures and error messages. `enabled` means configured to start at boot, **not** currently running. Service names vary (`sshd`, `chronyd`, `NetworkManager`, etc.).

## 6. System logs and recent changes

```bash
journalctl -p warning..alert --since '2 hours ago' --no-pager
journalctl -k --since '2 hours ago' --no-pager
journalctl -b -p err --no-pager
journalctl --list-boots
last -x | head -25          # Recent logins, reboots, shutdowns
who -b                      # Last system boot
sudo tail -n 100 /var/log/messages  # If rsyslog is installed/configured
sudo tail -n 100 /var/log/secure    # If present; authentication-related events
```

**Look for:** First error before the alert, boot/restart events, I/O errors, authentication failures, SELinux denials and related service errors. RHEL deployments may rely primarily on the systemd journal; not every `/var/log/*` file will exist.

## 7. Networking and connectivity

```bash
ip -br addr
ip route
ip -s link
nmcli device status
nmcli connection show --active
resolvectl status            # If systemd-resolved is enabled
cat /etc/resolv.conf
getent hosts <hostname>      # Resolver test through system NSS
ping -c 4 <gateway-or-host>  # ICMP may be filtered
tracepath <host>             # If tracepath is installed
ss -tulpen                   # Listening TCP/UDP sockets and owning processes
ss -tan state established    # Established TCP connections
curl -I --max-time 10 https://<approved-host>  # HTTP(S) response headers
```

**Look for:** Missing IP/default route, interface errors, DNS failure, port not listening, connection timeout or application-level HTTP errors. `ping` failure alone does not prove a host is down. `curl` to internal endpoints should use approved destinations. `resolvectl` and `tracepath` may not be installed or applicable.

### Quick network isolation sequence

1. Check local address: `ip -br addr`.
2. Check default route: `ip route`.
3. Check DNS lookup: `getent hosts <hostname>`.
4. Check application listener locally: `ss -tulpen`.
5. Check approved remote endpoint: `curl -I --max-time 10 https://<host>`.
6. Record which step failed and whether the issue affects one host or multiple hosts.

## 8. SSH and access incidents

```bash
systemctl status sshd --no-pager
ss -tlnp | grep ':22 '
journalctl -u sshd --since '1 hour ago' --no-pager
id <username>
getent passwd <username>
groups <username>
sudo -l                     # Lists *your* permitted sudo commands
sudo faillock --user <username>  # If faillock is configured and access approved
```

**Look for:** SSH service state, listening port, authentication errors, locked accounts, wrong group membership and network reachability. Do not unlock accounts or change SSH configuration without the access-management process.

## 9. SELinux and firewall checks (RHEL)

```bash
getenforce
sestatus
sudo ausearch -m AVC,USER_AVC -ts recent  # If audit tools/logs available
sudo firewall-cmd --state                   # If firewalld is running
sudo firewall-cmd --list-all                # Active default-zone rules
sudo firewall-cmd --get-active-zones
```

**Look for:** SELinux enforcing mode, relevant AVC denials, missing allowed service/port, unexpected zone assignment. **Never** disable SELinux or the firewall as a shortcut to diagnosis.

## 10. Packages, patching and scheduled work

```bash
rpm -q <package>
rpm -qa | grep -i <keyword>
dnf history list | head -25
uname -r
cat /etc/redhat-release
systemctl list-timers --all --no-pager
crontab -l                   # Current user's crontab
sudo ls -l /etc/cron.d /etc/cron.daily  # Scheduled task locations
```

**Look for:** Recent package changes, kernel mismatch after reboot, scheduled tasks overlapping the incident window. `dnf check-update` may contact repositories and change metadata; use only if appropriate for your environment.

## 11. Performance and hardware indicators

```bash
vmstat 1 5
cat /proc/loadavg
journalctl -k -p warning --since '4 hours ago' --no-pager
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

If installed and approved:

```bash
iostat -xz 1 5             # From sysstat; storage latency/utilisation
sar -u 1 5                 # From sysstat; CPU trends
sar -r 1 5                 # Memory trends
```

**Look for:** Repeated disk I/O errors, high device latency, sustained I/O wait and a clear change from normal baselines. A missing utility is not itself a system fault.

## 12. Common alerts: first checks

| Alert | First commands | Typical escalation evidence |
|---|---|---|
| Host unreachable | `ip -br addr`, `ip route`, `ping -c 4 <host>` | Scope, last successful contact, route/DNS tests |
| CPU high | `uptime`, `nproc`, `top`, `ps aux --sort=-%cpu` | Duration, top PID/process, load vs CPU count |
| Memory high | `free -h`, `vmstat 1 5`, `ps aux --sort=-%mem` | Available RAM, swap activity, top process |
| Filesystem full | `df -hT`, `df -i`, `lsblk -f` | Affected mount, usage %, inode %, growth evidence |
| Service down | `systemctl status <service>`, `journalctl -u <service> -n 100` | State, exit code, first error, impact |
| SSH failed | `systemctl status sshd`, `journalctl -u sshd`, `id <user>` | Exact error, username, source, reachability |
| DNS issue | `getent hosts <host>`, `cat /etc/resolv.conf` | Lookup result, affected names, other hosts affected |
| Network/port issue | `ip route`, `ss -tulpen`, `curl -I --max-time 10 <url>` | Route, listener, timeout/refusal/HTTP code |
| Recent reboot | `uptime`, `who -b`, `last -x`, `journalctl -b -p err` | Boot time, preceding errors, change window |
| SELinux denial | `getenforce`, `sudo ausearch -m AVC -ts recent` | Denial timestamp, process, target context |

## 13. L1 incident workflow and handover template

**A. Validate:** Alert name, hostname, timestamp, severity, whether still firing.  
**B. Scope:** Single host, service, site, or multiple systems? User/business impact?  
**C. Correlate:** Recent changes, deployments, patching, maintenance, reboots.  
**D. Collect:** Run the smallest relevant read-only command set and capture errors.  
**E. Act:** Follow an approved runbook only; otherwise escalate.  
**F. Communicate:** Record updates and hand over evidence.

```text
Incident / Ticket:
Alert and severity:
Host / environment:
Start time and timezone:
Business impact / affected users:
Current state (ongoing / recovered / intermittent):
Checks performed (commands + times):
Key findings (include exact error messages):
Recent changes / related incidents:
Actions taken (runbook / approval reference):
Escalation team and next requested action:
```

## 14. Commands to avoid without authorization

- `systemctl restart|stop|disable <service>` — can interrupt production.
- `kill`, `kill -9`, `pkill` — can terminate business workloads.
- `rm -rf`, `find ... -delete`, log truncation — destroys evidence/data.
- `chmod`, `chown`, `usermod`, `passwd`, `faillock --reset` — changes access or ownership.
- `lvextend`, `lvremove`, `mkfs`, `fsck`, `mount`, `umount` — changes storage or availability.
- `setenforce 0`, `systemctl stop firewalld` — weakens security controls.
- `dnf upgrade`, `reboot`, network configuration changes — require change approval.

---

**Portfolio note:** Commands are examples for an authorized enterprise environment. Adapt service names, monitoring thresholds, escalation paths and access controls to your organisation's procedures. Document *observed* results separately from expected examples.
