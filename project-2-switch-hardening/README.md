Project 2 — Switch Layer Hardening & Secure Administration

## Objective
Lock down physical switch access by disabling unused ports, preventing unauthorized 
device connections using sticky MAC port security, and replacing plaintext Telnet 
with encrypted SSH management.

---

## What Changed From Project 1
Project 1 left the network functional but physically vulnerable — empty ports were 
open doors and admin traffic was sent in plain text over Telnet. Project 2 fixes both.

---

## Part 1 — BlackHole VLAN + Unused Port Shutdown

Created VLAN 99 as a dead end — no sub-interface on router, no DHCP pool, no routing.
Assigned all unused ports to it and shut them down.

```bash
vlan 99
 name BlackHole

interface range fa0/9 - 24
 switchport mode access
 switchport access vlan 99
 shutdown
exit
```

**Three layers of protection on every unused port:**
- No IP address possible (no DHCP pool)
- No routing out (no sub-interface)
- Port administratively dead (shutdown)

---

## Part 2 — Port Security Sticky MAC

Locked every active port to its legitimate device automatically.

```bash
interface range fa0/1 - 3
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
exit

interface range fa0/4 - 6
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
exit

interface range fa0/7 - 8
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
exit
```

**What each command does:**
- `maximum 1` — one device per port, no exceptions
- `mac-address sticky` — first device that sends traffic gets locked in automatically
- `violation shutdown` — wrong MAC appears, port goes err-disabled instantly

---

## Part 3 — SSH Replacing Telnet

```bash
hostname SW1
ip domain-name cisco.local
crypto key generate rsa
 1024
username admin privilege 15 secret cisco123
line vty 0 15
 transport input ssh
 login local
exit
ip ssh version 2
```

**Why this order matters:**
- Hostname + domain name must come before RSA key generation
- RSA key must exist before SSH version 2 can be forced
- `transport input ssh` completely kills Telnet on all 16 VTY lines

---

## Verification Commands

```bash
show port-security
show port-security interface fa0/1
show port-security address
show ip ssh
show running-config | section vty
```

---

## Verification Results

**Port Security Summary:**
All 8 active ports showing CurrentAddr 1, SecurityViolation 0, Action Shutdown.

**SSH Status:**
SSH Enabled - version 2.0, Authentication timeout 120 secs, Retries 3.
Telnet completely blocked on all VTY lines 0 through 15.

---

## What I learned
Before this project the network was like a secure building with unlocked back doors 
everywhere. I learned that shutting unused ports alone is not enough. Parking them 
in a BlackHole VLAN adds a second layer because even if someone brings a port back up 
accidentally it still goes nowhere without a sub-interface or DHCP pool behind it.

Port security sticky MAC was satisfying to configure because the switch does the hard 
work automatically. It learns and locks legitimate devices without me manually typing 
a single MAC address. The violation shutdown behavior is immediate and unforgiving which 
is exactly what security should be.

SSH configuration taught me that encryption needs an identity — the hostname and domain 
name aren't just labels, they're the foundation the RSA key is built on. Understanding 
why the order of commands matters made the whole thing click.

## Biggest challenge
Understanding VTY lines confused me at first because I thought they were related to 
physical ports like fa0/1 to fa0/24. Once I understood that VTY lines are purely 
software channels like a private backdoor that only admins use to remotely access 
the CLI which is completely separate from the physical ports that carry everyone else's 
traffic. Physical ports are the hotel rooms, VTY lines are the 
manager's private office lines.
