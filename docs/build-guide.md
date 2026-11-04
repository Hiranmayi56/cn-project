# Build Guide

Step-by-step instructions for each build stage: what to do in Packet Tracer, which config to paste, **what every command does and why**, how to verify it, and what to do when it doesn't work.

Addresses and ports come from [addressing.md](addressing.md). Viva questions for each stage are in [viva-notes.md](viva-notes.md).

---

## Stage 1: Base and Layer 2

**Owner:** Member 2 (all four members must understand it)

**Goal:** every switch is named, secured, and reachable by SSH. VLANs exist, trunks and the EtherChannel are up, and every access port is in the right VLAN. At the end, the admin PC can SSH into all 6 campus switches.

### 1.1 Terms used in this stage

| Term | Meaning |
|---|---|
| **VLAN** (Virtual LAN) | A separate Layer 2 network created inside a switch. Devices in different VLANs can't talk to each other without a router, even when plugged into the same switch. |
| **Access port** | A switch port that belongs to exactly one VLAN. End devices (PCs, servers, APs) plug into access ports and never see VLAN tags. |
| **Trunk** | A link that carries many VLANs at once, usually between switches. |
| **802.1Q** | The standard for trunks: it inserts a 4-byte **tag** holding the VLAN ID into each frame, so the far switch knows which VLAN the frame belongs to. |
| **Native VLAN** | The one VLAN on a trunk whose frames travel **untagged**. We set it to an unused VLAN (999) for security. |
| **DTP** (Dynamic Trunking Protocol) | A Cisco protocol that lets ports negotiate whether to become trunks. We turn it off, so a plugged-in device can't trick a port into trunking. |
| **EtherChannel** | Bundles several physical links into one logical link (a **port-channel**). Adds bandwidth, and the link survives one cable failing. |
| **LACP** (Link Aggregation Control Protocol) | The open standard (IEEE 802.3ad) that two switches use to agree on forming an EtherChannel. |
| **VTP** (VLAN Trunking Protocol) | A Cisco protocol that copies VLANs between switches automatically. We use **transparent** mode, so each switch keeps its own VLAN list and can't be wiped by another switch. |
| **SVI** (Switched Virtual Interface) | A virtual interface for a VLAN (`interface vlan 99`). In this stage it gives each switch a management IP address. |
| **SSH** (Secure Shell) | Encrypted remote login. It replaces Telnet, which sends passwords in plain text. |

### 1.2 Build the topology (Packet Tracer GUI)

1. **Place devices:**
   - 2× 3650-24PS: `MLS1-Core`, `MLS2-Core`
   - 5× 2960-24TT: `SW1-Access-Admin`, `SW2-Access-Acad`, `SW3-Access-Lib`, `SW-Servers`, `SW-DMZ`
   - 2× 2911: `R1-Edge`, `ISP` (configured in Stage 3)
   - Servers: 3 on SW-Servers (DHCP, DNS, IoT), 3 on SW-DMZ (web, DNS, email)
   - End devices: 1 PC per VLAN per building for testing, plus the admin PC on SW1-Access-Admin Fa0/13
   - Access points and IoT devices can wait until Stage 7.
2. **Power the 3650s:** a new 3650 has no power supply, so its ports stay red. Click the switch → **Physical** tab → drag **AC-POWER-SUPPLY** into the empty bay. If it won't go in, turn the power switch off, insert it, and turn the power back on.
3. **Add serial modules to the routers now** (needed in Stage 3): on R1-Edge and ISP, **Physical** tab → power off → drag **HWIC-2T** into an empty slot → power on.
4. **Rename the icons:** Config tab → Display Name, using the exact device names above. Screenshots will match the configs.
5. **Cable using the port plan** ([addressing.md section 9](addressing.md#9-port-plan)). Pick the cable type yourself. Don't use the "automatically choose connection type" cable: it picks random ports.
   - **Switch ↔ switch:** Copper Cross-Over
   - **Switch ↔ PC, server or router:** Copper Straight-Through
   - **R1-Edge ↔ ISP:** Serial DCE. **Click the ISP router first**, so the ISP end becomes the DCE (the side that sets the clock rate).

   | From | To | Cable |
   |---|---|---|
   | MLS1-Core G1/0/1 | MLS2-Core G1/0/1 | Cross-over |
   | MLS1-Core G1/0/2 | MLS2-Core G1/0/2 | Cross-over |
   | MLS1-Core G1/0/3 | SW1-Access-Admin G0/1 | Cross-over |
   | MLS2-Core G1/0/3 | SW1-Access-Admin G0/2 | Cross-over |
   | MLS1-Core G1/0/4 | SW2-Access-Acad G0/1 | Cross-over |
   | MLS2-Core G1/0/4 | SW2-Access-Acad G0/2 | Cross-over |
   | MLS1-Core G1/0/5 | SW3-Access-Lib G0/1 | Cross-over |
   | MLS2-Core G1/0/5 | SW3-Access-Lib G0/2 | Cross-over |
   | MLS1-Core G1/0/6 | SW-Servers G0/1 | Cross-over |
   | MLS2-Core G1/0/6 | SW-Servers G0/2 | Cross-over |
   | MLS1-Core G1/0/24 | R1-Edge G0/0 | Straight-through |
   | MLS2-Core G1/0/24 | R1-Edge G0/1 | Straight-through |
   | R1-Edge G0/2 | SW-DMZ G0/1 | Straight-through |
   | ISP S0/0/0 | R1-Edge S0/0/0 | Serial DCE (start at ISP) |

   End devices go on the access ports listed in each config file's header.
6. **Set up the admin PC:** Desktop → IP Configuration → Static. IP `10.10.3.110`, mask `255.255.255.224`, gateway `10.10.3.97`, DNS `10.10.3.71`.

### 1.3 Paste the configs

Paste in this order: the core switches first, then the others.

1. [mls1-core.txt](../configs/mls1-core.txt)
2. [mls2-core.txt](../configs/mls2-core.txt)
3. [sw1-access-admin.txt](../configs/sw1-access-admin.txt)
4. [sw2-access-acad.txt](../configs/sw2-access-acad.txt)
5. [sw3-access-lib.txt](../configs/sw3-access-lib.txt)
6. [sw-servers.txt](../configs/sw-servers.txt)
7. [sw-dmz.txt](../configs/sw-dmz.txt)

**How to paste:**
- Click the switch → **CLI** tab. If asked *"Would you like to enter the initial configuration dialog?"*, type `no`.
- Copy **one `--- 1.x` section at a time** and paste it with right-click → Paste (or Ctrl+V). Packet Tracer sometimes drops lines from a very long paste.
- Watch for `% Invalid input detected`. Each file notes the two places where that error is expected and harmless.
- The last lines (`end`, `write memory`) save the config. Without them, a device reload loses everything.

**Logging in afterwards:** the console now asks for a username. It's `netadmin`, password `RVU@admin1`. The `enable` password is `RVU@enable1`. These are lab passwords; use the same ones on every device so nobody gets locked out.

**Expect a short wait after pasting.** Trunks, the EtherChannel and spanning tree take up to about 50 seconds to settle. Port lights go orange (still deciding) and then green. Click **Fast Forward Time** at the bottom of the window to speed this up.

### 1.4 What each command does, and why

Section numbers match the `--- 1.x` headings in the config files.

**1.1 Identity**

| Command | What it does | Why |
|---|---|---|
| `hostname MLS1-Core` | Names the device. The name appears in the prompt. | You always know which device you're typing on. SSH key generation also requires it. |
| `no ip domain-lookup` | Stops the switch treating a mistyped word as a hostname to look up | Without it, every typo freezes the CLI for about 30 seconds |

**1.2 Passwords and local account**

| Command | What it does | Why |
|---|---|---|
| `enable secret RVU@enable1` | Password for privileged (`#`) mode, stored as an MD5 hash (shown as type 5) | Never use `enable password`: it's stored in plain text |
| `service password-encryption` | Scrambles any remaining plain-text passwords in the config (type 7) | Type 7 is easy to reverse, but it stops someone reading passwords over your shoulder |
| `username netadmin privilege 15 secret RVU@admin1` | Creates a local user account with full privileges, password hashed | SSH needs a username and password. Individual accounts are better than one shared line password. |
| `banner motd # ... #` | Message shown before login | A legal warning that the system is monitored. The `#` marks where the text starts and ends. |

**1.3 SSH**

| Command | What it does | Why |
|---|---|---|
| `ip domain-name rvu-campus.local` | Sets a domain name | The RSA key is named *hostname.domain*, so both must exist before generating it |
| `crypto key generate rsa general-keys modulus 1024` | Creates the RSA key pair SSH uses for encryption | SSH can't run without a key. SSH version 2 needs at least 768 bits; 1024 is the usual lab size. |
| `ip ssh version 2` | Allows only SSH version 2 | Version 1 has known weaknesses |
| `line console 0` → `login local` | The console asks for a local username and password | Anyone with physical access to the console port must still log in |
| `logging synchronous` | Status messages no longer cut into what you're typing | Pure convenience |
| `exec-timeout 15 0` | Logs out an idle session after 15 minutes (10 for remote sessions) | An unattended open session is a security risk |
| `line vty 0 15` | VTY lines are the 16 virtual terminal lines used for remote logins | Remote access is configured here |
| `transport input ssh` | Allows only SSH on the VTY lines, so Telnet is refused | Telnet sends everything, including passwords, in plain text |

**1.4 VLANs**

| Command | What it does | Why |
|---|---|---|
| `vtp mode transparent` | The switch keeps its own VLAN list and ignores VTP updates from other switches | A VTP mistake can delete VLANs on every switch at once. Transparent mode also stores VLANs in the running-config, so they appear in our config backups. |
| `vlan 10` → `name STUDENTS` | Creates the VLAN and names it | Names make `show vlan brief` readable |
| `vlan 998` (BLACKHOLE) | A VLAN with no gateway that goes nowhere | Unused ports go here. If one is ever enabled by mistake, the device on it can't reach anything. |
| `vlan 999` (NATIVE-UNUSED) | A VLAN that no device uses | Used as the trunks' native VLAN (see 1.6) |

Each access switch only creates the VLANs it actually uses, matching [addressing.md](addressing.md). Fewer VLANs means less to go wrong and less to explain.

**1.5 EtherChannel (core switches only)**

| Command | What it does | Why |
|---|---|---|
| `interface range g1/0/1 - 2` | Configures both ports at once | Every member of a bundle must have identical settings |
| `shutdown` … `no shutdown` | Keeps the ports down while the bundle is built, then brings them up together | Avoids a moment where the two links are half-configured and mismatched |
| `channel-group 1 mode active` | Adds the ports to port-channel 1 and **actively starts** LACP negotiation | `active` sends LACP messages; `passive` only answers them. Active on both ends is the safest choice. |
| `interface port-channel 1` | The single logical interface representing the bundle | From now on, configure and verify trunk settings on **Po1**, not on the member ports |

**1.6 Trunks**

| Command | What it does | Why |
|---|---|---|
| `switchport trunk encapsulation dot1q` | Selects 802.1Q tagging (core switches only) | Some multilayer switches also support the old Cisco ISL method and need this first. If the 3650 rejects it, it only supports 802.1Q anyway. |
| `switchport mode trunk` | Forces the port to be a trunk | Fixed configuration, nothing left to negotiation |
| `switchport trunk native vlan 999` | Untagged frames on this trunk belong to VLAN 999 | Blocks **VLAN hopping** (an attacker sending double-tagged frames from the native VLAN). No device is in VLAN 999. **Must match on both ends** of the trunk. |
| `switchport trunk allowed vlan 20,50,99` | Only these VLANs cross this trunk | A building only receives the VLANs it uses. Less broadcast traffic, and less exposure if a switch is compromised. |
| `switchport nonegotiate` | Turns off DTP | A device plugged into the port can't negotiate a trunk |

**1.7 Access ports and unused ports**

| Command | What it does | Why |
|---|---|---|
| `switchport mode access` | The port is always an access port (also turns off DTP) | End devices never need trunks |
| `switchport access vlan 20` | Puts the port in VLAN 20 | This is what actually places a PC into a department |
| `description ...` | A label on the port | `show` output says what each port is for |
| Unused ports: `switchport access vlan 998` + `shutdown` | Disables the port and parks it in BLACKHOLE | A free wall socket can't be used to join the network |

**1.8 Management IP**

| Command | What it does | Why |
|---|---|---|
| `interface vlan 99` → `ip address ...` | Gives the switch an IP address in the Management VLAN | You need an IP to SSH into a switch. Management stays in its own VLAN, away from user traffic, instead of default VLAN 1. |
| `ip default-gateway 10.10.3.97` | Where the switch sends traffic for other subnets (access switches only) | Lets admins in other subnets manage it. 10.10.3.97 is the HSRP virtual IP, which starts answering in Stage 5. |

SW-DMZ is the exception: it's managed through VLAN 60 with R1-Edge as its gateway, because it isn't connected to the core switches.

### 1.5 Verify

Run these on each switch from the CLI tab (in privileged mode, prompt ending in `#`).

**On every switch**

| Command | What correct output looks like |
|---|---|
| `show vlan brief` | Your VLANs listed by number and name. Access ports appear next to the right VLAN (e.g. Fa0/1–8 under VLAN 20 on SW1-Access-Admin), and unused ports under VLAN 998. **Trunk ports don't appear here**, because they carry all their allowed VLANs. |
| `show interfaces trunk` | Each trunk listed with Mode `on`, Encapsulation `802.1q`, Status `trunking`, Native vlan `999`. "Vlans allowed on trunk" matches the config. |
| `show ip interface brief` | `Vlan99` with its IP, Status `up`, Protocol `up`. Unused ports show `administratively down`. |
| `show ip ssh` | `SSH Enabled - version 2.0` |
| `show vtp status` | `VTP Operating Mode : Transparent` |
| `show running-config` | Passwords show as `secret 5 $1$...` (hashed), never as readable text |

Example of a healthy trunk table on SW2-Access-Acad:
```
Port        Mode         Encapsulation  Status        Native vlan
Gig0/1      on           802.1q         trunking      999
Gig0/2      on           802.1q         trunking      999

Port        Vlans allowed on trunk
Gig0/1      10,20,50,99
Gig0/2      10,20,50,99
```
In the last section, *"Vlans in spanning tree forwarding state"*, one uplink may list fewer VLANs, or none. **That's correct.** Spanning tree (STP) is blocking one path to prevent a loop. You'll choose which path it blocks in Stage 4.

**On MLS1-Core and MLS2-Core only**

| Command | What correct output looks like |
|---|---|
| `show etherchannel summary` | `Po1(SU)` with protocol `LACP`, and both `Gig1/0/1(P)` and `Gig1/0/2(P)` |

```
Group  Port-channel  Protocol    Ports
------+-------------+-----------+------------------------------
1      Po1(SU)           LACP   Gig1/0/1(P) Gig1/0/2(P)
```
The flags: **S** = Layer 2, **U** = in use, **P** = port is bundled. In `show interfaces trunk`, the core switches list `Po1` instead of G1/0/1 and G1/0/2.

**From the admin PC** (Desktop → Command Prompt)

| Test | Expected |
|---|---|
| `ping 10.10.3.98`, `.99`, `.101`, `.102`, `.103`, `.104` | Replies from all six. The first ping to each may time out once while the PC learns the MAC address (ARP). |
| `ssh -l netadmin 10.10.3.101`, then the password | The banner, then the `SW1-Access-Admin>` prompt. Type `exit` and repeat for each IP above. |
| `telnet 10.10.3.101` | **Fails** (connection refused or closed), because only SSH is allowed |

SW-DMZ can't be reached from the admin PC until R1-Edge routes to the DMZ in Stage 6. For now, check it from its console with the commands above.

**Break it on purpose**
1. Start `ping -t 10.10.3.99` (continuous ping to MLS2-Core) on the admin PC.
2. Delete one of the two cables between MLS1-Core and MLS2-Core.
3. The ping should keep going, perhaps with one lost reply. `show etherchannel summary` now shows one member down, but Po1 stays up.
4. Put the cable back: the port rejoins the bundle with `(P)`.

### 1.6 Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| 3650 ports stay red | No power supply | Add AC-POWER-SUPPLY in the Physical tab |
| `% Invalid input` on `switchport trunk encapsulation dot1q` | This switch only supports 802.1Q | Harmless. Skip the line. |
| `Po1(SD)`, members show `(I)` or `(s)` | The other core switch isn't configured yet, or the member ports have different settings | Configure both sides. Compare `show running-config` for G1/0/1–2 on both switches. |
| Message `%CDP-4-NATIVE_VLAN_MISMATCH` | One end of a trunk still uses native VLAN 1 | Check `switchport trunk native vlan 999` on both ends |
| Trunk missing from `show interfaces trunk` | Port not set to trunk, cable in the wrong port, or port shut down | Check the cable against the port plan, then `show running-config` for that interface |
| `Vlan99` is `down` | No working port carries VLAN 99 (trunks down, or 99 not in the allowed list) | Fix the trunks first. The SVI comes up as soon as one port carrying VLAN 99 is up. |
| Admin PC can't ping any switch | PC in the wrong VLAN, wrong IP or mask, or Fa0/13 shut down | On SW1-Access-Admin, `show vlan brief` must show Fa0/13 under VLAN 99. Recheck the PC's IP settings. |
| Ping works but SSH says connection refused | No RSA key (hostname or domain name was missing), or VTY lines not set | `show ip ssh`. If it's disabled, re-run the `ip domain-name` and `crypto key generate` lines. |
| SSH login rejected | Wrong username or password | Username `netadmin`, password `RVU@admin1`, case-sensitive |
| A VLAN's ports work on one switch but not across buildings | VLAN missing from the trunk's allowed list on one side, or not created on one switch | Compare `show interfaces trunk` and `show vlan brief` on both ends |

### 1.7 Done checklist

- [ ] `show vlan brief` correct on all 7 switches
- [ ] All trunks `trunking` with native VLAN 999 and the right allowed list
- [ ] `Po1(SU)` with both members `(P)` on both core switches
- [ ] Admin PC pings and SSHes into all 6 campus switches; Telnet is refused
- [ ] SW-DMZ checked from its console
- [ ] EtherChannel cable-pull test done
- [ ] Saved as `packet-tracer/stage1.pkt`
- [ ] `show running-config` from each switch saved into `configs/` if anything was changed by hand
- [ ] Member 2 has taught Stage 1 to the other three, using [viva-notes.md](viva-notes.md)
