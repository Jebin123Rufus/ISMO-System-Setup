# Production Server Architecture & Deployment Plan

## 1. Objective

Deploy a physical company server that:

* Runs 24/7.
* Hosts the production application.
* Hosts the production database.
* Is reachable by customers over the Internet.
* Allows employees to access authorized internal resources.
* Allows authorized remote employees to access internal resources through VPN.
* Provides secure administrative access through SSH.
* Uses RAID 10 for storage redundancy.
* Maintains separate backups.
* Provides monitoring, logging, and security controls.

---

# 2. High-Level Architecture

```text
                              INTERNET
                                  |
                                  |
                           Public IP Address
                                  |
                                  v
                    +--------------------------+
                    |   EDGE FIREWALL/ROUTER   |
                    |                          |
                    | NAT                      |
                    | Firewall                 |
                    | VPN Gateway              |
                    | Routing                  |
                    +------------+-------------+
                                 |
                    +------------+-------------+
                    |                          |
                    v                          v
             PUBLIC WEB TRAFFIC           COMPANY LAN/VPN
                    |                          |
                    v                          |
              +-----------+                    |
              |   NGINX   |<-------------------+
              | Reverse   |
              | Proxy     |
              +-----+-----+
                    |
                    v
          +----------------------+
          |   APPLICATION        |
          | Node.js / Spring     |
          +----------+-----------+
                     |
                     | Private DB connection
                     v
          +----------------------+
          |      DATABASE        |
          | MongoDB / PostgreSQL |
          +----------+-----------+
                     |
                     v
                  RAID 10
                     |
              +------+------+
              |             |
            Disk          Disk
              |             |
              +------+------+
                     |
                     v
             SEPARATE BACKUPS
                     |
                     v
               OFF-SITE COPY
```

---

# 3. Network Architecture

The company network should not be completely flat.

A basic segmentation design:

```text
                    FIREWALL
                        |
        +---------------+----------------+
        |               |                |
        v               v                v
   VLAN 10          VLAN 20          VLAN 30
   Employees        Servers          Guests
        |               |                |
     Employee        Production        Guest
       PCs            Server           Devices
```

Recommended:

| Network         | Example subnet    | Purpose                       |
| --------------- | ----------------- | ----------------------------- |
| Employee VLAN   | `192.168.10.0/24` | Employee computers            |
| Server VLAN     | `192.168.20.0/24` | Production servers            |
| Guest VLAN      | `192.168.30.0/24` | Guest Wi-Fi                   |
| Management VLAN | `192.168.40.0/24` | Network/server administration |
| VPN network     | `10.20.0.0/24`    | Remote employees              |

These addresses are examples. The actual addressing should be chosen based on the company's existing network.

---

# 4. Hardware

A production server should ideally have:

* Server-grade CPU
* ECC RAM
* At least 16–32 GB RAM depending on workload
* Multiple SSDs
* RAID controller or supported software RAID
* Multiple network interfaces if appropriate
* Redundant PSU if supported
* Good cooling
* UPS
* Hardware monitoring

Example:

```text
CPU:       Server-grade CPU
RAM:       32 GB ECC
Storage:   4 × 2 TB enterprise SSD
RAID:      RAID 10
Network:   1/10 GbE depending on workload
PSU:       Redundant if available
UPS:       Yes
OS:        Ubuntu Server 24.04 LTS
```

Hardware sizing should ultimately be based on expected traffic, database size, IOPS, and growth.

---

# 5. RAID 10

Use four drives for a basic RAID 10 configuration.

```text
             RAID 10
                |
       +--------+--------+
       |                 |
     Mirror            Mirror
     Disk 1            Disk 3
       |                 |
     Disk 2            Disk 4
       |                 |
       +--------+--------+
                |
             Storage
```

RAID 10 combines:

* Mirroring
* Striping

### Advantages

* Good read performance
* Good write performance
* Drive redundancy
* Suitable for database workloads

### Capacity

With:

```text
4 × 2 TB
```

Raw capacity:

```text
8 TB
```

Approximate RAID 10 usable capacity:

```text
4 TB
```

Actual usable capacity will be somewhat lower after filesystem and system overhead.

### Important

RAID is **not a backup**.

```text
RAID = availability/redundancy

Backup = recovery
```

---

# 6. UPS

Power architecture:

```text
Utility Power
      |
      v
     UPS
      |
      v
   Server
```

The UPS protects against:

* Short power outages
* Power interruptions
* Abrupt shutdowns

Configure the server to perform a controlled shutdown if the UPS battery becomes critically low.

---

# 7. Operating System

Install:

```text
Ubuntu Server 24.04 LTS
```

Do not install Ubuntu Desktop unless there is a specific requirement for a graphical environment.

The server should primarily be administered through SSH.

```text
Administrator Laptop
        |
        | SSH
        v
Ubuntu Server
```

---

# 8. Ubuntu Installation

During installation:

1. Boot from the Ubuntu Server installation media.
2. Configure the server's RAID/storage system.
3. Install Ubuntu Server.
4. Configure hostname.

Example:

```text
Hostname:
prod-server-01
```

5. Configure a static private IP.

Example:

```text
IP:       192.168.20.20
Gateway:  192.168.20.1
DNS:      Company DNS / trusted resolver
```

6. Create a non-root administrative user.
7. Install OpenSSH Server.
8. Complete installation.
9. Reboot.

---

# 9. Initial Ubuntu Configuration

Update the system immediately:

```bash
sudo apt update
sudo apt upgrade
```

Install basic tools:

```bash
sudo apt install curl wget git vim htop unzip ca-certificates
```

Check system information:

```bash
hostnamectl
ip addr
ip route
lsblk
df -h
```

Check listening services:

```bash
sudo ss -tulpn
```

---

# 10. Create Administrative User

Create a dedicated administrator:

```bash
sudo adduser admin
```

Add the user to sudo:

```bash
sudo usermod -aG sudo admin
```

Do not use the root account for routine administration.

---

# 11. SSH Key Authentication

Generate an SSH key on the administrator's workstation:

```bash
ssh-keygen -t ed25519
```

This produces:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

The private key:

```text
id_ed25519
```

must remain on the administrator's device.

The public key:

```text
id_ed25519.pub
```

is placed on the server.

Example:

```bash
ssh-copy-id admin@192.168.20.20
```

The server stores the authorized public key in:

```text
/home/admin/.ssh/authorized_keys
```

Authentication conceptually works like:

```text
Administrator
     |
     | Private key
     v
Digital signature
     |
     v
SSH Server
     |
     | Verify using public key
     v
Authentication succeeds
```

The private key itself is never sent to the server.

---

# 12. SSH Hardening

After confirming key-based login works:

* Disable direct root SSH login.
* Prefer disabling password authentication.
* Restrict SSH to the management network/VPN.
* Use a non-root administrative account.
* Keep SSH patched.

Example configuration:

```text
/etc/ssh/sshd_config
```

Relevant settings:

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

After changes:

```bash
sudo systemctl restart ssh
```

Do not disable password authentication until key authentication has been tested successfully.

---

# 13. Host Firewall

Ubuntu should have its own firewall even though there is an external firewall.

Concept:

```text
Internet
   |
   v
Edge Firewall
   |
   v
Ubuntu Host Firewall
   |
   v
Services
```

Using UFW:

```bash
sudo apt install ufw
```

Default policy:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Allow HTTPS:

```bash
sudo ufw allow 443/tcp
```

Allow HTTP if needed for redirection:

```bash
sudo ufw allow 80/tcp
```

SSH should only be allowed from the management network or VPN.

Example:

```bash
sudo ufw allow from 192.168.40.0/24 to any port 22 proto tcp
```

Enable:

```bash
sudo ufw enable
```

Check:

```bash
sudo ufw status verbose
```

Do not expose the database publicly.

---

# 14. Public IP and DNS

The ISP provides the public IP.

Example:

```text
Public IP:
203.0.113.50
```

DNS:

```text
company.com
     |
     v
203.0.113.50
```

The public IP belongs to the company's network edge.

The production server can continue using a private IP:

```text
Public IP:
203.0.113.50

Server:
192.168.20.20
```

---

# 15. NAT / Port Forwarding

The firewall/router forwards only required public ports.

Example:

```text
Internet
   |
   | 203.0.113.50:443
   v
Firewall
   |
   | NAT
   v
192.168.20.20:443
```

Only required services should be forwarded.

Example:

```text
443 → Nginx
80  → Nginx (optional HTTP → HTTPS redirect)
```

Do NOT forward:

```text
27017 → MongoDB
5432  → PostgreSQL
3000  → Node.js
8080  → internal application
```

unless there is a specific, justified requirement.

---

# 16. Install Nginx

Install:

```bash
sudo apt install nginx
```

Enable and start:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

Check:

```bash
sudo systemctl status nginx
```

---

# 17. Nginx Reverse Proxy

The public traffic path becomes:

```text
Customer
   |
   | HTTPS
   v
Public IP
   |
   v
Firewall
   |
   v
Nginx
   |
   | Internal request
   v
Application
```

Suppose the application listens internally on:

```text
127.0.0.1:3000
```

Nginx can proxy requests to it.

Conceptual configuration:

```nginx
server {
    listen 80;
    server_name company.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

For production, configure HTTPS and redirect HTTP to HTTPS.

---

# 18. HTTPS/TLS

The public endpoint should use:

```text
https://company.com
```

instead of:

```text
http://company.com
```

Traffic:

```text
Customer
   |
   | Encrypted HTTPS
   v
Nginx
   |
   v
Application
```

Nginx can terminate TLS.

This is called:

```text
TLS termination
```

Use a trusted certificate and automate certificate renewal.

---

# 19. Application Deployment

Deploy the application behind Nginx.

Example:

```text
Nginx
  |
  v
Node.js / Express
```

or:

```text
Nginx
  |
  v
Spring Boot
```

The application should listen only on an internal interface where possible.

Example:

```text
127.0.0.1:3000
```

rather than:

```text
0.0.0.0:3000
```

if the application doesn't need direct network access.

---

# 20. Application Process Management

The application should automatically restart after failure.

Possible approaches:

* systemd
* Docker
* Docker Compose
* another appropriate process supervisor

Example architecture:

```text
Application
     |
     v
systemd
     |
     +--> restart if process fails
```

The service should also start automatically after reboot.

---

# 21. Database

Install the chosen database.

Examples:

```text
MongoDB
PostgreSQL
MySQL
```

The database should remain private.

Correct:

```text
Internet
   |
   v
Nginx
   |
   v
Application
   |
   v
Database
```

Incorrect:

```text
Internet
   |
   v
MongoDB :27017
```

---

# 22. Database Security

Create a dedicated database user for the application.

Do not use the database administrator account from the application.

Concept:

```text
Database
|
+-- Administrative account
|
+-- Application account
       |
       +-- Required database permissions only
```

Use:

* Strong credentials
* Authentication
* Encryption where appropriate
* Least privilege
* Network restrictions
* Regular backups

---

# 23. Application Secrets

Never put production secrets directly into source code.

Do not:

```javascript
const password = "ProductionPassword123";
```

Do not commit:

```text
.env
```

to GitHub.

Use:

* Environment variables
* Secure secret storage
* Proper filesystem permissions
* Secret management systems as the infrastructure grows

---

# 24. Employee Network

Employees should use the company LAN.

```text
Employee PC
     |
     v
Switch / Wi-Fi
     |
     v
Employee VLAN
     |
     v
Firewall
     |
     v
Authorized internal resources
```

Employees should not automatically have access to:

```text
Database
SSH
Network management
Other restricted services
```

Access should be based on business requirements.

---

# 25. Remote Employee VPN

Remote employees should not require public exposure of internal services.

Architecture:

```text
Remote Employee
       |
       | Encrypted VPN tunnel
       v
Internet
       |
       v
Company Firewall / VPN Gateway
       |
       v
Private Network
       |
       v
Authorized Resources
```

VPN technologies can include:

* WireGuard
* OpenVPN
* IPsec/IKEv2
* Enterprise firewall VPN

---

# 26. VPN Network

Example:

```text
VPN subnet:
10.20.0.0/24
```

An employee might receive:

```text
10.20.0.25
```

After authentication, the firewall controls which internal resources they can reach.

Example:

```text
VPN employee
     |
     +---- Internal application :443     ALLOW
     |
     +---- File server :445              ALLOW
     |
     +---- Database :27017               DENY
     |
     +---- SSH :22                       DENY
```

An administrator could receive different permissions.

---

# 27. VPN + SSH for Administrators

For remote server administration:

```text
Administrator
      |
      v
VPN
      |
      v
Company Network
      |
      v
SSH
      |
      v
Ubuntu Server
```

Then SSH uses the administrator's key:

```text
Private Key
     |
     v
Digital Signature
     |
     v
SSH Server
     |
     v
Public Key Verification
```

This creates two security boundaries:

```text
VPN → Can you reach the private network?

SSH → Are you authorized to log into this server?
```

---

# 28. Network Access Control

Example firewall policy:

```text
Internet
    |
    +--> TCP 443 --> Nginx              ALLOW
    |
    +--> TCP 80  --> Nginx              ALLOW/REDIRECT
    |
    +--> TCP 22  --> Production Server  DENY
    |
    +--> TCP 27017 --> Database         DENY
    |
    +--> Other ports                    DENY
```

Management network:

```text
Management VLAN
    |
    +--> SSH :22 --> Server             ALLOW
```

VPN administrators:

```text
Admin VPN
    |
    +--> SSH :22 --> Server             ALLOW
```

---

# 29. Application Architecture

A basic application flow:

```text
Customer
   |
   | HTTPS
   v
Nginx
   |
   | HTTP/internal connection
   v
Application
   |
   | Database protocol
   v
Database
```

Example request:

```text
GET /orders
```

Flow:

```text
Customer
   |
   v
Nginx
   |
   v
Application
   |
   v
Database
   |
   v
Application
   |
   v
Nginx
   |
   v
Customer
```

---

# 30. Security Headers

Configure appropriate security headers at the application or Nginx layer.

Examples include:

```text
Content-Security-Policy
Strict-Transport-Security
X-Content-Type-Options
Referrer-Policy
```

The exact policy should be tested against the application rather than copied blindly.

---

# 31. Rate Limiting

Nginx/application-level rate limiting can reduce abuse.

Example:

```text
Client
   |
   | 1000 requests/sec
   v
Nginx
   |
   | Rate limiting
   v
Application
```

This protects application resources against certain forms of abuse.

It is not a complete DDoS solution.

---

# 32. DDoS Protection

For a serious public application, consider placing upstream protection in front of the server:

```text
Internet
    |
    v
CDN / DDoS Protection / WAF
    |
    v
Firewall
    |
    v
Nginx
    |
    v
Application
```

The exact service depends on the company's requirements and provider.

---

# 33. Host Security

The Ubuntu server should have:

### Firewall

```text
UFW / nftables
```

### AppArmor

Restricts application capabilities.

### Least privilege

Services should not run as root unnecessarily.

### Updates

Regular security updates.

### SSH hardening

Key-based authentication and restricted access.

### Unnecessary services

Disable/remove anything that isn't required.

---

# 34. Monitoring

Monitor:

```text
Hardware
|
+-- CPU
+-- RAM
+-- Disk
+-- RAID
+-- Temperature
+-- Network
|
Services
|
+-- Nginx
+-- Application
+-- Database
|
Security
|
+-- SSH
+-- Firewall
+-- Authentication
```

Set alerts for:

```text
High CPU
High RAM
Low disk space
RAID disk failure
Application failure
Database failure
Server unreachable
Suspicious login attempts
Certificate expiration
```

---

# 35. Logging

Collect logs from:

```text
Nginx
Application
Database
SSH
Ubuntu
Firewall
```

Example:

```text
Customer request
      |
      v
Nginx access log
      |
      v
Application log
      |
      v
Database log
```

For a mature deployment, consider centralized logging so an attacker who compromises one machine cannot simply erase every useful log.

---

# 36. Backups

Use separate backup storage.

```text
Production Server
       |
       v
Backup System
       |
       v
Off-site Backup
```

Back up:

* Database
* Application configuration
* Important server configuration
* Certificates/keys where appropriate and securely stored
* Critical company data

Do not rely on RAID as a backup.

---

# 37. Backup Strategy

A practical strategy should include:

```text
Daily backups
+
Retention policy
+
Off-site copy
+
Encryption
+
Regular restore tests
```

A backup is not considered reliable until you have demonstrated that it can actually be restored.

---

# 38. Disaster Recovery

If the physical server dies:

```text
Server failure
     |
     v
Replace/repair hardware
     |
     v
Configure RAID
     |
     v
Install Ubuntu
     |
     v
Restore configuration
     |
     v
Deploy application
     |
     v
Restore database
     |
     v
Test
     |
     v
Return to production
```

Document this procedure before a disaster happens.

---

# 39. RAID Failure

If one drive fails:

```text
RAID 10
   |
   +-- Disk 1 ❌
   +-- Disk 2 ✅
   +-- Disk 3 ✅
   +-- Disk 4 ✅
```

The array can continue operating depending on the failure pattern.

Replace the failed drive and rebuild the array.

Monitor the rebuild carefully.

---

# 40. RAID Is Not Backup

Remember:

```text
RAID
 ↓
Hardware failure protection

Backup
 ↓
Data recovery
```

RAID does not protect against:

* Accidental deletion
* Application bugs
* Ransomware
* Malicious administrator actions
* Database corruption
* Fire
* Theft
* Complete server destruction

Separate backups address these scenarios.

---

# 41. Power Failure

Architecture:

```text
Power
  |
  v
UPS
  |
  v
Server
```

The UPS should be monitored.

If power remains unavailable:

```text
UPS battery low
      |
      v
Controlled server shutdown
```

---

# 42. Security Architecture

The security model should follow defense in depth:

```text
                    ATTACKER
                       |
                       v
                Edge Firewall
                       |
                       v
                 Network ACLs
                       |
                       v
                    Nginx
                       |
                       v
              Application Security
                       |
                       v
              Authentication
                       |
                       v
                Authorization
                       |
                       v
              Database Security
                       |
                       v
                   Backups
```

No single component should be considered the only security control.

---

# 43. Public Attack Surface

Keep the Internet-facing attack surface small.

Ideally:

```text
PUBLIC
  |
  +-- TCP 443 → Nginx
  |
  +-- TCP 80  → HTTP → HTTPS redirect
```

Avoid publicly exposing:

```text
MongoDB
PostgreSQL
Redis
Node.js
Spring Boot
SSH
Internal admin tools
```

unless there is a specific architectural reason.

---

# 44. Complete Traffic Flows

## Customer

```text
Customer
   |
   v
Internet
   |
   v
Public IP
   |
   v
Edge Firewall
   |
   v
Nginx
   |
   v
Application
   |
   v
Database
```

## Employee in office

```text
Employee
   |
   v
Company LAN
   |
   v
Employee VLAN
   |
   v
Firewall
   |
   v
Authorized Internal Resource
```

## Remote employee

```text
Employee
   |
   v
Internet
   |
   v
VPN
   |
   v
Company Firewall
   |
   v
Private Network
   |
   v
Authorized Resource
```

## Remote administrator

```text
Administrator
   |
   v
VPN
   |
   v
Private Network
   |
   v
SSH
   |
   v
Ubuntu Server
   |
   v
SSH Public-Key Authentication
```

## Application → Database

```text
Application
     |
     | Private network
     v
Database
```

The database does not need to be exposed to customers.

---

# 45. Production Deployment Order

The recommended implementation order is:

## Phase 1 — Hardware

* Install server hardware.
* Install RAID drives.
* Configure RAID 10.
* Connect redundant power if available.
* Connect UPS.
* Connect network interfaces.

## Phase 2 — Network

* Configure ISP connection.
* Configure firewall/router.
* Obtain public IP.
* Configure internal addressing.
* Configure VLANs.
* Configure server VLAN.
* Configure management network.
* Configure employee network.
* Configure guest network.
* Configure VPN.

## Phase 3 — Operating System

* Install Ubuntu Server 24.04 LTS.
* Configure hostname.
* Configure static IP.
* Create administrator.
* Install OpenSSH.
* Apply updates.
* Configure host firewall.
* Enable AppArmor.
* Disable unnecessary services.

## Phase 4 — SSH

* Generate Ed25519 SSH keys.
* Install public key on server.
* Test key authentication.
* Disable direct root login.
* Disable password authentication after verification.
* Restrict SSH to management/VPN networks.

## Phase 5 — Web Layer

* Install Nginx.
* Configure domain.
* Configure DNS.
* Configure HTTP/HTTPS.
* Install TLS certificate.
* Configure reverse proxy.
* Configure security headers.
* Configure access logging.
* Configure rate limiting where appropriate.

## Phase 6 — Application

* Deploy application.
* Configure environment variables/secrets.
* Create dedicated service user.
* Configure application process manager.
* Bind application to an internal interface.
* Test application.
* Configure automatic restart.

## Phase 7 — Database

* Install database.
* Enable authentication.
* Create application database/user.
* Restrict network access.
* Configure database storage.
* Configure database backups.
* Test database connection from application.

## Phase 8 — Employees

* Configure employee VLAN.
* Configure access rules.
* Configure internal applications/resources.
* Configure VPN access for remote employees.
* Apply least-privilege access.

## Phase 9 — Monitoring

* Monitor CPU.
* Monitor RAM.
* Monitor storage.
* Monitor RAID.
* Monitor network.
* Monitor Nginx.
* Monitor application.
* Monitor database.
* Monitor SSH/security events.
* Configure alerts.

## Phase 10 — Backups

* Configure automated database backups.
* Configure server configuration backups.
* Store backups separately.
* Maintain off-site copies.
* Encrypt sensitive backups.
* Test restoration.

## Phase 11 — Security Testing

Test:

```text
External:
- Port scanning
- HTTPS configuration
- Unnecessary exposed ports
- Application vulnerabilities

Internal:
- VLAN isolation
- Employee access restrictions
- VPN access
- Database accessibility

Server:
- SSH configuration
- Firewall
- Permissions
- Services
- Updates
```

---

# 46. Pre-Production Checklist

## Hardware

* [ ] RAID 10 configured
* [ ] RAID health verified
* [ ] UPS configured
* [ ] Cooling verified
* [ ] Hardware monitoring configured

## Network

* [ ] Public IP configured
* [ ] DNS configured
* [ ] Firewall configured
* [ ] NAT configured
* [ ] Server VLAN configured
* [ ] Employee VLAN configured
* [ ] Guest VLAN configured
* [ ] Management network configured
* [ ] VPN configured

## Ubuntu

* [ ] Ubuntu Server LTS installed
* [ ] System updated
* [ ] Static IP configured
* [ ] Administrator created
* [ ] SSH configured
* [ ] Host firewall enabled
* [ ] AppArmor enabled
* [ ] Unnecessary services disabled

## SSH

* [ ] Ed25519 keys generated
* [ ] Public key installed
* [ ] Key authentication tested
* [ ] Root SSH disabled
* [ ] Password SSH disabled
* [ ] SSH restricted to management/VPN

## Nginx

* [ ] Nginx installed
* [ ] Domain configured
* [ ] HTTPS configured
* [ ] TLS certificate installed
* [ ] Reverse proxy configured
* [ ] Security headers configured
* [ ] Logging configured

## Application

* [ ] Application deployed
* [ ] Application user created
* [ ] Secrets secured
* [ ] Application bound internally
* [ ] Automatic restart configured
* [ ] Health check configured

## Database

* [ ] Database installed
* [ ] Authentication enabled
* [ ] Application DB user created
* [ ] Least privilege applied
* [ ] Database not publicly accessible
* [ ] Backups configured

## Employees

* [ ] LAN access tested
* [ ] VLAN restrictions tested
* [ ] VPN tested
* [ ] Internal resources tested
* [ ] Least privilege verified

## Monitoring

* [ ] CPU monitoring
* [ ] RAM monitoring
* [ ] Disk monitoring
* [ ] RAID monitoring
* [ ] Network monitoring
* [ ] Application monitoring
* [ ] Database monitoring
* [ ] Security alerts

## Backup

* [ ] Database backup tested
* [ ] Configuration backup tested
* [ ] Off-site backup configured
* [ ] Restore procedure documented
* [ ] Restore test completed

---

# 47. Final Architecture

```text
                               INTERNET
                                   |
                                   |
                              Public IP
                                   |
                                   v
                     +-------------------------+
                     |    EDGE FIREWALL        |
                     |                         |
                     | NAT                     |
                     | Firewall                |
                     | VPN                     |
                     | Routing                 |
                     +------------+------------+
                                  |
              +-------------------+-------------------+
              |                   |                   |
              v                   v                   v
         Public Web          Employee LAN        Remote VPN
              |                   |                   |
              v                   v                   |
        +-----------+        Employee VLAN            |
        |   NGINX   |              |                  |
        | HTTPS      |              |                  |
        | Reverse    |              |                  |
        | Proxy      |              |                  |
        +-----+-----+              |                  |
              |                    |                  |
              +--------------------+------------------+
                                   |
                                   v
                       +-----------------------+
                       |   SERVER VLAN         |
                       |                       |
                       |  Ubuntu Server        |
                       |                       |
                       |  +----------------+  |
                       |  | Nginx           |  |
                       |  +-------+--------+  |
                       |          |            |
                       |  +-------v--------+  |
                       |  | Application    |  |
                       |  +-------+--------+  |
                       |          |            |
                       |  +-------v--------+  |
                       |  | Database       |  |
                       |  +----------------+  |
                       |                       |
                       |  Host Firewall        |
                       |  AppArmor              |
                       |  SSH                   |
                       |  Monitoring            |
                       |  Logging               |
                       +-----------+-----------+
                                   |
                                RAID 10
                                   |
                     +-------------+-------------+
                     |                           |
                 Production                  Separate
                    Data                     Backups
                                                 |
                                                 v
                                           Off-site Copy
```

---

# 48. Core Security Principle

The architecture should follow this rule:

```text
Internet
   ↓
Only expose what is necessary
   ↓
Authenticate users
   ↓
Authorize only required actions
   ↓
Keep databases/internal services private
   ↓
Segment networks
   ↓
Use least privilege
   ↓
Monitor everything important
   ↓
Maintain independent backups
   ↓
Test recovery
```

The most important distinction to retain is:

```text
Firewall  → Controls network access

VPN       → Provides authenticated private network access

SSH       → Authenticates administrators to the server

Nginx     → Public web entry point / reverse proxy

Application → Business logic

Database  → Data storage

RAID 10   → Storage redundancy

Backup    → Disaster recovery

Monitoring → Detects failures and suspicious activit
```
