# SSH Configuration on Rocky Linux
### Password Authentication & Key-Based Authentication

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Install & Enable SSH Server](#1-install--enable-ssh-server)
3. [SSH with Password Authentication](#2-ssh-with-password-authentication)
4. [SSH with Key-Based Authentication](#3-ssh-with-key-based-authentication)
5. [Harden SSH Configuration](#4-harden-ssh-configuration)
6. [Firewall Configuration](#5-firewall-configuration)
7. [Troubleshooting](#6-troubleshooting)
8. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| OS | Rocky Linux 8 / 9 |
| User | Root or sudo privileges |
| Network | SSH port (default: 22) accessible |
| Client | Any SSH client (Linux/macOS terminal, Windows PuTTY/WSL) |

---

## 1. Install & Enable SSH Server

### Step 1.1 — Update the system

```bash
sudo dnf update -y
```

### Step 1.2 — Install OpenSSH Server

```bash
sudo dnf install -y openssh-server openssh-clients
```

### Step 1.3 — Start and enable the SSH service

```bash
# Start SSH daemon
sudo systemctl start sshd

# Enable SSH to auto-start on boot
sudo systemctl enable sshd
```

### Step 1.4 — Verify SSH service is running

```bash
sudo systemctl status sshd
```

Expected output should show:
```
● sshd.service - OpenSSH server daemon
     Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled)
     Active: active (running)
```

---

## 2. SSH with Password Authentication

### Step 2.1 — Enable password authentication (Server Side)

Edit the SSH configuration file:

```bash
sudo nano /etc/ssh/sshd_config
```

Ensure the following lines are set:

```
PasswordAuthentication yes
PermitRootLogin no        # Recommended: disable direct root login
```

### Step 2.2 — Restart SSH to apply changes

```bash
sudo systemctl restart sshd
```

### Step 2.3 — Create a new user (if needed)

```bash
# Create user
sudo useradd -m samrat

# Set a strong password
sudo passwd samrat1234
```

### Step 2.4 — Connect from client using password

```bash
ssh samrat@<SERVER_IP>
```

**Example:**
```bash
ssh samrat@192.168.226.131
```

You will be prompted:
```
samrat@192.168.226.131's password:
```

---

## 3. SSH with Key-Based Authentication

Key-based authentication is more secure than passwords. It uses a **public/private key pair**.

---

### Step 3.1 — Generate SSH Key Pair (Client Side)

Run this on your **local machine** (not the server):

```bash
ssh-keygen -t ed25519 -C "Samrat"
```

> For legacy systems that don't support Ed25519, use RSA:
> ```bash
> ssh-keygen -t rsa -b 4096 -C "Samrat"
> ```

Follow the prompts:
```
Enter file in which to save the key (/home/user/.ssh/id_ed25519): [Press Enter for default]
Enter passphrase (empty for no passphrase): [Recommended: set a passphrase]
Enter same passphrase again:
```

This generates two files:
| File | Description |
|------|-------------|
| `~/.ssh/id_ed25519` | **Private key** — Keep this secret, never share |
| `~/.ssh/id_ed25519.pub` | **Public key** — Copy this to the server |

### Step 3.2 — Verify keys were created

```bash
ls -la ~/.ssh/
```

```
-rw-------  1 user user  411 Jan 01 10:00 id_ed25519
-rw-r--r--  1 user user  105 Jan 01 10:00 id_ed25519.pub
```

### Step 3.3 — Copy Public Key to Server

#### Method A — Using `ssh-copy-id` (Recommended)

```bash
ssh-copy-id samrat@192.168.226.131
```

Enter the user's password when prompted. The public key will be appended to `~/.ssh/authorized_keys` on the server.


### Step 3.4 — Set correct permissions on server (if not set)

```bash
# On the SERVER — critical for SSH to accept the key
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

> SSH will silently reject keys if permissions are too open.

### Step 3.5 — Enable key authentication on the server

```bash
sudo vi /etc/ssh/sshd_config
```

Ensure these lines are set:

```
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
```

Restart SSH:

```bash
sudo systemctl restart sshd
```

### Step 3.6 — Connect using SSH key

```bash
ssh samrat@192.168.226.131
```

SSH will automatically use your key. If you set a passphrase, you'll be prompted for it (this is the **key passphrase**, not the user's system password).

### Step 3.7 — Specify a key explicitly

```bash
ssh -i ~/.ssh/id_ed25519 samrat@192.168.226.131
```

---

## 4. Harden SSH Configuration

After setting up key authentication, harden the server by editing `/etc/ssh/sshd_config`:

```bash
sudo vi /etc/ssh/sshd_config
```

Apply the following hardened settings:

# Disable root login completely
PermitRootLogin no

# Disable password authentication (force key-only)
PasswordAuthentication no

# Limit authentication attempts
MaxAuthTries 3

```

Apply changes:

```bash
# Validate configuration before restarting
sudo sshd -t

# Restart only if no errors reported
sudo systemctl restart sshd
```

---

## 5. Firewall Configuration

### Step 5.1 — Allow SSH through firewalld (default port 22)

```bash
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```



## 6. Troubleshooting

### Check SSH service status

```bash
sudo systemctl status sshd
sudo journalctl -u sshd -n 50 --no-pager
```

### View live authentication logs

```bash
# Rocky Linux 9
sudo tail -f /var/log/secure

# Or using journalctl
sudo journalctl -fu sshd
```


### Verify SSH port is listening

```bash
sudo ss -tlnp | grep sshd
# or
sudo netstat -tlnp | grep :22
```

### Fix permission issues on client

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/config
```

### Common Error Messages

| Error | Cause | Fix |
|-------|-------|-----|
| `Connection refused` | sshd not running or port blocked | Start sshd, check firewall |
| `Permission denied (publickey)` | Wrong key or bad permissions | Check `authorized_keys` and file permissions |
| `Host key verification failed` | Server key changed | Run `ssh-keygen -R <IP>` |
| `Too many authentication failures` | Too many keys tried | Use `-i` to specify exact key |
| `No route to host` | Network issue or firewall | Check network/firewall rules |
| `WARNING: UNPROTECTED PRIVATE KEY FILE!` | Key permissions too open | `chmod 600 ~/.ssh/id_ed25519` |

---

## 7. Quick Reference Cheat Sheet

```bash
# ── SERVER SETUP ──────────────────────────────────────────────
sudo dnf install -y openssh-server          # Install SSH server
sudo systemctl enable --now sshd            # Start & enable
sudo systemctl restart sshd                 # Restart after config changes
sudo sshd -t                                # Test config for syntax errors

# ── KEY GENERATION (Client) ───────────────────────────────────
ssh-keygen -t ed25519 -C "comment"          # Generate Ed25519 key pair
ssh-keygen -t rsa -b 4096                   # Generate RSA key pair (legacy)
ssh-keygen -t ed25519 -C "personal" -f ~/.ssh/personal_key

# ── COPY KEY TO SERVER ────────────────────────────────────────
ssh-copy-id user@server                     # Copy public key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server  # Specify key file

# ── CONNECT ───────────────────────────────────────────────────
ssh samrat@192.168.226.131                      # Connect (default port 22)
ssh -p 2222 samrat@192.168.226.131              # Connect on custom port
ssh -i ~/.ssh/id_ed25519 user@192.168.1.100 # Specify key file explicitly

# ── FIREWALL ──────────────────────────────────────────────────
sudo firewall-cmd --permanent --add-service=ssh     # Allow SSH (port 22)
sudo firewall-cmd --permanent --add-port=2222/tcp   # Allow custom port
sudo firewall-cmd --reload                          # Apply rules

emanage port -a -t ssh_port_t -p tcp 2222     # Allow port in SELinux
```

---

## Security Summary

| Setting | Recommended Value |
|---------|------------------|
| Default port | Change from 22 to a high port (e.g., 2222) |
| Root login | `PermitRootLogin no` |
| Password auth | `PasswordAuthentication no` (use keys only) |
| Key type | Ed25519 (preferred) or RSA 4096-bit |
| Key passphrase | Always set one |
| Allowed users | Restrict with `AllowUsers` |
| Idle timeout | `ClientAliveInterval 300` |

---


