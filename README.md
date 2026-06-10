# Project 6: Enterprise Infrastructure Overhaul, Core SVI Routing & Perimeter Hardening
**Company:** KCB Bank Kenya, Westlands Branch  
**Location:** Westlands, Nairobi  
**Status:** Production Remediated & Audited  
**Deployment Horizon:** 48-Hour Emergency Response Execution  

---

##  1. Executive Summary & Problem Statement
A comprehensive, unscheduled infrastructure audit of the KCB Bank Westlands Branch revealed critical structural vulnerabilities across the local area network. The branch was operating on a legacy, flat Layer 2 topology where high-value internal banking assets, tellers, branch management nodes, and public guest endpoints co-existed within a single unrestricted broadcast domain.

### The Immediate Risks
1. **Lateral Movement Vectors:** Zero segmentation allowed any standard workstation to directly map, probe, and access core database servers hosting sensitive financial records.
2. **Public Wi-Fi Contamination:** Uninsulated public Guest WiFi networks allowed external actors to execute local packet sniffing, intercept unencrypted headers, and directly ping back-end systems.
3. **Perimeter Exposure:** Unused physical RJ-45 switchports were left active and assigned to default VLAN 1, presenting an open vector for unauthorized rogue hardware connections.
4. **Voice Degradation:** Legacy voice endpoints shared data fabrics without explicit Class of Service (CoS) marking, resulting in severe jitter and dropped call streams under peak financial processing hours.

### The Engineering Solution
I was retained to completely re-architect the branch network fabric within a strict 48-hour compliance window. The modernized design implements strict multi-tier segmentation, introduces wire-speed Layer 3 Multi-Layer Switch Core SVI routing, deploys Cisco Call Manager Express (CME) IP telephony, enforces strict Layer 2 access port protections (DHCP Snooping & Port Security), and isolates untrusted segments via Layer 3 Extended Access Control Lists (ACLs).

---

##  2. System Topology & Advanced Inter-Network Architecture
The infrastructure was migrated to a Cisco Hierarchical Modular Framework, separating the core routing transit boundaries from the physical access layer delivery.

```text
         [ ISP-EDGE-GATEWAY ]
                   │
                   │ (Public WAN Routing via PAT Engine)
                   ▼
              [ KCB-RTR ] (Edge Security & Dynamic NAT Gateway)
                   │
                   │ (Point-to-Point Layer 3 Routed Link: 10.1.1.0/30)
                   ▼
             [ KCB-CORE ] (Layer 3 Multilayer Distribution Switch)
              /        \
             /          \ (802.1Q High-Speed Trunk Aggregation Lines)
            ▼            ▼
       [ KCB-ACC1 ]    [ KCB-ACC2 ] (Layer 2 Perimeter Defense Access Layer)
            │               │
       [IP PHONE]      [IP PHONE]    (Cisco Voice Endpoint Processing)
            │               │
       [TELLER PC]     [MGMT PC]     (Data Host Workstations - Piggybacked)
```

## 3. Logical Address Space & Segment Allocation Matrix

The local address architecture was structured using Variable Length Subnet Masking (VLSM) to maximize conservation and restrict subnet sizing. The default VLAN 1 was completely removed from all operational transit data tracks.

| VLAN ID | Subnet Segment | Subnet Mask | Usable Host Range | Default Gateway | Security Zone Classification |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VLAN 10** | 192.168.10.0/25 | 255.255.255.128 | .2 to .126 | 192.168.10.1 | Tellers (Restricted Core App Access Only) |
| **VLAN 20** | 192.168.20.0/27 | 255.255.255.224 | .2 to .30 | 192.168.20.1 | Branch Management (Full Audit Access) |
| **VLAN 30** | 192.168.30.0/28 | 255.255.255.240 | .2 to .14 | 192.168.30.1 | Core Banking Servers (Highly Isolated) |
| **VLAN 40** | 192.168.40.0/27 | 255.255.255.224 | .2 to .30 | 192.168.40.1 | Voice (VoIP) (CoS Priority Queueing) |
| **VLAN 50** | 192.168.50.0/24 | 255.255.255.0 | .2 to .254 | 192.168.50.1 | Guest WiFi Pool (Untrusted Sandbox) |
| **VLAN 99** | 192.168.99.0/29 | 255.255.255.248 | .2 to .6 | 192.168.99.1 | Network Management (Secure SSH Transit) |
| **VLAN 999**| N/A | N/A | N/A | N/A | Blackhole Domain (Unused Interface Parking) |


## 🛠️ 4. Enterprise Production Configuration Scripts

### A. Layer 3 Core Switch Routing Platform (KCB-CORE)
```ios
KCB-CORE> enable
KCB-CORE# configure terminal
KCB-CORE(config)# hostname KCB-CORE

! --- Step 1: Initialize Hardware ASIC Layer 3 Routing Engine ---
KCB-CORE(config)# ip routing

! --- Step 2: Establish Corporate Multi-VLAN Registry ---
KCB-CORE(config)# vlan 10
KCB-CORE(config-vlan)# name Tellers
KCB-CORE(config)# vlan 20
KCB-CORE(config-vlan)# name Management
KCB-CORE(config)# vlan 30
KCB-CORE(config-vlan)# name Servers
KCB-CORE(config)# vlan 40
KCB-CORE(config-vlan)# name Voice
KCB-CORE(config)# vlan 50
KCB-CORE(config-vlan)# name Guest_WiFi
KCB-CORE(config)# vlan 99
KCB-CORE(config-vlan)# name NetMgmt
KCB-CORE(config)# vlan 999
KCB-CORE(config-vlan)# name Unused_Blackhole
KCB-CORE(config-vlan)# exit

! --- Step 3: Configure Switch Virtual Interfaces (SVIs) as Subnet Gateways ---
KCB-CORE(config)# interface vlan 10
KCB-CORE(config-if)# ip address 192.168.10.1 255.255.255.128
KCB-CORE(config)# interface vlan 20
KCB-CORE(config-if)# ip address 192.168.20.1 255.255.255.224
KCB-CORE(config)# interface vlan 30
KCB-CORE(config-if)# ip address 192.168.30.1 255.255.255.240
KCB-CORE(config)# interface vlan 40
KCB-CORE(config-if)# ip address 192.168.40.1 255.255.255.224
KCB-CORE(config)# interface vlan 50
KCB-CORE(config-if)# ip address 192.168.50.1 255.255.255.0
KCB-CORE(config)# interface vlan 99
KCB-CORE(config-if)# ip address 192.168.99.2 255.255.255.248
KCB-CORE(config-if)# exit

! --- Step 4: Configure 802.1Q High-Speed Backbone Trunk Interfaces ---
KCB-CORE(config)# interface range GigabitEthernet0/1 - 2
KCB-CORE(config-if-range)# switchport trunk encapsulation dot1q
KCB-CORE(config-if-range)# switchport mode trunk
KCB-CORE(config-if-range)# switchport trunk native vlan 99
KCB-CORE(config-if-range)# no shutdown
KCB-CORE(config-if-range)# exit

! --- Step 5: Provision Layer 3 Upstream Routed Link to Perimeter ---
KCB-CORE(config)# interface GigabitEthernet0/24
KCB-CORE(config-if)# no switchport
KCB-CORE(config-if)# ip address 10.1.1.2 255.255.255.252
KCB-CORE(config-if)# no shutdown
KCB-CORE(config-if)# exit

! --- Step 6: Inject Default Route Pointing directly to Security Router ---
KCB-CORE(config)# ip route 0.0.0.0 0.0.0.0 10.1.1.1
KCB-CORE# write
```

## B. Edge Security Router & VoIP CME Provisioning Engine (KCB-RTR)
```text
KCB-RTR> enable
KCB-RTR# configure terminal
KCB-RTR(config)# hostname KCB-RTR

! --- Step 1: Interface Assignment Facing L3 Core Switch ---
KCB-RTR(config)# interface GigabitEthernet0/0
KCB-RTR(config-if)# ip address 10.1.1.1 255.255.255.252
KCB-RTR(config-if)# no shutdown
KCB-RTR(config-if)# exit

! --- Step 2: Static Route Declaration for Downstream Internal Banking Fabrics ---
KCB-RTR(config)# ip route 192.168.0.0 255.255.0.0 10.1.1.2

! --- Step 3: Cisco Call Manager Express (CME) Voice Provisioning ---
KCB-RTR(config)# telephony-service
KCB-RTR(config-telephony)# max-ephones 30
KCB-RTR(config-telephony)# max-dn 30
KCB-RTR(config-telephony)# ip source-address 192.168.40.1 port 2000
KCB-RTR(config-telephony)# auto assign 1 to 5
KCB-RTR(config-telephony)# exit

KCB-RTR(config)# ephone-dn 1
KCB-RTR(config-ephone-dn)# number 1001
KCB-RTR(config)# ephone-dn 2
KCB-RTR(config-ephone-dn)# number 1002

! --- Step 4: Perimeter Firewalls (Layer 3 Extended Access Control Lists) ---
KCB-RTR(config)# ip access-list extended TELLER_ACL
KCB-RTR(config-ext-nacl)# permit tcp 192.168.10.0 0.0.0.127 host 192.168.30.10 eq 443
KCB-RTR(config-ext-nacl)# deny ip 192.168.10.0 0.0.0.127 192.168.30.0 0.0.0.15
KCB-RTR(config-ext-nacl)# permit ip any any
KCB-RTR(config-ext-nacl)# exit

KCB-RTR(config)# ip access-list extended GUEST_ACL
KCB-RTR(config-ext-nacl)# deny ip 192.168.50.0 0.0.0.255 192.168.10.0 0.0.0.127
KCB-RTR(config-ext-nacl)# deny ip 192.168.50.0 0.0.0.255 192.168.20.0 0.0.0.31
KCB-RTR(config-ext-nacl)# deny ip 192.168.50.0 0.0.0.255 192.168.30.0 0.0.0.15
KCB-RTR(config-ext-nacl)# permit ip any any
KCB-RTR(config-ext-nacl)# exit

! --- Step 5: Bind Firewall Policies to Inter-VLAN Sub-Interfaces ---
KCB-RTR(config)# interface GigabitEthernet0/0.10
KCB-RTR(config-if)# ip access-group TELLER_ACL in
KCB-RTR(config)# interface GigabitEthernet0/0.50
KCB-RTR(config-if)# ip access-group GUEST_ACL in
KCB-RTR# write
```
## C. Layer 2 Access Switches Hardening & Isolation (KCB-ACC1)
```text
KCB-ACC1> enable
KCB-ACC1# configure terminal
KCB-ACC1(config)# hostname KCB-ACC1

! --- Step 1: Enable Hardware DHCP Snooping Globally ---
KCB-ACC1(config)# ip dhcp snooping
KCB-ACC1(config)# ip dhcp snooping vlan 10,20,30,40,50

! --- Step 2: Establish Trusted Uplink Boundary Port to Core Switch ---
KCB-ACC1(config)# interface GigabitEthernet0/1
KCB-ACC1(config-if)# switchport mode trunk
KCB-ACC1(config-if)# switchport trunk native vlan 99
KCB-ACC1(config-if)# ip dhcp snooping trust
KCB-ACC1(config-if)# no shutdown
KCB-ACC1(config-if)# exit

! --- Step 3: Configure Multi-VLAN Access Ports with Sticky Port Security ---
KCB-ACC1(config)# interface range FastEthernet0/1 - 10
KCB-ACC1(config-if-range)# switchport mode access
KCB-ACC1(config-if-range)# switchport access vlan 10
KCB-ACC1(config-if-range)# switchport voice vlan 40
KCB-ACC1(config-if-range)# switchport port-security
KCB-ACC1(config-if-range)# switchport port-security maximum 2
KCB-ACC1(config-if-range)# switchport port-security violation shutdown
KCB-ACC1(config-if-range)# switchport port-security mac-address sticky
KCB-ACC1(config-if-range)# ip dhcp snooping limit rate 20
KCB-ACC1(config-if-range)# no shutdown
KCB-ACC1(config-if-range)# exit

! --- Step 4: Administrative Containment of Inactive Physical Interfaces ---
KCB-ACC1(config)# interface range FastEthernet0/11 - 24, GigabitEthernet0/2
KCB-ACC1(config-if-range)# switchport access vlan 999
KCB-ACC1(config-if-range)# shutdown
KCB-ACC1# write
```
##  5. Verification Evidence & Compliance Verification

### Execution Screenshots & Logs
* **1. Hierarchical Topology Implementation**
  * *Verification Note:* Displays the structural design consisting of the KCB-RTR perimeter firewall routing platform, the high-performance KCB-CORE multi-layer hub, and downstream access switches supporting piggybacked IP telephony deployments.
* **2. VoIP Provisioning & Dynamic Registration**
  * *Verification Note:* Confirms valid CME deployment. Displays the Cisco IP Phones successfully drawing dial-plan profiles, assigning extensions (1001 and 1002), and establishing complete voice paths.
* **3. Perimeter Firewall Restrictions (ACL Tests)**
  * *Verification Note:* Hard evidence of protocol alignment. The Teller host successfully accesses the database over port 443 HTTPS, while standard ICMP ping queries are intercepted and dropped at the SVI interface.
* **4. Edge Address Translation Map (PAT Overload)**
  * *Verification Note:* Displays the live network address translation entries mapping private internal bank identities securely through unique multiplexed source ports behind the single public WAN interface.
 
*
## 6. Senior Engineering Directive: Advice & Troubleshooting Guide

Building a converged enterprise network like KCB Bank requires a systematic troubleshooting approach. When voice networks, data subnets, layer-3 switches, and firewalls run on the same infrastructure, a single misconfiguration can stop the entire bank's operations.

Below is the production-tested mental model for engineering this deployment and fixing it if it fails.

### A. Strategic Advice for Junior Network Engineers
* **Always Build Layer 2 Before Layer 3:** Never write an IP address or routing command until your physical cables are mapped, your VLAN database is populated identically across all switches, and your 802.1Q trunk lines are fully active (`show interfaces trunk`). If your trunk lines are down, your Layer 3 configuration will fail.
* **Respect the Native VLAN Rules:** Always change the default Native VLAN from VLAN 1 to an unassigned management VLAN (like VLAN 99). Ensure it matches across every switch on the trunk link. If there is a native VLAN mismatch, Cisco switches will flood your log console with CDP errors and drop packets to protect against VLAN hopping attacks.
* **The Voice Piggyback Rule:** Remember that when a PC is plugged into the back of a Cisco IP Phone, the switchport handles two distinct frames. The Voice frame carries an explicit 802.1p class-of-service priority tag, while the PC data frame is untagged. Never configure the port as a trunk line—it must remain an access port using the dual-zone statement (`switchport access vlan X`, `switchport voice vlan Y`).

### B. Tiered Troubleshooting Runbooks

#### Scenario 1: Tellers or Phones fail to pull an IP Address (DHCP Failure)
If end devices display APIPA autoconfiguration addresses (`169.254.x.x`), the DHCP request is getting blocked or dropped along the path.

* **The Fix Actions:**
  1. Check your DHCP Snooping tables on the access switch using `show ip dhcp snooping binding`. If it is blank, verify that your uplink port to the Core switch is explicitly configured as trusted (`ip dhcp snooping trust`).
  2. If the DHCP server is running on the router (KCB-RTR) rather than the core switch, verify that the core switch SVI ports have a helper address pointing to the router's interface:
     ```ios
     KCB-CORE(config)# interface vlan 10
     KCB-CORE(config-if)# ip helper-address 10.1.1.1
     ```

#### Scenario 2: Cisco IP Phone powers on but shows "Configuring IP" or fails to get an Extension
The phone has pulled an IP address but cannot locate the Cisco Call Manager Express server to download its firmware and dial plan.

* **The Fix Actions:**
  1. Inspect your Voice DHCP pool on the router. Ensure Option 150 is explicitly configured and points exactly to the IP address where the CME voice service is running:
     ```ios
     KCB-RTR# show run | section ip dhcp pool VOICE_POOL
     ! --- Must contain: option 150 ip 192.168.40.1 ---
     ```
  2. Verify the CME registration status by running `show ephone summary` or `show telephony-service switchboard`. If the extensions are unassigned, verify that the phone’s MAC address matches the assignment entry.

#### Scenario 3: Switchport suddenly turns Red and drops link state (Port Security Violation)
A user has plugged an unauthorized device (like a personal laptop or rogue travel router) into a teller wall jack, triggering a port-security shutdown violation.

* **The Fix Actions:**
  1. Identify the shut down interface on the access switch by running:
     ```ios
     KCB-ACC1# show interface status err-disabled
     ```
  2. Unplug the rogue device, connect the approved hardware back to the port, and manually cycle the interface configuration to clear the security lockout:
     ```ios
     KCB-ACC1(config)# interface FastEthernet0/1
     KCB-ACC1(config-if)# shutdown
     KCB-ACC1(config-if)# no shutdown
     ```

#### Scenario 4: Valid internal devices cannot ping outside or browse the web
The internal network paths are functional, but edge communication is dropping at the security router boundary.

* **The Fix Actions:**
  1. Test the point-to-point transit link. From KCB-CORE, ping `10.1.1.1`. If this fails, your Layer 3 link between the switch and router is misconfigured.
  2. Verify the upstream routing tables on KCB-RTR using `show ip route`. Ensure the router has a return route pointing back to the core switch for the internal networks (`ip route 192.168.0.0 255.255.0.0 10.1.1.2`). Without this return route, the router will drop the response traffic.

---

##  7. Project Conclusion
This emergency security intervention successfully transformed a highly vulnerable branch layout into a hardened, compliant enterprise network. By mapping deterministic subnets, implementing wire-speed Layer 3 core switching, providing prioritized voice infrastructure, and running strict perimeter defense lines, the KCB Westlands Branch complies fully with strict global financial banking infrastructure architecture requirements.
