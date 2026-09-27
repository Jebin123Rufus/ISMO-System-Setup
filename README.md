# Production Architecture: Single-Server Deployment

## 1. Architectural Foundation & Optimal Technology Stack

This architecture establishes a hardened, single-node bare-metal production deployment that co-locates the reverse proxy, application runtime, and persistence database on a single physical host. 

The core architectural mandate is **hardware-level performance with operating-system-level isolation**: inter-service traffic executes in-memory over the virtual loopback adapter (`127.0.0.1`) at microsecond latencies, while security, resource limits, and privilege separation are strictly enforced by the Linux kernel, systemd cgroups, and network firewalls. The architecture is fully **language-agnostic and database-agnostic**.

### Optimal Production Technology Stack & Versions

| Architectural Layer | Recommended Technology | Optimal Production Version | Core Architectural Role |
|---|---|---|---|
| **Operating System** | Ubuntu Server LTS | **24.04 LTS (Noble Numbat)** | Linux Kernel 6.8+ providing cgroups v2 resource envelopes, AppArmor, eBPF telemetry hooks, and long-term security maintenance. |
| **Server Hardware** | Enterprise 1U/2U Rack Server (Dell PowerEdge / HPE ProLiant) | **Latest Gen (AMD EPYC / Intel Xeon)** | 16+ Cores, 64GB+ ECC RAM, Dual Redundant Hot-Swap PSUs, IPMI/iDRAC out-of-band management on isolated VLAN. |
| **Storage Subsystem** | 4x Enterprise NVMe SSDs | **PCIe Gen 4/5 Enterprise** | Arranged in RAID 10 (Striped Mirrors) for high random-write IOPS, zero parity latency, and multi-drive failure tolerance. |
| **Power Continuity** | Smart On-Line UPS with USB/SNMP | **Network UPS Tools (NUT) / apcupsd** | Battery backup with dual-rail power feeds; triggers automated database buffer flush and graceful shutdown on power failure. |
| **Edge Firewall / Router** | OPNsense / pfSense | **OPNsense 24.7 / pfSense Plus 24.03** | Perimeter NAT gateway, Layer-3 VLAN routing, stateful packet filtering, and WireGuard VPN tunnel endpoint. |
| **Reverse Proxy & Ingress** | Nginx | **1.26 LTS** | High-concurrency event-driven TLS 1.3 termination, HTTP security headers, leaky-bucket rate limiting, and request buffering. |
| **Host-Level Firewall** | UFW / Netfilter (iptables) | **UFW 0.36+ (iptables 1.8.10+)** | Autonomous host packet filtering with default-deny ingress; drops external access to internal application and database ports. |
| **Administrative Bastion** | WireGuard | **1.0.0+ (In-Kernel Module)** | Encrypted Noise protocol tunnel (ChaCha20-Poly1305) with Ed25519 key authentication; eliminates public SSH port exposure. |
| **Process Supervision** | systemd | **v255+** | Native PID 1 process lifecycle management, automated crash recovery, hard memory ceilings (`MemoryMax`), and filesystem sandboxing. |
| **Observability Telemetry** | Prometheus Node Exporter & Promtail | **Node Exporter 1.8+, Promtail 3.1+** | Out-of-band metrics export and real-time inotify log streaming to external Grafana and Loki clusters. |
| **Disaster Recovery Storage** | S3-Compatible Object Storage | **AWS S3 / Wasabi (WORM Mode)** | Off-site immutable backup repository with Object Lock in Compliance Mode, providing immunity against ransomware. |

### The Four Isolation Rings of the Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│  RING 1: HARDWARE & STORAGE FAULT TOLERANCE                              │
│  • ECC RAM (Auto-corrects bit-flips; prevents silent database corruption)│
│  • NVMe RAID 10 (High IOPS + survives up to 2 physical drive failures)   │
│  • Dual Hot-Swap PSUs + Smart UPS (Automated transaction-safe shutdown)  │
├──────────────────────────────────────────────────────────────────────────┤
│  RING 2: NETWORK & PERIMETER ZERO-TRUST                                  │
│  • Edge Firewall: NAT forwards only TCP 443 & 80 to private server IP    │
│  • Corporate LAN (VLAN 10): Direct access to server VLAN 20 is BLOCKED   │
│  • Management Plane: SSH reachable ONLY via encrypted WireGuard VPN      │
├──────────────────────────────────────────────────────────────────────────┤
│  RING 3: OPERATING SYSTEM & KERNEL SANDBOXING                            │
│  • POSIX Separation: 'appuser' cannot read database files or /etc        │
│  • Systemd Cgroups: 'MemoryMax' caps app RAM; prevents DB starvation     │
│  • Filesystem Sandboxing: 'ProtectSystem=strict' & 'PrivateTmp=true'     │
├──────────────────────────────────────────────────────────────────────────┤
│  RING 4: SERVICE & DATA PIPELINE ISOLATION                               │
│  • Loopback Binding: DB and App listen strictly on 127.0.0.1             │
│  • Privilege Scoping: Application user holds DML-only permissions        │
│  • Backup Immutability: Off-site S3 Object Lock prevents deletion        │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Core Architecture & Inter-Service Workflow

The backend application service and the database engine run co-located on a single physical host. Communication between tiers stays strictly inside memory over the virtual loopback adapter (`127.0.0.1`) and local Unix domain sockets. The architecture operates identically regardless of the chosen backend programming language or database engine.

![Macro Architecture Blueprint](assets/01-macro-architecture.jpg)

### End-to-End Traffic Pipeline

1. **Incoming Traffic Ingestion:**
   - **Public Users:** Inbound web traffic arrives via public DNS and hits the edge perimeter on TCP port 443 (HTTPS).
   - **Office Staff:** Employees on the office network reach the application through the same public HTTPS gateway.
   - **Remote Employees / Admins:** Connect through an encrypted WireGuard VPN tunnel on UDP port 51820.

2. **Perimeter Edge Firewall / NAT Gateway (OPNsense / pfSense):**
   - Enforces perimeter firewall policies and network address translation (NAT).
   - Forwards only TCP port 443 and port 80 to the private server IP (`192.168.20.10`).
   - Acts as the WireGuard VPN termination endpoint for authorized administrative access.
   - Drops all unsolicited scans and probe attempts before they reach the server switchport.

3. **Ingress & Reverse Proxy Layer (Nginx :443):**
   - Terminates TLS 1.3 encryption using modern cryptographic ciphers.
   - Applies dynamic rate limiting and validates HTTP request headers.
   - Proxies validated requests directly to the application service via internal loopback (`127.0.0.1:3000`).

4. **Application Service Tier (Backend Runtime):**
   - Runs under a dedicated, unprivileged system user (`appuser`) with no interactive login shell.
   - Bounded by systemd cgroups (`MemoryMax` and `CPUQuota`) to prevent memory leaks from impacting the host.
   - Processes business logic and communicates with the database strictly over loopback (`127.0.0.1:<port>`).

5. **Persistence Engine Tier (Database Engine):**
   - Bound strictly to the localhost loopback interface (`127.0.0.1`); rejects all external physical network connections.
   - Authenticates application connections using salted cryptographic password hashes.
   - Stores all data files in `/data/db` with exclusive directory permissions (`0700`, accessible only by the database service).

6. **Storage Subsystem & Off-Site Data Pipeline:**
   - Operates on a high-speed enterprise NVMe RAID 10 storage array divided into `/data/db`, `/data/app`, and `/data/backups`.
   - Executes daily automated encrypted backups that are pushed off-site to AWS S3/Wasabi with Object Lock enabled for ransomware defense.

---

## 3. Zero-Trust Access Model: Public, Office LAN & Remote VPN

Every incoming request is verified and authenticated regardless of network origin. Physical presence on the corporate network grants zero implicit trust or access to the server.

![Zero-Trust Access Model](assets/02-zero-trust-access-model.jpg)

### User Access Profiles & Routing Paths

1. **Profile 1: Public Customers (Global Internet):**
   - Resolves the application domain via Anycast DNS and connects over public HTTPS (TCP 443).
   - Traffic enters through the edge gateway, is NAT-forwarded to Nginx, and proxies internally to the backend application.

2. **Profile 2: Office Employees (Corporate LAN - VLAN 10):**
   - Employee workstations, laptops, and Wi-Fi devices reside in an isolated subnet (VLAN 10: `192.168.10.0/24`).
   - **Layer-3 Access Rule:** All direct IP traffic from VLAN 10 to the Production Server VLAN 20 (`192.168.20.10`) is **STRICTLY BLOCKED** at the router switchport.
   - Employees access business applications through the public HTTPS domain like external customers, authenticating through application identity providers.
   - Physical presence on the office network provides zero access to internal databases, administrative shells, or server management ports.

3. **Profile 3: Remote System Administrators (WireGuard VPN):**
   - Systems engineers establish an encrypted WireGuard VPN tunnel on UDP port 51820.
   - Connections authenticate using pre-configured, non-exportable Ed25519 cryptographic public keys.
   - Once the tunnel is verified, the administrator receives an internal VPN IP (`10.20.0.0/24`).
   - Only from this VPN subnet is SSH access (TCP 22) permitted by the server's firewall. Direct root login is disabled, and administrative commands require audited `sudo` elevation.

### Host-Level Firewall (UFW) Security Policy

The physical server runs an autonomous host-level firewall (UFW / Netfilter) enforcing strict stateful packet filtering:

| Port / Protocol | Allowed Source Interface | Direction | Firewall Action | Architectural Guarantee |
|---|---|---|---|---|
| **TCP 443 (HTTPS)** | Any (`0.0.0.0/0`) | Inbound | **ACCEPT** | Passes public web traffic directly to Nginx reverse proxy. |
| **TCP 80 (HTTP)** | Any (`0.0.0.0/0`) | Inbound | **ACCEPT** | Passes traffic to Nginx for immediate 301 redirect to HTTPS. |
| **TCP 22 (SSH)** | VPN Subnet (`10.20.0.0/24`) ONLY | Inbound | **ACCEPT** | Administrative shell access permitted exclusively over WireGuard VPN. |
| **Application Port (3000)** | Loopback (`127.0.0.1`) ONLY | Inbound | **DROP on Physical NICs** | Inaccessible from external cables; accepts packets only from Nginx. |
| **Database Port (5432/27017)**| Loopback (`127.0.0.1`) ONLY | Inbound | **DROP on Physical NICs** | Inaccessible from external cables; accepts packets only from App. |
| **All Other Ports** | Any | Inbound | **DEFAULT DROP** | Silent packet drop with zero ICMP feedback to port scanners. |

---

## 4. Ingress Traffic & Reverse Proxy Architecture

Nginx serves as the single public-facing software component on the physical server, shielding internal services from raw network traffic:

```
[ Inbound Request ] ──► [ Nginx 1.26 LTS ] ──► [ Localhost Proxy ] ──► [ Application Service ]
                              │
                      • TLS 1.3 Termination
                      • HSTS, CSP, X-Frame-Options
                      • Leaky-Bucket Rate Limiting
                      • Client Request Buffering
```

- **TLS Offloading:** Nginx terminates TLS 1.3 handshakes using hardware-accelerated elliptic-curve cryptography, freeing application worker threads from CPU-intensive cryptographic processing.
- **Header Hardening:** Injects mandatory HTTP security headers (`Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Content-Security-Policy`) to neutralize Cross-Site Scripting (XSS), Clickjacking, and MIME-sniffing exploits.
- **Abuse Prevention & Rate Limiting:** Enforces memory-backed leaky-bucket rate limiting zones:
  - *Authentication Endpoints:* Capped at 5 requests/minute per IP to prevent credential-stuffing and brute-force attacks.
  - *General API Routes:* Capped at 30 requests/second per IP to prevent denial-of-service scraping.
- **Slowloris & Buffer Shielding:** Nginx buffers slow, streaming client uploads completely in memory/disk before passing the request to the backend service, preventing slow-client connection exhaustion.

---

## 5. Hardware Sizing & Resource Allocation Model

Because compute, memory, and storage buses are shared between the application service and the database engine, resources are statically partitioned to prevent starvation:

![Hardware Sizing & Resource Allocation](assets/03-hardware-and-resource-allocation.jpg)

### Resource Partitioning Breakdown

1. **CPU Core Pinning (16 Physical Cores):**
   - **Cores 0–3 (4 Cores):** Dedicated to the Linux Kernel, network interface interrupts, system daemons, and Nginx ingress processing.
   - **Cores 4–9 (6 Cores):** Dedicated to the Application Service runtime, handling business logic, worker threads, and garbage collection.
   - **Cores 10–15 (6 Cores):** Dedicated to the Database Engine, running query planner workers, background write threads, and transaction commit flushing.

2. **Memory Allocation (64 GB ECC RAM Budget):**
   - **32 GB (50%):** Dedicated to Database Shared Buffers, index caching, and query execution working memory.
   - **16 GB (25%):** Allocated to Application Service heap space, strictly capped by systemd cgroups (`MemoryMax=16G`).
   - **10 GB (16%):** Allocated to Linux Virtual Filesystem (VFS) page cache to accelerate repeated file and database reads.
   - **6 GB (9%):** Reserved for OS kernel overhead, telemetry daemons, and security tools.

3. **Storage Subsystem:**
   - Four enterprise NVMe SSDs arranged in RAID 10 (Striped Mirrors), delivering high random-write IOPS and monitored continuously via SMART health telemetry.

4. **Physical Chassis & Thermal Cooling:**
   - Rack-mounted 1U/2U server chassis with front-to-back airflow, redundant cooling fans, and IPMI hardware monitoring.

5. **Power Resilience & Dual-Rail UPS:**
   - Dual hot-swap power supply units (PSU 1 & PSU 2) connected to independent power circuits (Feed A and Feed B).
   - Backed by an intelligent on-line Uninterruptible Power Supply (UPS) providing 5+ minutes of battery runtime.
   - An integrated daemon (`nut` / `apcupsd`) initiates an automated flush of in-memory database buffers and a safe system shutdown if utility power remains unrecovered.

---

## 6. Storage Architecture & Partition Isolation

Persistence relies on four enterprise NVMe SSDs configured in **RAID 10**, combining striping throughput with mirror redundancy:

![Storage Architecture RAID 10 & Partitioning](assets/04-storage-raid10-partitioning.jpg)

### Storage Assembly & Partitioning Workflow

1. **Physical NVMe Drives:** 4 identical enterprise PCIe NVMe SSDs (e.g., 1 TB each).
2. **Mirrored Pairs (RAID 1):** Drive 1 and Drive 2 form Mirror Pair 1; Drive 3 and Drive 4 form Mirror Pair 2. Each pair provides complete data mirroring.
3. **Striping (RAID 0 $ightarrow$ RAID 10):** The two mirrored pairs are striped together, creating a unified high-speed 2 TB usable storage volume with zero parity write penalties and multi-drive failure tolerance.
4. **Partition Allocation:**
   - `/data/db` (~1 TB / 50%): Dedicated to database tables, indexes, and write-ahead transaction journals (WAL). Mounted with `noatime` and restricted to `chmod 0700`.
   - `/data/app` (~600 GB / 30%): Dedicated to application releases, user-uploaded assets, and runtime logs. Storage quotas prevent application logs from consuming database disk space.
   - `/data/backups` (~400 GB / 20%): Local staging area for daily database snapshot creation and encryption prior to off-site cloud synchronization.

5. **Performance Mount Options:**
   - All data filesystems are mounted with `noatime`, disabling write overhead for file access timestamp updates on read operations and boosting database I/O performance.

---

## 7. Operating System Hardening & Process Sandboxing

The OS baseline is minimized: compilers (`gcc`, `make`), package build tools, and unnecessary network daemons are barred from the production server.

```
┌──────────────────────────────────────────────────────────────┐
│                    SECURITY SANDBOX                          │
│                                                              │
│  [ Application Sandbox ]                                     │
│  • Runs as unprivileged 'appuser' (shell: /sbin/nologin)     │
│  • MemoryMax = 16GB (Hard cgroup ceiling via systemd)        │
│  • ProtectSystem = strict (OS filesystems mounted read-only) │
│  • PrivateTmp = true (Isolated temporary filesystem)         │
│  • Permissions: Zero read access to database data directory  │
│                                                              │
│  [ Database Sandbox ]                                        │
│  • Runs as dedicated database system user                    │
│  • Listens strictly on 127.0.0.1                             │
│  • Data directory permissions restricted to mode 0700        │
└──────────────────────────────────────────────────────────────┘
```

- **Process Isolation via Systemd:** The application service executes under an unprivileged `appuser`. Systemd enforces `MemoryMax=16G` (preventing memory leaks from causing host OOM crashes), `ProtectSystem=strict` (mounting system binaries read-only), and `PrivateTmp=true` (isolating `/tmp`).
- **Kernel Parameter Hardening (Sysctl):** Enforces strict reverse path filtering (`rp_filter = 1`) to defeat IP spoofing, activates TCP SYN cookies (`tcp_syncookies = 1`) to neutralize SYN floods, and disables ICMP redirects.
- **Audit Subsystem (`auditd`):** Hooks into kernel system calls to maintain an immutable audit trail of modifications to sensitive files (`/etc/shadow`, `/etc/ssh/sshd_config`, and application secrets).

---

## 8. Application Lifecycle & Zero-Downtime Deployment

Software deployments use an **Atomic Symlink Strategy** with pre-compiled artifacts built off-host in Continuous Integration:

![Application Lifecycle & Zero-Downtime Deployment](assets/05-application-deployment-lifecycle.jpg)

### Five-Stage Deployment Workflow

1. **Stage 1 (Build Artifact in CI):** Application code is built, tested, and packaged into a production artifact (e.g., zip/tarball) on an external CI runner. No compilers or package managers run on the production server.
2. **Stage 2 (Copy & Extract):** The pre-compiled artifact is copied to the server and extracted into a new timestamped directory under `/data/app/releases/<timestamp>/`.
3. **Stage 3 (Link Shared Resources):** Shared configuration secrets (`.env`, stored in `/data/app/shared/` with mode `640`) and persistent customer uploads are symlinked into the new release directory.
4. **Stage 4 (Validate Release):** The application is health-checked on an ephemeral local port (e.g., 3001) to verify database connectivity and dependency health prior to routing live traffic.
5. **Stage 5 (Atomic Switch):** If validation passes, the `current` symbolic link is atomically repointed to the new release directory using `ln -sfn`. Nginx continues serving incoming requests with zero downtime.

### Instant Rollback Capability
If a critical error is detected post-deployment, the `current` symlink is repointed back to the previous stable release directory in a single command. The application reverts to the known-good version in seconds without re-downloading or re-building code.

---

## 9. Inter-Tier Communication & Data Isolation

The application service and database engine communicate strictly over internal system boundaries:

```
[ Nginx Reverse Proxy ]
          │
          ▼ (Unix Domain Socket / Localhost HTTP - Sub-10 microsecond latency)
[ Application Service ]
          │
          ▼ (Loopback TCP: 127.0.0.1 - Sub-millisecond query execution)
[ Database Engine ]
```

- **Loopback Binding:** The database engine binds strictly to `127.0.0.1`. Remote TCP connections are rejected at the socket layer.
- **Privilege Scoping:** The application connects using a non-administrative database user with credentials stored in an out-of-tree `.env` file (mode 640). The application role holds standard Data Manipulation Language (DML) rights but cannot execute destructive schema drops (`DROP TABLE`, `ALTER SYSTEM`).

---

## 10. Observability & Centralized Telemetry Pipeline

Telemetry collection runs out-of-band: local lightweight collectors gather metrics and stream logs to a dedicated external observability cluster:

```
[ Production Server ]
  • Node Exporter   ──(Metrics scrape over VPN)──►  [ Central Prometheus & Grafana ]
  • Promtail        ──(TLS log streaming)────────►  [ Central Grafana Loki ]
```

- **Proactive Alerting:** Out-of-band alerts notify on-call engineers via Slack or PagerDuty if disk utilization exceeds 80%, RAM exceeds 85%, or HTTP 5xx error rates exceed 1%.
- **Immutable Log Storage:** Promtail streams logs in real time to external Write-Once-Read-Many (WORM) storage, ensuring audit records remain preserved even if the host is compromised.

---

## 11. Backup Strategy & Disaster Recovery Architecture

Data durability follows the **3-2-1 Backup Strategy**:

```
[ 1. Live Data ]             Database files on NVMe RAID 10
      │
      ▼ (Continuous Write Log Streaming)
[ 2. Local Backup ]          Encrypted staging in /data/backups (15-min recovery target)
      │
      ▼ (Encrypted TLS Push)
[ 3. Off-Site Storage ]      AWS S3 / Wasabi with Object Lock (WORM Immutability)
```

- **RPO (Recovery Point Objective):** `< 5 Minutes` via continuous database transaction write-log streaming.
- **RTO (Recovery Time Objective):** `< 4 Hours` for complete bare-metal or virtual standby provisioning, snapshot rehydration, and DNS record swing.
- **Ransomware Defense:** Off-site storage enforces Object Lock in Compliance Mode. Backup archives cannot be deleted or overwritten by host administrative credentials.

---

## 12. Architectural Limitations & Strategic Solutions

| Architectural Limitation | Technical Impact | Architectural Solution |
|---|---|---|
| **Single Point of Failure (SPOF)** | Motherboard or CPU failure halts execution until hardware is repaired. | Maintain a standby host receiving continuous database transaction log replication. Repoint DNS (TTL 300s) to the standby within 15–30 minutes during a major hardware failure. |
| **Hardware Scaling Ceiling** | Single-node vertical scaling is bounded by motherboard socket and RAM capacity. | Decouple read traffic via a secondary read replica, or migrate the database to its own dedicated bare-metal server while maintaining this host for the application service. |
| **Shared Resource Competition** | Heavy database batch jobs compete with the application runtime for CPU and I/O. | Enforce systemd cgroup bounds (`CPUQuota`, `MemoryMax`) on the app, configure database query timeouts (`statement_timeout = 30s`), and route analytics off-host. |
| **Single Machine Blast Radius** | Root compromise exposes both application files and local database storage. | Enforce unprivileged user execution, lock file permissions (`0700` for DB, `640` for secrets), encrypt sensitive database fields in the application, and isolate backups off-site. |
| **Reboot Downtime for Kernel Patches** | Kernel security updates require a system reboot. | Deploy Linux live-patching (`kpatch` / Canonical Livepatch) for in-memory updates. Schedule mandatory physical maintenance during designated low-traffic hours. |
| **DDoS Vulnerability** | Direct volumetric packet floods can saturate the physical edge link. | Place an upstream Anycast CDN / reverse proxy (such as Cloudflare) in front of the domain to absorb volumetric DDoS attacks before they reach the edge router. |

---

## 13. Operational Guarantees

- **Sub-Millisecond Inter-Tier Performance:** Direct in-memory loopback communication between application and persistence tiers.
- **Language & Database Independence:** Identical architectural guarantees across any backend runtime or database technology.
- **Zero-Trust Access Control:** Total isolation between Public Internet, Corporate Office LAN, and Administrative WireGuard VPN.
- **Hardware Fault Tolerance:** RAID 10 storage redundancy and dual-rail UPS battery protection.
- **Deterministic Business Continuity:** Continuous transaction log archiving with `< 5 min` RPO and `< 4 hr` RTO.
