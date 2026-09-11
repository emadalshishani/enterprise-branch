# L2-Branch1 Configuration

## Overview

L2-Branch1 operates as an access-layer switch for the Branch network.

It provides redundant trunk connectivity toward the Layer 3 switching layer and connects end devices to VLAN 50 and VLAN 60.

## Trunk Interfaces

Ethernet0/0 and Ethernet0/1 are configured as 802.1Q trunk interfaces carrying VLANs 50 and 60:

```text
interface Ethernet0/0
 switchport trunk allowed vlan 50,60
 switchport trunk encapsulation dot1q
 switchport mode trunk

interface Ethernet0/1
 switchport trunk allowed vlan 50,60
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

These two trunk links provide redundant connectivity from the access switch toward the Layer 3 switching layer.

## Access Interfaces

### Ethernet0/2

Ethernet0/2 connects an end device to VLAN 50:

```text
interface Ethernet0/2
 switchport access vlan 50
 switchport mode access
```

### Ethernet0/3

Ethernet0/3 connects an end device to VLAN 60:

```text
interface Ethernet0/3
 switchport access vlan 60
 switchport mode access
```

## Configuration Summary

L2-Branch1 provides:

* Two redundant trunk uplinks
* VLAN 50 and VLAN 60 trunk transport
* Access connectivity for VLAN 50
* Access connectivity for VLAN 60
* Layer 2 connectivity between end devices and the redundant Layer 3 gateway layer
