# Enterprise Network Infrastructure Simulation

A simulated enterprise network built in Cisco Packet Tracer, demonstrating VLAN segmentation, inter-VLAN routing, DHCP, DNS/HTTP services, ACL-based security controls, and firewall placement.

## Objective

Design and configure a multi-department enterprise network with proper security segmentation, mirroring how a small-to-mid-sized organization would structure its internal network across IT, HR, Sales, and shared server resources.

## Topology

```
                [Internet]  (not simulated — out of scope)
                     |
                  [Firewall - ASA 5505]
                     |
                  [Router - 2911]  (router-on-a-stick)
                     |
              [Core Switch]  (trunk links)
        /        |        |        \
 [Switch-IT] [Switch-HR] [Switch-Sales] [Switch-Server]
   /    \      /    \       /    \            |
 [PC1] [PC2] [PC1] [PC2]  [PC1] [PC2]      [Server-99]
```

## IP Addressing & VLAN Plan

| Department | VLAN | Subnet | Gateway (Router) |
|---|---|---|---|
| IT | 10 | 192.168.10.0/24 | 192.168.10.1 |
| HR | 20 | 192.168.20.0/24 | 192.168.20.1 |
| Sales | 30 | 192.168.30.0/24 | 192.168.30.1 |
| Servers | 99 | 192.168.99.0/24 | 192.168.99.1 |
| Router↔Firewall link | — | 192.168.1.0/24 | Router: .2, ASA: .1 |

Server-99 (DNS/HTTP) is statically addressed at `192.168.99.10`. All client PCs use DHCP.

![Physical Topology](screenshots/01-physical-topology.png)
*Figure 1: Full physical topology — firewall, router, core switch, 4 department switches, PCs, and server.*

## What Was Built

### 1. Physical Topology
Router, ASA 5505 firewall, 1 core switch, 4 access switches (IT/HR/Sales/Server), 6 client PCs, and 1 server — cabled and verified link-up across every connection.

### 2. VLAN Segmentation
Each department switch carries its own VLAN (10/20/30/99), with access ports for end devices and a trunk uplink to the Core Switch carrying all VLANs.

### 3. Router-on-a-Stick (Inter-VLAN Routing)
A single physical router interface (Gig0/0) with four 802.1Q sub-interfaces, one per VLAN, providing default gateway services and inter-VLAN routing.

![Router Sub-interfaces](screenshots/04-router-subinterfaces.png)
*Figure 2: All four VLAN sub-interfaces up/up with correct gateway IPs.*

**Verification:** Cross-VLAN pings (e.g., IT → HR, Sales → Server) returned `TTL=127`, confirming traffic passed through exactly one router hop.

![Inter-VLAN ping showing TTL=127](screenshots/05a-interVLAN-ping-ttl127.png)
*Figure 3: PC1-IT pinging its own gateway (TTL=128, local) then the HR gateway (TTL=255, router itself responding).*

![Sales to Server cross-VLAN ping](screenshots/05b-sales-to-server-ping.png)
*Figure 4: PC1-Sales successfully reaching the Server (VLAN 99) — TTL=127 confirms a single router hop.*

### 4. DHCP
Three DHCP pools (IT, HR, Sales) on the router, each excluding the first 9 addresses (reserved for gateways/infrastructure) and pushing the DNS server address (`192.168.99.10`) automatically via the `dns-server` option. Servers remain statically addressed by design.

![DHCP pool configuration](screenshots/06a-dhcp-config-section.png)
*Figure 5: DHCP pool configuration for IT, HR, and Sales — excluded addresses and network scopes.*

![PC receiving DHCP lease](screenshots/06b-dhcp-client-success.png)
*Figure 6: PC1-IT successfully receiving an IP, gateway, and DNS server via DHCP.*

![Single DHCP binding](screenshots/06c-dhcp-single-binding.png)
*Figure 7: Router confirming the lease in its DHCP binding table.*

![All DHCP bindings](screenshots/06d-dhcp-binding-all.png)
*Figure 8: Full binding table — all 6 client PCs leased correctly within their department subnet.*

### 5. DNS & HTTP Services
Server-99 runs DNS (A record: `intranet.company.com` → `192.168.99.10`) and HTTP. Verified end-to-end: a client resolved the hostname via `nslookup` and successfully loaded the webpage by name.

![nslookup resolving intranet.company.com](screenshots/07-dns-nslookup.png)
*Figure 9: DNS resolution confirmed via nslookup — name resolves to 192.168.99.10.*

![Webpage loaded via hostname](screenshots/08-http-pageload.png)
*Figure 10: Default HTTP page loading successfully using the resolved hostname.*

### 6. Security Segmentation (ACL)
An extended ACL (110) applied inbound on the Sales VLAN sub-interface enforces least-privilege access:
- Sales → HR: **denied**
- Sales → IT: **denied**
- Sales → Servers: **permitted** (shared resources remain accessible)
- All other inter-department traffic: **unaffected**

```
access-list 110 deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
access-list 110 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
access-list 110 permit ip any any
```

Verified via `show access-lists` (showing non-zero match counts on both deny rules) and live ping tests from Sales toward HR (failed) and Servers (succeeded).

![ACL enforcement test](screenshots/09-acl-enforcement-test.png)
*Figure 11: PC1-Sales — HR (blocked), Server (allowed), IT (blocked) — confirming least-privilege segmentation.*

### 7. Firewall Placement
An ASA 5505 sits between the router and the (unsimulated) internet boundary, with its inside interface (Ethernet0/0, VLAN 1) configured and connected to the router on a dedicated `192.168.1.0/24` link. This demonstrates correct architectural placement of a perimeter firewall ahead of the internal routed network. No outside-facing internet circuit was simulated, so NAT/outbound policy was out of scope for this lab.

![Router to firewall link up](screenshots/10-firewall-link-up.png)
*Figure 12: Router's Gig0/1 up with IP 192.168.1.2, successful ping to the ASA at 192.168.1.1.*

![ASA interface brief](screenshots/11-asa-interface-brief.png)
*Figure 13: ASA confirming Ethernet0/0 and VLAN1 ("inside") are up with the matching IP.*

## Key Configuration Snippets

**Router sub-interface (example, VLAN 10):**
```
interface gigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
```

**DHCP pool (example, IT):**
```
ip dhcp excluded-address 192.168.10.1 192.168.10.9
ip dhcp pool IT_POOL
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 192.168.99.10
```

**Switch trunk/access (example, Switch-IT):**
```
vlan 10
 name IT
interface range fastEthernet0/1-2
 switchport mode access
 switchport access vlan 10
interface gigabitEthernet0/1
 switchport mode trunk
```

## Verification Summary

| Test | Result |
|---|---|
| IT ↔ HR (cross-VLAN) | Success, TTL=127 |
| Sales ↔ Server (cross-VLAN) | Success, TTL=127 |
| HR ↔ IT (post-ACL, unaffected dept) | Success, TTL=127 |
| Sales → HR (post-ACL) | Blocked — destination unreachable |
| Sales → IT (post-ACL) | Blocked — destination unreachable |
| Sales → Server (post-ACL) | Still succeeds — shared resource exempted |
| DHCP lease (all 6 PCs) | All received correct subnet + DNS server automatically |
| DNS resolution | `intranet.company.com` → `192.168.99.10` |
| HTTP via hostname | Page loaded successfully |
| Router ↔ Firewall link | Up, ping successful |

## Skills Demonstrated

- VLAN design and trunking
- Router-on-a-stick inter-VLAN routing
- DHCP scope design and option configuration
- DNS/HTTP service deployment
- Extended ACL design for least-privilege network segmentation
- Firewall placement within enterprise topology
- Systematic verification and troubleshooting (TTL analysis, port/link diagnostics, binding tables, ACL match counters)

## Notes for Reviewers

This project intentionally favors router ACLs over deep ASA firewall policy, reflecting how many small-to-mid-sized enterprises segment internal traffic in practice. The ACL work is the security-relevant centerpiece of this lab — demonstrating the kind of "should this host be allowed to reach that subnet" thinking relevant to SOC analysis and network defense.
