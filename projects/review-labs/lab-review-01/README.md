# Lab Review 01 - Device Hardening and Basic Security

## Objective

Configure foundational security settings on a router and switch from scratch without referencing previous labs. This lab reinforces the first thing a professional network engineer does when deploying any new device: harden it before connecting it to anything else.

## Devices Configured

| Device | Type | Hostname |
|---|---|---|
| Router | Cisco ISR | CORP-R1 |
| Switch | Cisco 2960 | CORP-SW1 |

## Topology

[Lab-01 Topology](./images/review-lab01-top
.png)

---

## Full Configuration

### CORP-R1

```
hostname CORP-R1
no ip domain-lookup
ip domain-name lab.local
enable secret CCNA2024!
username admin privilege 15 secret CCNA2024!
service password-encryption
banner motd #
Authorized access only. Unauthorized access is prohibited and may result in prosecution.
#
crypto key generate rsa modulus 1024
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
line con 0
 login local
 exec-timeout 5 0
 logging synchronous
line vty 0 4
 transport input ssh
 login local
 exec-timeout 5 0
```

### CORP-SW1

```
hostname CORP-SW1
no ip domain-lookup
ip domain-name lab.local
enable secret CCNA2024!
username admin privilege 15 secret CCNA2024!
service password-encryption
banner motd #
Authorized access only. Unauthorized access is prohibited and may result in prosecution.
#
crypto key generate rsa modulus 1024
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
interface vlan 1
 ip address 192.168.1.2 255.255.255.0
 no shutdown
ip default-gateway 192.168.1.1
line con 0
 login local
 exec-timeout 5 0
 logging synchronous
line vty 0 15
 transport input ssh
 login local
 exec-timeout 5 0
```

---

## Necessary Commands and What they Do 

| Command | Purpose |
|---|---|
| `hostname` | Names the device which is required before RSA key generation |
| `no ip domain-lookup` | Stops DNS resolution of mistyped commands |
| `enable secret` | MD5-encrypted privileged access password that always wins over enable password |
| `username admin privilege 15 secret` | Creates local user with full access for SSH and console login |
| `service password-encryption` | Encrypts plaintext passwords in running-config with type 7 |
| `banner motd # message #` | Legal warning displayed before login |
| `ip domain-name` | Required before RSA key generation alongside hostname |
| `crypto key generate rsa modulus 1024` | Generates RSA keypair for SSH encryption |
| `ip ssh version 2` | Forces SSH v2 which requires minimum 1024-bit RSA key |
| `transport input ssh` | Blocks Telnet, allows SSH only on VTY lines |
| `login local` | Authenticates using local username database |
| `exec-timeout 5 0` | Logs out idle sessions after 5 minutes 0 seconds |
| `logging synchronous` | Prevents console messages from interrupting typed commands |
| `interface vlan 1 / ip address / no shutdown` | Switch management SVI is how switches get a management IP |
| `ip default-gateway` | Tells switch where to send traffic outside its local subnet |
| `copy running-config startup-config` | Saves config to NVRAM so it survives a reboot |

---

## Key Concepts Reviewed

**Three prerequisites for RSA key generation:**
Hostname must be set, domain name must be configured, and a local username must exist. Missing any one of these causes the crypto key generate command to fail.

**enable secret vs enable password:**
Enable secret uses MD5 type 5 encryption and always takes priority. Enable password uses type 7 which is reversible with free online tools. If both are configured enable secret wins and enable password is completely ignored. Never use enable password on any modern device.

**login vs login local:**
`login` authenticates using a single shared line password set with the `password` command. `login local` authenticates using the full username and password database. Login local is more secure because it ties access to named individual accounts and works with SSH.

**Why switches need a default gateway:**
A switch SVI only knows about its directly connected subnet. Without a default gateway it cannot send or receive traffic from devices on other networks. Routers do not need a default gateway because they have a full routing table.

**SSH VTY lines -- routers vs switches:**
Routers use `line vty 0 4` covering 5 sessions. Switches use `line vty 0 15` covering 16 sessions. Leaving any VTY lines unconfigured on a switch can allow Telnet access through those lines even if the lower lines are locked to SSH only.

**RSA key role vs username and password:**
The RSA key encrypts the entire SSH session tunnel so captured traffic cannot be read. The username and password authenticate the actual login inside that tunnel. The RSA key alone does not grant access and the password alone without encryption is no better than Telnet, so both username and password are required for ssh to work. 

---

## Troubleshooting Encountered

**SSH connection timeout:** Initial SSH attempt failed because no cable connected R1 to SW1. Devices must have a physical link in Packet Tracer before any communication is possible. Connected via copper straight-through cable using the auto-connect tool. Once the link turned green and both SVIs showed up/up SSH worked immediately.

---

## Lessons Learned

Device hardening is always the first phase of any professional network deployment and it is easy to underestimate how many interdependencies exist between seemingly simple commands. This lab reinforced that the order of operations matters: hostname and domain name must come before RSA key generation, exclusions must come before DHCP pools, and physical connectivity must exist before any remote management can be tested.

The SSH timeout troubleshooting was a practical reminder that no matter how correct the configuration is, a missing physical link will prevent everything from working. In a real network this translates to verifying physical layer health before spending time on logical troubleshooting. Always check the cable before blaming the config.

The difference between routers and switches for VTY line configuration is a commonly missed detail. Switches support 16 VTY sessions and all 16 must be configured or some lines may default to allowing Telnet. This is a real security gap that shows up in network audits and is tested on the CCNA exam.

One Packet Tracer limitation worth noting: the `crypto key generate rsa modulus 1024` inline syntax may not work in all PT versions. The workaround is to run `crypto key generate rsa` and enter 1024 at the interactive prompt. On real Cisco hardware the inline modulus syntax works correctly. Always test SSH functionality after key generation rather than assuming the key was created successfully.

The session termination command `exit` closes an SSH session cleanly and returns to the originating device prompt. The Cisco escape sequence `Ctrl + Shift + 6` then `X` suspends the session without closing it which saves time and is useful when you need to check something on the local device and return to the remote session.

---

## Verification Commands

```
show running-config
show ip ssh
show users
show ip interface brief
show line vty 0 4
```
