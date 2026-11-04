# CLAUDE.md — Computer Networks CIE3 Mini Project

Read this file fully before doing anything in this repo. It explains what the project is, who we are, how we are graded, and how you should help.

**[README.md](README.md) is the detailed design spec** (topology, VLAN table, security policy, build plan, tests, demo script). This file summarizes it. If you change the design in one, update the other so they never disagree.

---

## 1. Project at a glance

- **Title:** Enterprise Network Design and Security: Implementation of a Fault-Tolerant Smart Campus Network
- **Domain:** Enterprise Network Design and Security
- **Course:** Computer Networks (CIE-3 Mini Project, 15 marks)
- **Tool:** Cisco Packet Tracer (software-only, no physical hardware)
- **Team size:** 4 students
- **Theme:** A smart campus network modeled on a university campus (RV University, Bengaluru), with 3 buildings (Administration, Academics, Library), segmentation, redundancy, layered security, and IoT-based smart campus services.

## 2. Grading context (this drives everything)

The faculty guidelines say:

- Objective: create and simulate a network using Cisco Packet Tracer.
- Marks are for the **project setup and equipment**, and for **each student's knowledge of the project and all associated technologies and protocols** (a viva).
- So a working topology is not enough. **Every team member must be able to explain every protocol in it**, and what each verification command shows.

**Implication for you (Claude Code):** optimize for our understanding, not just for a finished config. Explain the why behind every command.

## 3. About us (background)

- We are second-year B.Tech Computer Science and Engineering students (AIML major) at RV University, Bengaluru.
- Our background is mostly programming, AI/ML, and embedded systems. **Networking is not our strongest area yet**, so do not assume CCNA-level knowledge. Define terms the first time you use them.
- Our long-term goal is to become strong problem solvers and target top-tier tech companies, so we want real understanding rather than copy-paste configs.

<!-- TEAM: fill in below -->
| Member | Role | Owns |
|---|---|---|
| (Team lead) | Lead | Core routing and redundancy: OSPF, STP, HSRP |
| Member 2 | | VLANs, trunks, EtherChannel, inter-VLAN routing, DHCP |
| Member 3 | | Security: ACLs, port security, trunk hardening, (optional) VPN |
| Member 4 | | Servers, DMZ, NAT, wireless, smart campus IoT, documentation |

## 4. How to help us (important)

1. **Explain, don't just solve.** When we hit a bug, identify what is wrong and explain why, and point us to the fix. Give the full corrected config only if we ask.
2. **Be practical.** Prefer simple, cost-effective, low-risk choices. If there are tradeoffs (for example, VPN vs. extra ACLs), tell us which is more pragmatic and why.
3. **Teach in viva style.** After each feature, give us 2 or 3 likely viva questions with short model answers.
4. **Always include a verification command** (for example `show vlan brief`, `show ip ospf neighbor`, `show standby brief`, `show etherchannel summary`, `show access-lists`) and what correct output looks like.
5. **Keep scope tight.** Say so if something is over-scoped. Do not add features from the Future scope list (README section 13) unless we ask.
6. **Be honest about Packet Tracer limits.** Some IOS features and IoT behaviors are limited or behave differently there. Flag this rather than guessing.

## 5. What Claude Code can and cannot do here

Packet Tracer is a GUI app and its `.pkt` files are binary. **You cannot open or run them.** Your job is everything around the build:

- Generate **IOS configuration scripts** we paste into each device's CLI, one block per device, ready to copy.
- Maintain the **IP addressing plan and VLAN table** (`docs/addressing.md`).
- Give step-by-step GUI instructions for things that aren't CLI-configured in Packet Tracer (servers, IoT devices and automation rules, access points).
- Review config snippets we paste in, and find mistakes.
- Write **test plans** and expected results.
- Draft the **report and documentation**.
- Prepare **viva Q&A** and short learning notes per topic.

If we paste CLI output (`show` commands, errors, ping results), diagnose from that.

## 6. Planned design (scope)

Full detail is in README sections 3–6. Summary:

**Topology:** 3 buildings (Administration, Academics, Library) with a collapsed core/distribution layer, an access layer, a server block, a DMZ, and one edge router to the internet.

- **Core:** two multilayer switches (`MLS1-Core`, `MLS2-Core`, 3650) joined by an EtherChannel. They do inter-VLAN routing (SVIs), HSRP, DHCP relay and internal ACLs. We chose these over router-on-a-stick because a single router would be a single point of failure.
- **Access:** one 2960 per building (`SW1-Access-Admin`, `SW2-Access-Acad`, `SW3-Access-Lib`), each with one uplink to each core switch.
- **Edge:** `R1-Edge` (2911) runs NAT/PAT, outside ACLs and the DMZ, and connects to a simulated `ISP` router over a serial link (HWIC-2T module in both routers, because the 2911's 3 Gigabit ports are used by MLS1, MLS2 and the DMZ).
- **Server block:** `SW-Servers` (in Administration), uplinked to both core switches, holding the DHCP, internal DNS and IoT servers.
- **DMZ:** `SW-DMZ` off R1-Edge, holding the public web, DNS and email servers.
- **Routing:** OSPF single area 0 between MLS1, MLS2 and R1-Edge, with a default route from R1-Edge.

**VLANs**

| VLAN | Name | Where | HSRP active |
|---|---|---|---|
| 10 | STUDENTS | Academics, Library | MLS1 |
| 20 | FACULTY (faculty + admin staff) | Administration, Academics | MLS2 |
| 30 | SERVERS | Server block (Administration) | MLS1 |
| 40 | GUEST | Library (Guest Wi-Fi) | MLS2 |
| 50 | IOT | All 3 buildings | MLS1 |
| 60 | DMZ | SW-DMZ only (not trunked to the core) | — (R1-Edge is the gateway) |
| 99 | MANAGEMENT | All devices, admin PC in Administration | MLS2 |
| 998 | BLACKHOLE | Unused access ports, shut down | — |
| 999 | NATIVE-UNUSED | Trunk native VLAN only | — |

The STP root bridge for each VLAN is the same switch as its HSRP active router. Subnets are VLSM, recorded in `docs/addressing.md`.

**Smart campus features** (Packet Tracer IoT devices registered to the IoT server):
- **Academics:** smart classrooms (motion sensor turns lights and AC/fan on).
- **Library:** climate monitoring (temperature sensor turns AC on) and fire safety (smoke detector triggers alarm).
- **Administration:** server room smart door lock and fire safety.
- Central IoT dashboard opened from a Faculty PC. IoT devices are isolated in VLAN 50 and can reach only the IoT server.

**Security:** SSH-only management from VLAN 99, encrypted passwords, banners, port security, unused ports shut down, restricted trunks with an unused native VLAN, segmentation ACLs (Guest internet-only, IoT to IoT server only, Students blocked from Faculty/Management), NAT, outside ACLs allowing inbound only to the DMZ.

**Optional, only if time allows:** site-to-site VPN. Do not put it in the report unless it actually works.

**Out of scope (Future scope):** second edge router / dual ISP, multi-area OSPF, AAA (RADIUS/TACACS+), Syslog/SNMP/NTP/NetFlow, QoS and Voice VLAN, IPv6, cost analysis, BGP.

**Fault-tolerance demo (the "wow" moment):** during the viva, start a continuous ping, then unplug an uplink or power off the active HSRP switch, and show traffic failing over (ping recovers within seconds).

## 7. Build order (stage by stage, save a `.pkt` copy after each stage)

0. Plan: confirm device models and the lab's Packet Tracer version, user counts, VLSM addressing, diagram.
1. Base and Layer 2: cabling, hostnames, SSH/passwords, VLANs, trunks (native VLAN 999, allowed VLAN lists, DTP off), EtherChannel, unused ports shut.
2. Inter-VLAN routing (SVIs on the core switches) and DHCP relay.
3. OSPF single area, default route from R1-Edge.
4. STP root bridge placement matched to HSRP active.
5. HSRP, then test failover.
6. Services and DMZ: internal DNS, NAT/PAT, DMZ web/DNS/email.
7. Smart campus and wireless: APs and SSIDs, IoT server, devices, automation rules.
8. Security: segmentation ACLs, port security, VTY access restricted to the Management VLAN.
9. (Optional) VPN.
10. Final testing, screenshots, report, viva prep.

After each stage, the owner teaches it to the other three. If a stage breaks something, roll back to the previous saved copy instead of piling fixes on top.

## 8. Repo layout

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
├── configs/
│   ├── r1-edge.txt
│   ├── isp.txt
│   ├── mls1-core.txt
│   ├── mls2-core.txt
│   ├── sw1-access-admin.txt
│   ├── sw2-access-acad.txt
│   ├── sw3-access-lib.txt
│   ├── sw-servers.txt
│   └── sw-dmz.txt
├── packet-tracer/         # stage1.pkt, stage2.pkt ...
└── screenshots/
```

## 9. Conventions

- Device names are fixed and used identically in configs, docs, and the Packet Tracer file: `R1-Edge`, `ISP`, `MLS1-Core`, `MLS2-Core`, `SW1-Access-Admin`, `SW2-Access-Acad`, `SW3-Access-Lib`, `SW-Servers`, `SW-DMZ`.
- Every config file starts with a comment header: device, model, role, what it connects to.
- Use only the IOS commands supported by the Packet Tracer device models we use.
- Record every IP, VLAN, and HSRP virtual IP in `docs/addressing.md` **before** writing configs, so docs and configs never disagree.

## 10. Topics we must be able to explain (viva checklist)

For each, we need: what problem it solves, how it works, and which command verifies it.

VLANs and 802.1Q trunking · VTP (transparent mode) · Subnetting and VLSM · DHCP (DORA) and DHCP relay · Inter-VLAN routing (SVIs vs router-on-a-stick) · STP and root bridge election · EtherChannel (LACP) · OSPF (neighbors, router ID, cost, areas) · HSRP (active/standby, virtual IP, preempt) · NAT/PAT · DMZ · ACLs (standard vs extended, placement) · Port security · SSH vs Telnet · IoT segmentation and the IoT server · Wireless SSIDs and VLAN mapping · (optional) IPsec VPN

## 11. Open items (update as we go)

- [ ] Confirm team members and roles above
- [ ] Confirm the Packet Tracer version on the lab machines
- [ ] Confirm device models (2911, 3650-24PS, 2960-24TT) work in that version
- [ ] Check that the IoT devices and automation rules in section 6 work in that version
- [x] Get user counts per VLAN, then fill in `docs/addressing.md` (Students 500, Faculty 100, Guest 100, IoT 50)
- [ ] Confirm CIE-3 demo date with faculty
- [ ] Decide whether the VPN is in or out
