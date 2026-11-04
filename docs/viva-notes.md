# Viva Notes

Likely viva questions for each stage, with short model answers. **Every member should be able to answer every question, not only their own stage's.** Practice by quizzing each other on stages you don't own.

---

## Stage 1: Base and Layer 2

### VLANs

**Q: What is a VLAN and why did you use them?**
A VLAN is a separate Layer 2 network inside a switch. Devices in different VLANs can't talk directly, even on the same switch. We used them to separate Students, Faculty, Guests, IoT devices, servers and management, so each group can be given its own security rules and broadcasts stay within one group.

**Q: How do you check which VLAN a port is in?**
`show vlan brief`. It lists each VLAN with its access ports. Trunk ports don't appear there; we check those with `show interfaces trunk`.

**Q: Why didn't you use VLAN 1?**
VLAN 1 is the default for every port and carries some control traffic, so attackers target it first. We use VLAN 99 for management, VLAN 998 for unused ports, and VLAN 999 as the native VLAN, so nothing important sits in VLAN 1.

### Trunks and 802.1Q

**Q: What is a trunk, and how does the receiving switch know which VLAN a frame belongs to?**
A trunk is a link that carries several VLANs at once. 802.1Q adds a 4-byte tag containing a 12-bit VLAN ID to each frame. The receiving switch reads the tag, removes it, and forwards the frame only within that VLAN.

**Q: What is the native VLAN, and why did you change it to 999?**
It's the one VLAN whose frames cross a trunk untagged. The default is VLAN 1. An attacker in the native VLAN can send double-tagged frames to jump into another VLAN (VLAN hopping). By making the native VLAN an unused one with no devices, there's nobody in it to attack from.

**Q: Why restrict the allowed VLANs on each trunk?**
Each building only gets the VLANs it uses, e.g. the Library trunk carries 10, 40, 50 and 99 but not Faculty (20). This reduces unnecessary broadcast traffic and limits exposure if a switch is compromised.

**Q: What does `switchport nonegotiate` do?**
It turns off DTP, Cisco's trunk negotiation protocol. Without it, a device could pretend to be a switch and negotiate a trunk, gaining access to every VLAN.

### EtherChannel

**Q: What is EtherChannel and why use it between the core switches?**
It bundles several physical links into one logical link. We get the combined bandwidth of both cables, and if one cable fails the bundle stays up on the other. Spanning tree sees it as a single link, so it doesn't block one of the cables.

**Q: What's the difference between LACP `active` and `passive`?**
`active` sends LACP messages to start negotiation; `passive` only responds. Active–active or active–passive forms a bundle, but passive–passive never does, because neither side starts. We used active on both ends.

**Q: What does `Po1(SU)` mean in `show etherchannel summary`?**
Port-channel 1 is a Layer 2 bundle (S) that is in use (U). Member ports marked (P) are bundled. (D) would mean down, (I) or (s) a member that couldn't join the bundle.

### Device security

**Q: Why SSH instead of Telnet?**
Telnet sends everything, including usernames and passwords, in plain text, so anyone capturing traffic can read them. SSH encrypts the whole session. We enforce it with `transport input ssh` on the VTY lines.

**Q: What does SSH need before it works on a Cisco switch?**
A hostname, a domain name, an RSA key (`crypto key generate rsa`), a local username, and VTY lines set to `login local` and `transport input ssh`. `show ip ssh` confirms it's enabled.

**Q: What's the difference between `enable secret` and `service password-encryption`?**
`enable secret` stores the password as a one-way hash (type 5). `service password-encryption` only scrambles other plain-text passwords with a weak, reversible method (type 7). It's better than nothing, but not real protection.

**Q: Why are unused ports shut down and put in VLAN 998?**
So a free wall socket can't be used to join the network. Even if someone enables the port by mistake, VLAN 998 has no gateway and no other devices, so the device can't reach anything.

### VTP

**Q: Why is VTP in transparent mode?**
VTP can copy VLANs between switches automatically, but a switch with a higher revision number can overwrite, and even delete, VLANs on every other switch. In transparent mode, each switch keeps its own VLAN list, and the VLANs are saved in the running-config where we can see and back them up.
