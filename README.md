# KCB-BANK-KENYA
Enterprise VLAN
# KCB Bank Kenya — Westlands Branch
## Network Security Hardening & Infrastructure Mitigation Project

###  Project Overview
* **Client:** KCB Bank Kenya
* **Branch:** Westlands Branch, Nairobi
* **Status:** Fully Remediated & Audited
* **Deployment Horizon:** 48 Hours (Emergency Response)

###  Scenario & Audit Failure Context
Following a critical security audit failure, the Westlands branch was flagged for running a highly vulnerable **"flat network"** topology. The core vulnerabilities identified were:
1. **Zero Segmentation:** Tellers, management, and backend servers coexisted on the same broadcast domain, allowing unauthorized internal lateral movement.
2. **Rogue Access Risks:** Public Guest WiFi traffic was uninsulated, permitting guests to sniff packet headers and directly ping internal core banking infrastructure.
3. **Lack of Edge Hardening:** Unused physical switchports were left active on default VLAN 1, creating a massive insider-threat vector for rogue device cross-connections.

**Objective:** Design, simulate, and implement a complete Layer 2 and Layer 3 security mitigation strategy within a strict 48-hour window using Cisco architecture.

---

##  Architecture & Network Design

### Subnetting & VLAN Design Matrix
The network was re-engineered utilizing Variable Length Subnet Masking (VLSM) to achieve strict department isolation and efficient address allocation, completely eliminating the use of the default VLAN 1 for data traffic.

| VLAN ID | Department/Purpose | Network Subnet | Subnet Mask | Assignable Range | Role / Gateway |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VLAN 10** | TELLERS | `192.168.10.0/25` | `255.255.255.128` | `.1` to `.126` | Host Access (120 Hosts) |
| **VLAN 20** | MANAGEMENT | `192.168.20.0/27` | `255.255.255.224` | `.1` to `.30` | Executive Access (30 Hosts) |
| **VLAN 30** | SERVERS (DMZ) | `192.168.30.0/28` | `255.255.255.240` | `.1` to `.14` | Core DB Room (14 Hosts) |
| **VLAN 40** | VOICE (VoIP) | `192.168.40.0/27` | `255.255.255.224` | `.1` to `.30` | Cisco IP Phones (30 Devices) |
| **VLAN 50** | GUEST WiFi | `192.168.50.0/24` | `255.255.255.0` | `.1` to `.254` | Public Internet Only (200 Hosts) |
| **VLAN 99** | NETMGMT | `192.168.99.0/29` | `255.255.255.248` | `.1` to `.6` | Native Trunk & Admin Transit |
| **VLAN 999**| UNUSED | N/A | N/A | N/A | Administrative Black-Hole |

---

## 🔒 Security Implementations

### 1. Perimeter Access Control Lists (ACLs)
Strict extended firewall filtering rules were bound to the sub-interface gateways on the perimeter router (`KCB-RTR`) to govern inter-VLAN transit traffic:

* **Guest Isolation Rule:** The `GUEST_ACL` explicitly drops any packet sourced from the guest subnet (`192.168.50.0/24`) destined for *any* internal banking subnet, preserving complete quarantine while allowing outbound public web surfing.
* **Teller Protocol Restriction:** Tellers are restricted via `TELLER_ACL`. They are granted exclusive access to the Core Database Server (`192.168.30.10`) strictly over secure web layer protocols (**HTTPS / TCP Port 443**). Standard infrastructure tracking tools, such as ICMP Pings, are dropped at the gateway to prevent reconnaissance.

### 2. Layer-2 Infrastructure Hardening
* **Switchport Security:** All access interfaces on `KCB-ACC1` and `KCB-ACC2` utilize MAC Address tracking filters. If an unauthorized rogue hardware MAC address is detected on an interface, the engine triggers a security violation and drops the interface state to `err-disabled`.
* **DHCP Snooping:** Activated globally across all operational VLANs to establish a security boundary between trusted server-facing ports and untrusted user ports, completely neutralizing rogue DHCP spoofing attacks.
* **Port Containment:** Every single unused physical interface on the access layers was administratively forced down using the `shutdown` command and assigned to the dead-end **VLAN 999 Black-Hole**.

---

##  Verification Evidence & Audit Logs

### Execution Screenshots

#### 1. Hierarchical Topology Implementation
KCB NETWORK TOPOLOGY.png
> **Verification Note:** Displays the structural layout consisting of the `KCB-RTR` edge routing platform, the high-performance `KCB-CORE` multi-layer hub, and the downstream client access switches.

#### 2. Dynamic Addressing Matrix (DHCP)
*[Insert screenshot of TELLER-PC or MGMT-PC Desktop IP configuration]*
> **Verification Note:** Confirms that endpoints successfully escape default autoconfiguration and lease valid, structured network parameters across independent broadcast domains.

#### 3. Perimeter Firewall Restrictions (ACL Tests)
*[Insert screenshot of the Teller browser loading the secure page, alongside a failed ping window]*
> **Verification Note:** Hard evidence of protocol compliance. The Teller node successfully communicates over HTTPS to the secure database web application, while a simultaneous ICMP Ping request is explicitly dropped by the router's firewall filter policy.

#### 4. Guest WiFi Quarantine Isolation Audit
*[Insert screenshot of GUEST-PC command prompt trying to ping internal resources]*
> **Verification Note:** Proves complete environment isolation. All attempts by guest assets to poll internal infrastructure segments return an immediate block message.

#### 5. Edge Translation Map (Static NAT)
*[Insert screenshot of the KCB-RTR terminal running: show ip nat translations]*
> **Verification Note:** Displays the live network address translation table mapping the private DMZ address (`192.168.30.14`) securely to the public-facing edge WAN block (`196.201.45.3`).

---

##  Project Conclusion
The emergency security intervention successfully transformed the vulnerable branch environment into an enterprise-grade, hardened infrastructure block. By mapping deterministic subnets, defining clean Layer-4 access control list policies, and running strict switchport boundary protections, the **KCB Westlands Branch** complies with strict banking infrastructure compliance requirements.
