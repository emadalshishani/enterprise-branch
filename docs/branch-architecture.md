# Branch Site Architecture

## Overview

The Branch site is designed as a redundant Layer 3 switching environment with two Layer 3 switches providing gateway redundancy for the internal VLANs.

The Branch firewall (SRX) provides the Layer 3 connection between the Branch internal network and the external network. The two Layer 3 switches provide redundant gateway functionality using VRRP for the internal VLANs.

## Topology

The Branch consists of:

* 1 × Juniper SRX firewall
* 2 × Layer 3 switches
* 2 × Layer 2 access switches
* 4 × end devices (PCs)

The Layer 2 switches are connected to both Layer 3 switches using trunk links, providing redundant connectivity between the access and Layer 3 layers.

## SRX to Layer 3 Switches

| Device        | Interface | IP Address    |
| ------------- | --------- | ------------- |
| SRX           | ge-0/0/2  | 10.10.20.1/30 |
| L3-SW-Branch1 | Eth0/0    | 10.10.20.2/30 |
| SRX           | ge-0/0/0  | 10.10.20.5/30 |
| L3-SW-Branch2 | Eth0/0    | 10.10.20.6/30 |

## VLANs and VRRP

### VLAN 50

* Network: `192.168.50.0/29`
* L3-SW-Branch1 SVI: `192.168.50.6`
* L3-SW-Branch2 SVI: `192.168.50.5`
* VRRP Virtual IP: `192.168.50.1`
* L3-SW-Branch1 VRRP priority: `120`
* L3-SW-Branch2 VRRP priority: `100`

L3-SW-Branch1 is the preferred VRRP Master for VLAN 50.

### VLAN 60

* Network: `192.168.60.0/29`
* L3-SW-Branch1 SVI: `192.168.60.6`
* L3-SW-Branch2 SVI: `192.168.60.5`
* VRRP Virtual IP: `192.168.60.1`
* L3-SW-Branch1 VRRP priority: `100`
* L3-SW-Branch2 VRRP priority: `120`

L3-SW-Branch2 is the preferred VRRP Master for VLAN 60.

## Gateway Redundancy

VRRP is used to provide a virtual default gateway for the Branch VLANs.

The design uses both Layer 3 switches as active gateway devices:

* L3-SW-Branch1 is preferred for VLAN 50.
* L3-SW-Branch2 is preferred for VLAN 60.

If the active Layer 3 gateway for a VLAN becomes unavailable, the other Layer 3 switch can assume the VRRP Master role and provide the virtual gateway.

This provides gateway redundancy while distributing the preferred gateway role between the two Layer 3 switches.

## Redundant Access Layer

The Layer 2 access switches have redundant trunk connectivity toward the Layer 3 switching layer.

The required Branch VLANs are carried across the trunk links:

* VLAN 50
* VLAN 60

This provides redundant paths between the access layer and the Layer 3 gateways.

## Verification

### 1. VRRP Gateway Redundancy Test

Connectivity from the Branch PCs to the VRRP virtual gateways was tested using ICMP traffic:

* VLAN 50 gateway: `192.168.50.1`
* VLAN 60 gateway: `192.168.60.1`

The initial VRRP state confirmed that:

* L3-SW-Branch1 was the Master for VLAN 50 with priority `120`.
* L3-SW-Branch2 was the Master for VLAN 60 with priority `120`.
* Each Layer 3 switch operated as the Backup router for the other VLAN.

A continuous ping was initiated from Branch-PC3 toward the VLAN 50 virtual gateway:

```text
ping 192.168.50.1 -t
```

The VLAN 50 SVI on L3-SW-Branch1 was then administratively shut down:

```text
interface vlan 50
shutdown
```

The VRRP state changed from Master to Init on L3-SW-Branch1:

```text
%VRRP-6-STATECHANGE: Vl50 Grp 1 state Master -> Init
```

L3-SW-Branch2 subsequently assumed the Master role for VLAN 50.

The continuous ping remained operational after the failover, confirming successful VRRP gateway failover.

After restoring L3-SW-Branch1, VLAN 50 on L3-SW-Branch2 was also tested by administratively shutting down its VLAN 50 SVI.

With both Layer 3 switches unavailable as gateways for VLAN 50, connectivity to the virtual gateway `192.168.50.1` was lost and the continuous ping produced consecutive timeouts.

Both VLAN 50 interfaces were subsequently restored using:

```text
no shutdown
```

VRRP reconverged and connectivity to the virtual gateway was restored.

### 2. Branch PC to SRX Connectivity Test

End-to-end connectivity between the Branch LAN and the Branch SRX was tested using ICMP.

The SRX uses the following static routing configuration to reach the Branch internal networks:

```text
set routing-options static route 192.168.0.0/16 next-hop 10.10.20.2
set routing-options static route 192.168.0.0/16 next-hop 10.10.20.6
```

The configured next-hops are:

* `10.10.20.2` → L3-SW-Branch1
* `10.10.20.6` → L3-SW-Branch2

A continuous ping was initiated from Branch-PC3 toward the SRX:

```text
ping 10.10.20.1 -t
```

The ping initially received continuous replies.

The VLAN 50 SVI on L3-SW-Branch1 was then administratively shut down:

```text
interface vlan 50
shutdown
```

The VRRP state on L3-SW-Branch1 changed from Master to Init, allowing the gateway function for VLAN 50 to fail over to L3-SW-Branch2.

However, the ping from Branch-PC3 to the SRX temporarily stopped responding:

```text
10.10.20.1 icmp_seq=57 timeout
10.10.20.1 icmp_seq=58 timeout
...
10.10.20.1 icmp_seq=67 timeout
```

Although the SRX has two static next-hops toward the Branch networks, the static routing configuration does not dynamically detect the failure of the downstream VLAN path associated with L3-SW-Branch1.

As a result, the SRX did not automatically move the return traffic to the alternate next-hop `10.10.20.6` when the VLAN 50 path through L3-SW-Branch1 became unavailable.

After the VLAN 50 SVI on L3-SW-Branch1 was restored using:

```text
no shutdown
```

VRRP reconverged:

```text
%VRRP-6-STATECHANGE: Vl50 Grp 1 state Init -> Backup
%VRRP-6-STATECHANGE: Vl50 Grp 1 state Backup -> Master
```

ICMP replies to the SRX resumed successfully.

### Observation

This test demonstrated an important limitation in the current Branch-to-SRX design.

VRRP successfully provides redundant default-gateway functionality for the Branch VLANs. However, the SRX currently relies on static routes with two configured next-hops.

The presence of multiple static next-hops does not by itself provide dynamic end-to-end path-failure detection for the downstream VLANs.

Therefore, the current design provides:

* **LAN gateway redundancy through VRRP**
* **Two static next-hops configured on the SRX**
* **Redundant Layer 3 gateways for the Branch VLANs**
* **No dynamic detection of downstream VLAN path failure on the SRX yet**

This limitation was intentionally identified through testing and can be addressed in a later phase using a more robust routing and failover mechanism.
