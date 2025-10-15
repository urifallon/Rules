# TÀI LIỆU QUY HOẠCH MẠNG VMWARE vSPHERE
## PRODUCTION NETWORK DESIGN & IP ALLOCATION STANDARD

**Phiên bản:** 1.0  
**Ngày cập nhật:** 15/10/2025  
**Người soạn:** IT Infrastructure Team  
**Phạm vi áp dụng:** Toàn bộ hệ thống VMware vSphere Production

---

## MỤC LỤC

1. [Nguyên tắc Quy hoạch IP](#nguyen-tac)
2. [Cấu trúc Naming Convention](#naming-convention)
3. [Phân bổ IP theo Site](#ip-allocation)
4. [Chi tiết VLAN](#vlan-details)
5. [Cấu hình ESXi Networking](#esxi-networking)
6. [Bảng phân bổ IP đầy đủ](#ip-table)
7. [Firewall Rules](#firewall-rules)
8. [DNS Records](#dns-records)
9. [Checklist triển khai](#checklist)

---

<a name="nguyen-tac"></a>
## 1. NGUYÊN TẮC QUY HOẠCH IP

### 1.1. Dải IP Private sử dụng

```
RFC 1918 Private Address Space:
├── 10.0.0.0/8        → Sử dụng cho toàn bộ infrastructure
├── 172.16.0.0/12     → Dự phòng cho expansion/DR site
└── 192.168.0.0/16    → Không sử dụng (tránh conflict với home network)

Dải loại trừ:
├── 127.0.0.0/8       → Loopback (KHÔNG dùng cho VM)
├── 169.254.0.0/16    → Link-local (KHÔNG dùng)
└── 224.0.0.0/4       → Multicast (KHÔNG dùng cho host)
```

### 1.2. Cấu trúc IP Addressing (10.x.y.z)

```
10.SITE.VLAN.HOST
   │    │    │
   │    │    └── Host ID (1-254)
   │    └────── VLAN ID (10-255)
   └─────────── Site ID (10-99)

Ví dụ:
10.10.100.51
│  │  │   │
│  │  │   └── Web Server 01
│  │  └────── VLAN 100 (Web Tier)
│  └───────── VLAN ID = 100
└──────────── Site HCM (10)
```

### 1.3. Phân bổ Site ID

```
10.10.0.0/16  → HCM Site (Hồ Chí Minh)
10.20.0.0/16  → HN Site (Hà Nội)
10.30.0.0/16  → DN Site (Đà Nẵng)
10.40.0.0/16  → CT Site (Cần Thơ)
10.50.0.0/16  → HP Site (Hải Phòng)

10.100.0.0/16 → AWS Cloud
10.200.0.0/16 → Azure Cloud
10.250.0.0/16 → DR Site (Disaster Recovery)
```

### 1.4. Phân bổ VLAN ID chuẩn

```
VLAN 1        → Native VLAN (KHÔNG dùng)
VLAN 10       → Management Network
VLAN 20       → vMotion Network
VLAN 30       → iSCSI/Storage Network
VLAN 40       → vSAN Network (nếu có)
VLAN 50       → Backup Network
VLAN 60       → Replication Network

VLAN 90-99    → DMZ / Public-facing
VLAN 100-109  → Web Tier
VLAN 110-119  → App Tier
VLAN 120-129  → Database Tier
VLAN 130-139  → Management VMs (Ansible, Monitoring, etc)
VLAN 140-149  → Development Environment
VLAN 150-159  → UAT Environment
VLAN 160-169  → Integration/Staging
VLAN 200-209  → User Workstations
VLAN 210-219  → Guest WiFi / Isolated
```

### 1.5. Quy tắc phân bổ Host ID (.z)

```
.1 - .9       → Network devices (Gateway, Router, Firewall, Core Switch)
  .1          → Default Gateway
  .2          → Secondary Gateway (HSRP/VRRP)
  .3          → Firewall Primary
  .4          → Firewall Secondary
  .5-9        → Core Switches, Load Balancers

.10 - .49     → Infrastructure Services
  .10         → vCenter Server
  .11         → vCenter DB (external)
  .12         → NSX Manager
  .13-15      → NSX Controllers
  .16         → vRealize Automation
  .17         → vRealize Operations
  .18         → vRealize Log Insight
  .19         → Site Recovery Manager (SRM)
  .20         → Backup Server (Veeam/Commvault)
  .21-39      → ESXi Hosts (vmk0 Management)
  .40-49      → Storage Arrays, NAS, SAN

.50 - .99     → Production Servers (VMs)
  .50-69      → Tier 1 Applications
  .70-89      → Tier 2 Applications
  .90-99      → Tier 3 Applications

.100 - .199   → Workstations & End-user devices
  .100-149    → Desktop VDI
  .150-199    → Laptop / Mobile

.200 - .254   → DHCP Pool
  .200-254    → Dynamic IP assignment
```

---

<a name="naming-convention"></a>
## 2. CẤU TRÚC NAMING CONVENTION

### 2.1. Format chuẩn

```
[LOCATION]-[ENV]-[TYPE]-[FUNCTION]-[NUMBER]
    │        │      │       │         │
    │        │      │       │         └── Số thứ tự (01, 02, 03...)
    │        │      │       └─────────── Chức năng cụ thể
    │        │      └─────────────────── Loại thiết bị/VM
    │        └────────────────────────── Môi trường
    └─────────────────────────────────── Địa điểm

Ví dụ: hcm-prod-esxi-mgmt-01
```

### 2.2. Mã Location (LOCATION)

```
hcm  → Hồ Chí Minh
hn   → Hà Nội
dn   → Đà Nẵng
ct   → Cần Thơ
hp   → Hải Phòng
aws  → AWS Cloud
az   → Azure Cloud
dr   → DR Site
```

### 2.3. Mã Environment (ENV)

```
prod  → Production
dev   → Development
uat   → User Acceptance Testing
stg   → Staging
int   → Integration
lab   → Lab/Test
dr    → Disaster Recovery
```

### 2.4. Mã Type (TYPE)

```
Infrastructure:
├── vc     → vCenter Server
├── esxi   → ESXi Host
├── nsx    → NSX Manager/Controller
├── vro    → vRealize Operations
├── vra    → vRealize Automation
├── vrli   → vRealize Log Insight
├── srm    → Site Recovery Manager
├── bkp    → Backup Server
└── stor   → Storage Array

Virtual Machines:
├── web    → Web Server
├── app    → Application Server
├── db     → Database Server
├── cache  → Cache Server (Redis, Memcached)
├── mq     → Message Queue (RabbitMQ, Kafka)
├── lb     → Load Balancer
├── dns    → DNS Server
├── dhcp   → DHCP Server
├── ad     → Active Directory
├── file   → File Server
├── mail   → Mail Server
├── proxy  → Proxy Server
├── vpn    → VPN Server
├── fw     → Firewall VM
├── ids    → IDS/IPS
├── jump   → Jump/Bastion Host
├── mon    → Monitoring Server
├── log    → Log Server
└── repo   → Repository Server
```

### 2.5. Ví dụ đầy đủ

```
Infrastructure:
hcm-prod-vc-01              → vCenter HCM Production
hcm-prod-esxi-mgmt-01       → ESXi Host 01 HCM
hn-prod-nsx-mgr-01          → NSX Manager Hà Nội
hcm-prod-stor-iscsi-01      → iSCSI Storage HCM

Virtual Machines:
hcm-prod-web-frontend-01    → Web Frontend Server 01
hcm-prod-app-api-01         → API Application Server 01
hcm-prod-db-postgres-01     → PostgreSQL Database 01
hcm-prod-cache-redis-01     → Redis Cache Server 01
hn-dev-web-test-01          → Web Test Server HN Dev

Management:
hcm-prod-mon-zabbix-01      → Zabbix Monitoring Server
hcm-prod-log-elk-01         → ELK Log Server
hcm-prod-jump-linux-01      → Linux Jump Host
```

---

<a name="ip-allocation"></a>
## 3. PHÂN BỔ IP THEO SITE

### 3.1. HCM SITE (10.10.0.0/16)

#### VLAN 10 - Management Network (10.10.10.0/24)

```
10.10.10.1      → Gateway (Physical Router/Firewall)
10.10.10.2      → Gateway Secondary (HSRP/VRRP)
10.10.10.3      → Firewall Primary
10.10.10.4      → Firewall Secondary

10.10.10.10     → hcm-prod-vc-01 (vCenter Server)
10.10.10.11     → hcm-prod-vc-db-01 (vCenter External DB)
10.10.10.12     → hcm-prod-nsx-mgr-01
10.10.10.13     → hcm-prod-nsx-ctrl-01
10.10.10.14     → hcm-prod-nsx-ctrl-02
10.10.10.15     → hcm-prod-nsx-ctrl-03

10.10.10.21     → hcm-prod-esxi-01 (vmk0)
10.10.10.22     → hcm-prod-esxi-02 (vmk0)
10.10.10.23     → hcm-prod-esxi-03 (vmk0)
10.10.10.24     → hcm-prod-esxi-04 (vmk0)

10.10.10.50     → hcm-prod-ad-dc-01
10.10.10.51     → hcm-prod-dns-01
10.10.10.52     → hcm-prod-dhcp-01

10.10.10.200-254 → DHCP Pool for Management Access
```

#### VLAN 20 - vMotion Network (10.10.20.0/24)

```
10.10.20.1      → Gateway (không cần gateway thực tế, để dự phòng)

10.10.20.21     → hcm-prod-esxi-01 (vmk1 - vMotion)
10.10.20.22     → hcm-prod-esxi-02 (vmk1 - vMotion)
10.10.20.23     → hcm-prod-esxi-03 (vmk1 - vMotion)
10.10.20.24     → hcm-prod-esxi-04 (vmk1 - vMotion)

Note: Không cần DHCP, MTU = 9000 (Jumbo Frame)
```

#### VLAN 30 - iSCSI Storage Network (10.10.30.0/24)

```
10.10.30.1      → Gateway (không cần)

10.10.30.21     → hcm-prod-esxi-01 (vmk2 - iSCSI-A)
10.10.30.22     → hcm-prod-esxi-02 (vmk2 - iSCSI-A)
10.10.30.23     → hcm-prod-esxi-03 (vmk2 - iSCSI-A)
10.10.30.24     → hcm-prod-esxi-04 (vmk2 - iSCSI-A)

10.10.30.31     → hcm-prod-esxi-01 (vmk3 - iSCSI-B)
10.10.30.32     → hcm-prod-esxi-02 (vmk3 - iSCSI-B)
10.10.30.33     → hcm-prod-esxi-03 (vmk3 - iSCSI-B)
10.10.30.34     → hcm-prod-esxi-04 (vmk3 - iSCSI-B)

10.10.30.100    → hcm-prod-stor-iscsi-01 (Port A)
10.10.30.101    → hcm-prod-stor-iscsi-01 (Port B)

Note: Không cần gateway, MTU = 9000, Multipathing enabled
```

#### VLAN 50 - Backup Network (10.10.50.0/24)

```
10.10.50.1      → Gateway

10.10.50.10     → hcm-prod-bkp-veeam-01
10.10.50.11     → hcm-prod-bkp-proxy-01
10.10.50.12     → hcm-prod-bkp-repo-01

10.10.50.21     → hcm-prod-esxi-01 (vmk4 - Backup)
10.10.50.22     → hcm-prod-esxi-02 (vmk4 - Backup)
10.10.50.23     → hcm-prod-esxi-03 (vmk4 - Backup)
10.10.50.24     → hcm-prod-esxi-04 (vmk4 - Backup)

10.10.50.100    → hcm-prod-stor-backup-01 (NAS Backup)
```

#### VLAN 90 - DMZ (10.10.90.0/24)

```
10.10.90.1      → Gateway (External Firewall)
10.10.90.2      → Gateway Secondary

10.10.90.10     → hcm-prod-lb-external-01 (HAProxy/F5)
10.10.90.11     → hcm-prod-lb-external-02

10.10.90.50     → hcm-prod-web-public-01
10.10.90.51     → hcm-prod-web-public-02
10.10.90.52     → hcm-prod-api-public-01

10.10.90.60     → hcm-prod-vpn-01
10.10.90.61     → hcm-prod-jump-dmz-01
```

#### VLAN 100 - Web Tier (10.10.100.0/24)

```
10.10.100.1     → Gateway
10.10.100.10    → hcm-prod-lb-web-01 (Internal LB)

10.10.100.51    → hcm-prod-web-frontend-01
10.10.100.52    → hcm-prod-web-frontend-02
10.10.100.53    → hcm-prod-web-frontend-03
10.10.100.54    → hcm-prod-web-admin-01
10.10.100.55    → hcm-prod-web-cdn-01
```

#### VLAN 110 - App Tier (10.10.110.0/24)

```
10.10.110.1     → Gateway

10.10.110.51    → hcm-prod-app-api-01
10.10.110.52    → hcm-prod-app-api-02
10.10.110.53    → hcm-prod-app-worker-01
10.10.110.54    → hcm-prod-app-worker-02
10.10.110.55    → hcm-prod-app-scheduler-01

10.10.110.60    → hcm-prod-cache-redis-01
10.10.110.61    → hcm-prod-cache-redis-02
10.10.110.62    → hcm-prod-mq-rabbitmq-01
```

#### VLAN 120 - Database Tier (10.10.120.0/24)

```
10.10.120.1     → Gateway

10.10.120.51    → hcm-prod-db-postgres-01
10.10.120.52    → hcm-prod-db-postgres-02 (Replica)
10.10.120.53    → hcm-prod-db-mysql-01
10.10.120.54    → hcm-prod-db-mysql-02 (Replica)
10.10.120.55    → hcm-prod-db-mongo-01
10.10.120.56    → hcm-prod-db-mongo-02
10.10.120.57    → hcm-prod-db-mongo-03

Note: Isolated VLAN, no internet access, strict firewall rules
```

#### VLAN 130 - Management VMs (10.10.130.0/24)

```
10.10.130.1     → Gateway

10.10.130.51    → hcm-prod-mon-zabbix-01
10.10.130.52    → hcm-prod-mon-grafana-01
10.10.130.53    → hcm-prod-mon-prometheus-01
10.10.130.54    → hcm-prod-log-elk-01
10.10.130.55    → hcm-prod-log-graylog-01

10.10.130.60    → hcm-prod-ansible-01
10.10.130.61    → hcm-prod-gitlab-01
10.10.130.62    → hcm-prod-jenkins-01
10.10.130.63    → hcm-prod-harbor-01 (Container Registry)

10.10.130.70    → hcm-prod-jump-linux-01
10.10.130.71    → hcm-prod-jump-windows-01
```

---

### 3.2. HN SITE (10.20.0.0/16)

```
Cấu trúc tương tự HCM, chỉ thay đổi octet thứ 2:

10.20.10.0/24   → Management
10.20.20.0/24   → vMotion
10.20.30.0/24   → iSCSI
10.20.50.0/24   → Backup
10.20.90.0/24   → DMZ
10.20.100.0/24  → Web Tier
10.20.110.0/24  → App Tier
10.20.120.0/24  → Database Tier
10.20.130.0/24  → Management VMs

Ví dụ:
10.20.10.10     → hn-prod-vc-01
10.20.10.21     → hn-prod-esxi-01
10.20.100.51    → hn-prod-web-01
```

---

<a name="vlan-details"></a>
## 4. CHI TIẾT VLAN CONFIGURATION

### 4.1. VLAN Properties Table

| VLAN ID | Name | Subnet | Gateway | MTU | DHCP | Security |
|---------|------|--------|---------|-----|------|----------|
| 10 | Management | 10.x.10.0/24 | .1 | 1500 | .200-254 | High |
| 20 | vMotion | 10.x.20.0/24 | None | 9000 | No | Medium |
| 30 | iSCSI | 10.x.30.0/24 | None | 9000 | No | High |
| 40 | vSAN | 10.x.40.0/24 | None | 9000 | No | High |
| 50 | Backup | 10.x.50.0/24 | .1 | 9000 | No | Medium |
| 60 | Replication | 10.x.60.0/24 | None | 9000 | No | Medium |
| 90 | DMZ | 10.x.90.0/24 | .1 | 1500 | No | Critical |
| 100 | Web Tier | 10.x.100.0/24 | .1 | 1500 | No | Medium |
| 110 | App Tier | 10.x.110.0/24 | .1 | 1500 | No | High |
| 120 | DB Tier | 10.x.120.0/24 | .1 | 1500 | No | Critical |
| 130 | Mgmt VMs | 10.x.130.0/24 | .1 | 1500 | No | Medium |
| 140 | Dev | 10.x.140.0/24 | .1 | 1500 | .200-254 | Low |
| 200 | Workstations | 10.x.200.0/24 | .1 | 1500 | .100-254 | Low |

### 4.2. VLAN Security Levels

#### Critical Security (DB Tier, DMZ)
```
- No internet access (outbound only through proxy)
- Micro-segmentation enabled
- IDS/IPS monitoring
- Encrypted traffic only
- Strict firewall rules (whitelist only)
- Regular security audits
```

#### High Security (Management, iSCSI, App Tier)
```
- Limited internet access
- Jump host required for access
- MFA enabled
- Firewall rules (specific allow)
- Monitoring enabled
```

#### Medium Security (Web Tier, Backup, vMotion)
```
- Controlled internet access
- Firewall rules (general allow with deny list)
- Basic monitoring
```

#### Low Security (Dev, Workstations)
```
- Full internet access
- Basic firewall rules
- Best-effort monitoring
```

---

<a name="esxi-networking"></a>
## 5. CẤU HÌNH ESXI NETWORKING

### 5.1. ESXi Host Network Configuration Template

```
ESXi Host: hcm-prod-esxi-01 (10.10.10.21)

Physical NICs:
├── vmnic0  → 10 Gbps (Management + vMotion)
├── vmnic1  → 10 Gbps (Management + vMotion) - Redundant
├── vmnic2  → 10 Gbps (iSCSI-A)
├── vmnic3  → 10 Gbps (iSCSI-B)
├── vmnic4  → 10 Gbps (VM Network)
└── vmnic5  → 10 Gbps (VM Network) - Redundant

Virtual Switches:
├── vSwitch0 (Management + vMotion)
│   ├── vmnic0, vmnic1 (Active-Active)
│   ├── Management Network (VLAN 10)
│   │   └── vmk0: 10.10.10.21/24, GW: 10.10.10.1
│   └── vMotion Network (VLAN 20)
│       └── vmk1: 10.10.20.21/24, No GW, MTU 9000
│
├── vSwitch1 (iSCSI)
│   ├── vmnic2 → iSCSI-A
│   │   └── vmk2: 10.10.30.21/24, No GW, MTU 9000
│   └── vmnic3 → iSCSI-B
│       └── vmk3: 10.10.30.31/24, No GW, MTU 9000
│
└── vSwitch2 (VM Networks) hoặc VDS (Distributed Switch)
    ├── vmnic4, vmnic5 (Active-Active)
    ├── VLAN 90 (DMZ)
    ├── VLAN 100 (Web Tier)
    ├── VLAN 110 (App Tier)
    ├── VLAN 120 (DB Tier)
    └── VLAN 130 (Mgmt VMs)
```

### 5.2. VMkernel Adapter Configuration

#### vmk0 - Management

```yaml
Interface: vmk0
IP Address: 10.10.10.21
Subnet Mask: 255.255.255.0
Default Gateway: 10.10.10.1
MTU: 1500
VLAN ID: 10
Services Enabled:
  - Management Traffic
  - Provisioning (if needed)
TCP/IP Stack: Default
```

#### vmk1 - vMotion

```yaml
Interface: vmk1
IP Address: 10.10.20.21
Subnet Mask: 255.255.255.0
Default Gateway: None
MTU: 9000
VLAN ID: 20
Services Enabled:
  - vMotion
TCP/IP Stack: vMotion
```

#### vmk2 - iSCSI-A

```yaml
Interface: vmk2
IP Address: 10.10.30.21
Subnet Mask: 255.255.255.0
Default Gateway: None
MTU: 9000
VLAN ID: 30
Port Binding: vmnic2 only
Services Enabled:
  - None (iSCSI binding)
iSCSI Port Binding: Yes
```

#### vmk3 - iSCSI-B

```yaml
Interface: vmk3
IP Address: 10.10.30.31
Subnet Mask: 255.255.255.0
Default Gateway: None
MTU: 9000
VLAN ID: 30
Port Binding: vmnic3 only
Services Enabled:
  - None (iSCSI binding)
iSCSI Port Binding: Yes
```

### 5.3. vSphere Distributed Switch (VDS) Configuration

```
VDS Name: hcm-prod-vds-01
Version: 7.0.0 or later
MTU: 9000
Uplinks: 2 per host (vmnic4, vmnic5)

Port Groups:
├── VLAN-090-DMZ
│   ├── VLAN ID: 90
│   ├── Teaming: Route based on originating virtual port
│   └── Security: Promiscuous mode = Reject
│
├── VLAN-100-Web
│   ├── VLAN ID: 100
│   ├── Teaming: Route based on physical NIC load
│   └── Security: Promiscuous mode = Reject
│
├── VLAN-110-App
│   ├── VLAN ID: 110
│   ├── Teaming: Route based on physical NIC load
│   └── Security: Promiscuous mode = Reject
│
├── VLAN-120-Database
│   ├── VLAN ID: 120
│   ├── Teaming: Route based on physical NIC load
│   ├── Security: Promiscuous mode = Reject
│   └── Traffic Shaping: Enabled (if needed)
│
└── VLAN-130-MgmtVMs
    ├── VLAN ID: 130
    ├── Teaming: Route based on originating virtual port
    └── Security: Promiscuous mode = Reject
```

### 5.4. ESXi Firewall Rules

```bash
# Allow vCenter access
esxcli network firewall ruleset set --ruleset-id=vSphereClient --enabled=true

# Allow SSH (disable sau khi config xong)
esxcli network firewall ruleset set --ruleset-id=sshServer --enabled=true

# Allow NTP
esxcli network firewall ruleset set --ruleset-id=ntpClient --enabled=true

# Allow vMotion
esxcli network firewall ruleset set --ruleset-id=vMotion --enabled=true

# Custom rule cho iSCSI
esxcli network firewall ruleset set --ruleset-id=softwareISCSI --enabled=true
```

---

<a name="ip-table"></a>
## 6. BẢNG PHÂN BỔ IP ĐẦY ĐỦ

### 6.1. HCM Site - Infrastructure

| IP Address | Hostname | Type | VLAN | Function | OS | Note |
|------------|----------|------|------|----------|----|----- |
| 10.10.10.1 | hcm-gw-01 | Gateway | 10 | Primary Gateway | - | Physical |
| 10.10.10.2 | hcm-gw-02 | Gateway | 10 | Secondary GW (HSRP) | - | Physical |
| 10.10.10.10 | hcm-prod-vc-01 | vCenter | 10 | vCenter Server | VCSA 8.0 | 14GB RAM |
| 10.10.10.11 | hcm-prod-vc-db-01 | Database | 10 | vCenter Postgres | RHEL 8 | External DB |
| 10.10.10.12 | hcm-prod-nsx-mgr-01 | NSX | 10 | NSX Manager | NSX-T | - |
| 10.10.10.21 | hcm-prod-esxi-01 | ESXi | 10 | ESXi Host 01 | ESXi 8.0 | 256GB RAM |
| 10.10.10.22 | hcm-prod-esxi-02 | ESXi | 10 | ESXi Host 02 | ESXi 8.0 | 256GB RAM |
| 10.10.10.23 | hcm-prod-esxi-03 | ESXi | 10 | ESXi Host 03 | ESXi 8.0 | 256GB RAM |
| 10.10.10.24 | hcm-prod-esxi-04 | ESXi | 10 | ESXi Host 04 | ESXi 8.0 | 256GB RAM |
| 10.10.10.50 | hcm-prod-ad-dc-01 | VM | 10 | Active Directory | Win 2022 | - |
| 10.10.10.51 | hcm-prod-dns-01 | VM | 10 | DNS Server | Ubuntu 22.04 | Bind9 |
| 10.10.10.52 | hcm-prod-dhcp-01 | VM | 10 | DHCP Server | Ubuntu 22.04 | ISC DHCP |
| 10.10.20.21 | hcm-prod-esxi-01 | ESXi vmk1 | 20 | vMotion 01 | ESXi 8.0 | MTU 9000 |
| 10.10.20.22 | hcm-prod-esxi-02 | ESXi vmk1 | 20 | vMotion 02 | ESXi 8.0 | MTU 9000 |
| 10.10.20.23 | hcm-prod-esxi-03 | ESXi vmk1 | 20 | vMotion 03 | ESXi 8.0 | MTU 9000 |
| 10.10.20.24 | hcm-prod-esxi-04 | ESXi vmk1 | 20 | vMotion 04 | ESXi 8.0 | MTU 9000 |
| 10.10.30.21 | hcm-prod-esxi-01 | ESXi vmk2 | 30 | iSCSI-A 01 | ESXi 8.0 | MTU 9000 |
| 10.10.30.22 | hcm-prod-esxi-02 | ESXi vmk2 | 30 | iSCSI-A 02 | ESXi 8.0 | MTU 9000 |
| 10.10.30.31 | hcm-prod-esxi-01 | ESXi vmk3 | 30 | iSCSI-B 01 | ESXi 8.0 | MTU 9000 |
| 10.10.30.32 | hcm-prod-esxi-02 | ESXi vmk3 | 30 | iSCSI-B 02 | ESXi 8.0 | MTU 9000 |
| 10.10.30.100 | hcm-prod-stor-01 | Storage | 30 | iSCSI Target A | - | SAN Array |
| 10.10.30.101 | hcm-prod-stor-01 | Storage | 30 | iSCSI Target B | - | SAN Array |

### 6.2. HCM Site - Production VMs

| IP Address | Hostname | Type | VLAN | Function | OS | vCPU/RAM |
|------------|----------|------|------|----------|----|----- |
| 10.10.90.10 | hcm-prod-lb-ext-01 | VM | 90 | External LB | Ubuntu 22.04 | 4C/8GB |
| 10.10.90.50 | hcm-prod-web-public-01 | VM | 90 | Public Web | Ubuntu 22.04 | 4C/16GB |
| 10.10.90.51 | hcm-prod-web-public-02 | VM | 90 | Public Web | Ubuntu 22.04 | 4C/16GB |
| 10.10.90.60 | hcm-prod-vpn-01 | VM | 90 | VPN Gateway | pfSense | 2C/4GB |
| 10.10.100.10 | hcm-prod-lb-web-01 | VM | 100 | Web LB | HAProxy | 2C/4GB |
| 10.10.100.51 | hcm-prod-web-fe-01 | VM | 100 | Frontend Web | Ubuntu 22.04 | 4C/16GB |
| 10.10.100.52 | hcm-prod-web-fe-02 | VM | 100 | Frontend Web | Ubuntu 22.04 | 4C/16GB |
| 10.10.100.53 | hcm-prod-web-fe-03 | VM | 100 | Frontend Web | Ubuntu 22.04 | 4C/16GB |
| 10.10.110.51 | hcm-prod-app-api-01 | VM | 110 | API Server | Ubuntu 22.04 | 8C/32GB |
| 10.10.110.52 | hcm-prod-app-api-02 | VM | 110 | API Server | Ubuntu 22.04 | 8C/32GB |
| 10.10.110.53 | hcm-prod-app-worker-01 | VM | 110 | Background Worker | Ubuntu 22.04 | 4C/16GB |
| 10.10.110.60 | hcm-prod-cache-redis-01 | VM | 110 | Redis Master | Ubuntu 22.04 | 4C/16GB |
| 10.10.110.61 | hcm-prod-cache-redis-02 | VM | 110 | Redis Slave | Ubuntu 22.04 | 4C/16GB |
| 10.10.110.62 | hcm-prod-mq-rabbit-01 | VM | 110 | RabbitMQ | Ubuntu 22.04 | 4C/16GB |
| 10.10.120.51 | hcm-prod-db-pgsql-01 | VM | 120 | PostgreSQL Primary | Ubuntu 22.04 | 16C/64GB |
| 10.10.120.52 | hcm-prod-db-pgsql-02 | VM | 120 | PostgreSQL Standby | Ubuntu 22.04 | 16C/64GB |
| 10.10.120.53 | hcm-prod-db-mysql-01 | VM | 120 | MySQL Master | Ubuntu 22.04 | 8C/32GB |
| 10.10.120.54 | hcm-prod-db-mysql-02 | VM | 120 | MySQL Slave | Ubuntu 22.04 | 8C/32GB |

### 6.3. HCM Site - Management VMs

| IP Address | Hostname | Type | VLAN | Function | OS | Purpose |
|------------|----------|------|------|----------|----|----- |
| 10.10.130.51 | hcm-prod-mon-zabbix-01 | VM | 130 | Monitoring | Ubuntu 22.04 | Zabbix Server |
| 10.10.130.52 | hcm-prod-mon-grafana-01 | VM | 130 | Visualization | Ubuntu 22.04 | Grafana |
| 10.10.130.53 | hcm-prod-mon-prometheus-01 | VM | 130 | Metrics | Ubuntu 22.04 | Prometheus |
| 10.10.130.54 | hcm-prod-log-elk-01 | VM | 130 | Logging | Ubuntu 22.04 | ELK Stack |
| 10.10.130.60 | hcm-prod-ansible-01 | VM | 130 | Automation | Ubuntu 22.04 | Ansible Tower |
| 10.10.130.61 | hcm-prod-gitlab-01 | VM | 130 | SCM | Ubuntu 22.04 | GitLab CE |
| 10.10.130.62 | hcm-prod-jenkins-01 | VM | 130 | CI/CD | Ubuntu 22.04 | Jenkins |
| 10.10.130.63 | hcm-prod-harbor-01 | VM | 130 | Registry | Ubuntu 22.04 | Harbor |
| 10.10.130.70 | hcm-prod-jump-linux-01 | VM | 130 | Jump Host | Ubuntu 22.04 | Bastion |
| 10.10.130.71 | hcm-prod-jump-win-01 | VM | 130 | Jump Host | Win 2022 | Bastion |

---

<a name="firewall-rules"></a>
## 7. FIREWALL RULES

### 7.1. Management VLAN (10) Rules

#### Inbound Rules

```
Rule 1: Allow vCenter Management
Source: 10.10.130.0/24 (Mgmt VMs)
Destination: 10.10.10.10 (vCenter)
Protocol: TCP
Ports: 443, 9443
Action: ALLOW
Description: vCenter Web Client & API access

Rule 2: Allow ESXi Management
Source: 10.10.10.10 (vCenter)
Destination: 10.10.10.21-24 (ESXi Hosts)
Protocol: TCP
Ports: 443, 902
Action: ALLOW
Description: vCenter to ESXi management

Rule 3: Allow SSH from Jump Hosts
Source: 10.10.130.70-71 (Jump Hosts)
Destination: 10.10.10.21-24 (ESXi Hosts)
Protocol: TCP
Port: 22
Action: ALLOW
Description: SSH access via Jump Host only

Rule 4: Allow DNS
Source: ANY (internal)
Destination: 10.10.10.51 (DNS)
Protocol: TCP/UDP
Port: 53
Action: ALLOW
Description: DNS queries

Rule 5: Allow NTP
Source: 10.10.10.0/24
Destination: 10.10.10.1 (Gateway - NTP relay)
Protocol: UDP
Port: 123
Action: ALLOW
Description: Time synchronization

Rule 6: Deny Direct SSH from User Networks
Source: 10.10.200.0/24 (Workstations)
Destination: 10.10.10.0/24
Protocol: TCP
Port: 22
Action: DENY
Log: Yes
Description: Force SSH via Jump Host
```

#### Outbound Rules

```
Rule 1: Allow Internet via Proxy
Source: 10.10.10.0/24
Destination: Internet
Protocol: TCP
Port: 80, 443
Via: Proxy 10.10.130.80
Action: ALLOW
Description: Updates & patches

Rule 2: Allow NTP to Internet
Source: 10.10.10.1 (NTP relay)
Destination: pool.ntp.org
Protocol: UDP
Port: 123
Action: ALLOW
Description: External NTP sync
```

---

### 7.2. vMotion VLAN (20) Rules

```
Rule 1: Allow vMotion Traffic
Source: 10.10.20.0/24
Destination: 10.10.20.0/24
Protocol: TCP
Port: 8000
Action: ALLOW
Description: vMotion between ESXi hosts

Rule 2: Deny All Other Traffic
Source: ANY
Destination: 10.10.20.0/24
Protocol: ANY
Action: DENY
Description: Isolated vMotion network
```

---

### 7.3. iSCSI VLAN (30) Rules

```
Rule 1: Allow iSCSI Traffic
Source: 10.10.30.21-34 (ESXi vmk2/vmk3)
Destination: 10.10.30.100-101 (Storage)
Protocol: TCP
Port: 3260
Action: ALLOW
Description: iSCSI initiator to target

Rule 2: Deny All Other Traffic
Source: ANY
Destination: 10.10.30.0/24
Protocol: ANY
Action: DENY
Description: Isolated storage network
```

---

### 7.4. DMZ VLAN (90) Rules

#### Inbound (from Internet)

```
Rule 1: Allow HTTPS to Web Servers
Source: Internet (ANY)
Destination: 10.10.90.50-52 (Public Web)
Protocol: TCP
Port: 443
Action: ALLOW
Rate Limit: 1000 req/sec per IP
Description: Public HTTPS access

Rule 2: Allow HTTP (redirect to HTTPS)
Source: Internet (ANY)
Destination: 10.10.90.50-52
Protocol: TCP
Port: 80
Action: ALLOW
Description: HTTP to HTTPS redirect

Rule 3: Allow VPN
Source: Internet (ANY)
Destination: 10.10.90.60 (VPN)
Protocol: UDP
Port: 1194, 500, 4500
Action: ALLOW
Description: OpenVPN & IPSec

Rule 4: Deny All Other Inbound
Source: Internet
Destination: 10.10.90.0/24
Protocol: ANY
Action: DENY
Log: Yes
```

#### Outbound (to Internal)

```
Rule 1: Allow DMZ to App Tier (API calls)
Source: 10.10.90.50-52 (DMZ Web)
Destination: 10.10.110.51-52 (App API)
Protocol: TCP
Port: 8080, 8443
Action: ALLOW
Description: Web to API backend

Rule 2: Deny DMZ to Database
Source: 10.10.90.0/24
Destination: 10.10.120.0/24 (Database)
Protocol: ANY
Action: DENY
Log: Yes
Description: No direct DB access from DMZ

Rule 3: Deny DMZ to Management
Source: 10.10.90.0/24
Destination: 10.10.10.0/24, 10.10.130.0/24
Protocol: ANY
Action: DENY
Log: Yes
Description: Isolate DMZ from management
```

---

### 7.5. Web Tier VLAN (100) Rules

```
Rule 1: Allow from Load Balancer
Source: 10.10.90.10 (External LB)
Destination: 10.10.100.51-53 (Web servers)
Protocol: TCP
Port: 80, 443
Action: ALLOW

Rule 2: Allow to App Tier
Source: 10.10.100.0/24
Destination: 10.10.110.0/24 (App Tier)
Protocol: TCP
Port: 8080, 8443, 9000-9100
Action: ALLOW
Description: Web to App communication

Rule 3: Allow to Cache
Source: 10.10.100.0/24
Destination: 10.10.110.60-61 (Redis)
Protocol: TCP
Port: 6379
Action: ALLOW

Rule 4: Deny to Database (direct)
Source: 10.10.100.0/24
Destination: 10.10.120.0/24
Protocol: ANY
Action: DENY
Log: Yes
Description: Web must go through App tier

Rule 5: Allow Internet Access (via Proxy)
Source: 10.10.100.0/24
Destination: Internet
Protocol: TCP
Port: 80, 443
Via: Proxy
Action: ALLOW
Description: External API calls, CDN
```

---

### 7.6. App Tier VLAN (110) Rules

```
Rule 1: Allow from Web Tier
Source: 10.10.100.0/24 (Web)
Destination: 10.10.110.51-53 (App)
Protocol: TCP
Port: 8080, 8443
Action: ALLOW

Rule 2: Allow to Database
Source: 10.10.110.51-55 (App servers)
Destination: 10.10.120.51-54 (Databases)
Protocol: TCP
Port: 5432, 3306, 27017
Action: ALLOW
Description: App to DB queries

Rule 3: Allow Redis Internal
Source: 10.10.110.0/24
Destination: 10.10.110.60-61 (Redis)
Protocol: TCP
Port: 6379, 16379
Action: ALLOW
Description: Redis master-slave replication

Rule 4: Allow RabbitMQ
Source: 10.10.110.51-53 (App)
Destination: 10.10.110.62 (RabbitMQ)
Protocol: TCP
Port: 5672, 15672
Action: ALLOW

Rule 5: Allow Internet (Restricted)
Source: 10.10.110.51-55
Destination: Internet
Protocol: TCP
Port: 443
Via: Proxy
Action: ALLOW
Description: External API calls only
```

---

### 7.7. Database Tier VLAN (120) Rules - CRITICAL

```
Rule 1: Allow from App Tier ONLY
Source: 10.10.110.51-55 (App servers)
Destination: 10.10.120.0/24 (All DBs)
Protocol: TCP
Port: 5432, 3306, 27017
Action: ALLOW
Description: Database queries

Rule 2: Allow DB Replication
Source: 10.10.120.51 (PostgreSQL Primary)
Destination: 10.10.120.52 (PostgreSQL Standby)
Protocol: TCP
Port: 5432
Action: ALLOW
Description: PostgreSQL streaming replication

Rule 3: Allow MySQL Replication
Source: 10.10.120.53 (MySQL Master)
Destination: 10.10.120.54 (MySQL Slave)
Protocol: TCP
Port: 3306
Action: ALLOW

Rule 4: Allow Backup Server
Source: 10.10.50.10 (Backup server)
Destination: 10.10.120.0/24
Protocol: TCP
Port: 5432, 3306, 22
Action: ALLOW
Description: Database backups

Rule 5: Allow from Jump Host (Admin only)
Source: 10.10.130.70 (Linux Jump)
Destination: 10.10.120.0/24
Protocol: TCP
Port: 22, 5432, 3306
Action: ALLOW
Description: DBA access

Rule 6: DENY ALL OTHER INBOUND
Source: ANY (except above)
Destination: 10.10.120.0/24
Protocol: ANY
Action: DENY
Log: Yes
Alert: Yes
Description: Critical - Unauthorized DB access attempt

Rule 7: DENY INTERNET ACCESS
Source: 10.10.120.0/24
Destination: Internet
Protocol: ANY
Action: DENY
Log: Yes
Description: Databases isolated from Internet
```

---

### 7.8. Management VMs VLAN (130) Rules

```
Rule 1: Allow Monitoring to All
Source: 10.10.130.51-53 (Monitoring servers)
Destination: ANY (internal)
Protocol: TCP
Port: 161, 10050, 9100, 8080
Action: ALLOW
Description: SNMP, Zabbix, Prometheus agents

Rule 2: Allow Ansible to Managed Hosts
Source: 10.10.130.60 (Ansible)
Destination: ANY (internal VMs)
Protocol: TCP
Port: 22, 5986
Action: ALLOW
Description: Configuration management

Rule 3: Allow Log Collection
Source: ANY (internal)
Destination: 10.10.130.54 (ELK)
Protocol: TCP
Port: 5044, 9200
Action: ALLOW
Description: Logstash & Elasticsearch

Rule 4: Jump Host Access
Source: 10.10.200.0/24 (Workstations - authorized IPs only)
Destination: 10.10.130.70-71 (Jump hosts)
Protocol: TCP
Port: 22, 3389
Action: ALLOW
Description: User access to jump hosts

Rule 5: Jump Host to Infrastructure
Source: 10.10.130.70-71
Destination: ANY (internal)
Protocol: TCP
Port: 22, 443, 3389
Action: ALLOW
Description: Admin access from jump hosts
```

---

### 7.9. Firewall Rule Summary Matrix

| Source VLAN | Dest VLAN | Allowed Ports | Action | Notes |
|-------------|-----------|---------------|--------|-------|
| 130 (Mgmt) | 10 (Management) | 443, 22 | ALLOW | Admin access |
| 130 (Mgmt) | ANY | 22, 443 | ALLOW | Jump host access |
| 90 (DMZ) | 110 (App) | 8080, 8443 | ALLOW | DMZ to App |
| 90 (DMZ) | 120 (DB) | ANY | DENY | No DB access |
| 100 (Web) | 110 (App) | 8080, 8443 | ALLOW | Web to App |
| 100 (Web) | 120 (DB) | ANY | DENY | Must via App |
| 110 (App) | 120 (DB) | 5432, 3306 | ALLOW | App to DB |
| 120 (DB) | Internet | ANY | DENY | Isolated |
| 200 (Users) | 10 (Mgmt) | 22 | DENY | No direct SSH |
| ANY | 20 (vMotion) | ANY | DENY | Isolated |
| ANY | 30 (iSCSI) | ANY | DENY | Isolated |

---

<a name="dns-records"></a>
## 8. DNS RECORDS

### 8.1. Forward DNS Zone (lab.local)

```
; Infrastructure
vcenter.lab.local.              A    10.10.10.10
hcm-prod-vc-01.lab.local.       A    10.10.10.10

esxi01.lab.local.               A    10.10.10.21
hcm-prod-esxi-01.lab.local.     A    10.10.10.21

esxi02.lab.local.               A    10.10.10.22
hcm-prod-esxi-02.lab.local.     A    10.10.10.22

esxi03.lab.local.               A    10.10.10.23
hcm-prod-esxi-03.lab.local.     A    10.10.10.23

; Production VMs
web01.lab.local.                A    10.10.100.51
hcm-prod-web-fe-01.lab.local.   A    10.10.100.51

app01.lab.local.                A    10.10.110.51
hcm-prod-app-api-01.lab.local.  A    10.10.110.51

db01.lab.local.                 A    10.10.120.51
hcm-prod-db-pgsql-01.lab.local. A    10.10.120.51

; Management
monitoring.lab.local.           A    10.10.130.51
gitlab.lab.local.               A    10.10.130.61
jenkins.lab.local.              A    10.10.130.62

; Load Balancers (VIPs)
www.lab.local.                  A    10.10.90.10
api.lab.local.                  A    10.10.110.10
```

### 8.2. Reverse DNS Zone (10.10.10.0/24)

```
; 10.10.10.x
10.10.in-addr.arpa.    PTR    hcm-prod-vc-01.lab.local.
21.10.10.10.in-addr.arpa.  PTR    hcm-prod-esxi-01.lab.local.
22.10.10.10.in-addr.arpa.  PTR    hcm-prod-esxi-02.lab.local.
```

### 8.3. DNS Server Configuration (Bind9)

```bash
# /etc/bind/named.conf.local

zone "lab.local" {
    type master;
    file "/etc/bind/zones/db.lab.local";
    allow-transfer { 10.10.10.52; }; # Secondary DNS
};

zone "10.10.10.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.10.10.10";
    allow-transfer { 10.10.10.52; };
};

# Forwarders for external queries
options {
    forwarders {
        8.8.8.8;
        1.1.1.1;
    };
    recursion yes;
    allow-recursion { 10.10.0.0/16; };
    listen-on { 10.10.10.51; };
};
```

---

<a name="checklist"></a>
## 9. CHECKLIST TRIỂN KHAI

### 9.1. Pre-Deployment Checklist

#### Physical Infrastructure
```
☐ Cabling hoàn thành (data, power, IPMI)
☐ Network switches cấu hình VLAN tags
☐ Uplink trunk ports configured
☐ Jumbo frames (MTU 9000) enabled on storage VLANs
☐ STP (Spanning Tree) configured
☐ Port security enabled
☐ LACP/Link Aggregation configured
```

#### ESXi Hosts
```
☐ ESXi installed (version 8.0 trở lên)
☐ Management IP assigned (10.x.10.21-29)
☐ Hostname set (hcm-prod-esxi-01)
☐ Root password set (strong password)
☐ NTP configured (10.10.10.1 or pool.ntp.org)
☐ SSH enabled (để cấu hình, disable sau)
☐ Lockdown mode = Disabled (until vCenter deployed)
☐ License key applied
☐ Hardware health checked (no warnings)
```

#### Storage
```
☐ iSCSI targets created
☐ LUNs provisioned
☐ Multipathing configured (MPIO)
☐ CHAP authentication enabled (recommended)
☐ Network isolation verified (VLAN 30)
☐ MTU 9000 verified
☐ Storage performance baseline tested
```

#### Network
```
☐ VLANs created on physical switches
☐ DHCP server configured (if needed)
☐ DNS server installed & configured
☐ Gateway/Router configured
☐ Firewall rules created
☐ Internet connectivity tested
☐ Inter-VLAN routing tested
```

---

### 9.2. vCenter Deployment Checklist

```
☐ OVA file downloaded & verified (SHA checksum)
☐ Deployment size selected (Tiny/Small/Medium)
☐ IP address planned: 10.10.10.10
☐ FQDN decided: hcm-prod-vc-01.lab.local
☐ DNS A and PTR records created
☐ Passwords prepared (root, SSO admin)
☐ Datastore selected with sufficient space
☐ Network port group selected (VLAN 10)
☐ NTP server configured
☐ Thin provisioning enabled
☐ SSH enabled (for troubleshooting)

Stage 1 Deployment:
☐ Deploy OVF completed (20-30 mins)
☐ vCenter VM powered on
☐ Web UI accessible (https://10.10.10.10:443)
☐ Root SSH login tested (if needed)

Stage 2 Setup:
☐ SSO configuration completed
☐ vCenter services started
☐ vSphere Client accessible
☐ License applied
☐ CEIP opted out (if required)

Post-Deployment:
☐ Add ESXi hosts to vCenter
☐ Create Datacenter object
☐ Create Cluster (enable DRS, HA)
☐ Configure vSphere HA (admission control)
☐ Configure DRS (automation level)
☐ Disable SSH on vCenter (security)
```

---

### 9.3. ESXi Networking Checklist (per host)

#### vSwitch0 (Management + vMotion)
```
☐ Created with 2 NICs (vmnic0, vmnic1)
☐ Teaming policy: Route based on originating port ID
☐ Failover order: Active-Active
☐ Management Network port group:
   ☐ VLAN ID: 10
   ☐ vmk0 created with IP 10.10.10.21
   ☐ Gateway: 10.10.10.1
   ☐ MTU: 1500
   ☐ Management traffic enabled
☐ vMotion Network port group:
   ☐ VLAN ID: 20
   ☐ vmk1 created with IP 10.10.20.21
   ☐ No gateway
   ☐ MTU: 9000
   ☐ vMotion traffic enabled
   ☐ TCP/IP stack: vMotion
```

#### vSwitch1 (iSCSI) - Standard Switch
```
☐ vmnic2: iSCSI-A port group
   ☐ VLAN ID: 30
   ☐ vmk2: 10.10.30.21
   ☐ MTU: 9000
   ☐ Bind to vmnic2 only
☐ vmnic3: iSCSI-B port group
   ☐ VLAN ID: 30
   ☐ vmk3: 10.10.30.31
   ☐ MTU: 9000
   ☐ Bind to vmnic3 only
☐ Software iSCSI adapter enabled
☐ Port binding configured
☐ Dynamic discovery target added: 10.10.30.100
☐ CHAP configured (if used)
☐ Rescan storage adapters
☐ Datastores visible & mounted
```

#### Distributed Switch (VDS) - VM Networks
```
☐ VDS created: hcm-prod-vds-01
☐ Version: 7.0 or later
☐ MTU: 9000
☐ Add hosts: esxi-01 to esxi-04
☐ Migrate vmnic4, vmnic5 to VDS
☐ Port groups created:
   ☐ VLAN-090-DMZ (VLAN 90)
   ☐ VLAN-100-Web (VLAN 100)
   ☐ VLAN-110-App (VLAN 110)
   ☐ VLAN-120-Database (VLAN 120)
   ☐ VLAN-130-MgmtVMs (VLAN 130)
☐ Security policy:
   ☐ Promiscuous mode: Reject
   ☐ MAC changes: Reject
   ☐ Forged transmits: Reject
☐ Teaming policy: Load balancing
☐ NetFlow configured (optional)
☐ Port mirroring configured (optional)
```

---

### 9.4. VM Deployment Checklist (Template)

#### Before Creating VM
```
☐ Hostname decided: hcm-prod-web-01
☐ IP address assigned: 10.10.100.51
☐ VLAN determined: 100 (Web Tier)
☐ Resources planned: 4 vCPU, 16GB RAM, 100GB disk
☐ OS decided: Ubuntu 22.04 LTS
☐ DNS A record created
☐ Firewall rules created
☐ Application dependencies identified
```

#### VM Creation
```
☐ Deploy from template (if available)
☐ Or create new VM:
   ☐ Name: hcm-prod-web-01
   ☐ Datastore selected
   ☐ Folder organized (by tier/function)
   ☐ Resource pool assigned (if used)
   ☐ Network: VLAN-100-Web
   ☐ Guest OS: Ubuntu Linux (64-bit)
   ☐ CPU: 4 sockets x 1 core (or 2x2)
   ☐ Memory: 16 GB
   ☐ Disk: 100 GB (thin provision)
   ☐ SCSI controller: VMware Paravirtual
   ☐ Network adapter: VMXNET3
☐ VM Hardware version: 19 or later
☐ VMware Tools installation planned
```

#### Post-Deployment
```
☐ OS installed & updated
☐ Static IP configured (10.10.100.51/24)
☐ Gateway configured (10.10.100.1)
☐ DNS configured (10.10.10.51)
☐ Hostname set (hcm-prod-web-01)
☐ Timezone set (Asia/Ho_Chi_Minh)
☐ NTP configured
☐ VMware Tools installed
☐ SSH key-based authentication configured
☐ Firewall configured (ufw/firewalld)
☐ Monitoring agent installed (Zabbix/Prometheus)
☐ Log forwarding configured (rsyslog to ELK)
☐ Backup job created
☐ VM snapshot taken (clean state)
☐ Application installed & tested
☐ Documentation updated
```

---

### 9.5. Security Hardening Checklist

#### ESXi Host Security
```
☐ Change default root password (complex password)
☐ Enable lockdown mode (after vCenter integration)
☐ Disable unnecessary services:
   ☐ Disable SSH (sau khi config xong)
   ☐ Disable ESXi Shell
   ☐ Disable MOB (Managed Object Browser)
☐ Configure NTP authentication
☐ Enable certificate checking
☐ Configure syslog forwarding
☐ Set account lockout policy
☐ Configure password complexity
☐ Enable audit logging
☐ Join Active Directory (optional)
☐ Configure firewall rules (only necessary ports)
☐ Disable unnecessary VIBs
☐ Enable Secure Boot (if supported)
☐ Regular patching schedule established
```

#### vCenter Security
```
☐ Strong SSO password (administrator@vsphere.local)
☐ Enable password expiration
☐ Configure password policy:
   ☐ Minimum length: 12 characters
   ☐ Complexity: uppercase, lowercase, number, special
   ☐ Maximum lifetime: 90 days
   ☐ Reuse restriction: 5 passwords
☐ Enable account lockout (5 failed attempts)
☐ Configure session timeout (15 minutes)
☐ Enable audit logging
☐ Configure syslog forwarding
☐ Integrate with LDAP/AD (centralized authentication)
☐ Implement RBAC (Role-Based Access Control):
   ☐ No users with full admin except emergency
   ☐ Separate roles for: Network Admin, Storage Admin, VM Admin
☐ Enable vSphere Certificate Authority
☐ Replace self-signed certificates (production)
☐ Enable vCenter HA (High Availability)
☐ Backup vCenter configuration regularly
☐ Disable unused services (Update Manager if not used)
```

#### Network Security
```
☐ Change default VLAN 1 (don't use native VLAN)
☐ Enable BPDU Guard on access ports
☐ Enable Root Guard on uplinks
☐ Configure Port Security (MAC address limits)
☐ Disable unused ports
☐ Enable DHCP Snooping
☐ Enable Dynamic ARP Inspection (DAI)
☐ Enable IP Source Guard
☐ Implement private VLANs (if needed)
☐ Configure storm control
☐ Enable logging on switches
☐ Restrict management access (specific IPs only)
☐ Use SNMPv3 (not v1/v2c)
☐ Disable CDP/LLDP on untrusted ports
```

#### VM Security
```
☐ Minimize installed packages (least privilege)
☐ Disable root login via SSH
☐ Use SSH keys (disable password auth)
☐ Configure host-based firewall (allow only necessary)
☐ Disable unused services
☐ Regular OS updates (automated)
☐ Implement antivirus/antimalware
☐ Enable SELinux/AppArmor (Linux)
☐ Configure audit logging (auditd)
☐ Implement file integrity monitoring (AIDE/Tripwire)
☐ Use encrypted filesystems for sensitive data
☐ Implement network segmentation
☐ Configure intrusion detection (OSSEC/Wazuh)
☐ Regular vulnerability scanning
☐ Application-level WAF (for web servers)
```

---

### 9.6. High Availability Checklist

#### vSphere HA Configuration
```
☐ HA enabled on cluster
☐ Admission control enabled:
   ☐ Policy: Host failures cluster tolerates
   ☐ Value: 1 (or more based on SLA)
☐ Host monitoring enabled
☐ VM monitoring enabled:
   ☐ VM Monitoring sensitivity: Medium
   ☐ Failure interval: 30 seconds
   ☐ Window: 120 seconds
   ☐ Max failures: 3
   ☐ Max resets: 3
☐ Datastore heartbeating configured
☐ Isolation response: Power off VMs
☐ VM restart priority configured:
   ☐ High: Critical VMs (DB, vCenter)
   ☐ Medium: App servers
   ☐ Low: Development VMs
☐ Host isolation response tested
☐ HA failure scenarios tested:
   ☐ Host power off
   ☐ Network isolation
   ☐ Storage failure
```

#### vSphere DRS Configuration
```
☐ DRS enabled on cluster
☐ Automation level: Fully Automated
☐ Migration threshold: Apply priority 3 recommendations
☐ VM-Host affinity rules (if needed):
   ☐ Should rules for licensing
   ☐ Must rules for compliance
☐ VM-VM affinity rules:
   ☐ Anti-affinity: DB replicas on different hosts
   ☐ Affinity: Web + App if latency-sensitive
☐ Resource pools created (if needed):
   ☐ Production (High priority)
   ☐ Development (Low priority)
☐ DRS testing:
   ☐ Manual vMotion test
   ☐ DRS recommendation verification
   ☐ Resource contention simulation
```

#### Application-Level HA
```
☐ Database replication configured:
   ☐ PostgreSQL: Streaming replication
   ☐ MySQL: Master-Slave replication
   ☐ Automatic failover tested
☐ Load balancers configured:
   ☐ Health checks enabled (every 5 seconds)
   ☐ Failover thresholds: 3 consecutive failures
   ☐ Session persistence (if needed)
   ☐ SSL offloading configured
☐ Redis Sentinel for cache HA
☐ RabbitMQ clustering configured
☐ Application connection pooling
☐ Graceful degradation tested
☐ Circuit breakers implemented
```

---

### 9.7. Backup & Disaster Recovery Checklist

#### Backup Strategy
```
☐ Backup software deployed (Veeam/Commvault/Rubrik)
☐ Backup repository configured:
   ☐ Primary: Local NAS (10.10.50.100)
   ☐ Secondary: Offsite/Cloud
☐ Backup schedules defined:
   ☐ Critical VMs (vCenter, DB): Daily full + hourly incremental
   ☐ Production VMs: Daily incremental + Weekly full
   ☐ Development VMs: Weekly full
☐ Retention policy:
   ☐ Daily: 7 days
   ☐ Weekly: 4 weeks
   ☐ Monthly: 12 months
   ☐ Yearly: 7 years (compliance)
☐ Application-consistent backups (VSS/pre-freeze scripts)
☐ Database backup (dedicated):
   ☐ PostgreSQL pg_dump daily
   ☐ MySQL mysqldump daily
   ☐ Transaction logs every 15 minutes
☐ Configuration backups:
   ☐ vCenter configuration
   ☐ ESXi configuration
   ☐ Switch configurations
   ☐ Firewall rules
☐ Backup verification:
   ☐ Automated restore tests (monthly)
   ☐ Full DR drill (quarterly)
☐ Offsite replication (3-2-1 rule):
   ☐ 3 copies of data
   ☐ 2 different media types
   ☐ 1 offsite copy
☐ Backup encryption enabled
☐ Backup job alerts configured
```

#### Disaster Recovery
```
☐ DR site established (10.250.0.0/16)
☐ Site Recovery Manager (SRM) deployed
☐ Replication configured:
   ☐ vSphere Replication for VMs
   ☐ Storage-based replication (if available)
☐ RPO (Recovery Point Objective) defined:
   ☐ Critical: 15 minutes
   ☐ Important: 1 hour
   ☐ Normal: 4 hours
☐ RTO (Recovery Time Objective) defined:
   ☐ Critical: 1 hour
   ☐ Important: 4 hours
   ☐ Normal: 8 hours
☐ Recovery plans created:
   ☐ Network remapping configured
   ☐ IP customization configured
   ☐ Boot order priority set
   ☐ Post-recovery scripts prepared
☐ DR testing schedule:
   ☐ Planned failover test: Quarterly
   ☐ Unplanned failover test: Bi-annually
   ☐ Full site failover: Annually
☐ DR documentation:
   ☐ Runbook created
   ☐ Contact list updated
   ☐ Escalation procedures defined
☐ Disaster declaration criteria defined
☐ Communication plan established
```

---

### 9.8. Monitoring & Alerting Checklist

#### Infrastructure Monitoring
```
☐ Zabbix/Prometheus server deployed
☐ Grafana dashboards created
☐ ESXi hosts monitored:
   ☐ CPU utilization (alert > 80%)
   ☐ Memory usage (alert > 85%)
   ☐ Datastore space (alert < 20% free)
   ☐ Network throughput
   ☐ Hardware health (sensors, fans, PSU)
☐ vCenter monitoring:
   ☐ Service health
   ☐ Database size
   ☐ Certificate expiration (alert 30 days before)
   ☐ Backup job status
☐ Network monitoring:
   ☐ Interface errors
   ☐ Bandwidth utilization
   ☐ Latency between sites
   ☐ Packet loss
☐ Storage monitoring:
   ☐ IOPS performance
   ☐ Latency (alert > 20ms)
   ☐ Queue depth
   ☐ Array health
```

#### Application Monitoring
```
☐ Application performance monitoring (APM) configured
☐ Web servers:
   ☐ HTTP response time (alert > 2 seconds)
   ☐ Error rate (alert > 1%)
   ☐ Request rate
   ☐ SSL certificate expiration
☐ App servers:
   ☐ API response time
   ☐ Queue length
   ☐ Thread pool utilization
   ☐ Memory leaks
☐ Databases:
   ☐ Query performance (slow query log)
   ☐ Connection pool usage
   ☐ Replication lag (alert > 10 seconds)
   ☐ Deadlocks
   ☐ Database size growth
☐ Cache (Redis):
   ☐ Hit rate (alert < 80%)
   ☐ Memory fragmentation
   ☐ Evicted keys
☐ Message queue (RabbitMQ):
   ☐ Queue depth (alert > 1000 messages)
   ☐ Consumer lag
   ☐ Message rate
```

#### Logging
```
☐ Centralized logging (ELK/Graylog) deployed
☐ Log sources configured:
   ☐ ESXi hosts (syslog)
   ☐ vCenter logs
   ☐ Firewall logs
   ☐ Application logs
   ☐ Web server access logs
   ☐ Database logs
☐ Log retention:
   ☐ Hot storage: 30 days
   ☐ Warm storage: 90 days
   ☐ Cold storage: 1 year
☐ Log parsing/filtering configured
☐ Alerting rules created:
   ☐ Failed login attempts (> 5 in 5 min)
   ☐ Error rate spike (> 10x baseline)
   ☐ Disk full warnings
   ☐ Critical application errors
☐ Dashboards created for:
   ☐ Security events
   ☐ Application errors
   ☐ Performance metrics
   ☐ Audit trail
```

#### Alerting
```
☐ Alert channels configured:
   ☐ Email (PagerDuty/OpsGenie)
   ☐ Slack/Teams webhook
   ☐ SMS (critical alerts only)
☐ Alert severity levels:
   ☐ P1 (Critical): Immediate response, 24/7
   ☐ P2 (High): Response within 1 hour
   ☐ P3 (Medium): Response within 4 hours
   ☐ P4 (Low): Response within 24 hours
☐ Escalation policies defined
☐ On-call rotation schedule
☐ Alert fatigue prevention:
   ☐ Thresholds tuned (avoid false positives)
   ☐ Flapping detection enabled
   ☐ Maintenance windows configured
☐ Alert documentation (runbooks):
   ☐ Each alert has troubleshooting steps
   ☐ Known issues documented
   ☐ Resolution procedures defined
```

---

### 9.9. Performance Optimization Checklist

#### ESXi Host Optimization
```
☐ CPU configuration:
   ☐ Hyperthreading enabled (if beneficial)
   ☐ Power management: High Performance
   ☐ CPU C-states: Disabled (for low-latency apps)
☐ Memory configuration:
   ☐ Large pages enabled
   ☐ Memory compression enabled
   ☐ TPS (Transparent Page Sharing): Consider security vs performance
☐ Storage optimization:
   ☐ VAAI (vStorage APIs for Array Integration) enabled
   ☐ SIOC (Storage I/O Control) configured
   ☐ Datastore alignment verified
☐ Network optimization:
   ☐ Jumbo frames enabled (MTU 9000) on storage
   ☐ TSO (TCP Segmentation Offload) enabled
   ☐ LRO (Large Receive Offload) enabled
   ☐ NetQueue/RSS enabled
☐ Advanced settings:
   ☐ Disk.SchedNumReqOutstanding = 32 (for SSD)
   ☐ Net.TcpipHeapSize = 32 (high network traffic)
   ☐ Mem.MemZipEnable = 1
```

#### VM Optimization
```
☐ Right-sizing performed:
   ☐ vCPU count matches workload (avoid over-provisioning)
   ☐ Memory allocated based on actual usage
   ☐ Memory reservation for critical VMs only
☐ Virtual hardware:
   ☐ Hardware version: Latest compatible
   ☐ SCSI controller: Paravirtual (best performance)
   ☐ Network adapter: VMXNET3
☐ VMware Tools up to date
☐ VM swap file location optimized
☐ VM snapshots policy:
   ☐ No long-lived snapshots (> 24 hours)
   ☐ Automated snapshot cleanup
☐ Disk optimization:
   ☐ Thin provisioning for dev/test
   ☐ Thick provisioning for production databases
   ☐ Separate VMDKs for OS, data, logs
☐ CPU optimization:
   ☐ CPU hot-add disabled (unless needed)
   ☐ NUMA optimization (vCPU ≤ physical cores per socket)
☐ Memory optimization:
   ☐ Memory hot-add disabled (fragments memory)
   ☐ Memory reservation for databases
   ☐ Memory shares adjusted by priority
```

#### Database Optimization
```
☐ PostgreSQL tuning:
   ☐ shared_buffers = 25% of RAM
   ☐ effective_cache_size = 50% of RAM
   ☐ work_mem, maintenance_work_mem adjusted
   ☐ max_connections limited
   ☐ Connection pooling (PgBouncer)
☐ MySQL tuning:
   ☐ innodb_buffer_pool_size = 70% of RAM
   ☐ Query cache (deprecated in 8.0)
   ☐ Connection pooling (ProxySQL)
☐ Regular maintenance:
   ☐ VACUUM (PostgreSQL) - automated
   ☐ OPTIMIZE TABLE (MySQL) - scheduled
   ☐ Index optimization
   ☐ Statistics update
☐ Slow query monitoring enabled
☐ Query execution plans reviewed
```

---

### 9.10. Documentation Checklist

#### Required Documentation
```
☐ Network topology diagram (Visio/Draw.io)
☐ IP address allocation spreadsheet
☐ VLAN configuration document
☐ Firewall rules matrix
☐ DNS records documentation
☐ VM inventory spreadsheet:
   ☐ Hostname, IP, VLAN, vCPU, RAM, Disk
   ☐ Owner, Purpose, Environment
   ☐ Backup schedule, DR tier
☐ Service dependencies map
☐ Change management process
☐ Runbooks for common procedures:
   ☐ Adding new ESXi host
   ☐ Deploying new VM
   ☐ Expanding datastore
   ☐ vCenter upgrade procedure
   ☐ ESXi patching procedure
☐ Disaster recovery plan
☐ Business continuity plan
☐ Incident response procedures
☐ Escalation matrix (contacts)
☐ Password management (KeePass/1Password)
☐ License key inventory
☐ Warranty/Support contracts
☐ Compliance documentation (ISO27001, PCI-DSS, etc.)
```

---

## 10. MAINTENANCE PROCEDURES

### 10.1. Monthly Maintenance Tasks

```
☐ Review monitoring alerts and trends
☐ Check disk space on all datastores (< 80% used)
☐ Review VM resource utilization (right-sizing opportunities)
☐ Check for VM snapshots older than 24 hours
☐ Verify backup job success rate (> 95%)
☐ Test one random backup restore
☐ Review firewall logs for anomalies
☐ Check certificate expiration dates
☐ Review and archive old logs
☐ Update documentation for any changes
☐ Review user access (remove terminated employees)
☐ Check for available patches (ESXi, vCenter)
☐ Review capacity planning (6-month forecast)
```

### 10.2. Quarterly Maintenance Tasks

```
☐ Major version updates review (ESXi, vCenter)
☐ DR failover test (planned)
☐ Full backup restore test
☐ Performance baseline update
☐ Security audit:
   ☐ Password rotation
   ☐ Review firewall rules (remove obsolete)
   ☐ Review user permissions (least privilege)
☐ Disaster recovery plan review/update
☐ Business continuity plan table-top exercise
☐ Hardware health check (firmware updates)
☐ License compliance review
☐ Capacity planning review (1-year forecast)
☐ Vendor support contract renewal check
```

### 10.3. Annual Maintenance Tasks

```
☐ Full DR drill (unplanned failover test)
☐ Infrastructure architecture review
☐ Major upgrade planning (vSphere version)
☐ Hardware refresh planning (3-5 year cycle)
☐ Security penetration test
☐ Compliance audit (ISO27001, PCI-DSS)
☐ Disaster recovery plan full update
☐ Business continuity plan full update
☐ Review SLAs with business stakeholders
☐ Training budget and planning for team
☐ Vendor relationship review
☐ Technology roadmap update (3-year plan)
```

---

## 11. TROUBLESHOOTING GUIDE

### 11.1. Common Issues & Resolution

#### Issue: VM không ping được

```
Troubleshooting Steps:
1. Check VM power state (powered on?)
2. Check network adapter connected (device status)
3. Verify correct port group/VLAN assigned
4. Check guest OS network configuration:
   - IP address correct?
   - Gateway configured?
   - DNS configured?
5. Check physical network:
   - VLAN tagged on switch?
   - Trunk port configured?
   - Cable connected?
6. Check firewall rules (both VM and network)
7. Verify routing between VLANs
8. Check ARP table (arp -a)
9. Packet capture (tcpdump/Wireshark)
```

#### Issue: vMotion failed

```
Common Causes & Fixes:
1. Different CPU families → Enable EVC (Enhanced vMotion Compatibility)
2. Network issue → Check vMotion VMkernel connectivity
3. Storage issue → Verify shared storage accessible by both hosts
4. Resource constraints → Check destination host has sufficient resources
5. Snapshot > 32GB → Consolidate snapshots first
6. VM hardware version mismatch → Upgrade VM hardware
7. Advanced CPU features → Disable if not needed

Commands:
# Test vMotion network
vmkping -I vmk1 10.10.20.22

# Check EVC mode
vim-cmd /hostsvc/hosthardware/get_cpuid_info
```

#### Issue: Datastore full

```
Resolution Steps:
1. Identify space consumers:
   # SSH to ESXi
   cd /vmfs/volumes/datastore_name
   du -sh *
   
2. Common culprits:
   - Old snapshots (*.vmdk-delta files)
   - VM swap files (*.vswp)
   - Log files (vmware*.log)
   - Orphaned VMDKs
   
3. Clean up:
   - Delete old snapshots (Snapshot Manager)
   - Delete unnecessary VMs
   - Storage vMotion VMs to another datastore
   - Expand datastore (add LUN)
   
4. Prevention:
   - Enable Storage DRS
   - Set datastore alerts (80% threshold)
   - Implement snapshot retention policy
   - Regular cleanup automation
```

#### Issue: ESXi host not responding

```
Diagnosis:
1. Physical access → Check iLO/iDRAC/IPMI
   - Host powered on?
   - PSOD (Purple Screen of Death)?
   - Hardware failure?
   
2. Network access → Check management network
   - Cable unplugged?
   - Switch port down?
   - IP conflict?
   
3. vCenter shows disconnected:
   - ESXi vpxa service down?
   - Certificate expired?
   - Firewall blocking port 443/902?

Resolution:
# Restart management agents (if SSH accessible)
/etc/init.d/hostd restart
/etc/init.d/vpxa restart

# If completely unresponsive:
- Check IPMI → Power cycle if needed
- HA should failover VMs automatically
- After restart, investigate logs: /var/log/vmkernel.log

Prevention:
- Enable ESXi HA
- Configure IPMI monitoring
- Regular hardware health checks
```

---

## 12. EMERGENCY CONTACT & ESCALATION

### 12.1. On-Call Rotation

```
Primary On-Call:   [Name]  [Phone]  [Email]
Secondary On-Call: [Name]  [Phone]  [Email]
Manager:           [Name]  [Phone]  [Email]
```

### 12.2. Vendor Support

```
VMware Support:
- Phone: [Regional Support Number]
- Portal: https://customerconnect.vmware.com
- License Key: [XXXXX-XXXXX-XXXXX-XXXXX]
- Contract: [Support Contract Number]

Hardware Vendor (Dell/HPE):
- Support Phone: [Vendor Support Number]
- Service Tag: [Tag Number]
- Contract Expiry: [Date]

Network Vendor (Cisco/Juniper):
- TAC Phone: [TAC Number]
- Contract: [SmartNet/JTAC Number]

Storage Vendor:
- Support: [Support Contact]
- Serial Numbers: [SN List]
```

### 12.3. Escalation Matrix

```
Severity P1 (Critical - Production Down):
- Immediate notification: On-Call Engineer
- If no response in 15 min: Secondary On-Call
- If no resolution in 30 min: Escalate to Manager
- If no resolution in 1 hour: Escalate to CTO + Engage vendor support

Severity P2 (High - Partial Outage):
- Notification: On-Call Engineer
- Response SLA: 1 hour
- Escalation after 2 hours: Manager

Severity P3 (Medium - Degraded Performance):
- Notification: On-Call Engineer
- Response SLA: 4 hours
- Escalation after 8 hours: Manager

Severity P4 (Low - Minor Issue):
- Notification: Email to team
- Response SLA: Next business day
- No escalation unless becomes higher priority
```

---

## 13. CHANGE MANAGEMENT

### 13.1. Change Request Process

```
1. Submit Change Request (CR)
   ☐ Description of change
   ☐ Justification/Business reason
   ☐ Impact assessment
   ☐ Risk assessment
   ☐ Rollback plan
   ☐ Testing plan
   ☐ Maintenance window requested
   
2. Change Review
   ☐ Technical review by team
   ☐ Manager approval
   ☐ Stakeholder notification
   
3. Change Implementation
   ☐ Follow documented procedure
   ☐ Take pre-change snapshot/backup
   ☐ Execute change
   ☐ Verify success
   ☐ Update documentation
   
4. Post-Change Review
   ☐ Document results
   ☐ Lessons learned (if issues occurred)
   ☐ Close change ticket
```

### 13.2. Maintenance Windows

```
Standard Maintenance Windows:
- Production: Sunday 02:00-06:00 (monthly, 3rd weekend)
- Development: Saturday 00:00-08:00 (weekly)
- Emergency: As needed with 4-hour notice

Change Freeze Periods:
- Year-end: Dec 15 - Jan 5
- Peak business seasons: [Company-specific]
- Major product launches: [As announced]
```

---

## PHỤ LỤC A: QUICK REFERENCE COMMANDS

### ESXi Useful Commands

```bash
# Network
esxcli network ip interface list
esxcli network vswitch standard list
esxcli network nic list
vmkping -I vmk0 10.10.10.1

# Storage
esxcli storage vmfs extent list
esxcli storage core adapter list
esxcli storage core path list

# VM Management
vim-cmd vmsvc/getallvms
vim-cmd vmsvc/power.on <vmid>
vim-cmd vmsvc/power.off <vmid>

# System Info
esxcli system version get
esxcli hardware cpu list
esxcli hardware memory get
esxcli system uuid get

# Firewall
esxcli network firewall get
esxcli network firewall ruleset list
esxcli network firewall ruleset set --ruleset-id=sshServer --enabled=true

# Logs
tail -f /var/log/vmkernel.log
tail -f /var/log/hostd.log
```

### vCenter API Examples (PowerCLI)

```powershell
# Connect to vCenter
Connect-VIServer -Server 10.10.10.10 -User administrator@vsphere.local

# Get all VMs
Get-VM

# Get VMs by resource pool
Get-ResourcePool -Name "Production" | Get-VM

# vMotion a VM
Get-VM "hcm-prod-web-01" | Move-VM -Destination (Get-VMHost "hcm-prod-esxi-02")

# Create VM snapshot
Get-VM "hcm-prod-db-01" | New-Snapshot -Name "Pre-Patch" -Description "Before OS patching"

# Get datastore usage
Get-Datastore | Select Name, CapacityGB, FreeSpaceGB, @{N="UsedPercent";E={[math]::Round(($_.CapacityGB - $_.FreeSpaceGB)/$_.CapacityGB*100,2)}}

# Get VM CPU/Memory usage
Get-VM | Select Name, NumCpu, MemoryGB, @{N="CPUUsage";E={$_.ExtensionData.Summary.QuickStats.OverallCpuUsage}}

# Disconnect
Disconnect-VIServer -Confirm:$false
```

---

## PHỤLỤC B: NETWORK DIAGRAM

```
[Internet]
    |
[Firewall 10.10.10.3]
    |
[Core Switch]
    |
    +---[VLAN 10 - Management]---[vCenter 10.10.10.10]
    |                           [ESXi 10.10.10.21-24]
    |
    +---[VLAN 20 - vMotion]-----[ESXi vmk1]
    |
    +---[VLAN 30 - iSCSI]-------[ESXi vmk2/vmk3]---[Storage 10.10.30.100]
    |
    +---[VLAN 90 - DMZ]---------[Public Web VMs]
    |
    +---[VLAN 100 - Web]--------[Web Tier VMs]
    |
    +---[VLAN 110 - App]--------[App Tier VMs]
    |
    +---[VLAN 120 - Database]---[Database VMs] (Isolated, no Internet)
    |
    +---[VLAN 130 - Mgmt VMs]---[Monitoring, Automation]
```

---

## KẾT LUẬN

Tài liệu này cung cấp một framework toàn diện để thiết kế, triển khai và vận hành hạ tầng VMware vSphere theo chuẩn production.

### Nguyên tắc quan trọng cần nhớ:

1. **Security First**: Luôn ưu tiên bảo mật (network segmentation, least privilege, monitoring)
2. **High Availability**: Thiết kế để tránh single point of failure
3. **Scalability**: Quy hoạch cho tương lai (room to grow)
4. **Documentation**: Tài liệu phải luôn được cập nhật
5. **Automation**: Tự động hóa các tác vụ lặp đi lặp lại
6. **Monitoring**: "You can't manage what you don't measure"
7. **Regular Maintenance**: Ngăn chặn vấn đề trước khi xảy ra
8. **Disaster Recovery**: Luôn có plan B (và test thường xuyên)
