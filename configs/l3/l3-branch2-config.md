# L3-SW-Branch2 Configuration

## Overview

L3-SW-Branch2 provides Layer 3 gateway services for the Branch VLANs and participates in the redundant routing design with L3-SW-Branch1.

The configuration evolved from the initial VLAN gateway and VRRP configuration to dynamic OSPF routing toward the Branch SRX.

---

## 1. SRX Uplink

The routed uplink toward the Branch SRX is configured on Ethernet0/0.

```text
interface Ethernet0/0
 no switchport
 ip address 10.10.20.6 255.255.255.252
```

Addressing:

| Device | Interface | IP Address |
|---|---|---|
| SRX-BRANCH | ge-0/0/0 | 10.10.20.5/30 |
| L3-SW-Branch2 | Ethernet0/0 | 10.10.20.6/30 |

---

## 2. VLAN Trunk Configuration

The switch provides trunk connectivity toward the Branch access layer and the redundant L3 switch.

```text
interface Ethernet0/1
 switchport trunk allowed vlan 50,60
 switchport trunk encapsulation dot1q
 switchport mode trunk

interface Ethernet0/2
 switchport trunk allowed vlan 50,60
 switchport trunk encapsulation dot1q
 switchport mode trunk

interface Ethernet0/3
 switchport trunk allowed vlan 50,60
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

The trunks carry:

- VLAN 50
- VLAN 60

---

## 3. VLAN 50 Gateway

VLAN 50 uses the following addressing:

- Network: `192.168.50.0/29`
- VRRP virtual gateway: `192.168.50.1`
- Branch1 SVI: `192.168.50.6`
- Branch2 SVI: `192.168.50.5`

Branch2 uses the default VRRP priority for VLAN 50, allowing Branch1 to act as the preferred Master.

```text
interface Vlan50
 ip address 192.168.50.5 255.255.255.248
 vrrp 1 ip 192.168.50.1
```

---

## 4. VLAN 60 Gateway

VLAN 60 uses the following addressing:

- Network: `192.168.60.0/29`
- VRRP virtual gateway: `192.168.60.1`
- Branch1 SVI: `192.168.60.6`
- Branch2 SVI: `192.168.60.5`

Branch2 is configured with a higher VRRP priority and therefore acts as the preferred Master for VLAN 60.

```text
interface Vlan60
 ip address 192.168.60.5 255.255.255.248
 vrrp 1 ip 192.168.60.1
 vrrp 1 priority 120
```

This creates an active/active gateway design across the two VLANs:

| VLAN | Preferred VRRP Master |
|---|---|
| VLAN 50 | L3-SW-Branch1 |
| VLAN 60 | L3-SW-Branch2 |

---

## 5. OSPF Configuration

After the initial static-routing design was tested, dynamic routing was introduced using OSPF.

OSPF was enabled on the routed uplink toward the Branch SRX.

```text
router ospf 1
 network 192.168.0.0 0.0.255.255 area 0

interface Ethernet0/0
 ip ospf 1 area 0
```

The switch forms an OSPF adjacency with the Branch SRX over the `10.10.20.4/30` transit network.

### OSPF Area

Area 0

---

## 6. OSPF Verification

OSPF neighbor relationships were verified after configuration.

The Branch SRX successfully formed a Full OSPF adjacency with L3-SW-Branch2 using:

10.10.20.6

The SRX routing table subsequently learned the Branch VLAN networks dynamically through OSPF.

Relevant networks:

- `192.168.50.0/29`
- `192.168.60.0/29`

---

## 7. VRRP Failover Verification

VRRP gateway redundancy was tested by shutting down a VLAN interface on the preferred gateway.

For VLAN 50, Branch1 was the preferred VRRP Master.

When Branch1's VLAN 50 interface was shut down, Branch2 assumed the VRRP Master role for VLAN 50.

Example:

```text
interface Vlan50
 shutdown
```

After restoring the interface:

```text
interface Vlan50
 no shutdown
```

the VRRP state returned according to the configured priorities.

The test confirmed that Branch2 can provide gateway redundancy when the preferred Branch1 gateway becomes unavailable.

---

## 8. OSPF Path Failover Verification

OSPF path failover was tested by shutting down VLAN 60 on L3-SW-Branch2.

```text
interface Vlan60
 shutdown
```

The test was performed using Branch-PC2 on VLAN 60.

Branch-PC2 continuously pinged the SRX interface:

```text
ping 10.10.20.5 -t
```

Before the failure, traffic entered the SRX through Branch2:

```text
ge-0/0/0.0
```

After VLAN 60 was shut down on Branch2, traffic was able to use the alternate Layer 3 path through L3-SW-Branch1.

SRX flow verification showed the post-failure traffic entering through:

```text
ge-0/0/2.0
```

instead of the original:

```text
ge-0/0/0.0
```

The ping experienced timeouts during the transition, followed by successful replies after the alternate path became available.

This confirmed that the redundant Layer 3 path through Branch1 was used after the Branch2 VLAN 60 failure.

---

## 9. Configuration Evolution

The Branch2 configuration evolved through the following stages:

| Stage | Change |
|---|---|
| Initial | VLAN 50 and VLAN 60 gateway configuration |
| Initial | VRRP gateway redundancy |
| Initial | Active/active gateway preference across VLANs |
| Phase 2 | OSPF enabled toward the Branch SRX |
| Verification | OSPF adjacency established |
| Failover Test | VLAN 60 failure simulated on Branch2 |
| Verification | Traffic moved to the alternate Layer 3 path through Branch1 |

---

## Current State

L3-SW-Branch2 currently provides:

- Layer 3 gateway services for VLAN 50 and VLAN 60
- VRRP gateway redundancy
- Preferred VRRP Master for VLAN 60
- OSPF connectivity toward the Branch SRX
- Redundant Layer 3 connectivity through the Branch topology
- Alternate gateway availability for VLAN 50 when Branch1 fails