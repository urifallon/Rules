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
| 10.10.10.51 | hcm-prod-dns-01
