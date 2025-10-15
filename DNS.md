# QUY TẮC DOMAIN & DNS CONFIGURATION
## PRODUCTION ENVIRONMENT STANDARD

**Phiên bản:** 1.0  
**Ngày:** 15/10/2025  
**Phạm vi:** Toàn bộ hệ thống VMware vSphere & Production VMs

---

## MỤC LỤC

1. [Nguyên tắc Domain Naming](#nguyen-tac-domain)
2. [Phân loại Domain theo mục đích](#phan-loai-domain)
3. [Cấu trúc DNS Records](#cau-truc-dns)
4. [Cấu hình DNS Server](#cau-hinh-dns-server)
5. [Active Directory Integration](#active-directory)
6. [SSL Certificate Management](#ssl-certificate)
7. [Troubleshooting DNS](#troubleshooting)

---

<a name="nguyen-tac-domain"></a>
## 1. NGUYÊN TẮC DOMAIN NAMING

### 1.1. Quy tắc chung

#### ✅ NÊN:
```
- Sử dụng lowercase (chữ thường)
- Độ dài: 3-63 ký tự
- Ký tự hợp lệ: a-z, 0-9, dấu gạch ngang (-)
- Dễ đọc, dễ nhớ, có ý nghĩa
- Nhất quán trong toàn bộ hệ thống
```

#### ❌ KHÔNG NÊN:
```
- Bắt đầu hoặc kết thúc bằng dấu gạch ngang
- Sử dụng ký tự đặc biệt (_, @, #, etc.)
- Sử dụng UPPERCASE (CHỮ HOA)
- Domain quá dài (>63 ký tự)
- Domain trùng với public domain (google.com, microsoft.com)
- Sử dụng .local cho production (deprecated)
```

### 1.2. Cấu trúc Domain Standard

```
Format: <subdomain>.<environment>.<company>.<tld>

Ví dụ:
vcenter.prod.company.internal
web01.prod.company.internal
db-master.prod.company.internal
api.staging.company.internal
```

---

<a name="phan-loai-domain"></a>
## 2. PHÂN LOẠI DOMAIN THEO MỤC ĐÍCH

### 2.1. Internal Domain (Chỉ dùng nội bộ)

#### A. Infrastructure Domain

**Mục đích:** Quản lý hạ tầng VMware, ESXi, Storage

**TLD khuyên dùng:** `.internal`, `.corp`, `.local` (legacy)

**Cấu trúc:**
```yaml
company.internal              # Root domain
├── prod.company.internal     # Production environment
│   ├── vcenter.prod.company.internal          # vCenter Server
│   ├── esxi01.prod.company.internal           # ESXi Host 01
│   ├── esxi02.prod.company.internal           # ESXi Host 02
│   ├── storage.prod.company.internal          # Storage Array
│   └── nsx-mgr.prod.company.internal          # NSX Manager
│
├── dev.company.internal      # Development environment
│   ├── vcenter.dev.company.internal
│   └── esxi01.dev.company.internal
│
└── mgmt.company.internal     # Management tools
    ├── monitoring.mgmt.company.internal       # Zabbix/Grafana
    ├── backup.mgmt.company.internal           # Veeam
    ├── ansible.mgmt.company.internal          # Automation
    └── gitlab.mgmt.company.internal           # SCM
```

**Ví dụ đầy đủ:**
```
# vCenter Servers
hcm-prod-vc-01.prod.company.internal      → 10.10.10.10
hn-prod-vc-01.prod.company.internal       → 10.20.10.10
vcenter.prod.company.internal             → 10.10.10.10 (CNAME)

# ESXi Hosts
hcm-prod-esxi-01.prod.company.internal    → 10.10.10.21
hcm-prod-esxi-02.prod.company.internal    → 10.10.10.22
hn-prod-esxi-01.prod.company.internal     → 10.20.10.21

# Storage
hcm-prod-stor-iscsi-01.prod.company.internal  → 10.10.30.100
hn-prod-stor-nas-01.prod.company.internal     → 10.20.30.100
```

---

#### B. Active Directory Domain

**Mục đích:** Windows authentication, Group Policy, LDAP

**TLD khuyên dùng:** `.corp`, `.ad.company.internal`

**Cấu trúc:**
```yaml
ad.company.corp                      # AD Forest Root
├── _msdcs.ad.company.corp           # Domain Controllers SRV records
├── _sites.ad.company.corp           # AD Sites
├── _tcp.ad.company.corp             # TCP services
└── _udp.ad.company.corp             # UDP services

Domain Controllers:
├── dc01.ad.company.corp             # Primary DC (HCM)
├── dc02.ad.company.corp             # Secondary DC (HCM)
└── dc03.ad.company.corp             # DC (HN site)

Organizational Units:
ad.company.corp
├── Computers
│   ├── Servers
│   │   ├── Production
│   │   ├── Development
│   │   └── Management
│   └── Workstations
├── Users
│   ├── IT Department
│   ├── Sales Department
│   └── HR Department
└── Groups
    ├── Domain Admins
    ├── vCenter Admins
    └── Application Users
```

**DNS Records cần thiết:**
```dns
; Active Directory DNS Zone: ad.company.corp

; Domain Controllers
dc01.ad.company.corp.           A       10.10.10.50
dc02.ad.company.corp.           A       10.10.10.51
dc03.ad.company.corp.           A       10.20.10.50

; Domain Root (khi user login @ad.company.corp)
ad.company.corp.                A       10.10.10.50
ad.company.corp.                A       10.10.10.51

; LDAP Service Records (tự động tạo bởi AD)
_ldap._tcp.ad.company.corp.     SRV     0 100 389 dc01.ad.company.corp.
_ldap._tcp.ad.company.corp.     SRV     0 100 389 dc02.ad.company.corp.

; Kerberos Service Records
_kerberos._tcp.ad.company.corp. SRV     0 100 88  dc01.ad.company.corp.
_kerberos._udp.ad.company.corp. SRV     0 100 88  dc01.ad.company.corp.

; Global Catalog
_gc._tcp.ad.company.corp.       SRV     0 100 3268 dc01.ad.company.corp.

; Reverse DNS (quan trọng cho AD)
50.10.10.10.in-addr.arpa.       PTR     dc01.ad.company.corp.
51.10.10.10.in-addr.arpa.       PTR     dc02.ad.company.corp.
```

**Tích hợp vCenter với Active Directory:**
```yaml
vCenter Configuration:
  Identity Source: Active Directory
  Domain: ad.company.corp
  LDAP Server: ldaps://dc01.ad.company.corp:636
  Base DN: dc=ad,dc=company,dc=corp
  User Base DN: ou=Users,dc=ad,dc=company,dc=corp
  Group Base DN: ou=Groups,dc=ad,dc=company,dc=corp

vCenter Permissions:
  - Group: vCenter Admins@ad.company.corp
    Role: Administrator
    Scope: Root vCenter
    
  - Group: vCenter ViewOnly@ad.company.corp
    Role: Read-Only
    Scope: Root vCenter
    
  - Group: vCenter NetworkAdmins@ad.company.corp
    Role: Network Administrator
    Scope: Networking folder
```

---

#### C. Application Domain

**Mục đích:** Internal applications, APIs, databases

**TLD:** `.app.company.internal` hoặc sử dụng chung `.prod.company.internal`

**Cấu trúc:**
```yaml
app.company.internal
├── web.app.company.internal              # Web servers
│   ├── web01.app.company.internal        → 10.10.100.51
│   ├── web02.app.company.internal        → 10.10.100.52
│   └── web-lb.app.company.internal       → 10.10.100.10 (Load Balancer VIP)
│
├── api.app.company.internal              # API servers
│   ├── api01.app.company.internal        → 10.10.110.51
│   ├── api02.app.company.internal        → 10.10.110.52
│   └── api-lb.app.company.internal       → 10.10.110.10 (VIP)
│
├── db.app.company.internal               # Databases
│   ├── db-master.app.company.internal    → 10.10.120.51
│   ├── db-slave.app.company.internal     → 10.10.120.52
│   ├── postgres.app.company.internal     → 10.10.120.51 (CNAME)
│   └── mysql.app.company.internal        → 10.10.120.53 (CNAME)
│
└── cache.app.company.internal            # Cache servers
    ├── redis01.cache.app.company.internal → 10.10.110.60
    └── redis02.cache.app.company.internal → 10.10.110.61
```

---

### 2.2. External/Public Domain

#### A. Public Website Domain

**Mục đích:** Customer-facing websites, landing pages

**TLD:** `.com`, `.vn`, `.com.vn`

**Cấu trúc:**
```yaml
company.com                           # Main website
├── www.company.com                   # Website (CNAME to CDN or A to LB)
├── blog.company.com                  # Blog subdomain
├── shop.company.com                  # E-commerce
└── support.company.com               # Help center

DNS Records (Public DNS - Cloudflare/Route53):
www.company.com.        A       203.0.113.10    # Public IP của Load Balancer
www.company.com.        A       203.0.113.11    # Secondary LB
blog.company.com.       CNAME   company.cdn.cloudflare.net.
shop.company.com.       A       203.0.113.20

# Hoặc sử dụng CDN
www.company.com.        CNAME   d123456.cloudfront.net.
```

#### B. API Public Domain

**Mục đích:** Public APIs, webhooks, integrations

**TLD:** `.com`, hoặc subdomain riêng

**Cấu trúc:**
```yaml
api.company.com                       # Public API endpoint
├── api-v1.company.com                # API version 1 (deprecated)
├── api-v2.company.com                # API version 2 (current)
└── webhooks.company.com              # Webhook endpoint

DNS Records:
api.company.com.        A       203.0.113.30    # API Gateway Public IP
api.company.com.        A       203.0.113.31    # Secondary
api-v2.company.com.     CNAME   api.company.com.

# Với API Gateway (AWS API Gateway example)
api.company.com.        CNAME   abcd1234.execute-api.ap-southeast-1.amazonaws.com.
```

#### C. Email Domain

**Mục đích:** Corporate email, notifications

**TLD:** Sử dụng chung với website `.com`

**DNS Records:**
```dns
; MX Records (Mail Exchange)
company.com.            MX      10 mx1.emailprovider.com.
company.com.            MX      20 mx2.emailprovider.com.

; SPF Record (Sender Policy Framework)
company.com.            TXT     "v=spf1 include:_spf.emailprovider.com ~all"

; DKIM Record (DomainKeys Identified Mail)
default._domainkey.company.com. TXT "v=DKIM1; k=rsa; p=MIGfMA0GCS..."

; DMARC Record (Domain-based Message Authentication)
_dmarc.company.com.     TXT     "v=DMARC1; p=quarantine; rua=mailto:dmarc@company.com"

; Autodiscover (for Outlook/Exchange)
autodiscover.company.com.  CNAME   autodiscover.emailprovider.com.

; Mail server (if self-hosted)
mail.company.com.       A       203.0.113.40
```

---

### 2.3. Hybrid Domain (Internal + External)

#### A. VPN/Remote Access Domain

**Mục đích:** VPN portal, remote desktop gateway

**Cấu trúc:**
```yaml
# Public DNS
vpn.company.com.                A       203.0.113.50    # VPN Gateway Public IP
remote.company.com.             A       203.0.113.51    # Remote Desktop Gateway

# Internal DNS (split-horizon DNS)
vpn.company.internal            A       10.10.90.60     # Internal VPN server
rdp-gateway.company.internal    A       10.10.90.61     # RDP Gateway
```

#### B. Monitoring/Management Access

**Cấu trúc:**
```yaml
# Public (với VPN hoặc IP whitelist)
vcenter.company.com             A       203.0.113.60    # NAT đến 10.10.10.10
monitoring.company.com          A       203.0.113.61    # NAT đến 10.10.130.51

# Internal (truy cập trực tiếp)
vcenter.prod.company.internal   A       10.10.10.10
monitoring.mgmt.company.internal A      10.10.130.51

# Best Practice: Chỉ expose qua VPN, không public trực tiếp
```

---

<a name="cau-truc-dns"></a>
## 3. CẤU TRÚC DNS RECORDS

### 3.1. Record Types và Mục đích

#### A Record (Address Record)
```dns
; Map hostname to IPv4 address
; Format: <hostname> A <IPv4>

vcenter.prod.company.internal.      A       10.10.10.10
esxi01.prod.company.internal.       A       10.10.10.21
web01.app.company.internal.         A       10.10.100.51
```

**Khi nào dùng:** 
- ✅ Tất cả các servers, VMs có IP tĩnh
- ✅ Infrastructure devices (ESXi, vCenter, Storage)
- ✅ Load Balancer VIPs

**Quy tắc:**
```yaml
Rule 1: Mỗi IP quan trọng PHẢI có A record
Rule 2: Mỗi A record PHẢI có corresponding PTR record (reverse DNS)
Rule 3: Hostname PHẢI match với actual server hostname (để reverse lookup work)
```

---

#### AAAA Record (IPv6 Address)
```dns
; Map hostname to IPv6 address
; Format: <hostname> AAAA <IPv6>

vcenter.prod.company.internal.      AAAA    2001:db8::10
web01.app.company.internal.         AAAA    2001:db8:100::51
```

**Khi nào dùng:**
- ⚠️ Nếu có IPv6 deployment
- ⚠️ Dual-stack environment (IPv4 + IPv6)

**Lưu ý:** 
- Hầu hết VMware environments chỉ dùng IPv4
- Chỉ implement IPv6 nếu có yêu cầu cụ thể

---

#### CNAME Record (Canonical Name)
```dns
; Alias một hostname đến hostname khác
; Format: <alias> CNAME <canonical-name>

; Service aliases
vcenter.company.internal.           CNAME   hcm-prod-vc-01.prod.company.internal.
postgres.app.company.internal.      CNAME   db-master.app.company.internal.
redis.app.company.internal.         CNAME   redis01.cache.app.company.internal.

; Environment aliases
prod-web.company.internal.          CNAME   web01.app.company.internal.
staging-web.company.internal.       CNAME   web-staging-01.dev.company.internal.

; Load balancer aliases
www.company.internal.               CNAME   web-lb.app.company.internal.
api.company.internal.               CNAME   api-lb.app.company.internal.
```

**Khi nào dùng:**
- ✅ Alias để dễ nhớ (vcenter thay vì hcm-prod-vc-01)
- ✅ Service abstraction (postgres → db-master)
- ✅ Failover flexibility (đổi CNAME thay vì đổi nhiều A records)
- ✅ Load balancer VIPs

**Quy tắc:**
```yaml
Rule 1: CNAME KHÔNG thể trỏ đến IP, chỉ trỏ đến hostname khác
Rule 2: CNAME record KHÔNG thể coexist với record khác cùng tên
Rule 3: Root domain (@) KHÔNG thể là CNAME (phải dùng A record)
Rule 4: MX và NS records KHÔNG thể trỏ đến CNAME
```

**Ví dụ SAI:**
```dns
❌ vcenter.company.internal.  CNAME   10.10.10.10   # CNAME không trỏ IP
❌ company.internal.          CNAME   web01.app.company.internal.  # Root không thể CNAME
```

**Ví dụ ĐÚNG:**
```dns
✅ vcenter.company.internal.  CNAME   hcm-prod-vc-01.prod.company.internal.
✅ www.company.internal.      CNAME   web-lb.app.company.internal.
```

---

#### PTR Record (Pointer Record - Reverse DNS)
```dns
; Map IP address to hostname (reverse lookup)
; Format: <reversed-IP>.in-addr.arpa PTR <hostname>

; Forward DNS
vcenter.prod.company.internal.      A       10.10.10.10

; Reverse DNS (MUST MATCH)
10.10.10.10.in-addr.arpa.           PTR     vcenter.prod.company.internal.

; More examples
21.10.10.10.in-addr.arpa.           PTR     esxi01.prod.company.internal.
51.100.10.10.in-addr.arpa.          PTR     web01.app.company.internal.
51.120.10.10.in-addr.arpa.          PTR     db-master.app.company.internal.
```

**Khi nào dùng:**
- ✅ **BẮT BUỘC** cho Active Directory servers
- ✅ **BẮT BUỘC** cho Mail servers (chống spam)
- ✅ **KHUYÊN DÙNG** cho tất cả infrastructure servers
- ✅ Troubleshooting network issues

**Quy tắc:**
```yaml
Rule 1: Mỗi A record quan trọng PHẢI có PTR record tương ứng
Rule 2: PTR record PHẢI trỏ đến FQDN (có dấu chấm cuối)
Rule 3: Hostname trong PTR PHẢI match với server's actual hostname
Rule 4: Reverse zone phải được tạo (10.10.10.in-addr.arpa zone)
```

**Kiểm tra:**
```bash
# Forward lookup
dig vcenter.prod.company.internal
# Kết quả: 10.10.10.10

# Reverse lookup
dig -x 10.10.10.10
# Kết quả: vcenter.prod.company.internal

# Cả hai PHẢI consistent
```

---

#### SRV Record (Service Record)
```dns
; Define location of services
; Format: _service._proto.domain TTL class SRV priority weight port target

; Active Directory LDAP
_ldap._tcp.ad.company.corp.         SRV     0 100 389  dc01.ad.company.corp.
_ldap._tcp.ad.company.corp.         SRV     0 100 389  dc02.ad.company.corp.

; Kerberos
_kerberos._tcp.ad.company.corp.     SRV     0 100 88   dc01.ad.company.corp.
_kerberos._udp.ad.company.corp.     SRV     0 100 88   dc01.ad.company.corp.

; Global Catalog
_gc._tcp.ad.company.corp.           SRV     0 100 3268 dc01.ad.company.corp.

; SIP (VoIP example)
_sip._tcp.company.com.              SRV     10 60 5060 sipserver1.company.com.
_sip._tcp.company.com.              SRV     10 40 5060 sipserver2.company.com.

; XMPP (Jabber)
_xmpp-client._tcp.company.com.      SRV     0 5 5222  xmpp.company.com.
```

**Khi nào dùng:**
- ✅ **BẮT BUỘC** cho Active Directory
- ✅ SIP/VoIP services
- ✅ XMPP/Jabber chat services
- ✅ Auto-discovery services (Outlook, Skype)

**Format giải thích:**
```
_service._protocol.domain  SRV  priority weight port target
    │         │       │           │        │     │     │
    │         │       │           │        │     │     └── Hostname của server
    │         │       │           │        │     └──────── Port number
    │         │       │           │        └────────────── Weight (load balancing)
    │         │       │           └─────────────────────── Priority (thấp = ưu tiên)
    │         │       └─────────────────────────────────── Domain name
    │         └─────────────────────────────────────────── Protocol (tcp/udp)
    └───────────────────────────────────────────────────── Service name
```

**Quy tắc:**
```yaml
Rule 1: Service name PHẢI bắt đầu với underscore (_ldap, _kerberos)
Rule 2: Protocol PHẢI là _tcp hoặc _udp
Rule 3: Priority thấp hơn = ưu tiên cao hơn (0 = cao nhất)
Rule 4: Weight dùng để load balancing giữa các servers cùng priority
```

---

#### MX Record (Mail Exchange)
```dns
; Define mail servers for domain
; Format: domain MX priority mail-server

company.com.                        MX      10 mx1.emailprovider.com.
company.com.                        MX      20 mx2.emailprovider.com.
company.com.                        MX      30 mx3.emailprovider.com.

; Internal mail
company.internal.                   MX      10 mail.mgmt.company.internal.
```

**Khi nào dùng:**
- ✅ Email services
- ✅ Mỗi domain cần email PHẢI có MX record

**Quy tắc:**
```yaml
Rule 1: Priority thấp = ưu tiên cao (10 trước 20)
Rule 2: MX KHÔNG thể trỏ đến CNAME, phải trỏ A record
Rule 3: Mail server PHẢI có valid PTR record (reverse DNS)
Rule 4: Nên có ít nhất 2 MX records (redundancy)
```

---

#### TXT Record (Text Record)
```dns
; Store arbitrary text data
; Common uses: SPF, DKIM, DMARC, verification

; SPF (Sender Policy Framework)
company.com.            TXT     "v=spf1 ip4:203.0.113.0/24 include:_spf.google.com ~all"

; DKIM (DomainKeys Identified Mail)
default._domainkey.company.com. TXT "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4G..."

; DMARC (Domain-based Message Authentication)
_dmarc.company.com.     TXT     "v=DMARC1; p=quarantine; rua=mailto:dmarc-reports@company.com; pct=100"

; Domain verification (Google, Microsoft)
company.com.            TXT     "google-site-verification=rXOxyZounnZasA8Z7oaD3c14JdjS9aKSWvsR1EbUSIQ"
company.com.            TXT     "MS=ms12345678"

; Custom metadata
_infrastructure.company.internal. TXT "contact=ops@company.com; owner=IT Department"
```

**Khi nào dùng:**
- ✅ Email authentication (SPF, DKIM, DMARC)
- ✅ Domain verification (Google Workspace, Microsoft 365)
- ✅ SSL certificate validation (CAA records)
- ✅ Custom metadata

---

#### NS Record (Name Server)
```dns
; Delegate subdomain to different nameservers
; Format: subdomain NS nameserver

company.internal.                   NS      dns1.company.internal.
company.internal.                   NS      dns2.company.internal.

; Subdomain delegation
aws.company.internal.               NS      ns1.awsdns-cloud.com.
azure.company.internal.             NS      ns1-01.azure-dns.com.

; Glue records (nếu NS trong cùng zone)
dns1.company.internal.              A       10.10.10.51
dns2.company.internal.              A       10.10.10.52
```

**Khi nào dùng:**
- ✅ Khi tạo DNS zone mới
- ✅ Delegate subdomain to different DNS servers
- ✅ Hybrid cloud (AWS Route53, Azure DNS)

---

#### CAA Record (Certification Authority Authorization)
```dns
; Specify which CAs can issue certificates
; Format: domain CAA flags tag value

company.com.            CAA     0 issue "letsencrypt.org"
company.com.            CAA     0 issue "digicert.com"
company.com.            CAA     0 issuewild "letsencrypt.org"
company.com.            CAA     0 iodef "mailto:security@company.com"
```

**Khi nào dùng:**
- ✅ Security best practice (prevent unauthorized SSL certs)
- ✅ Compliance requirements (PCI-DSS)

---

### 3.2. DNS Record Examples - Production Environment

#### Complete DNS Zone File Example

```dns
; Zone file for: company.internal
; File: /var/named/company.internal.zone
; Type: Master zone
; Updated: 2025-10-15

$TTL 86400      ; 24 hours default TTL
@       IN      SOA     dns1.company.internal. admin.company.internal. (
                        2025101501      ; Serial (YYYYMMDDnn)
                        3600            ; Refresh (1 hour)
                        1800            ; Retry (30 minutes)
                        604800          ; Expire (7 days)
                        86400 )         ; Minimum TTL (24 hours)

; Name Servers
@                       IN      NS      dns1.company.internal.
@                       IN      NS      dns2.company.internal.

; DNS Servers themselves
dns1                    IN      A       10.10.10.51
dns2                    IN      A       10.10.10.52

; ============================================================
; INFRASTRUCTURE - Management VLAN (10.10.10.0/24)
; ============================================================

; Gateways
gateway                 IN      A       10.10.10.1
gw-secondary            IN      A       10.10.10.2

; vCenter Servers
vcenter                 IN      CNAME   hcm-prod-vc-01.prod.company.internal.
hcm-prod-vc-01.prod     IN      A       10.10.10.10
hn-prod-vc-01.prod      IN      A       10.20.10.10

; ESXi Hosts - HCM Site
hcm-prod-esxi-01.prod   IN      A       10.10.10.21
hcm-prod-esxi-02.prod   IN      A       10.10.10.22
hcm-prod-esxi-03.prod   IN      A       10.10.10.23
hcm-prod-esxi-04.prod   IN      A       10.10.10.24

; ESXi Hosts - HN Site
hn-prod-esxi-01.prod    IN      A       10.20.10.21
hn-prod-esxi-02.prod    IN      A       10.20.10.22

; ESXi Aliases (short names)
esxi01                  IN      CNAME   hcm-prod-esxi-01.prod.company.internal.
esxi02                  IN      CNAME   hcm-prod-esxi-02.prod.company.internal.

; NSX
hcm-prod-nsx-mgr.prod   IN      A       10.10.10.12
hcm-prod-nsx-ctrl-01.prod IN    A       10.10.10.13
hcm-prod-nsx-ctrl-02.prod IN    A       10.10.10.14

; ============================================================
; STORAGE - iSCSI VLAN (10.10.30.0/24)
; ============================================================
hcm-prod-stor-iscsi-01.storage  IN  A   10.10.30.100
hcm-prod-stor-iscsi-02.storage  IN  A   10.10.30.101
storage                         IN  CNAME hcm-prod-stor-iscsi-01.storage.company.internal.

; ============================================================
; APPLICATION SERVERS
; ============================================================

; Load Balancers (VIPs)
web-lb.app              IN
