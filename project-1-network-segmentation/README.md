# Project 1 — Core Infrastructure & Network Segmentation

## Objective
Convert a flat 192.168.100.0/24 network into a segmented business layout with 
VLAN isolation, inter-VLAN routing, DHCP automation, and ACL-based Guest network lockdown.

## Topology
![Network Topology](./images/topology.png)

## IP Addressing Table (VLSM)

| VLAN | Department | Network | Subnet Mask | Gateway | Hosts Available |
|------|------------|---------|-------------|---------|-----------------|
| 10 | HR | 192.168.100.0 | /25 (255.255.255.128) | 192.168.100.1 | 126 |
| 20 | Engineering | 192.168.100.128 | /26 (255.255.255.192) | 192.168.100.129 | 62 |
| 30 | Guest Wi-Fi | 192.168.100.192 | /27 (255.255.255.224) | 192.168.100.193 | 30 |

---

## Switch Configuration — VLANs & Ports

```bash
vlan 10
 name HR_Dept
vlan 20
 name Eng_Dept
vlan 30
 name Guest_WiFi

interface fa0/10
 switchport mode access
 switchport access vlan 10

interface fa0/15
 switchport mode access
 switchport access vlan 20

interface fa0/20
 switchport mode access
 switchport access vlan 30

interface gi0/1
 switchport mode trunk
```

---

## Router Configuration — Router-on-a-Stick (Sub-interfaces)

```bash
interface gi0/0
 no shutdown

interface gi0/0.10
 encapsulation dot1Q 10
 ip address 192.168.100.1 255.255.255.128

interface gi0/0.20
 encapsulation dot1Q 20
 ip address 192.168.100.129 255.255.255.192

interface gi0/0.30
 encapsulation dot1Q 30
 ip address 192.168.100.193 255.255.255.224
```

---

## DHCP Configuration

```bash
ip dhcp excluded-address 192.168.100.1
ip dhcp excluded-address 192.168.100.129
ip dhcp excluded-address 192.168.100.193

ip dhcp pool HR_POOL
 network 192.168.100.0 255.255.255.128
 default-router 192.168.100.1

ip dhcp pool ENG_POOL
 network 192.168.100.128 255.255.255.192
 default-router 192.168.100.129

ip dhcp pool GUEST_POOL
 network 192.168.100.192 255.255.255.224
 default-router 192.168.100.193
```

---

## ACL Firewall — Guest Network Isolation

**Security goal:** Guest VLAN 30 cannot reach HR or Engineering. Guest cannot SSH or Telnet to any device.

```bash
ip access-list extended SECURE_GUEST_ACL
 deny ip 192.168.100.192 0.0.0.31 192.168.100.0 0.0.0.127
 deny ip 192.168.100.192 0.0.0.31 192.168.100.128 0.0.0.63
 deny tcp 192.168.100.192 0.0.0.31 any eq 22
 deny tcp 192.168.100.192 0.0.0.31 any eq telnet
 permit ip any any

interface gi0/0.30
 ip access-group SECURE_GUEST_ACL in
```

---

## Verification Commands

```bash
show running-config
show ip interface gi0/0.30
show ip access-lists
clear ip access-list counters
```

---

## What I learned
Before this project I had no idea how VLANs actually worked. I learned how to create 
them on a switch, then assign specific ports to specific VLANs so that only authorised 
devices in that department get access. Then I learned why a trunk port is needed 
between the switch and router which is because traffic from multiple VLANs needs to travel 
down that one cable, so it carries all of them tagged.

The Router-on-a-Stick concept clicked when I understood that a real router has limited 
physical interfaces, so instead of needing one port per VLAN you virtually slice that 
one interface into sub-interfaces and each sub-interface only lets its own VLAN 
traffic through based on the 802.1Q tag.

DHCP pooling was straightforward once the subnets were set we can just point each pool 
at the right network and the router hands out IPs automatically to whatever host joins 
that VLAN. No manual configuration needed on the PCs.

For the ACL firewall I deployed it on sub-interface gi0/0.30 inbound, meaning any 
traffic coming FROM the Guest network gets filtered right at the entry point. It blocks 
Guest from reaching HR or Engineering, and also blocks SSH and Telnet so Guest users 
can't try to attack the gateway.

## Biggest challenge
The hardest part was wrapping my head around how a packet actually travels through 
the trunk. Like a PC in HR sends data, the switch tags it with VLAN 10, it goes up 
the trunk cable to the router, the router receives it on sub-interface gi0/0.10 because 
that sub-interface is configured for dot1Q tag 10, routes it, then sends it back DOWN 
the trunk tagged for the destination VLAN, and the switch reads that tag and forwards 
it to the correct port. Once I visualised that full journey in my head it finally made 
sense that the tag is basically the packet's label that tells every device which VLAN it 
belongs to at every step.
