# Linux File Permissions and Mount Modes — RHEL 9 L1 Triage Guide

A practical reference for understanding Linux `r`, `w`, `x`, `ro`, and `rw`, interpreting permissions, and investigating common access incidents on Red Hat Enterprise Linux 9.

## 1. File permissions: `r`, `w`, `x`

Linux permissions are assigned separately to the **owner**, **group**, and **others**.

| Symbol | Meaning | On a regular file | On a directory |
|---|---|---|---|
| `r` | Read | Read file contents | List directory entry names |
| `w` | Write | Modify file contents | Create, delete, or rename entries **when combined with `x`** |
| `x` | Execute | Execute a program/script (subject to other requirements) | Traverse/enter the directory and access known entries |
| `-` | Permission absent | Access not granted by that permission bit | Access not granted by that permission bit |

> **Directory nuance:** `r` lets you list names, while `x` lets you traverse the directory. Both are typically needed for a useful `ls -l` listing. Directory write access is particularly sensitive because it can allow removal of files even when the user cannot edit their contents; sticky-bit restrictions may apply.

### Read a permission string

```bash
ls -l /etc/passwd
```

Example output (illustrative):

```text
-rw-r--r-- 1 root root 2456 Oct 22 10:30 /etc/passwd
```

Breakdown of `-rw-r--r--`:

| Characters | Applies to | Meaning |
|---|---|---|
| `-` | File type | Regular file (`d` = directory, `l` = symbolic link) |
| `rw-` | Owner | Read and write |
| `r--` | Group | Read only |
| `r--` | Others | Read only |

The owner and group in this example are both `root`.

## 2. Numeric permissions and `chmod`

| Permission | Numeric value |
|---|---:|
| Read (`r`) | 4 |
| Write (`w`) | 2 |
| Execute (`x`) | 1 |
| `rwx` | 7 |
| `rw-` | 6 |
| `r-x` | 5 |
| `r--` | 4 |
| `---` | 0 |

Add the values for **owner / group / others**:

| Mode | Symbolic | Typical use |
|---|---|---|
| `644` | `rw-r--r--` | Regular text/config file readable by others |
| `755` | `rwxr-xr-x` | Executable or traversable directory |
| `600` | `rw-------` | Private user file |
| `700` | `rwx------` | Private directory or executable |

```bash
chmod 755 script.sh     # Owner rwx; group/others r-x
chmod 600 private.txt   # Owner read/write only
chmod u+x script.sh     # Add execute for the owner
chmod g-w file.txt      # Remove group write
chmod o-r file.txt      # Remove read for others
```

`chmod` changes permission bits; it does **not** change ownership.

```bash
ls -l file.txt          # View owner, group, and permissions
stat file.txt           # Detailed metadata, including numeric mode
```

## 3. Ownership and identity

```bash
whoami                 # Current username
id                     # Current UID and group memberships
id myapp               # UID and groups for application account
ls -l file.txt         # File owner and group
ls -ld /var/log/myapp   # Directory owner and permissions
```

Commands that **modify** ownership require appropriate authorisation:

```bash
sudo chown user:group file.txt   # Change owner and group
sudo chgrp group file.txt        # Change group only
```

> On production systems, verify the intended application account and approved permissions before changing ownership or mode. Do not apply recursive `chmod` or `chown` as a quick fix.

## 4. Filesystem mount modes: `ro` versus `rw`

These describe the **mount state**, not the per-file permission bits.

| Mode | Meaning | Effect |
|---|---|---|
| `ro` | Read-only | The filesystem cannot be written through that mount |
| `rw` | Read-write | Writes are possible **if** permissions, ACLs, SELinux, capacity, and other controls allow them |

```bash
findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS
findmnt -T /var/log
mount | grep ' on / '
```

Illustrative output:

```text
TARGET SOURCE                  FSTYPE OPTIONS
/      /dev/mapper/rhel-root   xfs    rw,relatime
/boot  /dev/sda1               xfs    rw,relatime
```

A `rw` mount **does not** grant every user write access to every file. A `ro` mount prevents writes even when file permissions otherwise permit them.

## 5. L1 incident triage: “Permission denied”

**Example ticket:** Application cannot write to `/var/log/myapp/application.log`.

Start with read-only checks:

```bash
ls -l /var/log/myapp/application.log  # File permissions and ownership
ls -ld /var/log/myapp                 # Parent directory permissions
namei -l /var/log/myapp/application.log  # Permissions along the path
id myapp                              # Application account and groups
findmnt -T /var/log/myapp/application.log # Filesystem and mount options
df -h /var/log                        # Available disk space
df -i /var/log                        # Available inodes
ls -Z /var/log/myapp/application.log  # SELinux label
getenforce                            # SELinux enforcement mode
getfacl /var/log/myapp/application.log # ACLs, if available
```

### How to interpret findings

| Finding | Possible explanation | L1 next action |
|---|---|---|
| File lacks write permission for app account | Ownership/mode mismatch | Record current values; compare with expected configuration |
| Parent directory lacks `x` | Application cannot traverse path | Inspect each path component with `namei -l` |
| Mount options include `ro` | Read-only filesystem | Check logs and escalate; do not remount without approval |
| Filesystem at 100% | No free blocks | Capture `df` and identify large paths safely |
| Inodes at 100% | Too many filesystem entries | Capture `df -i`; escalate cleanup decision |
| SELinux enforcing with relevant denials | Mandatory access control blocked action | Capture label and audit evidence; do not disable SELinux |
| ACL differs from basic mode | Additional access rules | Inspect `getfacl` and expected ACL policy |

Additional RHEL log checks:

```bash
sudo journalctl -p warning -b --no-pager -n 50
sudo journalctl -u myapp.service -b --no-pager -n 50
sudo ausearch -m AVC,USER_AVC -ts recent   # If audit tooling is installed
```

> A “Permission denied” error is **not automatically a `chmod` problem**. Gather evidence before making changes.

## 6. Safe hands-on lab exercise

Run these commands as your ordinary lab user in your own home directory:

```bash
cd ~
touch permissions-test.txt
ls -l permissions-test.txt
chmod 600 permissions-test.txt
ls -l permissions-test.txt
chmod 644 permissions-test.txt
ls -l permissions-test.txt
stat permissions-test.txt
```

Expected mode changes:

```text
600  ->  -rw-------
644  ->  -rw-r--r--
```

Optional directory exercise:

```bash
mkdir -p ~/permissions-lab
chmod 700 ~/permissions-lab
ls -ld ~/permissions-lab
chmod 755 ~/permissions-lab
ls -ld ~/permissions-lab
```

Clean up when finished:

```bash
rm ~/permissions-test.txt
rmdir ~/permissions-lab
```

## 7. Quick reference

| Command | L1 purpose | Changes system? |
|---|---|---|
| `ls -l` | View file permissions/owner/group | No |
| `ls -ld` | Inspect directory itself | No |
| `stat` | Detailed file metadata | No |
| `id` | Check user identity and groups | No |
| `namei -l` | Check permissions along path | No |
| `getfacl` | Inspect access control lists | No |
| `findmnt -T PATH` | Identify filesystem and mount mode | No |
| `df -h` / `df -i` | Check space and inodes | No |
| `ls -Z` | Inspect SELinux context | No |
| `getenforce` | Check SELinux mode | No |
| `chmod` | Change file mode | **Yes** |
| `chown` / `chgrp` | Change ownership/group | **Yes** |

### Avoid these shortcuts

- **Do not use `chmod 777`** to “fix” an access error. It grants broad read/write/execute permissions.
- **Do not disable SELinux** to work around an unexplained denial.
- **Do not remount `ro` filesystems as `rw`** without investigating the cause and following change procedures.
- **Do not run recursive permission changes** against production paths without an approved plan.

## 8. Suggested incident handover notes

```text
Incident ID:
Hostname:
Application/service:
Affected path:
Observed error and timestamp:
Effective application user/groups:
File and parent directory permissions:
ACL / SELinux findings:
Filesystem mount mode:
Free space and inode usage:
Relevant journal/audit entries:
Actions taken (read-only diagnostics):
Escalation / next owner:
```

---

**Training context:** RHEL 9 system administration lab; safe first-line diagnostics. Production procedures and escalation rules always take precedence.
