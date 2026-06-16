**Small Enterprise Network**

## 1. Overview

This lab simulates a small enterprise network using Cisco technologies. It demonstrates VLAN segmentation, dynamic routing, centralized network services, and multi-layered security policies (Port Security & ACLs).

**Key Technologies**

* Multi-VLAN architecture (VLAN 10 & VLAN 20)
* Inter-VLAN routing (Router-on-a-Stick)
* DHCP Relay service (`ip helper-address`)
* Layer 2 STP redundancy (Cross-over link)
* Dynamic routing with OSPF (Single Area)
* NAT overload (PAT)
* Layer 2 Port Security (Sticky MAC)
* ACL-based traffic filtering (Security Policy)
* Internet simulation via upstream ISP router

## 2. Network Topology

                                [ Internet Server ] (8.8.8.8)
                                        |
                                    [ R2 (ISP) ] 
                                   (Gig0/1) .1
                                        |
                            10.0.0.0/30 | OSPF Area 0
                                        |
                                   (Gig0/0) .2
                            [ R1 (HQ Gateway & NAT) ]
                                   (Gig0/1) 
                                        |
                                  (Trunk Link)
                                        |
                                     [ SW1 ] (Core Switch)
                                     /     \
                       (Trunk Link) /       \ (Trunk Link)
                                   /         \
                 (VLAN 10 - STAFF)             (VLAN 20 - IT)
                     [ SW0 ] <== STP Backup ==> [ SW2 ]
                     /     \                     /    \
                 PC0      PC1               Laptop0  DHCP+DNS Server
                (DHCP)   (DHCP)             (DHCP)   (192.168.20.10)

```

## 3. IP Addressing Plan

**Device IPs & Subnets**

| Device | Interface | IP Address / Subnet | Network / Role |
| --- | --- | --- | --- |
| **R1** | G0/1.10 | 192.168.10.1 /24 | VLAN 10 Gateway |
| **R1** | G0/1.20 | 192.168.20.1 /24 | VLAN 20 Gateway |
| **R1** | G0/0 | 10.0.0.2 /30 | WAN Link to ISP |
| **R2** | G0/2 | 10.0.0.1 /30 | WAN Link to HQ |
| **R2** | G0/1 | 8.8.8.1 /24 | Internet Gateway |
| **Server** | Fa0 | 192.168.20.10 /24 | Internal DHCP+DNS |
| **Server** | Fa0 | 8.8.8.8 /24 | External Web Server |

## 4. VLAN & Layer 2 Redundancy

* **VLAN 10:** STAFF (`192.168.10.0/24`)
* **VLAN 20:** IT_SERVER (`192.168.20.0/24`)
* **Trunking:** Configured between R1, SW1, SW0, and SW2.
* **STP Redundancy:** A cross-over cable connects SW0 and SW2. Spanning Tree Protocol (STP) automatically blocks one port to prevent broadcast storms, while providing a failover path if the main link to SW1 drops.

## 5. Layer 2 Security (Port Security)

Configured on SW0 access ports (connected to STAFF PCs):

* **Mode:** Access
* **Sticky MAC:** Dynamically learns and binds the first connected MAC address.
* **Max MAC:** `1` per port.
* **Violation Action:** `Shutdown` (Automatically disables the port if an unauthorized/rogue laptop is connected).

## 6. Inter-VLAN Routing & OSPF

* **Router-on-a-Stick:** R1 uses sub-interfaces (`G0/1.10`, `G0/1.20`) to route traffic between VLANs.
* **OSPF (Area 0):** Replaced static routing with dynamic OSPF. R1 and R2 automatically form neighbor adjacency to route traffic between the HQ LANs and the Internet simulation network.

## 7. Centralized DHCP & DNS

* **DHCP Server:** Located in VLAN 20 (`192.168.20.10`).
* **DHCP Relay:** R1 interface `G0/1.10` is configured with `ip helper-address 192.168.20.10` to forward DHCP broadcasts from VLAN 10 to the server.
* **DNS Service:** Resolves `google.com` to `8.8.8.8`.

## 8. NAT (Internet Access)

* **PAT (NAT Overload):** Configured on R1.
* **Inside:** VLAN sub-interfaces (`G0/1.10`, `G0/1.20`).
* **Outside:** WAN link to ISP (`G0/0`).
* Enables all internal PCs to browse the simulated Internet using a single public IP (`10.0.0.2`).

## 9. ACL (Security Policy)

**Extended ACL 100** applied inbound on R1 (`G0/1.10`):

* **VLAN 10 Restrictions:**
* ❌ Cannot `ping` (ICMP) VLAN 20 (Protects IT Servers from internal scanning).


* **VLAN 10 Permissions:**
* ✅ Can access the Internet (e.g., browse `8.8.8.8`).
* ✅ Can receive DHCP IP addresses.



## 10. Verification & Testing

* **DHCP Test:** PCs in VLAN 10 successfully receive IPs from the DHCP server in VLAN 20 via DHCP Relay.
* **OSPF Test:** `show ip route` on R1 displays OSPF (`O`) routes learned from R2.
* **Port Security Test:** Disconnecting PC0 and plugging in a new rogue laptop immediately turns the switch port status to `down/red`.
* **ACL Test:** PC0 `ping 192.168.20.10` results in `Destination host unreachable`, while `ping 8.8.8.8` replies successfully.

## 11. Conclusion

This lab demonstrates a realistic, cost-effective enterprise network design. It highlights the ability to centralize network services (DHCP Relay), ensure high availability at Layer 2 (STP), automate routing (OSPF), and enforce strict physical and logical security boundaries (Port Security & ACLs).
