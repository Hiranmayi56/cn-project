# Fault-Tolerant Smart Campus Network

**Enterprise Network Design and Security: Implementation of a Fault-Tolerant Smart Campus Network**

Computer Networks · CIE-3 Mini Project (15 marks) · RV University, Bengaluru · Team of 4 · Cisco Packet Tracer

---

## 1. Problem statement

A university campus serves students, faculty, guests, servers and a growing number of smart devices on the same physical network. Without design, this leads to:

- **No separation:** a student laptop can reach faculty machines and servers.
- **Downtime:** one failed switch or cable disconnects a whole building.
- **Exposure:** guests and weak IoT devices sit on the same network as internal systems, and public servers sit inside it.

This project designs and simulates a campus network that **separates users into zones, keeps working when a device or link fails, protects internal systems, and runs smart-campus services** such as automated lighting, climate control and safety alerts.

## 2. Objectives

1. Build a **hierarchical campus network** for 3 buildings (Administration, Academics, Library) with a redundant core.
2. **Segment** users into VLANs and enforce who can talk to whom with ACLs.
3. Make the network **self-healing**: no single switch or uplink failure inside the campus takes users offline (STP, EtherChannel, HSRP, OSPF).
4. Provide **central services**: DHCP, DNS, NAT/PAT, and a DMZ for public web, DNS and email servers.
5. Add a **smart campus layer**: IoT devices on an isolated VLAN, managed from a central IoT server with automation rules.
6. **Secure** the network at every layer: SSH-only management, port security, ACLs, guest and IoT isolation.
7. **Prove** everything with a test plan, and demonstrate live failover in the viva.

## 3. What makes it a "smart" campus

| Smart feature | What it does | How it's built in Packet Tracer |
|---|---|---|
| **Smart classrooms** | Lights and AC/fan switch on automatically when motion is detected, and off when the room is empty | Motion detector, smart light and fan/AC registered to the IoT server, with an automation rule |
| **Climate monitoring** | AC turns on when the temperature crosses a threshold | Temperature sensor + AC, automation rule on the IoT server |
| **Fire safety** | Smoke detection triggers an alarm (and sprinkler, if available) | Smoke detector + siren/sprinkler, automation rule |
| **Smart access** | Server room door lock controlled centrally | Smart door lock registered to the IoT server |
| **Central dashboard** | Facility staff monitor and control every device from a browser | IoT server's web dashboard, opened from a Faculty PC or smartphone |
| **IoT isolation** | IoT devices are kept in their own VLAN and only talk to the IoT server | IoT VLAN + ACL, so a compromised device can't reach users or servers |
| **Self-healing network** | Traffic reroutes automatically when a switch or link fails, with no manual action | HSRP, STP, EtherChannel and OSPF (see section 4) |
| **Campus-wide Wi-Fi** | Separate wireless networks for campus users and guests, each mapped to its own VLAN | Access points per building, one SSID per VLAN |

**Where the devices go:** Academics has the smart classrooms. The Library has climate monitoring and fire safety. Administration has the server room door lock and fire safety.

**Why this matters for the design:** IoT devices are easy to compromise, so a smart campus needs segmentation and access control even more than a normal one. The smart features and the security design support each other.

## 4. Topology

Three buildings (Administration, Academics, Library), a collapsed core/distribution layer (two multilayer switches), an access layer, a server block, a DMZ, and an edge router to the internet.

```
                         [ISP / Internet]
                                |
                           [R1-Edge] ─────── [SW-DMZ] ── Web · DNS · Email servers
                            /      \           (NAT and outside ACLs on R1-Edge)
                 routed links, OSPF area 0
                          /          \
              [MLS1-Core] ════════════ [MLS2-Core]      ← inter-VLAN routing, HSRP,
                         EtherChannel                     DHCP relay, internal ACLs

   Every switch below has one uplink to MLS1-Core and one to MLS2-Core:

 [SW1-Access-Admin]     [SW2-Access-Acad]     [SW3-Access-Lib]     [SW-Servers]
  Administration         Academics             Library              (in Administration)
  Staff PCs, admin PC    Student and           Student PCs          DHCP · DNS
  Door lock, smoke       faculty PCs           Campus + Guest Wi-Fi IoT server
  detector               Campus Wi-Fi          Temperature sensor,
                         Classroom IoT         smoke detector
```

**Key design decisions**

| Decision | Why |
|---|---|
| **Multilayer switches for inter-VLAN routing** (not router-on-a-stick) | Router-on-a-stick puts all routing on one router, which is a single point of failure. Two multilayer switches with HSRP give a backup gateway. |
| **Every access switch has an uplink to both core switches** | If one uplink or core switch fails, STP unblocks the other path. |
| **EtherChannel between the core switches** | Two cables act as one link: more bandwidth, and the link survives a cable failure. |
| **HSRP split across both core switches** | MLS1 is active gateway for half the VLANs, MLS2 for the other half, so both switches carry traffic and each backs up the other. |
| **STP root matches HSRP active** | For each VLAN, the same switch is both STP root and HSRP active, so traffic takes the shortest path to its gateway. |
| **Single-area OSPF** | 3 routing devices don't need multiple areas. One area is simpler to build, troubleshoot and explain. |
| **DMZ on the edge router** | Public servers are reachable from the internet but separated from internal VLANs. |

**Devices** (models to confirm against the lab's Packet Tracer version)

| Device | Model | Qty | Role |
|---|---|---|---|
| R1-Edge | Cisco 2911 + HWIC-2T | 1 | Internet edge, NAT/PAT, DMZ, outside ACLs. The HWIC-2T serial module provides the ISP link, since the 3 Gigabit ports go to MLS1, MLS2 and the DMZ |
| ISP | Cisco 2911 + HWIC-2T | 1 | Simulated internet: serial link to R1-Edge, LAN with an outside test PC and test server |
| MLS1-Core, MLS2-Core | Cisco 3650-24PS | 2 | Core/distribution, inter-VLAN routing, HSRP, OSPF |
| SW1-Access-Admin, SW2-Access-Acad, SW3-Access-Lib | Cisco 2960-24TT | 3 | Access layer (one per building), port security |
| SW-Servers, SW-DMZ | Cisco 2960-24TT | 2 | Server and DMZ segments |
| Access points | AccessPoint-PT | 3–4 | Campus Wi-Fi (Academics, Library), Guest Wi-Fi (Library) |
| Servers | Server-PT | 6 | DHCP, internal DNS, IoT server; DMZ web, DNS, email |
| IoT devices | Smart things | 8–12 | Sensors, lights, fan/AC, smoke detectors, alarm, door lock |
| End devices | PC, laptop, smartphone | 10–15 | 1–2 per VLAN per building |

> The 3650 may come without a power supply in Packet Tracer. If its ports stay red, add one in the device's Physical tab.

## 5. VLANs and addressing

| VLAN | Name | Who | HSRP active | Access rules |
|---|---|---|---|---|
| 10 | STUDENTS | Student PCs and Wi-Fi | MLS1 | Internet, DMZ, internal DNS. No Faculty, Servers (except DHCP/DNS), IoT or Management |
| 20 | FACULTY | Faculty and administrative staff PCs | MLS2 | Internet, DMZ, servers, IoT dashboard |
| 30 | SERVERS | DHCP, DNS, IoT server | MLS1 | Reachable as needed by each VLAN |
| 40 | GUEST | Guest Wi-Fi | MLS2 | **Internet only** (plus DHCP and DNS) |
| 50 | IOT | Smart devices | MLS1 | **IoT server only** (plus DHCP and DNS) |
| 60 | DMZ | Public web, DNS, email servers (only on SW-DMZ, not trunked to the core) | — (R1-Edge is the gateway) | Inbound from the internet to these servers only |
| 99 | MANAGEMENT | Switch/router management IPs, admin PC | MLS2 | Reachable only from the admin PC (in VLAN 99), SSH only |
| 998 | BLACKHOLE | Unused access ports, shut down | — | No traffic |
| 999 | NATIVE-UNUSED | Native VLAN on trunks, no devices | — | Prevents VLAN hopping attacks |

**VLANs per building**

| Building | Switch | VLANs with devices |
|---|---|---|
| Administration | SW1-Access-Admin | 20 Faculty (admin staff), 50 IoT, 99 Management (admin PC) |
| Administration (server room) | SW-Servers | 30 Servers |
| Academics | SW2-Access-Acad | 10 Students, 20 Faculty, 50 IoT |
| Library | SW3-Access-Lib | 10 Students, 40 Guest, 50 IoT |

Every switch also has a management IP in VLAN 99. Each trunk allows only the VLANs used in that building, plus VLAN 99.

- **Subnetting:** VLSM, sized from real user counts per VLAN. Point-to-point routed links use /30. Each routing device gets a loopback for its OSPF router ID.
- **HSRP:** each VLAN has a virtual gateway IP shared by MLS1 and MLS2.
- **Source of truth:** every subnet, gateway, HSRP virtual IP and device IP is recorded in [`docs/addressing.md`](docs/addressing.md) **before** any config is written.

## 6. Security design

| Layer | Controls |
|---|---|
| **Device access** | SSH only (no Telnet), encrypted passwords, local user accounts, login banners, management only from VLAN 99 |
| **Access ports** | Port security (MAC limit, shutdown on violation), unused ports shut down and placed in VLAN 998 (BLACKHOLE) |
| **Trunks** | Allowed VLAN list restricted, native VLAN changed to unused VLAN 999 |
| **Segmentation** | ACLs on the core switches enforce the access rules in section 5 (Guest internet-only, IoT to IoT server only, Students blocked from Faculty/Management) |
| **Perimeter** | NAT/PAT hides internal addresses. Outside ACLs on R1-Edge allow inbound traffic to the DMZ servers only, never to internal VLANs |
| **Optional** | Site-to-site IPsec VPN to a remote branch. Included in the report **only if it works** |

**Threats and countermeasures**

| Threat | Countermeasure |
|---|---|
| MAC flooding | Port security |
| Rogue device plugged into a wall port | Port security (shutdown), unused ports disabled |
| VLAN hopping | Native VLAN changed to an unused VLAN, trunking only on uplinks |
| Guest attacking internal systems | Guest VLAN + ACL, internet only |
| Compromised IoT device | IoT VLAN + ACL, can only reach the IoT server |
| Password sniffing on management | SSH instead of Telnet, encrypted passwords |
| Attack from the internet | NAT, outside ACLs, public servers in the DMZ only |

## 7. Build plan

Build one stage at a time. **Verify before moving on, save `packet-tracer/stageN.pkt` after each stage, and export each device's config to `configs/`.** If a stage breaks something, go back to the previous saved stage instead of piling fixes on top.

| # | Stage | Owner | Verify with |
|---|---|---|---|
| 0 | **Plan:** confirm device models and the lab's Packet Tracer version, user counts, VLSM addressing, diagram | Everyone | `docs/addressing.md` complete |
| 1 | **Base and Layer 2:** cabling, hostnames, SSH/passwords, VLANs, trunks (native VLAN 999, allowed VLAN lists, DTP off), EtherChannel, unused ports shut | Member 2 | `show vlan brief`, `show interfaces trunk`, `show etherchannel summary` |
| 2 | **Inter-VLAN routing and DHCP:** SVIs on the core switches, DHCP relay, central DHCP server | Member 2 | PCs get addresses, ping across VLANs |
| 3 | **OSPF:** routed links and loopbacks, single area 0, default route from R1-Edge | Lead | `show ip ospf neighbor`, `show ip route` |
| 4 | **STP:** root bridge placement matched to HSRP active per VLAN | Lead | `show spanning-tree`, unplug an uplink |
| 5 | **HSRP and failover:** virtual gateways, priorities, preempt, tracking | Lead | `show standby brief`, power off MLS1 during a continuous ping |
| 6 | **Services and DMZ:** internal DNS, NAT/PAT, DMZ web/DNS/email servers | Member 4 | Outside PC opens the website, inside PCs reach the internet |
| 7 | **Smart campus and wireless:** APs and SSIDs, IoT server, devices registered, automation rules | Member 4 | Dashboard shows all devices, rules fire in simulation |
| 8 | **Security:** segmentation ACLs, port security, VTY access restricted to the Management VLAN | Member 3 | `show access-lists`, `show port-security`, blocked pings fail |
| 9 | **(Optional) VPN** | Member 3 | `show crypto isakmp sa`, `show crypto ipsec sa` |
| 10 | **Test, document, prepare viva** | Everyone | Results table complete, screenshots taken |

**After every stage, the owner teaches it to the other three.** The viva can ask any member about any part.

## 8. Testing

Results are recorded in [`docs/test-plan.md`](docs/test-plan.md) as: **Test case | Expected | Actual | Pass/Fail | Notes**, with a screenshot for each.

| Category | Tests |
|---|---|
| Connectivity | DHCP assignment per VLAN, same-VLAN ping, inter-VLAN ping, internet access, DMZ website from outside |
| Fault tolerance | Unplug an access uplink (STP), unplug one EtherChannel cable, power off the active HSRP switch, shut a routed link (OSPF). Record pings lost and recovery time |
| Security | Guest blocked from internal VLANs, Students blocked from Faculty, IoT device blocked from user VLANs, rogue laptop triggers port security, Telnet refused and SSH allowed, outside PC can't reach internal hosts |
| Smart campus | Motion turns lights on, high temperature turns AC on, smoke triggers the alarm, door lock controlled from the dashboard, Student PC can't open the IoT dashboard |

## 9. Viva demo script (about 5 minutes)

1. **DHCP:** a new PC gets an address automatically. Explain DORA (Discover, Offer, Request, Acknowledge).
2. **Inter-VLAN routing:** ping between VLANs in **Simulation mode** so the packet path is visible.
3. **Failover:** start a continuous ping, power off the active HSRP switch, and show the ping recovering.
4. **Segmentation:** a Guest PC can't reach Faculty. A rogue laptop shuts its port down.
5. **DMZ:** an outside PC opens the campus website but can't reach internal VLANs.
6. **Smart campus:** trigger motion and smoke sensors, show the lights and alarm respond, and control the door lock from the dashboard.

## 10. Viva checklist

Every member can explain, for each topic: **what problem it solves, how it works, and which command verifies it.**

VLANs and 802.1Q trunking · VTP (transparent mode) · Subnetting and VLSM · DHCP (DORA) and DHCP relay · Inter-VLAN routing (SVIs vs router-on-a-stick) · STP and root bridge election · EtherChannel (LACP) · OSPF (neighbors, router ID, cost, areas) · HSRP (active/standby, virtual IP, preempt) · NAT/PAT · DMZ · ACLs (standard vs extended, placement) · Port security · SSH vs Telnet · IoT segmentation and the IoT server · Wireless SSIDs and VLAN mapping · (optional) IPsec VPN

Questions and model answers are kept in [`docs/viva-notes.md`](docs/viva-notes.md).

## 11. Deliverables

- Packet Tracer file (final + one saved copy per stage), saved in the lab's Packet Tracer version and kept in two places
- Logical and physical topology diagrams, with device names and VLAN/IP labels
- IP addressing plan and VLAN table
- Device configs, one file per device
- Test results table with screenshots
- Report: problem, design, implementation per stage, testing, limitations, future scope
- Viva notes

## 12. Known limitations

- **Single edge router and single ISP:** inside the campus there is no single point of failure, but R1-Edge and the ISP link are one. Accepted to keep scope manageable (see Future scope).
- **Single-homed end devices:** each PC connects to one access switch, so that switch failing affects its users. Normal for campus networks.
- **Simulated timing:** Packet Tracer doesn't model real hardware timing, so recovery times are relative comparisons, not real-world figures.
- **Packet Tracer feature support:** some IOS features are missing or limited. Anything we couldn't build is listed here, not claimed in the report.

## 13. Future scope

- Second edge router and dual ISP with automatic failover
- Multi-area OSPF for a larger campus with more buildings
- Site-to-site VPN to a branch campus (if not completed in stage 9)
- Centralized authentication (AAA with RADIUS/TACACS+)
- Monitoring: Syslog, SNMP, NTP, NetFlow
- QoS for voice and video, with a Voice VLAN
- IPv6 dual stack
- Cost-benefit analysis of redundancy

## 14. Repository layout

```
/
├── README.md
├── CLAUDE.md
├── docs/
│   ├── addressing.md      # IP plan, VLAN table, device list
│   ├── build-guide.md     # per-stage steps, command explanations, verification
│   ├── topology.md        # topology description, diagram notes
│   ├── test-plan.md       # test cases and results
│   ├── report.md          # final report draft
│   └── viva-notes.md      # per-topic notes and Q&A
├── configs/               # one file per device
├── packet-tracer/         # stage1.pkt, stage2.pkt, ...
└── screenshots/
```
