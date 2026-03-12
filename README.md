# Mid-Scale Enterprise Network Simulation

A simulated multi-building enterprise network built in **Cisco Packet Tracer**, designed to replicate a mid-scale organization with three separate departments across three buildings, a centralized core network, and full OSPF-based inter-building routing. This project is a deliberate step up in complexity from the previous small enterprise project — focusing on dynamic routing, multi-building architecture, and centralized network services.

> **Note:** NAT, ACL, DHCP Snooping, DAI, and device hardening are intentionally not configured in this project. This project was scoped specifically to focus on routing complexity, OSPF, and multi-building architecture. These features will be carried forward and implemented in the next project alongside the new technologies introduced here.

---

## Network Topology

![Network Topology](topology.png)

> *Three buildings each with a dedicated MLS, connected to a central core switch. Edge router connects the core network to the ISP via a serial WAN link. Standalone DHCP server sits in the core network segment.*

---

## Technology Summary

| Technology | Implementation Details |
|---|---|
| **VLSM** | Each building uses a dedicated /24 address space, subdivided into /26 subnets per VLAN |
| **VLANs** | Three department VLANs per building plus a native VLAN (60) and inactive VLAN (99) |
| **MLS Inter-VLAN Routing** | Each building uses its MLS for inter-VLAN routing — no ROAS |
| **OSPF** | Single-area OSPF (Area 0) across all MLS and edge router for inter-building routing |
| **Standalone DHCP Server** | Centralized DHCP server in the core network serving all three buildings via DHCP relay |
| **Routed Ports (no switchport)** | MLS uplinks to core switch configured as routed Layer 3 ports, not trunk switchports |

---

## Network Addressing

### Core Network — 10.10.1.0 /28

| Device | Interface | IP Address |
|---|---|---|
| Edge Router | G0/0/0 | 10.10.1.1 /28 |
| MLS1 (IT Dept) | G0/1 | 10.10.1.2 /28 |
| MLS2 (Administration) | G0/1 | 10.10.1.3 /28 |
| MLS3 (Engineers) | G0/1 | 10.10.1.4 /28 |
| DHCP Server | F0 | 10.10.1.5 /28 |
| Edge Router (WAN) | S0/1/0 | 200.125.35.2 /30 |
| ISP | S0/1/0 | 200.125.35.1 /30 |

### OSPF Router IDs

| Device | Router ID |
|---|---|
| Edge Router | 100.100.100.100 |
| MLS1 | 3.3.3.3 |
| MLS2 | 1.1.1.1 |
| MLS3 | 2.2.2.2 |

---

## VLAN & IP Addressing Table

### Building 1 — IT Department (192.168.3.0 /24)

| VLAN | Name | Subnet | Usable Range | Gateway |
|---|---|---|---|---|
| 10 | Security | 192.168.3.0 /26 | 192.168.3.2 – 192.168.3.63 | 192.168.3.1 |
| 20 | Infrastructure | 192.168.3.64 /26 | 192.168.3.66 – 192.168.3.127 | 192.168.3.65 |
| 30 | Customer Service | 192.168.3.128 /26 | 192.168.3.130 – 192.168.3.191 | 192.168.3.129 |
| 60 | Native VLAN | — | — | — |
| 99 | Inactive | — | — | — |

### Building 2 — Administration (192.168.1.0 /24)

| VLAN | Name | Subnet | Usable Range | Gateway |
|---|---|---|---|---|
| 10 | HR Department | 192.168.1.0 /26 | 192.168.1.2 – 192.168.1.63 | 192.168.1.1 |
| 20 | Finance Department | 192.168.1.64 /26 | 192.168.1.66 – 192.168.1.127 | 192.168.1.65 |
| 30 | Directors | 192.168.1.128 /26 | 192.168.1.130 – 192.168.1.191 | 192.168.1.129 |
| 60 | Native VLAN | — | — | — |
| 99 | Inactive | — | — | — |

### Building 3 — Engineers (192.168.2.0 /24)

| VLAN | Name | Subnet | Usable Range | Gateway |
|---|---|---|---|---|
| 10 | Senior SWE | 192.168.2.0 /26 | 192.168.2.2 – 192.168.2.63 | 192.168.2.1 |
| 20 | Junior SWE | 192.168.2.64 /26 | 192.168.2.66 – 192.168.2.127 | 192.168.2.65 |
| 30 | Intern | 192.168.2.128 /26 | 192.168.2.130 – 192.168.2.191 | 192.168.2.129 |
| 60 | Native VLAN | — | — | — |
| 99 | Inactive | — | — | — |

> **First usable address per subnet** is assigned to the default gateway (MLS SVI) across all three buildings. No additional addresses are excluded.

---

## OSPF Design

All devices participate in a single OSPF Area 0. Reference bandwidth is set to 10,000 on all routers to ensure accurate cost calculation for modern link speeds — the default OSPF reference bandwidth of 100 Mbps does not differentiate between FastEthernet and GigabitEthernet links.

Each MLS advertises its building subnets directly into OSPF Area 0. The edge router redistributes the default route using `default-information originate` to propagate the gateway of last resort to all OSPF participants. Traffic destined for unknown networks is forwarded through the WAN link toward the ISP — confirmed via simulation mode in Packet Tracer. The default route does not appear in MLS routing tables as an explicit OSPF entry, but packets are being forwarded correctly through the edge router.

---

## Design Decisions

**Why MLS instead of dedicated building routers?**
The original design included a dedicated router per building. During implementation it became clear that OSPF cannot advertise a network it is not directly part of — the building routers were using a separate /30 link address and could not advertise the internal building subnets (`192.168.X.0/24`) correctly. Removing the building routers and having each MLS connect directly to the core switch as a Layer 3 routed device resolved this cleanly. MLS devices are capable of handling both inter-VLAN routing and OSPF participation, making the separate router redundant.

**Why routed ports (no switchport) on MLS uplinks?**
Initial configuration used trunk switchports on the MLS uplink to the core switch. This caused the MLS to behave like a Layer 2 switch on that link, preventing routing. Converting the uplink to a routed port (`no switchport`) with a dedicated IP address allowed proper Layer 3 communication between the MLS and the core network.

**Why a single OSPF area instead of multi-area?**
Multi-area OSPF was considered during planning. Given the scale of this network and the fact that all three buildings connect to a single core, the overhead of multi-area OSPF was not justified. A single Area 0 is simpler to manage, easier to troubleshoot, and appropriate for this topology size.

**Why a standalone DHCP server instead of router-based DHCP?**
Centralizing DHCP on a dedicated server reflects real enterprise practice — it is easier to manage, audit, and scale than router-based DHCP pools. Each MLS is configured with `ip helper-address` to relay DHCP requests from hosts to the central server.

**Why is WLC not included?**
WLC and wireless networking were part of the original scope but removed during implementation. The routing and OSPF complexity was already significant, and adding WLC configuration on top would have compromised the quality of both. A dedicated standalone wireless network project will be the next build.

**Why no NAT, ACL, or device hardening?**
This project was deliberately scoped to focus on routing architecture, OSPF, and multi-building design. NAT, ACLs, and security hardening were present in the previous project and will be carried forward and combined with the new technologies in the next project. Devices are also intentionally left accessible so anyone downloading the project can explore it freely.

---

## Challenges & Troubleshooting

**1. Building routers not advertising internal subnets via OSPF**
The original design used dedicated building routers connected to the MLS via /30 point-to-point links. OSPF was not advertising the internal building subnets (`192.168.X.0/24`) to the rest of the network. After research, discovered that OSPF cannot advertise a network it is not directly part of — the router's interface was in the `10.X.X.0/30` space, so OSPF had no knowledge of the `192.168.X.0` subnets even with a network statement. Resolved by removing the building routers entirely and connecting MLS directly to the core switch as Layer 3 routed devices.

**2. MLS uplink behaving as a Layer 2 trunk port**
Inter-VLAN routing within each building was working but the MLS could not communicate with the building router or core network. The uplink port was configured as a trunk switchport, causing it to operate at Layer 2. Resolved by converting the uplink to a routed port using `no switchport` and assigning a dedicated IP address. This allowed proper Layer 3 routing between the MLS and the core network.

**3. OSPF neighbor adjacency failing on edge router**
OSPF was forming neighbors and populating routes correctly on all MLS devices but the edge router was not forming adjacencies. Discovered the MLS interfaces connecting to the core switch were configured with a /29 subnet mask (`255.255.255.248`) instead of the correct /28 (`255.255.255.240`). The subnet mismatch prevented OSPF hello packets from being accepted. Correcting the subnet mask resolved the adjacency issue immediately.

**4. Default route not appearing in MLS routing tables**
The edge router had a static default route configured with `default-information originate` applied. The default route was not appearing as an explicit OSPF entry in the MLS routing tables. Attempted `clear ip ospf process` on the edge router but this did not fully resolve the issue. However, using simulation mode in Packet Tracer confirmed that traffic destined for unknown networks was still being forwarded through the WAN link to the ISP — meaning the default route forwarding behavior was functioning even without appearing explicitly in the MLS routing tables. This remains an open observation and a known limitation of this build.

---

## Testing Results

| Test | Result |
|---|---|
| Inter-VLAN routing — Building 1 | ✅ Working |
| Inter-VLAN routing — Building 2 | ✅ Working |
| Inter-VLAN routing — Building 3 | ✅ Working |
| OSPF neighbor adjacencies | ✅ Working |
| OSPF route propagation across all devices | ✅ Working |
| Hosts pinging edge router | ✅ Working |
| DHCP address assignment | ✅ Working |
| Traffic forwarding toward ISP | ✅ Working (ISP not connected to external network — dropped at ISP as expected) |

---

## What's Missing / Future Scope

1. **NAT/PAT** — Required for actual internet access; will be implemented in the next project
2. **ACLs** — Inter-department traffic restriction; will be carried forward
3. **DHCP Snooping + DAI** — Layer 2 attack prevention; will be carried forward
4. **Device hardening & SSH** — Intentionally left open for lab accessibility; next project will include this
5. **Wireless Network** — WLC and AP configuration will be built as a dedicated standalone project
6. **Redundancy** — HSRP and EtherChannel for high availability
7. **Unique VLAN IDs per building** — All three buildings currently reuse VLAN 10, 20, 30; a production network would use unique IDs across the entire enterprise

---

## Tools Used

- **Cisco Packet Tracer** — Network simulation
- **Cisco ISR Router** — Edge routing, OSPF, default route redistribution
- **Cisco 3560 MLS** — Inter-VLAN routing, OSPF, routed uplinks
- **Standalone DHCP Server** — Centralized dynamic address assignment

---

## Testing the Project

Devices are intentionally left without passwords so anyone can download and explore the project freely in Cisco Packet Tracer.
