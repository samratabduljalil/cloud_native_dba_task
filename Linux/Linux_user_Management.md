# Linux User Management — Complete Guide

A comprehensive reference for managing users and groups on Linux systems.

---

## Table of Contents

- [View Users](#-view-users)
- [Create Users](#-create-users)
- [Modify Users](#-modify-users)
- [Delete Users](#-delete-users)
- [Password Management](#-password-management)
- [Group Management](#-group-management)
- [Sudo & Privileges](#-sudo--privileges)
- [Switch Users](#-switch-users)
- [User Information](#-user-information)
- [Lock & Unlock Accounts](#-lock--unlock-accounts)
- [Account Expiry](#-account-expiry)

---

## View Users

### List all users on the system
```bash
cat /etc/passwd
```


### List currently logged-in users
```bash
who
```

### List all logged-in users (detailed)
```bash
w
```

### Show current logged-in username
```bash
whoami
```


---

## Create Users

### Create a new user
```bash
sudo useradd username
```

### Create a user with a home directory
```bash
sudo useradd -m username
```

### Create a user with a specific UID
```bash
sudo useradd -m -u 1500 username
```

### Create a user and assign a primary group
```bash
sudo useradd -m -g groupname username
```

### Create a user with multiple supplementary groups
```bash
sudo useradd -m -G group1,group2,group3 username
```

### Create a user with a custom home directory path
```bash
sudo useradd -m -d /custom/home/path username
```

### Create a user with an expiry date
```bash
sudo useradd -m -e 2025-12-31 username
```

### See user info like expiry date 
```bash
sudo chage -l username
```

### Create a system user (no home directory, no login)
```bash
sudo useradd --system --no-create-home username
```

---

## Modify Users

### Rename a user
```bash
sudo usermod -l newusername oldusername
```

### Change a user's home directory
```bash
sudo usermod -d /new/home/path username
```

### Move user to a new home directory (and move files)
```bash
sudo usermod -d /new/home/path -m username
```

### Change a user's default shell
```bash
sudo usermod -s /bin/bash username
```

### Change a user's primary group
```bash
sudo usermod -g newgroup username
```

### Add user to a supplementary group
```bash
sudo usermod -aG groupname username
```

### Add user to multiple groups
```bash
sudo usermod -aG group1,group2,group3 username
```

### Remove user from a group
```bash
sudo gpasswd -d username groupname
```

### Change a user's UID
```bash
sudo usermod -u 2000 username
```

---

## Delete Users

### Delete a user (keep home directory)
```bash
sudo userdel username
```

### Delete a user and their home directory
```bash
sudo userdel -r username
```

---

## Password Management

### Set or change a user's password
```bash
sudo passwd username
```

### Change your own password
```bash
passwd
```


### Force user to change password on next login
```bash
sudo passwd -e username
```

### Set password expiry (days until password expires)
```bash
sudo chage -M 90 username
```

### Set minimum days between password changes
```bash
sudo chage -m 7 username
```

### Set warning days before password expires
```bash
sudo chage -W 14 username
```

### View password aging information
```bash
sudo chage -l username
```

### Remove password expiry (password never expires)
```bash
sudo chage -M -1 username
```

### Delete a user's password (disable password login)
```bash
sudo passwd -d username
```

---

## Group Management

### List all groups
```bash
cat /etc/group
```

### List groups a user belongs to
```bash
groups username
```

### Show group IDs for a user
```bash
id username
```

### Create a new group
```bash
sudo groupadd groupname
```

### Create a group with a specific GID
```bash
sudo groupadd -g 1500 groupname
```

### Rename a group
```bash
sudo groupmod -n newgroupname oldgroupname
```

### Change a group's GID
```bash
sudo groupmod -g 2000 groupname
```

### Delete a group
```bash
sudo groupdel groupname
```

### Set the group administrator
```bash
sudo gpasswd -A username groupname
```

### List members of a group
```bash
getent group groupname
```

---

## Sudo & Privileges


### Add user to wheel group (RHEL/CentOS/Fedora)
```bash
sudo usermod -aG wheel username
```

### Edit sudoers file safely
```bash
sudo visudo
```

### Grant a user full sudo privileges (add to sudoers)
```bash
echo "username ALL=(ALL:ALL) ALL" | sudo tee -a /etc/sudoers.d/username
```

### Grant a user passwordless sudo
```bash
echo "username ALL=(ALL) NOPASSWD:ALL" | sudo tee -a /etc/sudoers.d/username
```

### Allow user to run a specific command with sudo
```bash
echo "username ALL=(ALL) NOPASSWD:/usr/bin/apt" | sudo tee -a /etc/sudoers.d/username
```

### Check if a user has sudo access
```bash
sudo -l -U username
```

### Remove sudo access from a user
```bash
sudo gpasswd -d username wheel    # RHEL/CentOS
```

---

## User Information

### Show user ID info (UID, GID, groups)
```bash
id username
```

### Show login history
```bash
last username
```

### Show last login time for all users
```bash
lastlog
```

### Show failed login attempts
```bash
sudo lastb
```

### Show current user's info
```bash
id
```

### Show account details from /etc/passwd
```bash
getent passwd username
```

---

##  Lock & Unlock Accounts

### Lock a user account
```bash
sudo passwd -l username
```

### Unlock a user account
```bash
sudo passwd -u username
```

### Lock account using usermod
```bash
sudo usermod -L username
```

### Unlock account using usermod
```bash
sudo usermod -U username
```

### Disable login by setting shell to nologin
```bash
sudo usermod -s /sbin/nologin username
```

### Re-enable login by restoring shell
```bash
sudo usermod -s /bin/bash username
```

### Check if an account is locked
```bash
sudo passwd -S username
```

---

## Account Expiry

### Set account expiry date
```bash
sudo usermod -e 2025-12-31 username
```

### Set account expiry using chage
```bash
sudo chage -E 2025-12-31 username
```

### Remove account expiry (never expires)
```bash
sudo chage -E -1 username
```

### View account expiry details
```bash
sudo chage -l username
```

### Expire an account immediately
```bash
sudo chage -E 0 username
```

---


