# Branch SRX Configuration

## Overview

The Branch SRX provides Layer 3 connectivity between the Branch network and the two Layer 3 switches.

Two routed interfaces are configured toward the Layer 3 switching layer, providing separate paths to the Branch network.

## Interface Configuration

### ge-0/0/2

The first SRX interface connects to L3-SW-Branch1:

```text
set interfaces ge-0/0/2 unit 0 family inet address 10.10.20.1/30
```

* SRX address: `10.10.20.1/30`
* Connected L3 switch: L3-SW-Branch1
* L3 switch address: `10.10.20.2/30`

### ge-0/0/0

The second SRX interface connects to L3-SW-Branch2:

```text
set interfaces ge-0/0/0 unit 0 family inet address 10.10.20.5/30
```

* SRX address: `10.10.20.5/30`
* Connected L3 switch: L3-SW-Branch2
* L3 switch address: `10.10.20.6/30`

## Security Zone

Both routed interfaces are assigned to the `internal` security zone:

```text
set security zones security-zone internal interfaces ge-0/0/0.0
set security zones security-zone internal interfaces ge-0/0/2.0
```

ICMP ping is allowed as an inbound system service on the `internal` zone:

```text
set security zones security-zone internal host-inbound-traffic system-services ping
```

This allows the configured internal interfaces to receive ICMP ping traffic.

## Static Routing

The SRX uses static routing to reach the Branch internal networks:

```text
set routing-options static route 192.168.0.0/16 next-hop 10.10.20.2
set routing-options static route 192.168.0.0/16 next-hop 10.10.20.6
```

The configured next-hops are:

| Next-hop     | Connected Device |
| ------------ | ---------------- |
| `10.10.20.2` | L3-SW-Branch1    |
| `10.10.20.6` | L3-SW-Branch2    |

The `/16` summary route covers the Branch internal VLAN networks, including:

* `192.168.50.0/29`
* `192.168.60.0/29`

## Configuration Summary

The SRX configuration provides:

* Two routed connections toward the Branch Layer 3 switches
* Internal security-zone assignment for both interfaces
* ICMP reachability for verification
* Static routing toward the Branch internal networks
* Two configured next-hops toward the Branch Layer 3 switching layer

The behavior and limitations of the static routing design were verified separately in the Branch architecture and verification documentation.
