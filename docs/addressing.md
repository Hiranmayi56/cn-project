# Addressing Plan

**Source of truth** for every IP address, VLAN, gateway and HSRP virtual IP in the project. Update this file **before** changing any config. Design context is in [README.md](../README.md).

Status: draft · Last updated: 2026-10-31

---

## 1. Address space

| Range | Used for | Why |
|---|---|---|
| `10.10.0.0/16` | Everything inside the campus | Private range (RFC 1918), not routable on the internet, so NAT is needed to go outside |
| `10.10.0.0` – `10.10.3.151` | VLANs, DMZ, point-to-point links | Allocated with VLSM, largest subnet first (section 2) |
| `10.10.3.152` – `10.10.3.255`, `10.10.4.0` onward | Free | Room to grow |
| `10.10.255.0/24` | Loopbacks (OSPF router IDs) | Kept apart so they're easy to recognize |
| `203.0.113.0/24` | Simulated public internet: ISP link and public IPs for the DMZ | Reserved documentation range (RFC 5737), so it can't clash with real addresses |
| `198.51.100.0/24` | Simulated internet LAN behind the ISP router | Also a reserved documentation range |

## 2. Subnet sizing (VLSM)

**Rule:** pick the smallest block (a power of 2) that holds the hosts **plus 2** (the network address and the broadcast address can't be given to devices). Allocate the largest blocks first so they stay aligned and nothing overlaps.

| VLAN / segment | Hosts needed | Block size | Prefix | Subnet mask | Usable hosts |
|---|---|---|---|---|---|
| 10 Students | 500 | 512 | /23 | 255.255.254.0 | 510 |
| 20 Faculty | 100 | 128 | /25 | 255.255.255.128 | 126 |
| 40 Guest | 100 | 128 | /25 | 255.255.255.128 | 126 |
| 50 IoT | 50 | 64 | /26 | 255.255.255.192 | 62 |
| 30 Servers | ~10 | 32 | /27 | 255.255.255.224 | 30 |
| 99 Management | ~10 | 32 | /27 | 255.255.255.224 | 30 |
| 60 DMZ | ~6 | 16 | /28 | 255.255.255.240 | 14 |
| Router links (×2) | 2 each | 4 | /30 | 255.255.255.252 | 2 |

## 3. VLAN subnets

**Conventions for every user VLAN:**
- `.1` is the **HSRP virtual IP**: the default gateway every host uses. It stays the same whichever core switch is active.
- MLS1-Core and MLS2-Core take the next two addresses for their own SVIs (an SVI is the switch's virtual interface for a VLAN).
- The first 10 addresses are reserved for gateways and static devices. DHCP hands out the rest.

| VLAN | Name | Network | Mask | Gateway (HSRP VIP) | MLS1-Core | MLS2-Core | Broadcast | DHCP range |
|---|---|---|---|---|---|---|---|---|
| 10 | STUDENTS | 10.10.0.0/23 | 255.255.254.0 | 10.10.0.1 | 10.10.0.2 | 10.10.0.3 | 10.10.1.255 | 10.10.0.11 – 10.10.1.254 |
| 20 | FACULTY | 10.10.2.0/25 | 255.255.255.128 | 10.10.2.1 | 10.10.2.2 | 10.10.2.3 | 10.10.2.127 | 10.10.2.11 – 10.10.2.126 |
| 40 | GUEST | 10.10.2.128/25 | 255.255.255.128 | 10.10.2.129 | 10.10.2.130 | 10.10.2.131 | 10.10.2.255 | 10.10.2.139 – 10.10.2.254 |
| 50 | IOT | 10.10.3.0/26 | 255.255.255.192 | 10.10.3.1 | 10.10.3.2 | 10.10.3.3 | 10.10.3.63 | 10.10.3.11 – 10.10.3.62 |
| 30 | SERVERS | 10.10.3.64/27 | 255.255.255.224 | 10.10.3.65 | 10.10.3.66 | 10.10.3.67 | 10.10.3.95 | Static only |
| 99 | MANAGEMENT | 10.10.3.96/27 | 255.255.255.224 | 10.10.3.97 | 10.10.3.98 | 10.10.3.99 | 10.10.3.127 | Static only |
| 60 | DMZ | 10.10.3.128/28 | 255.255.255.240 | 10.10.3.129 (R1-Edge, no HSRP) | — | — | 10.10.3.143 | Static only |
| 998 | BLACKHOLE | No IP | | | | | | Unused ports, shut down |
| 999 | NATIVE-UNUSED | No IP | | | | | | Trunk native VLAN only |

VLAN 60 exists only on SW-DMZ and R1-Edge. It is not trunked to the core switches.

## 4. HSRP and STP

The **HSRP group number equals the VLAN ID** so it's easy to read in `show standby brief`. For each VLAN, the HSRP active switch is also the **STP root bridge**, so traffic goes straight to its gateway without crossing the EtherChannel.

| VLAN | Group | Virtual IP | Active (priority 110) + STP root primary | Standby (priority 100) + STP root secondary |
|---|---|---|---|---|
| 10 | 10 | 10.10.0.1 | MLS1-Core | MLS2-Core |
| 20 | 20 | 10.10.2.1 | MLS2-Core | MLS1-Core |
| 30 | 30 | 10.10.3.65 | MLS1-Core | MLS2-Core |
| 40 | 40 | 10.10.2.129 | MLS2-Core | MLS1-Core |
| 50 | 50 | 10.10.3.1 | MLS1-Core | MLS2-Core |
| 99 | 99 | 10.10.3.97 | MLS2-Core | MLS1-Core |

- **Preempt** is on for both switches, so the intended active switch takes its role back after it recovers.
- **Tracking:** each core switch tracks its uplink to R1-Edge and lowers its priority if that link fails, so the other switch becomes active.

## 5. Routed links, loopbacks and OSPF

| Link | Subnet | Side A | Side B |
|---|---|---|---|
| R1-Edge G0/0 ↔ MLS1-Core G1/0/24 | 10.10.3.144/30 | R1-Edge 10.10.3.145 | MLS1-Core 10.10.3.146 |
| R1-Edge G0/1 ↔ MLS2-Core G1/0/24 | 10.10.3.148/30 | R1-Edge 10.10.3.149 | MLS2-Core 10.10.3.150 |
| R1-Edge S0/0/0 ↔ ISP S0/0/0 (serial) | 203.0.113.0/30 | R1-Edge 203.0.113.2 | ISP 203.0.113.1 (DCE side, sets the clock rate) |

| Device | Loopback0 | OSPF router ID |
|---|---|---|
| R1-Edge | 10.10.255.1/32 | 10.10.255.1 |
| MLS1-Core | 10.10.255.2/32 | 10.10.255.2 |
| MLS2-Core | 10.10.255.3/32 | 10.10.255.3 |

**OSPF (area 0)**
- **Neighbors:** R1-Edge ↔ MLS1-Core and R1-Edge ↔ MLS2-Core, over the two /30 links.
- **Passive interfaces:** all SVIs on the core switches are advertised but passive (no OSPF hellos sent to user VLANs).
- **No direct OSPF link between the core switches:** if a core switch loses its uplink, HSRP tracking moves the gateway to the other switch instead.
- **Default route:** R1-Edge has a static default route to the ISP (203.0.113.1) and advertises it into OSPF with `default-information originate`.
- **ISP router:** has a static route for the public DMZ block `203.0.113.8/29` pointing to R1-Edge (203.0.113.2). It needs no route to `10.10.0.0/16`, since NAT hides those addresses.

## 6. Static device addresses

| Device | VLAN | IP / prefix | Gateway | DNS |
|---|---|---|---|---|
| DHCP server | 30 | 10.10.3.70/27 | 10.10.3.65 | 10.10.3.71 |
| Internal DNS server | 30 | 10.10.3.71/27 | 10.10.3.65 | 10.10.3.71 |
| IoT server | 30 | 10.10.3.72/27 | 10.10.3.65 | 10.10.3.71 |
| Admin PC (Administration) | 99 | 10.10.3.110/27 | 10.10.3.97 | 10.10.3.71 |
| MLS1-Core (SVI 99) | 99 | 10.10.3.98/27 | — | — |
| MLS2-Core (SVI 99) | 99 | 10.10.3.99/27 | — | — |
| SW1-Access-Admin (SVI 99) | 99 | 10.10.3.101/27 | 10.10.3.97 | — |
| SW2-Access-Acad (SVI 99) | 99 | 10.10.3.102/27 | 10.10.3.97 | — |
| SW3-Access-Lib (SVI 99) | 99 | 10.10.3.103/27 | 10.10.3.97 | — |
| SW-Servers (SVI 99) | 99 | 10.10.3.104/27 | 10.10.3.97 | — |
| DMZ web server | 60 | 10.10.3.130/28 | 10.10.3.129 | 10.10.3.131 |
| DMZ DNS server | 60 | 10.10.3.131/28 | 10.10.3.129 | 10.10.3.131 |
| DMZ email server | 60 | 10.10.3.132/28 | 10.10.3.129 | 10.10.3.131 |
| SW-DMZ (SVI 60) | 60 | 10.10.3.140/28 | 10.10.3.129 | — |
| ISP G0/0 (internet LAN) | — | 198.51.100.1/24 | — | — |
| Internet test server | — | 198.51.100.10/24 | 198.51.100.1 | 203.0.113.11 |
| Outside PC | — | 198.51.100.20/24 | 198.51.100.1 | 203.0.113.11 |

SW-DMZ is managed through VLAN 60, but its VTY lines only accept SSH from the Management VLAN (10.10.3.96/27).

## 7. DHCP pools

All pools live on the DHCP server (10.10.3.70). Both core switches relay requests to it with `ip helper-address 10.10.3.70` on SVIs 10, 20, 40 and 50.

| Pool | Default gateway | DNS | Start IP | Subnet mask | Max users |
|---|---|---|---|---|---|
| STUDENTS | 10.10.0.1 | 10.10.3.71 | 10.10.0.11 | 255.255.254.0 | 500 |
| FACULTY | 10.10.2.1 | 10.10.3.71 | 10.10.2.11 | 255.255.255.128 | 116 |
| GUEST | 10.10.2.129 | 10.10.3.71 | 10.10.2.139 | 255.255.255.128 | 116 |
| IOT | 10.10.3.1 | 10.10.3.71 | 10.10.3.11 | 255.255.255.192 | 52 |

**For the ACL stage:** Guest and IoT are blocked from internal networks, but they must still be allowed DHCP (UDP 67/68) and DNS (UDP 53 to 10.10.3.71), or they won't get addresses or resolve names.

## 8. NAT (on R1-Edge)

| Type | Inside | Outside | Purpose |
|---|---|---|---|
| PAT (overload) | Students 10.10.0.0/23, Faculty + Guest 10.10.2.0/24 | R1-Edge S0/0/0 (203.0.113.2) | User VLANs share one public IP to reach the internet |
| Static NAT | Web 10.10.3.130 | 203.0.113.10 | Campus website reachable from the internet |
| Static NAT | DMZ DNS 10.10.3.131 | 203.0.113.11 | Public DNS for the campus domain |
| Static NAT | Email 10.10.3.132 | 203.0.113.12 | Campus email reachable from the internet |

- **PAT ACL:** `permit 10.10.0.0 0.0.1.255` and `permit 10.10.2.0 0.0.0.255`.
- **IoT, Servers and Management get no internet access.** They don't need it, and it keeps them more secure.
- **Public block:** 203.0.113.8/29 (usable .9 – .14). .13 and .14 are spare.

## 9. Port plan

**Core switches (same on MLS1-Core and MLS2-Core)**

| Port | Mode | Connects to |
|---|---|---|
| G1/0/1 – 2 | Trunk, Port-channel 1 (LACP) | The other core switch (EtherChannel) |
| G1/0/3 | Trunk | SW1-Access-Admin (G0/1 from MLS1, G0/2 from MLS2) |
| G1/0/4 | Trunk | SW2-Access-Acad (G0/1 from MLS1, G0/2 from MLS2) |
| G1/0/5 | Trunk | SW3-Access-Lib (G0/1 from MLS1, G0/2 from MLS2) |
| G1/0/6 | Trunk | SW-Servers (G0/1 from MLS1, G0/2 from MLS2) |
| G1/0/24 | Routed (`no switchport`) | R1-Edge (G0/0 from MLS1, G0/1 from MLS2) |
| All others | Shut down, VLAN 998 | — |

All trunks use native VLAN 999.

**Access and server switches** (2960-24TT: ports Fa0/1 – 24, uplinks G0/1 – 2)

| Switch | Trunk VLANs allowed | Access ports | Unused (shut, VLAN 998) |
|---|---|---|---|
| SW1-Access-Admin | 20, 50, 99 | Fa0/1 – 8 VLAN 20 (staff PCs) · Fa0/9 – 12 VLAN 50 (door lock, smoke detector, alarm) · Fa0/13 VLAN 99 (admin PC) | Fa0/14 – 24 |
| SW2-Access-Acad | 10, 20, 50, 99 | Fa0/1 – 8 VLAN 10 (student PCs) · Fa0/9 – 14 VLAN 20 (faculty PCs) · Fa0/15 – 18 VLAN 50 (classroom IoT) · Fa0/19 VLAN 10 (Campus-WiFi AP) | Fa0/20 – 24 |
| SW3-Access-Lib | 10, 40, 50, 99 | Fa0/1 – 8 VLAN 10 (student PCs) · Fa0/9 VLAN 10 (Campus-WiFi AP) · Fa0/10 VLAN 40 (Guest-WiFi AP) · Fa0/11 – 14 VLAN 50 (temperature sensor, AC, smoke detector, alarm) | Fa0/15 – 24 |
| SW-Servers | 30, 99 | Fa0/1 DHCP · Fa0/2 internal DNS · Fa0/3 IoT server (all VLAN 30) | Fa0/4 – 24 |
| SW-DMZ | No trunk | G0/1 to R1-Edge G0/2 · Fa0/1 web · Fa0/2 DNS · Fa0/3 email (all VLAN 60) | Fa0/4 – 24 |

## 10. Wireless

AccessPoint-PT works at Layer 2 (it bridges wireless clients onto the VLAN of the switch port it's plugged into), so it needs no IP address. Each AP carries one SSID.

| SSID | VLAN | Location | Switch port |
|---|---|---|---|
| Campus-WiFi | 10 | Academics | SW2-Access-Acad Fa0/19 |
| Campus-WiFi | 10 | Library | SW3-Access-Lib Fa0/9 |
| Guest-WiFi | 40 | Library | SW3-Access-Lib Fa0/10 |

## 11. Likely viva questions

**Why is the Students subnet a /23?**
500 hosts + 2 reserved addresses needs a block of at least 502. The next power of 2 is 512 = 2⁹, which leaves 32 − 9 = 23 network bits.

**Why allocate the largest subnet first?**
Every subnet must start on a multiple of its own block size. Placing large blocks first keeps them aligned and leaves no gaps or overlaps when the smaller ones fill in after.

**Why is the gateway an HSRP virtual IP and not a switch's own IP?**
Hosts have one gateway configured. If it were MLS1's own address, hosts would lose their gateway when MLS1 fails. The virtual IP moves to whichever switch is active, so hosts never notice.

**Why use 203.0.113.0/24 for the "internet"?**
It's reserved for documentation and examples, so it can never conflict with a real public network.

## 12. Open items

- [ ] **R1-Edge needs 4 connections, but the 2911 has 3 Gigabit ports.** Install an HWIC-2T serial module in R1-Edge and the ISP router (power the router off in the Physical tab first) for the serial ISP link.
- [ ] Check whether each Packet Tracer IoT device can use a wired port. If some are wireless-only, add an IoT AP on VLAN 50 and update sections 9 and 10.
- [ ] Confirm the Packet Tracer DHCP server accepts all four relayed pools.
