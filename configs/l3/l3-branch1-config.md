# L3-SW-Branch1 Configuration

## Overview

L3-SW-Branch1 provides Layer 3 gateway functionality for the Branch VLANs and participates in VRRP gateway redundancy with L3-SW-Branch2.

The switch also provides trunk connectivity toward the Branch access layer and the second Layer 3 switch.

## Routed Uplink to SRX

The connection toward the Branch SRX is configured as a routed interface:

```text
interface Ethernet0/0
 no switchport
 ip address 10.10.20.2 255.255.255.252
```

* Interface: `Ethernet0/0`
* IP address: `10.10.20.2/30`
* Connected SRX: `10.10.20.1/30`

## Trunk Interfaces

The following interfaces operate as 802.1Q trunk links and carry VLANs 50 and 60:

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

The trunk links provide connectivity for the Branch VLANs across the Layer 2 and Layer 3 switching layers.

## VLAN 50

The Layer 3 SVI for VLAN 50 is configured with:

```text
interface Vlan50
 ip address 192.168.50.6 255.255.255.248
 vrrp 1 ip 192.168.50.1
 vrrp 1 priority 120
```

* SVI address: `192.168.50.6/29`
* VRRP Virtual IP: `192.168.50.1`
* VRRP priority: `120`
* Preferred role: **Master**

L3-SW-Branch1 is configured with the higher VRRP priority for VLAN 50, making it the preferred Master.

## VLAN 60

The Layer 3 SVI for VLAN 60 is configured with:

```text
interface Vlan60
 ip address 192.168.60.6 255.255.255.248
 vrrp 1 ip 192.168.60.1
```

* SVI address: `192.168.60.6/29`
* VRRP Virtual IP: `192.168.60.1`

L3-SW-Branch1 operates as the Backup router for VLAN 60, while L3-SW-Branch2 has the higher VRRP priority and is the preferred Master.

## Configuration Summary

L3-SW-Branch1 provides:

* Routed connectivity toward the Branch SRX
* Trunk connectivity carrying VLANs 50 and 60
* Layer 3 gateway functionality for VLAN 50 and VLAN 60
* VRRP gateway redundancy
* Preferred VRRP Master role for VLAN 50
* Backup VRRP role for VLAN 60

The VRRP behavior and failover scenarios are documented in the Branch architecture and verification documentation.
