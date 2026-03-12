# Linux Permission Management

## Understanding File Permissions

Every file and directory in Linux has three types of permissions for three categories of users.

### Permission Categories
| Category | Symbol | Description |
|----------|--------|-------------|
| Owner (User) | `u` | The file's creator/owner |
| Group | `g` | Users belonging to the file's group |
| Others | `o` | Everyone else |
| All | `a` | Owner + Group + Others |

### Permission Types
| Permission | Symbol | Numeric | Files | Directories |
|------------|--------|---------|-------|-------------|
| Read | `r` | `4` | View file contents | List directory contents |
| Write | `w` | `2` | Modify file | Create/delete files inside |
| Execute | `x` | `1` | Run as program | Enter (`cd`) directory |
| No Permission | `-` | `0` | — | — |

### Reading Permission Output
```bash
ls -l filename
# Output example:
# -rwxr-xr-- 1 alice devs 1024 Mar 11 10:00 script.sh
#  |||||||||||
#  |└──┬──┘└──┬──┘└──┬──┘
#  |  owner  group  others
#  |
#  └─ File type: - = file, d = directory, l = symlink
```

---

## `chmod` — Change File Permissions

### Symbolic Mode
```bash
# Syntax: chmod [who][operator][permission] file

# Add execute permission for owner
chmod u+x script.sh

# Remove write permission from group
chmod g-w file.txt

# Add read & write for owner, remove execute from others
chmod u+rw,o-x file.txt

# Set exact permissions for all (owner=rwx, group=rx, others=r)
chmod u=rwx,g=rx,o=r file.txt

# Add execute for everyone
chmod a+x script.sh

# Remove all permissions for others
chmod o= file.txt
```

### Numeric (Octal) Mode
```bash
# Syntax: chmod [octal] file
# Calculate: r=4, w=2, x=1 → add values for each category

chmod 777 file.txt      # rwxrwxrwx — full access for all (AVOID in production)
chmod 755 script.sh     # rwxr-xr-x — owner full, group/others read+execute
chmod 644 file.txt      # rw-r--r-- — owner read+write, others read only
 
# Apply recursively to directory and its contents
chmod -R 755 /var/www/html/

# Verbose output (show what was changed)
chmod -v 644 file.txt

```

---

## `chown` — Change File Owner and Group

```bash
# Syntax: chown [owner][:group] file

# Change owner only
chown alice file.txt

# Change owner and group
chown alice:devs file.txt

# Change group only (note the colon prefix)
chown :devs file.txt

# Change owner recursively
chown -R alice /home/alice/

# Change owner and group recursively
chown -R alice:devs /var/www/project/

# Verbose output
chown -v alice:devs file.txt

# Only change if the current owner matches
chown --from=bob alice file.txt
```

---

## `chgrp` — Change Group Ownership

```bash
# Change group of a file
chgrp devs file.txt

# Change group recursively
chgrp -R devs /var/www/html/

# Verbose output
chgrp -v devs file.txt


```
---

## `umask` — Default Permission Mask

```bash
# View current umask
umask            # e.g., 0022
umask -S         # Symbolic: u=rwx,g=rx,o=rx

# How umask works:
# Files:   666 - umask = default file permissions
# Dirs:    777 - umask = default directory permissions
# umask 022 → files: 644, dirs: 755
# umask 027 → files: 640, dirs: 750
# umask 077 → files: 600, dirs: 700

# Set umask for current session
umask 022
umask 027     # More restrictive: no perms for others

# Set umask permanently (add to ~/.bashrc or /etc/profile)
echo "umask 022" >> ~/.bashrc
```
---

## Quick Reference Cheat Sheet

```bash
# ── View Permissions ──────────────────────────────
ls -l file             # Long listing with permissions
ls -la                 # Include hidden files
stat file              # Detailed file info

# ── Change Permissions ────────────────────────────
chmod 755 file         # Numeric mode
chmod u+x file         # Symbolic mode
chmod -R 644 dir/      # Recursive

# ── Change Ownership ──────────────────────────────
chown user file        # Change owner
chown user:group file  # Change owner and group
chown :group file      # Change group only
chgrp group file       # Change group


# ── Default Permissions ───────────────────────────
umask                  # View current umask
umask 022              # Set umask



```
