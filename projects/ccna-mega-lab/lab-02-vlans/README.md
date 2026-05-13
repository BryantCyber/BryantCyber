# Lab 02 - VLAN Creation, Access Ports, and Trunk Configuration

## Objective

Create VLANs on both switches, assign PC-facing ports to the correct VLAN, and configure trunk links between switches and toward the routers. This lab establishes Layer 2 traffic segmentation and confirms that VLAN separation is enforced at the switch level before any routing is introduced.

## Devices Used

| Device | Type | Role |
|---|---|---|
| SW1 | Cisco 2960 | Access layer switch for PC1 through PC4 |
| SW2 | Cisco 2960 | Access layer switch for PC5 through PC8 |
| R1 | Cisco ISR 4331 | Gateway router connected to SW1 |
| R2 | Cisco ISR 4331 | Gateway router connected to SW2 |
| PC1 through PC8 | End devices | Used for connectivity testing |

## Topology

SW1 connects to SW2 via a trunk link. SW1 connects to R1 and SW2 connects to R2 via trunk uplinks. R1 and R2 are connected via a point-to-point link. PC-facing ports are access ports.

![Topology](./Images/lab-topology.png)

---

## Addressing

### PC Addressing

| Device | IP Address | Subnet Mask | VLAN | Switch |
|---|---|---|---|---|
| PC1 | 192.168.16.2 | 255.255.255.248 | 10 | SW1 |
| PC2 | 192.168.16.3 | 255.255.255.248 | 10 | SW1 |
| PC3 | 192.168.16.10 | 255.255.255.248 | 20 | SW1 |
| PC4 | 192.168.16.11 | 255.255.255.248 | 20 | SW1 |
| PC5 | 192.168.16.2 | 255.255.255.248 | 10 | SW2 |
| PC6 | 192.168.16.3 | 255.255.255.248 | 10 | SW2 |
| PC7 | 192.168.16.10 | 255.255.255.248 | 20 | SW2 |
| PC8 | 192.168.16.11 | 255.255.255.248 | 20 | SW2 |

### Router Addressing

| Device | Interface | IP Address | Subnet Mask | Connected To |
|---|---|---|---|---|
| R1 | G0/0 | 192.168.16.6 | 255.255.255.248 | SW1 |
| R1 | G0/1 | 192.168.16.17 | 255.255.255.252 | R2 |
| R2 | G0/0 | 192.168.16.14 | 255.255.255.248 | SW2 |
| R2 | G0/1 | 192.168.16.18 | 255.255.255.252 | R1 |

## VLAN Design

| VLAN | Name | Purpose | Ports Assigned |
|---|---|---|---|
| 10 | SALES | User traffic | SW1 Fa0/1-2, SW2 Fa0/1-2 |
| 20 | HR | User traffic | SW1 Fa0/3-4, SW2 Fa0/3-4 |
| 99 | MANAGEMENT | Switch management SVI | None yet - configured in Lab 04 |
| 100 | NATIVE | Native VLAN for trunks | Trunk ports Fa0/5 and Fa0/6 |

## Tools Used

- Cisco Packet Tracer
- Cisco IOS CLI

---

## Configuration Steps

---

### Step 1 - VLAN Creation on SW1

Enter global configuration mode and create all four VLANs with names.

```
enable
configure terminal
vlan 10
 name SALES
vlan 20
 name HR
vlan 99
 name MANAGEMENT
vlan 100
 name NATIVE
exit
```

**Verify:**

```
show vlan brief
```

All four VLANs should appear as active. All ports will still show under VLAN 1 at this point. Port assignments happen in Steps 3 and 4.

![SW1 show vlan brief after VLAN creation](./Images/sw1-sh-vl-br.png)

---

### Step 2 - VLAN Creation on SW2

Repeat the exact same VLAN creation on SW2.

```
enable
configure terminal
vlan 10
 name SALES
vlan 20
 name HR
vlan 99
 name MANAGEMENT
vlan 100
 name NATIVE
exit
```

**Verify:**

```
show vlan brief
```

![SW2 show vlan brief after VLAN creation](./Images/sw2-sh-vl-br.png)

**Why VLANs must be created manually on each switch:**

VLANs do not automatically propagate between switches. Each switch maintains its own independent VLAN database. VTP (VLAN Trunking Protocol) can automate this propagation but manual creation is best practice because it gives full control and avoids accidental VLAN changes being pushed across the network.

---

### Step 3 - Access Port Assignment SW1

Assign PC-facing ports to their correct VLANs using the range command for efficiency.

```
configure terminal
interface range Fa0/1 - 2
 switchport mode access
 switchport access vlan 10
interface range Fa0/3 - 4
 switchport mode access
 switchport access vlan 20
exit
```

**Two commands required per port:**

| Command | Purpose |
|---|---|
| `switchport mode access` | Locks the port as an access port, disables trunking negotiation |
| `switchport access vlan 10` | Assigns the port to the specified VLAN |

**Access port vs trunk port:**
- Access port carries traffic for one VLAN only, no VLAN tags on frames
- Trunk port carries traffic for multiple VLANs simultaneously using 802.1Q tags

**Default VLAN:** All ports are in VLAN 1 by default until explicitly assigned.

**Verify:**

```
show vlan brief
```

VLAN 10 should now list Fa0/1 and Fa0/2. VLAN 20 should list Fa0/3 and Fa0/4.

![SW1 show vlan brief after port assignment](./Images/sw1-sh-vl-br.png)

---

### Step 4 - Access Port Assignment SW2

Repeat the same port assignments on SW2.

```
configure terminal
interface range Fa0/1 - 2
 switchport mode access
 switchport access vlan 10
interface range Fa0/3 - 4
 switchport mode access
 switchport access vlan 20
exit
```

**Verify:**

```
show vlan brief
```

![SW2 show vlan brief after port assignment](./Images/sw2-sh-vl-br.png)

**Why SW2 needs the same VLANs as SW1:**

When a trunk is configured between SW1 and SW2, frames tagged with VLAN 10 or VLAN 20 cross that link. If the receiving switch does not have those VLANs in its database it drops the frames. Both switches must have matching VLAN databases for inter-switch communication to work correctly.

---

### Trunk Configuration - SW1 to R1 (Fa0/5)

This uplink carries all VLAN traffic from SW1 up to R1. Router-on-a-stick subinterfaces will be configured on R1 in Lab 06. This trunk lays the groundwork now.

```
configure terminal
interface Fa0/5
 switchport mode trunk
 switchport trunk native vlan 100
 switchport nonegotiate
exit
```

| Command | Purpose |
|---|---|
| `switchport mode trunk` | Forces port into permanent trunk mode |
| `switchport trunk native vlan 100` | Sets VLAN 100 as the native VLAN (untagged traffic) |
| `switchport nonegotiate` | Disables DTP so the port never tries to auto-negotiate |

**What is DTP?** Dynamic Trunking Protocol is a Cisco proprietary protocol that automatically negotiates trunk links. Disabling it with `switchport nonegotiate` is best practice because it prevents unauthorized devices from negotiating a trunk and gaining access to all VLANs.

---

### Trunk Configuration - SW1 to SW2 (Fa0/6)

This interswitch trunk carries VLAN 10, 20, 99, and 100 traffic between SW1 and SW2.

**On SW1:**

```
configure terminal
interface Fa0/6
 switchport mode trunk
 switchport trunk native vlan 100
 switchport trunk allowed vlan 10,20,99,100
 switchport nonegotiate
exit
```

**On SW2 -- mirror the same config:**

```
configure terminal
interface Fa0/6
 switchport mode trunk
 switchport trunk native vlan 100
 switchport trunk allowed vlan 10,20,99,100
 switchport nonegotiate
exit
```

**Why limit allowed VLANs on a trunk?**
Allowing only the VLANs that are actually needed reduces the attack surface. If a VLAN is not in the allowed list its traffic is dropped at the trunk even if that VLAN exists in the database. This is a security and traffic management best practice.

---

### Save Configuration on Both Switches

```
end
copy running-config startup-config
```

---

## Verification

### Show Interfaces Trunk

Run on SW1 to confirm both trunk ports are active:

```
show interfaces trunk
```
![SW1 show interfaces trunk](./Images/sw1-sh-ints-tr.png)

**Reading the output:**

| Field | What it means |
|---|---|
| Mode: on | Port is permanently set to trunk, not negotiating |
| Encapsulation: 802.1q | Industry standard VLAN tagging is in use |
| Status: trunking | Trunk is active and passing traffic |
| Native vlan: 100 | Untagged frames belong to VLAN 100 |
| Vlans allowed on trunk | What is configured to be allowed |
| Vlans allowed and active | Allowed VLANs that actually exist in the database |
| Forwarding state | VLANs actively forwarding traffic after STP converges |

---

### Ping Tests

Four ping tests confirm VLAN segmentation and trunk operation are working correctly.

---

**Test 1 - Same VLAN, same switch (expected: SUCCESS)**

PC1 pings PC2. Both are in VLAN 10 on SW1. Layer 2 forwarding works within the same VLAN on the same switch.

```
PC1> ping 192.168.16.3
```

![PC1 ping PC2 success](./Images/pc1-pc2-ping.png)

---

**Test 2 - Same VLAN, different switch (expected: SUCCESS)**

PC1 pings PC5. Both are in VLAN 10 but on different switches. The trunk between SW1 and SW2 carries VLAN 10 traffic -- no router needed because they are in the same VLAN.

```
PC1> ping 192.168.16.2
```

![PC1 ping PC5 success](./Images/pc1-pc5-ping.png)

---

**Test 3 - Same VLAN 20, different switch (expected: SUCCESS)**

PC3 pings PC7. Both are in VLAN 20 on different switches. Confirms the trunk is carrying VLAN 20 correctly as well.

```
PC3> ping 192.168.16.10
```

![PC3 ping PC7 success](./Images/pc3-pc7-ping.png)

---

**Test 4 - Different VLAN, any switch (expected: FAIL)**

PC1 pings PC3. VLAN 10 to VLAN 20. No inter-VLAN routing configured yet. This correctly fails and confirms VLAN isolation is enforced.

```
PC1> ping 192.168.16.10
```

![PC1 ping PC3 fail](./Images/pc1-pc3-ping.png)

---

### Summary of Test Results

| Test | From | To | VLAN | Result | What it proves |
|---|---|---|---|---|---|
| 1 | PC1 | PC2 | Same (10) same switch | Success | VLAN 10 forwarding on SW1 works |
| 2 | PC1 | PC5 | Same (10) diff switch | Success | Trunk carrying VLAN 10 correctly |
| 3 | PC3 | PC7 | Same (20) diff switch | Success | Trunk carrying VLAN 20 correctly |
| 4 | PC1 | PC3 | Different (10 to 20) | Fail | VLAN isolation enforced, routing needed |

---

## Key Concepts

**What is a VLAN?**
A VLAN is a logical grouping of ports that creates a separate broadcast domain at Layer 2. Devices in different VLANs cannot communicate without a Layer 3 router regardless of whether they are on the same physical switch.

**Why can PC1 reach PC5 without a router?**
PC1 and PC5 are in the same VLAN (VLAN 10). Same-VLAN communication is Layer 2 forwarding -- the switch uses MAC address tables to forward frames. No routing is involved. The trunk between SW1 and SW2 carries the VLAN 10 tagged frames across the interswitch link transparently.

**What is 802.1Q?**
IEEE 802.1Q is the industry standard protocol for VLAN tagging on trunk links. It inserts a 4-byte tag into the Ethernet frame header identifying which VLAN the frame belongs to.

**What is the native VLAN?**
The native VLAN is the one VLAN whose traffic crosses a trunk link without an 802.1Q tag. Both sides of a trunk must agree on the native VLAN or a mismatch error occurs. Best practice is to set it to an unused VLAN -- in this lab VLAN 100 serves that purpose.

**What happens if a VLAN is not in the allowed list?**
Its traffic is dropped at the trunk port even if the VLAN exists in the database on both switches.

**Can VLAN 1 be deleted?**
No. VLAN 1 is the default VLAN and cannot be deleted or renamed on Cisco switches. Best practice is to move all traffic off VLAN 1 and leave it unused.

---

## Lessons Learned

- VLANs must be manually created on every switch -- they do not propagate automatically without VTP
- Two commands are always required for an access port: `switchport mode access` and `switchport access vlan`
- Same-VLAN communication across switches is Layer 2 forwarding, not routing -- the trunk carries it transparently
- Inter-VLAN communication requires a router -- a failed ping between VLANs confirms isolation is working correctly
- Trunk ports do not appear under any VLAN in `show vlan brief` -- use `show interfaces trunk` to verify them
- Disabling DTP with `switchport nonegotiate` is always best practice on trunk ports
- Setting the native VLAN to an unused VLAN (100) prevents VLAN hopping attacks
- Limiting allowed VLANs on a trunk to only what is needed reduces the attack surface
- Always verify with both `show vlan brief` and `show interfaces trunk` after configuration
