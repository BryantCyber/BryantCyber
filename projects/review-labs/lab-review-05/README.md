# Lab Review 05 - Standard and Extended Access Control Lists

## Objective

Configure standard and extended Access Control Lists from scratch to filter traffic based on source IP, destination IP, protocol, and port number. Apply ACLs to physical interfaces and VTY lines, edit named ACLs using sequence numbers, and verify traffic filtering using show commands and ping tests.

## Devices Configured

| Device | Type | Role |
|---|---|---|
| CORP-R1 | Cisco ISR | ACL enforcement point, gateway router |
| CORP-R2 | Cisco ISR | Simulated internet router |

## Topology

![Lab-05 Topology](./images/review-lab05-top.png)

## Addressing

| Device | Interface | IP Address | Purpose |
|---|---|---|---|
| R1 | G0/0.10 | 192.168.10.1/24 | VLAN 10 gateway |
| R1 | G0/0.20 | 192.168.20.1/24 | VLAN 20 gateway |
| R1 | G0/1 | 203.0.113.1/30 | Uplink to R2 |
| R2 | G0/0 | 203.0.113.2/30 | Connected to R1 |
| R2 | Lo0 | 8.8.8.8/32 | Simulated web server |
| PC1 | NIC | 192.168.10.10/24 | VLAN 10 test host, blocked by standard ACL |
| PC2 | NIC | 192.168.10.20/24 | VLAN 10 test host, permitted |
| PC3 | NIC | 192.168.20.10/24 | VLAN 20 test host |
| PC4 | NIC | 192.168.20.20/24 | VLAN 20 test host |

---

## Full Configuration

### Part 1 - Standard ACL Blocking PC2 from Reaching R2

```
access-list 10 deny host 192.168.20.13
access-list 10 permit any

interface GigabitEthernet0/1
 ip access-group 10 out
```

### Part 2 - Named Extended ACL for Internet Traffic Policy

```
ip access-list extended INTERNET_POLICY
 permit tcp 192.168.20.0 0.0.0.255 host 8.8.8.8 eq 80
 permit tcp 192.168.20.0 0.0.0.255 host 8.8.8.8 eq 443
 deny tcp any any eq 23
 permit icmp any any
 permit ip any any

interface GigabitEthernet0/0.10
 ip access-group INTERNET_POLICY in
```

### Part 3 - Adding a Rule to an Existing Named ACL Using Sequence Numbers

```
ip access-list extended INTERNET_POLICY
 15 deny tcp 192.168.20.0 0.0.0.255 any eq 21
```

### Part 4 - ACL Applied to VTY Lines

```
access-list 20 permit host 192.168.99.5
access-list 20 deny any

line vty 0 4
 access-class 20 in
```

### Part 5 - Removing an ACL Completely

```
no ip access-group INTERNET_POLICY in - on interface. 

no ip access-list extended INTERNET_POLICY - global config. 
```

---

## Most Important Commands and What They Do

| Command | Purpose |
|---|---|
| `access-list 10 deny host 192.168.20.10` | Creates a standard ACL entry denying one specific host |
| `access-list 10 permit any` | Permits all traffic not matched by previous deny statements |
| `ip access-list extended INTERNET_POLICY` | Creates a named extended ACL with a descriptive identifier |
| `permit tcp src wildcard dst wildcard eq 80` | Permits HTTP traffic from a specific source to a specific destination |
| `deny tcp any any eq 23` | Blocks Telnet from all sources to all destinations |
| `permit icmp any any` | Allows ping testing from all sources to all destinations |
| `permit ip any any` | Permits all remaining IP traffic not matched by earlier statements |
| `ip access-group NAME in` | Applies an ACL inbound on an interface filtering arriving traffic |
| `ip access-group NAME out` | Applies an ACL outbound on an interface filtering leaving traffic |
| `ip access-class NAME in` | Applies an ACL to VTY lines to restrict remote management access |
| `show access-lists` | Displays all ACLs with match counters for each statement |
| `show ip interface G0/0.10` | Shows which ACL is applied inbound and outbound on an interface |
| `no ip access-group NAME in` | Removes an ACL from an interface before the ACL itself can be deleted |

---

## Key Concepts Reviewed

**Standard ACL placement rule and why it exists:**
A standard ACL filters traffic based on source IP address only. It has no ability to specify a destination, protocol, or port. Because of this limitation, applying a standard ACL near the source would block the filtered traffic from reaching every network beyond that point, not just the intended destination. Placing it close to the destination means the traffic can still reach other networks freely and is only stopped at the last possible point before the target network. This is not just a rule to memorize but a logical consequence of how source-only filtering works.

**Extended ACL placement rule and why it differs:**
An extended ACL can specify both source and destination along with protocol and port. Because the destination is part of the filter criteria, the ACL knows exactly which traffic to block without affecting flows to other destinations. Placing it close to the source is therefore safe and desirable because it drops unwanted traffic as early as possible, preventing it from consuming bandwidth traversing the network before being dropped later.

**The implicit deny and why permit any is always required:**
Every ACL on a Cisco device ends with an invisible deny any statement that is not shown in the configuration. Any traffic that reaches the end of the ACL without matching a permit statement is silently dropped. This is why every ACL that is not intended to block all unmatched traffic must end with either `permit any` for standard ACLs or `permit ip any any` for extended ACLs. Forgetting this line is one of the most common ACL misconfigurations and results in all traffic being blocked, which is always confusing when the ACL looks correct but connectivity is broken.

**Named ACLs versus numbered ACLs:**
Numbered ACLs use a number to identify them. Standard numbered ACLs use ranges 1-99 and 1300-1999. Extended numbered ACLs use ranges 100-199 and 2000-2699. Named ACLs use a descriptive text identifier instead, which makes them immediately readable in the configuration without needing to remember which number corresponds to which policy. Named ACLs also support in-place editing using sequence numbers, which is not available with numbered ACLs. In a professional environment named ACLs are always preferred because they are self-documenting and maintainable.

**Editing a named ACL using sequence numbers:**
When a named ACL is created each statement automatically receives a sequence number starting at 10 and incrementing by 10. To insert a new rule between existing statements, specify a sequence number that falls between the two surrounding entries. For example, to insert a rule between sequence 10 and sequence 20, use sequence 15. The router inserts the new statement at the correct position in the ACL without requiring the entire ACL to be deleted and rewritten. This is a critical operational skill for managing ACLs in production networks where deleting an active ACL to rewrite it would cause an outage.

**The difference between ip access-group and ip access-class:**
`ip access-group` applies an ACL to a physical or logical interface and filters all IP traffic transiting through that interface. `ip access-class` applies an ACL specifically to VTY lines and filters only management connections attempting to reach the router itself via Telnet or SSH. The two commands are not interchangeable. Applying an ACL with `ip access-group` to a physical interface does not restrict SSH access to the router's VTY lines, and applying `ip access-class` to VTY lines does not filter transit traffic. Each command operates at a completely different level of the device.

**Why both steps are required to fully remove an ACL:**
An ACL that is still applied to an interface with `ip access-group` cannot be seen as removed from a network policy perspective even if the ACL definition itself is deleted. The interface reference remains, and depending on the IOS version, this may result in a configuration that either permits all traffic or denies all traffic in an unpredictable way. The correct removal sequence is always: remove the ACL from the interface first with `no ip access-group`, then delete the ACL definition with `no ip access-list`. Skipping the first step leaves a dangling reference that can cause unexpected behavior.

**How the ACL match counter works and why it matters:**
Every statement in an ACL maintains a hit count that increments each time a packet matches that specific statement. This counter is visible in `show access-lists` and is one of the most useful troubleshooting tools in networking. A counter that is not incrementing when traffic should be hitting it confirms the traffic is either not reaching the ACL, not matching the statement as written, or being matched by an earlier statement before it reaches the one being examined. A counter that is incrementing unexpectedly confirms that traffic you did not intend to match is being caught by that statement.

---

## Extended ACL Syntax Reference

The order of fields in an extended ACL statement is fixed and must be followed exactly:

```
permit/deny   protocol   source   source-wildcard   destination   destination-wildcard   eq port
```

Common protocol keywords and port numbers:

| Service | Protocol | Port |
|---|---|---|
| HTTP | tcp | 80 |
| HTTPS | tcp | 443 |
| Telnet | tcp | 23 |
| SSH | tcp | 22 |
| DNS | udp | 53 |
| DHCP server | udp | 67 |
| DHCP client | udp | 68 |
| SNMP | udp | 161 |
| Ping | icmp | n/a |
| All IP traffic | ip | n/a |

---

## ACL Placement Decision Reference

| ACL Type | Filter criteria | Place it | Reason |
|---|---|---|---|
| Standard | Source IP only | Close to destination | Cannot specify destination so must be placed near target to avoid over-blocking |
| Extended | Source, destination, protocol, port | Close to source | Can specify exact destination so safe to drop early and save bandwidth |
| VTY ACL | Source IP only | Applied with access-class | Filters management access to the device itself, not transit traffic |

---

## Verification Commands

```
show access-lists
show ip interface GigabitEthernet0/0.10
show ip interface GigabitEthernet0/1
show running-config | section access-list
show running-config | section ip access-list
```

| Command | What it confirms |
|---|---|
| `show access-lists` | All ACL statements with current match counts per statement |
| `show ip interface` | Which ACL is applied inbound and outbound on each interface |
| `show running-config section access-list` | Numbered ACL definitions in the running configuration |
| `show running-config section ip access-list` | Named ACL definitions in the running configuration |

---

## Lessons Learned

The most important lesson from this lab is that ACL placement is not an arbitrary preference but a direct consequence of what the ACL can filter. Standard ACLs can only see the source address, so they must be positioned where the full list of affected destinations is narrowed down to exactly what you intend to block. Extended ACLs can see both endpoints of a conversation, so they should be positioned as early as possible in the path to conserve bandwidth by stopping unwanted traffic before it travels across the network.

The implicit deny at the end of every ACL is the single most common source of ACL-related outages in both real networks and lab environments. The symptom is always the same: an ACL is applied and suddenly all traffic stops rather than just the intended traffic. The fix is always the same: add a permit any or permit ip any any at the end. Building the habit of always writing this line before applying any ACL to an interface eliminates this category of mistake entirely.

Sequence numbers in named ACLs are a feature that is frequently overlooked by candidates who have only read about ACLs rather than configured them. The ability to insert a rule at sequence 15 between existing rules at 10 and 20 is not just a convenience, it is operationally essential because in a production network an active ACL cannot be deleted and rewritten without creating an outage during the window when no ACL is applied. Sequence-based insertion lets you modify a running policy without disrupting traffic.

The distinction between `ip access-group` and `ip access-class` trips up many candidates because both involve applying an ACL to control access, but they operate at completely different layers. One filters packets transiting the device. The other filters connections attempting to reach the device itself for management. Confusing the two results in either management access being unprotected when you thought it was, or transit traffic being unfiltered when you thought it was restricted.

Removing an ACL requires two steps in a specific order. Removing the ACL definition before removing it from the interface leaves a dangling reference. Always remove the interface binding first, then remove the definition. This sequence is tested on the exam and the order matters.