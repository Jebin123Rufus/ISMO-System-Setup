# Production Server Architecture & Deployment Plan

> **Scope:** Single physical server — Application + Database co-located on one machine.
> **Goal:** Secure, observable, recoverable, and efficient production deployment.

---

## Table of Contents

1. [Objective](#1-objective)
2. [Architecture Overview](#2-architecture-overview)
3. [Network Architecture](#3-network-architecture)
4. [Hardware Specification](#4-hardware-specification)
5. [Storage — RAID 10](#5-storage--raid-10)
6. [Power — UPS](#6-power--ups)
7. [Operating System](#7-operating-system)
8. [Ubuntu Installation](#8-ubuntu-installation)
9. [Initial System Configuration](#9-initial-system-configuration)
10. [User & Privilege Model](#10-user--privilege-model)
11. [SSH Hardening](#11-ssh-hardening)
12. [Host Firewall (UFW)](#12-host-firewall-ufw)
13. [Public IP & DNS](#13-public-ip--dns)
14. [NAT / Port Forwarding](#14-nat--port-forwarding)
15. [Nginx — Reverse Proxy & TLS](#15-nginx--reverse-proxy--tls)
16. [Application Deployment](#16-application-deployment)
17. [Process Management (systemd)](#17-process-management-systemd)
18. [Database — Co-located Setup](#18-database--co-located-setup)
19. [Database Security](#19-database-security)
20. [Secrets Management](#20-secrets-management)
21. [Internal Service Communication](#21-internal-service-communication)
22. [Employee Network & VLAN](#22-employee-network--vlan)
23. [Remote Access — WireGuard VPN](#23-remote-access--wireguard-vpn)
24. [Security Headers & Rate Limiting](#24-security-headers--rate-limiting)
25. [Host Security Hardening](#25-host-security-hardening)
26. [Monitoring & Alerting](#26-monitoring--alerting)
27. [Centralized Logging](#27-centralized-logging)
28. [Backup Strategy](#28-backup-strategy)
29. [Disaster Recovery](#29-disaster-recovery)
30. [Deployment Order](#30-deployment-order)
31. [Pre-Production Checklist](#31-pre-production-checklist)

---

## 1. Objective

Deploy a single physical server that:

| Requirement | Solution |
|---|---|
| Runs 24/7 | UPS + systemd auto-restart + monitored uptime |
| Hosts the production application | Node.js / Spring Boot via systemd |
| Hosts the production database (same machine) | PostgreSQL / MongoDB on `127.0.0.1` only |
| Reachable by customers over the Internet | Nginx HTTPS reverse proxy |
| Employee internal access | Firewall-controlled Employee VLAN |
| Authorized remote access | WireGuard VPN |
| Secure admin access | SSH over VPN only, key-based auth |
| Storage redundancy | RAID 10 across 4 drives |
| Data protection | Encrypted daily backups to off-site storage |
| Observability | Prometheus + Grafana + Loki |

> **Key constraint:** Application and database share one physical machine.
> Isolation is achieved through OS-level controls — users, filesystem permissions,
> firewall rules, and loopback binding — not network topology.

---

## 2. Architecture Overview

```
                          INTERNET
                              |
                         Public IP
                              |
                              v
                +---------------------------+
                |      EDGE FIREWALL        |
                |  (pfSense / OPNsense)     |
                |  NAT: 443, 80, 51820 only |
                |  VLAN routing             |
                |  WireGuard VPN endpoint   |
                +----------+----------------+
                           |
       +-------------------+--------------------+
       |                   |                    |
       v                   v                    v
 Public (443/80)     Employee VLAN        WireGuard VPN
                     192.168.10.0/24      10.20.0.0/24
                           |                    |
                     +-----+--------------------+
                     |   Internal access to server
                     v
       +--------------------------------------------+
       |       SERVER VLAN - 192.168.20.0/24        |
       |                                            |
       |  Ubuntu Server 24.04 LTS                  |
       |                                            |
       |  [ Nginx         :443 / :80          ]    |
       |           |                               |
       |  [ Application   127.0.0.1:3000      ]    |
       |           |                               |
       |  [ Database      127.0.0.1:5432      ]    |
       |           |                               |
       |  UFW / AppArmor / Fail2ban / auditd       |
       |  Prometheus Node Exporter (127.0.0.1)     |
       |  Promtail (log shipper -> off-server)     |
       +--------------------+-----------------------+
                            |
                         RAID 10
                            |
                 +----------+----------+
                 |                     |
             Production           Local backup
                Data             -> Off-site (S3)
```

### Traffic Boundaries

| Source | Destination | Allowed |
|---|---|---|
| Internet | Nginx :443 | Yes |
| Internet | Nginx :80 | Yes (redirect only) |
| Internet | SSH :22 | **No** |
| Internet | Any DB port | **No** |
| VPN Admin | SSH :22 | Yes (key auth only) |
| Employee VLAN | App :443 | Yes |
| Application | Database | Yes (loopback only) |
| Database | Internet | **No** |

---

## 3. Network Architecture

### VLAN Design

```
              EDGE FIREWALL
                   |
  +----------------+----------------+
  |        |            |           |
  v        v            v           v
VLAN 10  VLAN 20    VLAN 30    VLAN 40
Employee  Server     Guest    Management
.10/24    .20/24     .30/24     .40/24
```

| VLAN | Subnet | Purpose | Key Restrictions |
|---|---|---|---|
| Employee | 192.168.10.0/24 | Staff workstations | No DB, no SSH to server |
| Server | 192.168.20.0/24 | Production server | Only 443/80 from Internet |
| Guest | 192.168.30.0/24 | Guest Wi-Fi | Internet only, fully isolated |
| Management | 192.168.40.0/24 | Admin workstations | SSH to server allowed |
| VPN | 10.20.0.0/24 | Remote employees & admins | Per-role UFW rules |

### Inter-VLAN Firewall Rules

```
Default policy: DENY ALL inter-VLAN

Explicit allows:
  Employee    -> Server :443        ALLOW
  Management  -> Server :22         ALLOW
  VPN admin   -> Server :22         ALLOW
  VPN users   -> Server :443        ALLOW
  Internet    -> Server :443/:80    ALLOW

Explicit denies (defense-in-depth):
  Guest       -> Server VLAN        DENY all
  Guest       -> Employee VLAN      DENY all
  Employee    -> Server :22         DENY
  Employee    -> Server DB ports    DENY
  Internet    -> Server :22         DENY
  Internet    -> All other ports    DENY
```

---

## 4. Hardware Specification

### Minimum Recommended

```
CPU:      Server-grade 64-bit (Intel Xeon E or AMD EPYC)
RAM:      32 GB ECC  (ECC mandatory for DB reliability)
Storage:  4 x 2 TB Enterprise NVMe/SSD  =>  RAID 10 = ~4 TB usable
          + 1 x separate SSD for OS boot (not part of RAID)
Network:  2 x 1 GbE NICs
PSU:      Redundant if chassis supports it
UPS:      1500 VA minimum
Cooling:  Server room or rack with proper airflow
```

### Why ECC RAM?

```
Standard RAM bit-flip  ->  Silent data corruption  ->  DB corruption
ECC RAM bit-flip       ->  Auto-corrected          ->  No data loss
```

ECC RAM is non-negotiable when the database shares the machine.

### Sizing Guide

| Component | Sizing Basis |
|---|---|
| RAM | App working set + DB buffer pool + OS overhead |
| Storage | Projected data x 3 (growth + logs + backup staging) |
| CPU | Peak concurrent request throughput |

---

## 5. Storage — RAID 10

### Physical Layout

```
       RAID 10 Controller
           |
   +-------+-------+
   |               |
Mirror Set 1    Mirror Set 2
[Disk 1 + 2]   [Disk 3 + 4]
   |               |
   +-------+-------+
           |
     Striped across both mirrors
```

### Mount Strategy

```
Boot SSD (separate, not in RAID):
  /           root filesystem
  /boot
  /home
  /var/log    (or on a separate partition)

RAID 10 mounted at /data:
  /data/db/           database files
  /data/app/          application files
  /data/backups/      local backup staging
```

Keeping the OS on a separate drive means a RAID failure does **not** bring down the OS.

| Property | Value |
|---|---|
| Raw capacity | 4 x 2 TB = 8 TB |
| Usable capacity | ~4 TB (after filesystem overhead) |
| Fault tolerance | 1 drive per mirror set |
| Rebuild risk | High — take a backup before rebuilding |

> **RAID is not a backup.**
> RAID protects against disk hardware failure only.
> It does NOT protect against: deletion, ransomware, corruption, fire, or theft.

---

## 6. Power — UPS

### Power Path

```
Grid Power
    |
    v
UPS (1500 VA minimum)
    |
    +-- Server
    +-- Network switch
    +-- Edge firewall
```

### UPS Monitoring

```bash
sudo apt install apcupsd
```

```
# /etc/apcupsd/apcupsd.conf
BATTERYLEVEL 15
MINUTES 5
TIMEOUT 0
```

### Alert Levels

```
Power failure detected  ->  Alert admin immediately
Battery < 30%           ->  Warning alert
Battery < 15%           ->  Begin controlled shutdown
```

> **Limitation:** A UPS provides 5–30 minutes of runtime.
> It is not a generator. For extended outages, recovery depends
> on your off-site backups, not the UPS.

---

## 7. Operating System

```
Ubuntu Server 24.04 LTS — minimal install, no GUI
```

### Administration Model

```
Admin workstation
    |
    v  WireGuard VPN (step 1)
Company network
    |
    v  SSH with Ed25519 key (step 2)
Ubuntu Server (Management VLAN or VPN IPs only)
```

SSH is **never** exposed to the Internet.
Both authentication barriers must be cleared to reach the server.

---

## 8. Ubuntu Installation

1. Boot from Ubuntu Server 24.04 LTS ISO
2. Select **minimized installation**
3. Configure storage:
   - OS on the separate boot SSD
   - RAID 10 mounted at `/data`
4. Set hostname: `prod-server-01`
5. Configure static IP:
   ```
   IP:      192.168.20.20/24
   Gateway: 192.168.20.1
   DNS:     192.168.40.1
   ```
6. Create non-root admin user: `sysadmin`
7. Enable OpenSSH Server
8. Complete and reboot

### Post-Install Verification

```bash
hostnamectl
ip addr
lsblk            # Verify /data mounted on RAID
df -h
sudo ss -tulpn   # Check all listening services
```

---

## 9. Initial System Configuration

### Update First

```bash
sudo apt update && sudo apt upgrade -y
sudo apt autoremove -y
sudo reboot
```

### Install Essential Tools Only

```bash
sudo apt install -y curl wget git vim htop unzip \
  ca-certificates gnupg lsb-release fail2ban ufw net-tools
```

### Disable Unnecessary Services

```bash
sudo systemctl list-unit-files --state=enabled
sudo systemctl disable --now snapd
sudo systemctl disable --now avahi-daemon
sudo systemctl disable --now cups
```

### Enable Automatic Security Updates

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

```
# /etc/apt/apt.conf.d/50unattended-upgrades

Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};
Unattended-Upgrade::Automatic-Reboot "false";
Unattended-Upgrade::Mail "admin@company.com";
```

Security patches apply automatically. Reboots require manual approval.

---

## 10. User & Privilege Model

| Account | Purpose | Shell | sudo |
|---|---|---|---|
| `sysadmin` | Server administration | /bin/bash | Yes |
| `appuser` | Runs application process | /bin/false | No |
| `postgres` / `mongod` | DB process user | /bin/false | No |
| `root` | Emergency only — account locked | — | — |

### Create Application Service User

```bash
sudo adduser --system --no-create-home --shell /bin/false appuser
```

The application process runs as `appuser`, never as root or sysadmin.

### Verify Isolation

```bash
sudo -l -U appuser   # Must show: not allowed to run sudo
ls -la /data/app     # Must show: owned by appuser
ls -la /data/db      # Must show: owned by postgres or mongod
```

---

## 11. SSH Hardening

### Generate Key on Admin Workstation

```bash
ssh-keygen -t ed25519 -C "sysadmin@company.com"
```

```
~/.ssh/id_ed25519      <- Private key — NEVER leaves your workstation
~/.ssh/id_ed25519.pub  <- Public key — copied to server
```

### Copy Public Key and Test

```bash
ssh-copy-id sysadmin@192.168.20.20
ssh -i ~/.ssh/id_ed25519 sysadmin@192.168.20.20
```

Only proceed to hardening after confirming key login works.

### Harden /etc/ssh/sshd_config

```
Port 22
Protocol 2

PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthenticationMethods publickey
ChallengeResponseAuthentication no
UsePAM no

ListenAddress 192.168.20.20
AllowUsers sysadmin@192.168.40.0/24 sysadmin@10.20.0.0/24

ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 30
MaxAuthTries 3
MaxSessions 4

X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
PermitTunnel no
```

```bash
sudo sshd -t             # Must show no errors
sudo systemctl restart ssh
```

---

## 12. Host Firewall (UFW)

### Default Policy (deny everything)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed
```

### Allow Web Traffic

```bash
sudo ufw allow 443/tcp comment 'HTTPS public'
sudo ufw allow 80/tcp  comment 'HTTP redirect only'
```

### Allow SSH — Management and VPN Networks Only

```bash
sudo ufw allow from 192.168.40.0/24 to any port 22 proto tcp comment 'SSH management VLAN'
sudo ufw allow from 10.20.0.0/24    to any port 22 proto tcp comment 'SSH VPN admins'
```

### Explicitly Deny Internal Ports

```bash
sudo ufw deny 5432/tcp  comment 'PostgreSQL - internal only'
sudo ufw deny 27017/tcp comment 'MongoDB - internal only'
sudo ufw deny 3306/tcp  comment 'MySQL - internal only'
sudo ufw deny 3000/tcp  comment 'App port - internal only'
```

### Enable and Verify

```bash
sudo ufw enable
sudo ufw status verbose
```

Expected result:

```
443/tcp    ALLOW IN  Anywhere
80/tcp     ALLOW IN  Anywhere
22/tcp     ALLOW IN  192.168.40.0/24
22/tcp     ALLOW IN  10.20.0.0/24
5432/tcp   DENY IN   Anywhere
27017/tcp  DENY IN   Anywhere
3000/tcp   DENY IN   Anywhere
```

---

## 13. Public IP & DNS

```
ISP provides: 203.0.113.50  (static — request from ISP)

Internet -> 203.0.113.50 (Edge Firewall) --(NAT)--> 192.168.20.20 (Server)
```

### DNS Records

```
company.com.      A     203.0.113.50
www.company.com.  CNAME company.com.
```

Use a short TTL (300s) during initial setup for fast propagation.

---

## 14. NAT / Port Forwarding

Configure on the edge firewall only:

| Public Port | Forward To | Service |
|---|---|---|
| 443/TCP | 192.168.20.20:443 | Nginx HTTPS |
| 80/TCP | 192.168.20.20:80 | Nginx HTTP (redirect) |
| 51820/UDP | 192.168.20.20:51820 | WireGuard VPN |

**Never forward:**

```
22     ->  SSH         (access via VPN only)
5432   ->  PostgreSQL  (loopback binding — unreachable anyway)
27017  ->  MongoDB     (loopback binding — unreachable anyway)
3000   ->  App port    (loopback binding — unreachable anyway)
```

---

## 15. Nginx — Reverse Proxy & TLS

### Install

```bash
sudo apt install nginx
sudo systemctl enable --now nginx
```

### TLS Certificate via Certbot

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d company.com -d www.company.com
sudo certbot renew --dry-run   # Verify auto-renewal
```

### Rate Limiting (add to http block in nginx.conf)

```nginx
# /etc/nginx/nginx.conf — inside http { }
limit_req_zone $binary_remote_addr zone=api:10m rate=30r/s;
limit_conn_zone $binary_remote_addr zone=conn:10m;
```

### Production Site Configuration

```nginx
# /etc/nginx/sites-available/company.com

# HTTP -> HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name company.com www.company.com;
    return 301 https://$host$request_uri;
}

# HTTPS main block
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name company.com www.company.com;

    ssl_certificate     /etc/letsencrypt/live/company.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/company.com/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    server_tokens off;

    # Security headers
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Content-Type-Options    "nosniff" always;
    add_header X-Frame-Options           "SAMEORIGIN" always;
    add_header Referrer-Policy           "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy   "default-src 'self'; object-src 'none';" always;
    add_header Permissions-Policy        "geolocation=(), camera=(), microphone=()" always;

    # Rate limiting
    limit_req zone=api burst=20 nodelay;
    limit_req_status 429;

    # Public reverse proxy
    location / {
        proxy_pass         http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   Upgrade           $http_upgrade;
        proxy_set_header   Connection        "upgrade";
        proxy_read_timeout    60s;
        proxy_connect_timeout 10s;
    }

    # Internal admin panel — IP-restricted
    location /admin {
        allow 192.168.10.0/24;
        allow 10.20.0.0/24;
        deny all;
        proxy_pass http://127.0.0.1:3000/admin;
    }

    # Block sensitive file access
    location ~* \.(env|git|sql|bak|backup)$ {
        deny all;
        return 404;
    }

    access_log /var/log/nginx/company.access.log combined;
    error_log  /var/log/nginx/company.error.log warn;
}
```

```bash
sudo ln -s /etc/nginx/sites-available/company.com /etc/nginx/sites-enabled/
sudo nginx -t           # Must show: syntax is ok
sudo systemctl reload nginx
```

---

## 16. Application Deployment

### Directory Layout

```
/data/app/
  current/            <- Active release (symlink)
    .env              <- Secrets (chmod 640, NOT in git)
    index.js
    package.json
  releases/
    2026-09-20/       <- Previous release (rollback target)
  shared/
    logs/
```

### Permissions

```bash
sudo chown -R appuser:appuser /data/app
sudo chmod 750 /data/app
sudo chmod 640 /data/app/current/.env
```

### Application Binding Rules

```javascript
// CORRECT: bind to loopback only
app.listen(3000, '127.0.0.1');

// WRONG: never bind to all interfaces in production
app.listen(3000, '0.0.0.0');
```

```javascript
// CORRECT: read from environment
const dbPassword = process.env.DB_PASSWORD;

// WRONG: never hardcode secrets in source code
const dbPassword = "secret123";
```

```javascript
// CORRECT: connect to DB via loopback
const DB_HOST = '127.0.0.1';

// WRONG: never use a hostname that could resolve externally
const DB_HOST = 'localhost'; // Use 127.0.0.1 explicitly
```

---

## 17. Process Management (systemd)

### systemd vs Docker for a Single Server

| Criterion | systemd | Docker |
|---|---|---|
| Complexity | Low | Medium |
| Overhead | Minimal | Moderate |
| Auto-restart | Yes | Yes |
| Start on boot | Yes | Yes |
| Best fit for 1 service | Yes | Overkill |

Use systemd. Use Docker if you later need multiple isolated services or dev/prod parity.

### Application Service Unit

```ini
# /etc/systemd/system/app.service

[Unit]
Description=Production Application
After=network-online.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=appuser
Group=appuser
WorkingDirectory=/data/app/current

# Loads secrets from .env â€” not passed via environment directly
EnvironmentFile=/data/app/current/.env

ExecStart=/usr/bin/node /data/app/current/index.js
ExecReload=/bin/kill -HUP $MAINPID

# Restart policy
Restart=on-failure
RestartSec=5s
StartLimitIntervalSec=60s
StartLimitBurst=5

# Resource limits
LimitNOFILE=65536
MemoryMax=2G
CPUQuota=80%

# Security hardening directives
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/data/app /data/app/shared/logs
ProtectHome=true

# Logging goes to journald
StandardOutput=journal
StandardError=journal
SyslogIdentifier=app

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable app
sudo systemctl start app
sudo systemctl status app
```

### Verify Auto-Restart

```bash
# Kill the process â€” systemd should restart it within 5 seconds
sudo kill -9 $(pgrep -f "node /data/app")
sleep 6
sudo systemctl status app   # Must show: active (running)
```

### Rollback

```bash
sudo systemctl stop app
sudo ln -sfn /data/app/releases/2026-09-20 /data/app/current
sudo systemctl start app
sudo systemctl status app
```

---

## 18. Database â€” Co-located Setup

### Core Design Decision

The database runs on the **same physical machine** as the application.
All database connections stay on the **loopback interface** (`127.0.0.1`).

```
Application  (127.0.0.1:3000)
        |
        |  TCP on loopback â€” never leaves the machine
        v
Database     (127.0.0.1:5432)
        |
        v
RAID 10      (/data/db/)
```

Consequences:
- Database is **unreachable from any network** â€” by binding, not just by firewall
- Performance is optimal â€” no network latency for database queries
- No firewall misconfiguration can accidentally expose the database

### PostgreSQL â€” Bind to Loopback

```
# /etc/postgresql/16/main/postgresql.conf
listen_addresses = '127.0.0.1'
port = 5432
data_directory = '/data/db/postgresql/16/main'
```

```
# /etc/postgresql/16/main/pg_hba.conf
# TYPE  DATABASE  USER      ADDRESS         METHOD
local   all       postgres                  peer
host    appdb     appuser   127.0.0.1/32    scram-sha-256
```

Only `appuser` can connect to `appdb`, only from `127.0.0.1`.

### MongoDB â€” Bind to Loopback

```yaml
# /etc/mongod.conf
net:
  port: 27017
  bindIp: 127.0.0.1

security:
  authorization: enabled

storage:
  dbPath: /data/db/mongodb
  journal:
    enabled: true
```

### Verify After Every Restart

```bash
sudo ss -tlnp | grep -E '5432|27017'
```

Must show only `127.0.0.1` â€” never `0.0.0.0`:

```
LISTEN  127.0.0.1:5432   (postgres)
LISTEN  127.0.0.1:27017  (mongod)
```

---

## 19. Database Security

### Least-Privilege Application Account

```sql
-- PostgreSQL

CREATE DATABASE appdb;
CREATE USER appuser WITH PASSWORD 'use-a-long-random-password';

GRANT CONNECT ON DATABASE appdb TO appuser;
GRANT USAGE ON SCHEMA public TO appuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO appuser;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO appuser;

-- Revoke overly broad defaults
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON DATABASE appdb FROM PUBLIC;
```

### Access Model

```
postgres superuser  <-  Human DBA via local shell (sudo -u postgres psql)
appuser             <-  Application process only, password auth, 127.0.0.1
```

The application **never** uses the `postgres` superuser.

### File Permissions

```bash
sudo chown -R postgres:postgres /data/db/postgresql
sudo chmod 700 /data/db/postgresql

sudo chown -R mongod:mongod /data/db/mongodb
sudo chmod 700 /data/db/mongodb
```

### Verify Application Cannot Escalate

```bash
sudo -u appuser psql -U postgres -d appdb   # Must fail
```

---

## 20. Secrets Management

### Practical Approach for a Single Server

Use a `.env` file with tight filesystem permissions, loaded by systemd:

```bash
sudo touch /data/app/current/.env
sudo chown root:appuser /data/app/current/.env
sudo chmod 640 /data/app/current/.env
```

```bash
# /data/app/current/.env
DB_HOST=127.0.0.1
DB_PORT=5432
DB_NAME=appdb
DB_USER=appuser
DB_PASSWORD=long-random-secret-here
APP_SECRET=another-long-random-secret
NODE_ENV=production
```

### Rules

```
Never put secrets in source code
Never commit .env to git
Use a different secret for each purpose
Rotate secrets after any suspected compromise
```

### .gitignore (mandatory)

```
.env
.env.*
*.pem
*.key
secrets/
```

### Generating Secrets

```bash
openssl rand -hex 32
```

---

## 21. Internal Service Communication

### Loopback Trust Model

```
Nginx  -> 127.0.0.1:3000 -> Application -> 127.0.0.1:5432 -> Database
```

All service-to-service connections stay on the loopback interface.
No traffic hits the network stack.

### Controls at This Layer

| Security Question | Control |
|---|---|
| Who can connect to the DB? | pg_hba.conf â€” appuser from 127.0.0.1 only |
| What can the app do in the DB? | SQL grants â€” SELECT/INSERT/UPDATE/DELETE only |
| Can appuser read DB config files? | Filesystem permissions â€” No |
| Can DB process read app files? | Filesystem permissions â€” No |

### Adding Future Services

```bash
# Any new service (Redis, queues, etc.) must also bind to loopback
# redis.conf: bind 127.0.0.1

# Always add an explicit UFW deny for defense-in-depth
sudo ufw deny 6379/tcp comment 'Redis - internal only'
```

---

## 22. Employee Network & VLAN

### Access Policy

```
Employee PC (192.168.10.x)
    |
    v  Edge firewall + UFW enforces:
    |
    +-- company.com :443     ->  ALLOW  (web application)
    +-- /admin paths         ->  ALLOW  (Nginx IP restriction)
    +-- Server SSH :22       ->  DENY
    +-- Database ports       ->  DENY
    +-- Guest VLAN           ->  DENY
    +-- Management VLAN      ->  DENY
```

Employees have no shell access to the server and no direct database access.

---

## 23. Remote Access â€” WireGuard VPN

### Install

```bash
sudo apt install wireguard
```

### Server Configuration

```ini
# /etc/wireguard/wg0.conf

[Interface]
Address    = 10.20.0.1/24
ListenPort = 51820
PrivateKey = <server-private-key>
PostUp     = iptables -A FORWARD -i wg0 -j ACCEPT
PostDown   = iptables -D FORWARD -i wg0 -j ACCEPT

[Peer]  # Admin
PublicKey  = <admin-public-key>
AllowedIPs = 10.20.0.10/32

[Peer]  # Employee
PublicKey  = <employee-public-key>
AllowedIPs = 10.20.0.20/32
```

```bash
sudo systemctl enable --now wg-quick@wg0
```

### Per-Role Access via UFW

```bash
# Admin can SSH
sudo ufw allow from 10.20.0.10 to any port 22 proto tcp comment 'VPN admin SSH'

# All VPN users reach the app
sudo ufw allow from 10.20.0.0/24 to any port 443 proto tcp comment 'VPN HTTPS'

# VPN users cannot reach DB directly
sudo ufw deny from 10.20.0.0/24 to any port 5432 proto tcp comment 'VPN deny DB'
```

### Employee Client Config

```ini
[Interface]
PrivateKey = <employee-private-key>
Address    = 10.20.0.20/32
DNS        = 192.168.40.1

[Peer]
PublicKey           = <server-public-key>
Endpoint            = 203.0.113.50:51820
AllowedIPs          = 192.168.0.0/16, 10.20.0.0/24
PersistentKeepalive = 25
```

Only internal IP ranges are tunneled. General Internet traffic goes direct.

### Employee Offboarding

```bash
sudo wg set wg0 peer <employee-public-key> remove
sudo wg-quick save wg0
```

Access is revoked immediately. No full VPN rekey required.

---

## 24. Security Headers & Rate Limiting

Security headers are defined in the Nginx configuration in Section 15.

Test your configuration at: `https://securityheaders.com`

| Header | Protects Against |
|---|---|
| Strict-Transport-Security | HTTPS downgrade attacks |
| Content-Security-Policy | XSS, injection |
| X-Content-Type-Options | MIME sniffing |
| X-Frame-Options | Clickjacking |
| Referrer-Policy | Information leakage |
| Permissions-Policy | Unwanted browser API use |

### Rate Limiting Layers

```
Nginx:       30 req/sec per IP    ->  CPU protection
Application: per-endpoint limits  ->  fine-grained control
Fail2ban:    bans repeat 429s     ->  blocks persistent abusers
```

> This is not DDoS protection.
> For volumetric attacks, add Cloudflare or a CDN upstream.

---

## 25. Host Security Hardening

### Fail2ban

```bash
sudo apt install fail2ban
```

```ini
# /etc/fail2ban/jail.local

[DEFAULT]
bantime  = 3600
findtime = 600
maxretry = 5

[sshd]
enabled = true
port    = 22
logpath = /var/log/auth.log

[nginx-limit-req]
enabled = true
port    = http,https
logpath = /var/log/nginx/company.error.log
```

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

### AppArmor

```bash
sudo aa-status   # Should show profiles in enforce mode
sudo apt install apparmor-utils
sudo aa-enforce /etc/apparmor.d/usr.sbin.nginx
```

### Lock Root Account

```bash
sudo passwd -l root
```

Root access requires `sudo` from `sysadmin` only.

### Auditd â€” Tamper-Evident Audit Trail

```bash
sudo apt install auditd
sudo systemctl enable --now auditd
```

```
# /etc/audit/rules.d/production.rules

-a always,exit -F arch=b64 -S execve -F euid=0 -k root_commands
-w /data/app/current/.env -p rwa -k secrets_access
-w /etc/ssh/sshd_config -p rwa -k ssh_config
-w /etc/passwd -p rwa -k user_changes
```

### Kernel Hardening

```
# /etc/sysctl.d/99-production.conf

net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.tcp_syncookies = 1
fs.suid_dumpable = 0
```

```bash
sudo sysctl -p /etc/sysctl.d/99-production.conf
```

---

## 26. Monitoring & Alerting

### Stack

```
Node Exporter (system metrics)
PostgreSQL Exporter (DB metrics)
          |
          v
    Prometheus (collect + store)
          |
          v
     Grafana (dashboards + alerts)
```

### Install Node Exporter

```bash
sudo adduser --system --no-create-home prometheus
wget https://github.com/prometheus/node_exporter/releases/latest/download/node_exporter-1.8.2.linux-amd64.tar.gz
tar xzf node_exporter-*.tar.gz
sudo mv node_exporter-*/node_exporter /usr/local/bin/
```

```ini
# /etc/systemd/system/node-exporter.service

[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/node_exporter --web.listen-address="127.0.0.1:9100"
Restart=on-failure
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Node Exporter binds to `127.0.0.1:9100` â€” not network-accessible.

### Alert Thresholds

| Metric | Warning | Critical |
|---|---|---|
| CPU usage | > 85% for 5 min | â€” |
| RAM usage | > 90% | â€” |
| /data disk usage | > 80% | > 90% |
| RAID health | Any degraded | â€” |
| App response P99 | > 2s | â€” |
| App error rate | > 1% | > 5% |
| SSL cert expiry | < 30 days | < 7 days |
| Server unreachable | 1 missed check | â€” |
| Failed SSH logins | > 20 in 1 min | Security alert |

### Alert Delivery

```
Grafana alert fires
    |
    +-- Email to admin@company.com
    +-- Slack / Telegram webhook
```

---

## 27. Centralized Logging

### Why Off-Server Logging Is Required

```
Attacker compromises server
    |
    v
Deletes /var/log/* and clears journald
    |
    v
No forensic evidence
```

Ship logs off the server as they are written:

```
Nginx logs
Auth logs             Promtail     Loki (remote)    Grafana
App logs (journald)  --------->  ------------->  ----------> search
DB logs
```

### Promtail Configuration

```yaml
# /etc/promtail/config.yml

clients:
  - url: http://your-loki-host:3100/loki/api/v1/push

scrape_configs:
  - job_name: nginx
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx
          host: prod-server-01
          __path__: /var/log/nginx/*.log

  - job_name: system
    static_configs:
      - targets: [localhost]
        labels:
          job: system
          host: prod-server-01
          __path__: /var/log/auth.log

  - job_name: app
    journal:
      labels:
        job: app
        unit: app.service
```

Loki can run on a separate server or a cloud-hosted Grafana Cloud instance.

---

## 28. Backup Strategy

### 3-2-1 Rule

```
3 copies of data:
  1. Live RAID 10 data       ->  production
  2. Local /data/backups     ->  fast local restore
  3. Off-site S3 or NAS      ->  disaster recovery

2 storage types: NVMe RAID + Cloud
1 off-site location: S3, Backblaze B2, or remote NAS
```

### What to Back Up

| Item | Frequency | Method |
|---|---|---|
| Database dump | Daily at 2 AM | pg_dump + gzip |
| App config + .env | Daily | rsync (encrypted) |
| Nginx config | Daily | rsync |
| systemd units | Daily | rsync /etc/systemd |
| SSL certs | Auto-managed | certbot |

### Backup Script

```bash
#!/bin/bash
# /usr/local/bin/backup.sh
set -euo pipefail

TIMESTAMP=$(date +%Y-%m-%d_%H-%M)
DIR="/data/backups/${TIMESTAMP}"
BUCKET="s3://company-backups/prod-server-01"

mkdir -p "${DIR}"

# Database
pg_dump -U postgres appdb | gzip > "${DIR}/appdb.sql.gz"

# Application and config files
rsync -a /data/app/current/      "${DIR}/app/"
rsync -a /etc/nginx/             "${DIR}/nginx/"
rsync -a /etc/systemd/system/    "${DIR}/systemd/"

# Encrypt the DB dump
gpg --symmetric --cipher-algo AES256 \
    --passphrase-file /root/.backup-passphrase \
    "${DIR}/appdb.sql.gz"
rm "${DIR}/appdb.sql.gz"

# Upload off-site
aws s3 sync "${DIR}" "${BUCKET}/${TIMESTAMP}/" --storage-class STANDARD_IA

# Remove local backups older than 7 days
find /data/backups -maxdepth 1 -type d -mtime +7 -exec rm -rf {} +

echo "[${TIMESTAMP}] Backup complete"
```

```bash
# Schedule daily at 2 AM
sudo crontab -e
# Add: 0 2 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

### Recovery Objectives

| Scenario | RPO (data loss) | RTO (downtime) |
|---|---|---|
| App crash | 0 | 5 seconds (systemd restart) |
| Single disk failure | 0 | 0 (RAID continues) |
| DB corruption | Up to 24 hours | 2â€“4 hours |
| Full server loss | Up to 24 hours | 4â€“8 hours |

> These are targets. **Measure your actual RTO during restore drills.**
> Run a restore drill on a test machine every month.
> A backup that has never been tested is not a backup â€” it is an assumption.

### Monthly Restore Drill

```bash
# On a test machine
aws s3 cp s3://company-backups/prod-server-01/latest/appdb.sql.gz.gpg ./

gpg --decrypt --passphrase-file /root/.backup-passphrase \
    appdb.sql.gz.gpg | gunzip > appdb.sql

psql -U postgres -c "CREATE DATABASE appdb_test;"
psql -U postgres appdb_test < appdb.sql
psql -U postgres appdb_test -c "SELECT COUNT(*) FROM users;"

echo "Restore drill: PASSED"
```

---

## 29. Disaster Recovery

### Scenario Responses

| Scenario | Response |
|---|---|
| Application crash | systemd auto-restarts in 5s â€” no action needed |
| OOM kill | systemd restarts â€” investigate memory trend |
| One disk failure | RAID continues â€” replace drive, rebuild, monitor |
| All disks fail | Restore from off-site backup |
| Server hardware dies | Provision new server, restore from off-site |
| Ransomware | Wipe server, restore from off-site |
| Data accidentally deleted | Restore from backup |

### Full Server Rebuild Procedure

```
1.  Provision replacement hardware or cloud VM
2.  Install Ubuntu Server 24.04 LTS
3.  Configure RAID 10 on /data
4.  Install PostgreSQL / MongoDB, Nginx, Node.js
5.  Restore /etc/nginx and /etc/systemd from backup
6.  Download and decrypt DB backup from off-site
7.  Restore: psql appdb < appdb.sql
8.  Deploy application from git
9.  Restore .env from encrypted backup
10. Start: systemctl start postgresql nginx app
11. Update DNS if the IP address changed
12. Run smoke tests
13. Notify stakeholders
```

**Target: complete rebuild in under 8 hours with a documented and practiced procedure.**

Run a full drill on a test machine once per year. Update the procedure whenever steps are wrong.

---

## 30. Deployment Order

### Phase 1 â€” Hardware
- [ ] Install server, 4 RAID drives, and separate OS SSD
- [ ] Configure RAID 10
- [ ] Connect UPS and verify communication to server
- [ ] Connect redundant PSU if available

### Phase 2 â€” Network
- [ ] Configure edge firewall
- [ ] Obtain static public IP from ISP
- [ ] Configure VLANs 10, 20, 30, 40
- [ ] Set inter-VLAN rules (deny-all default + explicit allows)
- [ ] Configure NAT: 443, 80, 51820 only
- [ ] Configure WireGuard VPN
- [ ] Test: Guest VLAN cannot reach Server VLAN

### Phase 3 â€” OS
- [ ] Install Ubuntu Server 24.04 LTS (minimal)
- [ ] Verify /data RAID mount
- [ ] Set static IP: 192.168.20.20
- [ ] Apply system updates
- [ ] Configure unattended-upgrades
- [ ] Disable unnecessary services

### Phase 4 â€” SSH
- [ ] Generate Ed25519 keys on admin workstation
- [ ] Copy public key to server
- [ ] Test key-based login
- [ ] Apply hardened sshd_config
- [ ] Verify: VPN + SSH works for admin
- [ ] Verify: Internet -> SSH is denied (test externally)

### Phase 5 â€” Host Security
- [ ] Enable UFW with deny-all defaults
- [ ] Add allow rules (443, 80, SSH from VLAN/VPN)
- [ ] Add deny rules for DB and app ports
- [ ] Install and configure Fail2ban
- [ ] Verify AppArmor enforcing
- [ ] Apply sysctl hardening
- [ ] Install auditd with production rules
- [ ] Lock root account

### Phase 6 â€” Database
- [ ] Install PostgreSQL or MongoDB
- [ ] Bind to 127.0.0.1 only
- [ ] Verify: ss -tlnp shows only 127.0.0.1
- [ ] Create appdb and appuser
- [ ] Apply least-privilege grants
- [ ] Set /data/db permissions (mode 700)
- [ ] Test: psql -U appuser -h 127.0.0.1 appdb

### Phase 7 â€” Web Layer
- [ ] Install Nginx
- [ ] Configure DNS A record
- [ ] Obtain Let's Encrypt certificate
- [ ] Apply production Nginx config
- [ ] Test: nginx -t (no errors)
- [ ] Test: HTTPS in browser
- [ ] Test: HTTP redirects to HTTPS
- [ ] Test: curl -I shows all security headers

### Phase 8 â€” Application
- [ ] Create appuser service account
- [ ] Create /data/app directory structure
- [ ] Deploy application code
- [ ] Create .env (chmod 640, not in git)
- [ ] Install systemd service unit
- [ ] Enable and start: systemctl enable --now app
- [ ] Verify: app listening on 127.0.0.1:3000 only
- [ ] Test end-to-end: browser -> Nginx -> App -> DB

### Phase 9 â€” Monitoring
- [ ] Install Node Exporter (127.0.0.1:9100)
- [ ] Install DB exporter
- [ ] Connect Prometheus + Grafana
- [ ] Configure all alert thresholds
- [ ] Test: trigger an alert, verify delivery

### Phase 10 â€” Logging
- [ ] Install Promtail
- [ ] Configure shipping to Loki
- [ ] Verify logs appear in Grafana
- [ ] Test: generate a 404 in Nginx, verify it appears in Grafana

### Phase 11 â€” Backups
- [ ] Create /usr/local/bin/backup.sh
- [ ] Configure off-site storage credentials
- [ ] Run backup manually and verify all files
- [ ] Schedule daily cron at 2 AM
- [ ] Run restore drill on a test machine
- [ ] Document the tested restore procedure

### Phase 12 â€” Security Validation
- [ ] Port scan from Internet: only 80 and 443 should be visible
- [ ] Verify SSH unreachable from Internet
- [ ] Verify DB ports unreachable from Internet
- [ ] Check headers at securityheaders.com
- [ ] Run: sudo lynis audit system
- [ ] Confirm VPN + SSH works for admin
- [ ] Confirm Guest VLAN cannot reach Server VLAN

---

## 31. Pre-Production Checklist

### Hardware
- [ ] RAID 10 healthy (cat /proc/mdstat)
- [ ] UPS connected and apcupsd monitoring active
- [ ] Boot drive separate from RAID
- [ ] Temperatures normal

### Network
- [ ] Static public IP confirmed
- [ ] DNS resolves correctly
- [ ] NAT: only 443, 80, 51820 forwarded
- [ ] VLAN isolation verified
- [ ] Guest VLAN cannot reach Server VLAN (tested)
- [ ] WireGuard VPN working

### OS & Security
- [ ] Ubuntu 24.04 LTS, fully patched
- [ ] Root account locked
- [ ] sysadmin uses Ed25519 SSH key only
- [ ] UFW enabled with deny-all defaults
- [ ] Fail2ban active
- [ ] AppArmor enforcing
- [ ] auditd running
- [ ] unattended-upgrades enabled

### SSH
- [ ] Root login disabled
- [ ] Password authentication disabled
- [ ] Key authentication only
- [ ] SSH accessible only from Management VLAN + VPN
- [ ] Internet -> SSH blocked (tested from external)

### Database
- [ ] Listening on 127.0.0.1 only
- [ ] Authentication enabled
- [ ] appuser has only required grants
- [ ] DB admin account separate from application account
- [ ] /data/db owned by DB process user (mode 700)
- [ ] External access blocked (tested)

### Application
- [ ] Listening on 127.0.0.1:3000 only
- [ ] Running as appuser (not root)
- [ ] .env: chmod 640, not in git, not in repo history
- [ ] systemd: auto-restart enabled
- [ ] systemd: starts on boot
- [ ] App -> DB connection working

### Nginx & TLS
- [ ] HTTPS working
- [ ] HTTP redirects to HTTPS
- [ ] TLS 1.2 and 1.3 only
- [ ] All security headers present
- [ ] Rate limiting active
- [ ] Certbot auto-renewal verified (dry-run passes)

### Monitoring
- [ ] Node Exporter running on 127.0.0.1 only
- [ ] Grafana dashboards showing data
- [ ] All alerts configured and tested
- [ ] Alert delivery confirmed (email + Slack)

### Logging
- [ ] Promtail shipping logs to Loki
- [ ] Nginx, auth, and app logs visible in Grafana

### Backups
- [ ] Daily backup scheduled and running
- [ ] Off-site copy confirmed written
- [ ] Backup encrypted
- [ ] Restore drill completed on a test machine
- [ ] Restore procedure documented and reviewed

---

## Summary

### Architecture in One View

```
Internet
    |
    v  Ports 443, 80, 51820 only
Edge Firewall  (NAT + VLANs + WireGuard VPN)
    |
    v  HTTPS only
Nginx  (TLS termination, security headers, rate limiting)
    |
    v  127.0.0.1:3000  (loopback)
Application  (appuser, systemd, loopback bind)
    |
    v  127.0.0.1:5432  (loopback)
Database  (127.0.0.1 bind, least-privilege user)
    |
    v
RAID 10 /data  (survives 1 disk failure per mirror)
    |
    v
Encrypted daily backup  ->  Off-site storage
    |
Monitoring (Prometheus + Grafana, all on loopback)
Logging    (Promtail -> Loki, logs leave before they can be wiped)
```

### Component Responsibilities

| Component | Responsibility |
|---|---|
| Edge Firewall | Network perimeter: NAT, VLANs, VPN endpoint |
| UFW | Host defense-in-depth: blocks anything the edge misses |
| Nginx | Public entry: TLS, headers, rate limiting, proxy |
| Application | Business logic: loopback only, least-privilege user |
| Database | Data storage: loopback only, restricted grants |
| RAID 10 | Disk redundancy: survives one drive failure per mirror |
| Backups | Disaster recovery: survives total server destruction |
| Monitoring | Visibility: detect failures before users notice |
| WireGuard VPN | Authenticated network access for remote staff |
| SSH | Server administration: VPN + key both required |
