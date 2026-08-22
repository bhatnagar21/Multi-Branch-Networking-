# 🏢 Multi-Branch Corporate Network — VLAN Segmentation with Extended ACL

A simulated enterprise network built in Cisco Packet Tracer, demonstrating VLAN-based department isolation, inter-VLAN routing, centralized DHCP, and Extended ACL-based traffic filtering.

## 📖 Overview

This project simulates a small corporate network with two departments (HR and IT) that are logically separated using VLANs but can communicate through inter-VLAN routing configured on a central router. The network also includes automated IP assignment via DHCP and a security policy enforced through an Extended Access Control List.

## 🗺️ Topology

```
                        +-------------+
                        |   Router0   |
                        |   (1941)    |
                        +------+------+
                        Gi0/0  |  Gi0/1
                    (802.1Q)   |   (802.1Q)
                 +-------------+-------------+
                 |                           |
           +-----+-----+               +-----+-----+
           | Switch0   |               | Switch1   |
           +-----+-----+               +-----+-----+
             /       \                   /       \
          PC0        PC1              PC2        PC3
        (VLAN 10)   (VLAN 10)       (VLAN 20)   (VLAN 20)
           HR           HR             IT           IT
```

| Device | Role | VLAN | Subnet |
|---|---|---|---|
| Switch0 | Access switch | VLAN 10 (HR) | 192.168.10.0/24 |
| Switch1 | Access switch | VLAN 20 (IT) | 192.168.20.0/24 |
| Router0 | Router-on-a-Stick | Gateway for both VLANs | — |

## ✅ Features Implemented

- 🔀 **VLAN Segmentation** — HR (VLAN 10) and IT (VLAN 20) departments isolated at Layer 2
- 🔗 **802.1Q Trunking** — configured between each switch and the router
- 🌐 **Inter-VLAN Routing** — Router-on-a-Stick architecture using sub-interfaces (`Gi0/0.10`, `Gi0/1.20`)
- 📡 **DHCP Services** — centralized IP address assignment for both VLANs via separate DHCP pools
- 🔒 **Extended ACL** — protocol and destination-specific traffic filtering (ICMP block between a specific host pair, rather than blanket source-based blocking)

## ⚙️ Key Configuration

### 🔀 VLAN + Trunk (per switch)
```
vlan 10
name HR
exit

interface fastEthernet 0/1
switchport mode access
switchport access vlan 10
exit

interface fastEthernet 0/3
switchport mode trunk
exit
```

### 🌐 Router Sub-interfaces
```
interface gigabitEthernet 0/0.10
encapsulation dot1Q 10
ip address 192.168.10.1 255.255.255.0
exit

interface gigabitEthernet 0/1.20
encapsulation dot1Q 20
ip address 192.168.20.1 255.255.255.0
exit
```

### 📡 DHCP Pools
```
ip dhcp excluded-address 192.168.10.1
ip dhcp excluded-address 192.168.20.1

ip dhcp pool VLAN10_POOL
network 192.168.10.0 255.255.255.0
default-router 192.168.10.1
dns-server 8.8.8.8

ip dhcp pool VLAN20_POOL
network 192.168.20.0 255.255.255.0
default-router 192.168.20.1
dns-server 8.8.8.8
```

### 🔒 Extended ACL
```
access-list 100 deny icmp host 192.168.20.10 host 192.168.10.10
access-list 100 permit ip any any

interface gigabitEthernet 0/1.20
ip access-group 100 out
```

## 🔍 Troubleshooting Log

During implementation, cross-VLAN connectivity failed even though the router's own sub-interfaces were reachable. A systematic elimination process was used to isolate the cause:

1. Verified router sub-interface status (`show ip interface brief`) — both up, correct IPs.
2. Verified router could reach its own opposite-VLAN gateway (`ping`) — successful, confirming the router itself was correctly configured.
3. Verified switch trunk status (`show interfaces trunk`) and VLAN membership (`show vlan brief`) — both correct.
4. Since the router and switches were both healthy, reviewed the full running configuration (`show running-config`) line by line.
5. Found the root cause: the DHCP pool for VLAN 20 had `default-router 192.168.20.0` (the network address) instead of `192.168.20.1` (the actual gateway). Every host that received a DHCP lease was being handed an invalid gateway, which broke all outbound traffic for that VLAN even though the router and switches were fine.
6. Corrected the DHCP pool and renewed DHCP leases on the affected hosts — connectivity was restored and verified in both directions (PC0 ↔ PC2, PC1 ↔ PC3).

This highlighted that a fully correct router and switch configuration can still result in total connectivity failure if a downstream service (DHCP) is silently handing out bad values — and that layer-by-layer verification (router → switch → end host → DHCP) is the fastest way to isolate this kind of fault.

## ✅ Verification

| Test | Result |
|---|---|
| PC0 (VLAN 10) → PC2 (VLAN 20) | ✅ 100% success |
| PC1 (VLAN 10) → PC3 (VLAN 20) | ✅ 100% success |
| DHCP lease assignment (both VLANs) | ✅ Verified |
| Extended ACL (blocked host pair) | ✅ Verified deny + verified unaffected hosts still pass |

## 🛠️ Tools Used

- Cisco Packet Tracer 9.0.1

## 📂 Files

- `multi-branch-network.pkt` — Packet Tracer project file
- `/screenshots` — configuration and verification screenshots
