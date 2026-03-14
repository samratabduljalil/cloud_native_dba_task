# Apache, Local DNS & Load Balancing on Rocky Linux
> After completing this guide, typing `my.com` routes through a **Load Balancer VM** which distributes traffic across **2 Backend Web Server VMs**.

---

## Architecture Overview

```
Your Browser
     │
     ▼
┌─────────────────────────────┐
│   VM1 — Load Balancer       │  192.168.1.100  (my.com resolves here)
│   Apache + mod_proxy        │
└─────────────┬───────────────┘
              │  Round-Robin
     ┌────────┴────────┐
     ▼                 ▼
┌──────────────┐  ┌──────────────┐
│ VM2 Backend1 │  │ VM3 Backend2 │
│ 192.168.1.101│  │ 192.168.1.102│
│  Apache httpd│  │  Apache httpd│
└──────────────┘  └──────────────┘
```

| VM   | Role          | IP Address      |
|------|---------------|-----------------|
| VM1  | Load Balancer | 192.168.1.100   |
| VM2  | Backend 1     | 192.168.1.101   |
| VM3  | Backend 2     | 192.168.1.102   |

> **Replace the IPs above with your actual VM IPs.** Run `hostname -I` on each VM to find them.

---

## PART A — Backend Web Servers (Run on VM2 AND VM3)

> Run all commands in this section on **both VM2 and VM3** individually.

### A1 — Switch to Root

```bash
sudo su -
```

### A2 — Install Apache

```bash
dnf install httpd -y

systemctl start httpd
systemctl enable httpd
systemctl status httpd
```

### A3 — Configure Firewall

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
firewall-cmd --list-all
```

### A4 — Create Website Files

On **VM2** (Backend 1):

```bash
mkdir -p /var/www/my.com/html
chown -R apache:apache /var/www/my.com/html
chmod -R 755 /var/www/my.com

cat > /var/www/my.com/html/index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>my.com — Backend 1</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; padding: 80px; background: #e8f5e9; }
        h1   { color: #2e7d32; }
        p    { color: #555; font-size: 1.2em; }
    </style>
</head>
<body>
    <h1>✅ Welcome to my.com</h1>
    <p>You are being served by <strong>Backend Server 1 (VM2)</strong></p>
    <p>IP: 192.168.1.101</p>
</body>
</html>
EOF
```

On **VM3** (Backend 2):

```bash
mkdir -p /var/www/my.com/html
chown -R apache:apache /var/www/my.com/html
chmod -R 755 /var/www/my.com

cat > /var/www/my.com/html/index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>my.com — Backend 2</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; padding: 80px; background: #e3f2fd; }
        h1   { color: #1565c0; }
        p    { color: #555; font-size: 1.2em; }
    </style>
</head>
<body>
    <h1>✅ Welcome to my.com</h1>
    <p>You are being served by <strong>Backend Server 2 (VM3)</strong></p>
    <p>IP: 192.168.1.102</p>
</body>
</html>
EOF
```

> The different colors/labels help you visually confirm load balancing is working.

### A5 — Create Virtual Host Config (Both VM2 & VM3)

```bash
cat > /etc/httpd/conf.d/my.com.conf << 'EOF'
<VirtualHost *:80>
    ServerName   my.com
    ServerAlias  www.my.com
    DocumentRoot /var/www/my.com/html

    <Directory /var/www/my.com/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog  /var/log/httpd/my.com_error.log
    CustomLog /var/log/httpd/my.com_access.log combined
</VirtualHost>
EOF

apachectl configtest
systemctl reload httpd
```

### A6 — SELinux (Both VM2 & VM3)

```bash
restorecon -Rv /var/www/my.com/html
chcon -Rt httpd_sys_content_t /var/www/my.com/html
getenforce
```

### A7 — Quick Test on Each Backend

```bash
# On VM2 — should return Backend 1 page
curl http://192.168.1.101

# On VM3 — should return Backend 2 page
curl http://192.168.1.102
```

---

## PART B — Load Balancer Setup (Run on VM1 ONLY)

### B1 — Switch to Root

```bash
sudo su -
```

### B2 — Install Apache

```bash
dnf install httpd -y

systemctl start httpd
systemctl enable httpd
```

### B3 — Configure Firewall on Load Balancer

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

### B4 — Allow Apache Proxy Network Connections (SELinux)

```bash
# CRITICAL: Without this, SELinux will block all proxy connections
setsebool -P httpd_can_network_connect 1

# Verify
getsebool httpd_can_network_connect
```

### B5 — Verify Proxy Modules Are Available

```bash
httpd -M | grep -E "proxy|balancer|slotmem"
```

You should see output including:
```
proxy_module (shared)
proxy_http_module (shared)
proxy_balancer_module (shared)
lbmethod_byrequests_module (shared)
slotmem_shm_module (shared)
```

### B6 — Create Load Balancer Virtual Host Config

```bash
cat > /etc/httpd/conf.d/lb-my.com.conf << 'EOF'
# Load the required proxy modules
LoadModule proxy_module           modules/mod_proxy.so
LoadModule proxy_http_module      modules/mod_proxy_http.so
LoadModule proxy_balancer_module  modules/mod_proxy_balancer.so
LoadModule lbmethod_byrequests_module modules/mod_lbmethod_byrequests.so
LoadModule slotmem_shm_module     modules/mod_slotmem_shm.so

# Define the backend cluster
<Proxy "balancer://mycluster">
    # Backend 1 — VM2
    # retry=10: wait 10s before retrying a failed backend
    # timeout=5: give up on backend after 5 seconds
    BalancerMember http://192.168.1.101 loadfactor=1 route=web1 retry=10 timeout=5

    # Backend 2 — VM3
    BalancerMember http://192.168.1.102 loadfactor=1 route=web2 retry=10 timeout=5

    # Load balancing method: byrequests = Round-Robin
    # Alternatives: bybusyness (least busy), bytraffic (least traffic)
    ProxySet lbmethod=byrequests
</Proxy>

<VirtualHost *:80>
    ServerName  my.com
    ServerAlias www.my.com

    # Forward all requests to the balancer cluster
    ProxyPreserveHost On
    ProxyPass        "/"  "balancer://mycluster/"
    ProxyPassReverse "/"  "balancer://mycluster/"

    # Pass real client IP to backend servers
    RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}s"

    # Balancer Manager UI — accessible from local network only
    <Location "/balancer-manager">
        SetHandler balancer-manager
        Require ip 127.0.0.1 192.168.1.0/24
    </Location>

    ErrorLog  /var/log/httpd/lb-my.com_error.log
    CustomLog /var/log/httpd/lb-my.com_access.log combined
</VirtualHost>
EOF

# Test and apply
apachectl configtest
systemctl reload httpd
systemctl status httpd
```

### B7 — Configure Local DNS on Load Balancer VM

```bash
# my.com should resolve to the LB itself
echo "192.168.1.100   my.com www.my.com" >> /etc/hosts

grep "my.com" /etc/hosts
```

---

## PART C — Configure DNS on Your Client Machine

> Do this on the machine where you open the browser.

**Linux / Mac client:**

```bash
sudo su -
echo "192.168.1.100   my.com www.my.com" >> /etc/hosts

# Verify
grep "my.com" /etc/hosts
ping my.com
```

**Windows client** — edit as Administrator:

```
C:\Windows\System32\drivers\etc\hosts
```

Add this line:

```
192.168.1.100   my.com www.my.com
```

---

## PART D — Test Load Balancing

### D1 — Basic Test

```bash
# Run from any client — responses should alternate between backends
curl http://my.com
curl http://my.com
curl http://my.com
```

### D2 — Loop Test (Best Way to Confirm Round-Robin)

```bash
for i in {1..10}; do
  echo -n "Request $i → "
  curl -s http://my.com | grep -o 'Backend Server [0-9]*'
done
```

Expected output:
```
Request 1  → Backend Server 1
Request 2  → Backend Server 2
Request 3  → Backend Server 1
Request 4  → Backend Server 2
...
```

### D3 — Balancer Manager Web UI

Open in your browser:

```
http://my.com/balancer-manager
```

Or via curl on VM1:

```bash
curl http://127.0.0.1/balancer-manager
```

---

## PART E — Failover Test

```bash
# 1. Stop Apache on VM2 to simulate a server failure
#    (Run this on VM2)
systemctl stop httpd

# 2. On client — all requests now go to VM3 only (auto-failover)
for i in {1..6}; do
  echo -n "Request $i → "
  curl -s http://my.com | grep -o 'Backend Server [0-9]*'
done

# 3. Restart VM2 — it automatically rejoins the pool after retry= seconds
#    (Run this on VM2)
systemctl start httpd
```

---

## Quick Reference — All Commands

| Task | VM | Command |
|------|----|---------|
| Check Apache status | Any | `systemctl status httpd` |
| Restart Apache | Any | `systemctl restart httpd` |
| Reload config (no downtime) | Any | `systemctl reload httpd` |
| Test config syntax | Any | `apachectl configtest` |
| List loaded modules | Any | `httpd -M` |
| View LB error log | VM1 | `tail -f /var/log/httpd/lb-my.com_error.log` |
| View backend error log | VM2/3 | `tail -f /var/log/httpd/my.com_error.log` |
| List virtual hosts | Any | `apachectl -S` |
| Check SELinux proxy | VM1 | `getsebool httpd_can_network_connect` |
| View balancer stats UI | VM1 | `curl http://127.0.0.1/balancer-manager` |

---

## Troubleshooting

```bash
# 503 Bad Gateway on my.com?
# Test backend connectivity directly from LB (VM1):
curl http://192.168.1.101
curl http://192.168.1.102

# SELinux blocking proxy on VM1?
setsebool -P httpd_can_network_connect 1
ausearch -c 'httpd' --raw | audit2allow -M lbpol
semodule -i lbpol.pp

# Backend firewall blocking requests from LB?
# Run on VM2 and VM3:
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.100" service name="http" accept'
firewall-cmd --reload

# mod_proxy modules not loading?
httpd -M | grep proxy

# DNS not resolving my.com?
cat /etc/hosts | grep my.com
ping my.com

# Check which backend handled a request (from LB access log):
tail -f /var/log/httpd/lb-my.com_access.log
```

---

## Full Architecture Summary

```
Client Browser
      │
      │  DNS lookup: my.com → 192.168.1.100  (/etc/hosts)
      ▼
┌──────────────────────────────────────────┐
│            VM1 — Load Balancer           │
│            192.168.1.100                 │
│                                          │
│  Apache httpd                            │
│  mod_proxy + mod_proxy_balancer          │
│  VirtualHost my.com → mycluster         │
│  Method: Round-Robin (byrequests)        │
│  Failover: auto on backend timeout       │
└───────────────┬──────────────────────────┘
                │
        Round-Robin / Failover
                │
      ┌─────────┴──────────┐
      ▼                    ▼
┌──────────────┐    ┌──────────────┐
│     VM2      │    │     VM3      │
│ 192.168.1.101│    │ 192.168.1.102│
│  Backend 1   │    │  Backend 2   │
│  Apache httpd│    │  Apache httpd│
│  /var/www/   │    │  /var/www/   │
│  my.com/html │    │  my.com/html │
└──────────────┘    └──────────────┘
```
