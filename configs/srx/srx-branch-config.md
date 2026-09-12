# Branch SRX Configuration

## Overview

The Branch SRX provides Layer 3 connectivity between the Branch network and the two Layer 3 switches.

Two routed interfaces are configured toward the Layer 3 switching layer, providing separate paths to the Branch network.

The routing design was initially implemented using static routes and was later migrated to OSPF to provide dynamic route learning and better failover behavior.

---

## 1. Initial Interface Configuration

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

---

## 2. Security Zone Configuration

Both routed interfaces are assigned to the `internal` security zone:

```text
set security zones security-zone internal interfaces ge-0/0/0.0
set security zones security-zone internal interfaces ge-0/0/2.0
```

ICMP ping is allowed as an inbound system service on the `internal` zone:

```text
set security zones security-zone internal host-inbound-traffic system-services ping
```

This allows the configured internal interfaces to receive ICMP ping traffic for connectivity and troubleshooting verification.

---

## 3. Initial Static Routing Design

The initial routing design used static routes to reach the Branch internal networks:

```text
set routing-options static route 192.168.0.0/16 next-hop 10.10.20.2
set routing-options static route 192.168.0.0/16 next-hop 10.10.20.6
```

The configured next-hops were:

| Next-hop     | Connected Device |
| ------------ | ---------------- |
| `10.10.20.2` | L3-SW-Branch1    |
| `10.10.20.6` | L3-SW-Branch2    |

The `/16` summary route covered the Branch internal VLAN networks, including:

* `192.168.50.0/29`
* `192.168.60.0/29`

The static routing design was used during the initial Branch implementation and was later tested for failover behavior.

---

## 4. Migration from Static Routing to OSPF

After verifying the initial static routing design, the Branch routing was migrated to OSPF.

The static routes were removed:

```text
delete routing-options static route 192.168.0.0/16 10.10.20.2
delete routing-options static route 192.168.0.0/16 10.10.20.6
```

OSPF was then enabled on both routed interfaces:

```text
set protocols ospf area 0.0.0.0 interface ge-0/0/2.0
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0
```

The SRX therefore establishes OSPF adjacencies with both Branch Layer 3 switches.

The OSPF design uses Area 0:

```text
Area: 0.0.0.0
```

---

## 5. OSPF Host-Inbound Troubleshooting

After configuring OSPF, the configuration itself was correct, but OSPF adjacencies were initially not forming.

The issue was identified as the SRX security zone not permitting OSPF as an inbound host protocol.

The following configuration was added:

```text
set security zones security-zone internal host-inbound-traffic protocols ospf
```

After allowing OSPF on the `internal` zone, the SRX successfully established full OSPF adjacencies with both Layer 3 switches.

Verification:

```text
root@Branch> show ospf neighbor

Address          Interface              State           ID               Pri  Dead
10.10.20.6       ge-0/0/0.0             Full            192.168.60.5       1    32
10.10.20.2       ge-0/0/2.0             Full            192.168.60.6       1    39
```

The SRX also showed OSPF traffic in the security flow session table:

```text
In: 10.10.20.2/1 --> 224.0.0.5/1;ospf
If: ge-0/0/2.0
```

This confirmed that OSPF packets were being received by the SRX.

---

## 6. OSPF Route Verification

After the OSPF adjacencies reached the `Full` state, the Branch VLAN networks were learned dynamically through OSPF.

Example:

```text
root@Branch> show route
```

Relevant routes:

```text
192.168.50.0/29    *[OSPF/10]  metric 2
                       to 10.10.20.6 via ge-0/0/0.0
                    >  to 10.10.20.2 via ge-0/0/2.0

192.168.60.0/29    *[OSPF/10]  metric 2
                    >  to 10.10.20.6 via ge-0/0/0.0
                       to 10.10.20.2 via ge-0/0/2.0
```

The routing table confirmed that the SRX was no longer relying on the previous static `/16` route and was learning the Branch VLAN networks through OSPF.

Both Layer 3 switches also formed OSPF adjacencies with the SRX.

---

## 7. OSPF Failover Testing

To verify dynamic routing behavior, continuous ICMP traffic was generated from the Branch PC toward the SRX:

```text
VPC27> ping 10.10.20.5 -t
```

Before the failure, the traffic was successfully reaching the SRX through the path connected to L3-SW-Branch2.

The test then simulated a failure by shutting down VLAN 60 on L3-SW-Branch2:

```text
interface Vlan60
 shutdown
```

The existing traffic immediately experienced timeouts:

```text
84 bytes from 10.10.20.5 icmp_seq=5 ttl=255 time=0.940 ms
10.10.20.5 icmp_seq=6 timeout
10.10.20.5 icmp_seq=7 timeout
10.10.20.5 icmp_seq=8 timeout
```

The failure caused the traffic to use the alternate Layer 3 path through L3-SW-Branch1.

---

## 8. ICMP Security Policy Troubleshooting

During the failover test, the alternate path reached the SRX through a different interface.

The original traffic entered the SRX through:

```text
ge-0/0/0.0
```

After the VLAN 60 failure and path change, the traffic entered through:

```text
ge-0/0/2.0
```

The SRX flow session table was used to investigate the behavior.

After adding the required policy, the SRX showed the ICMP session matching the `internal-internal` security policy:

```text
Session ID: 134668, Policy name: internal-internal/6, Timeout: 2, Session State: Valid
  In: 192.168.60.2/32846 --> 10.10.20.5/227;icmp, Conn Tag: 0x0, If: ge-0/0/2.0, Pkts: 1, Bytes: 84,
  Out: 10.10.20.5/227 --> 192.168.60.2/32846;icmp, Conn Tag: 0x0, If: .local..0, Pkts: 1, Bytes: 84,
```

This confirmed that the ICMP traffic from the Branch PC was entering the SRX through the alternate `ge-0/0/2.0` interface and matching the new security policy.

The following policy was added:

```text
set security policies from-zone internal to-zone internal policy internal-internal match source-address any
set security policies from-zone internal to-zone internal policy internal-internal match destination-address any
set security policies from-zone internal to-zone internal policy internal-internal match application junos-icmp-all
set security policies from-zone internal to-zone internal policy internal-internal then permit
```

After the policy was added, the ICMP traffic resumed:

```text
10.10.20.5 icmp_seq=33 ttl=63 time=1.949 ms
10.10.20.5 icmp_seq=35 ttl=63 time=3.411 ms
10.10.20.5 icmp_seq=36 ttl=63 time=2.223 ms
10.10.20.5 icmp_seq=37 ttl=63 time=1.527 ms
```

This demonstrated that the traffic could successfully use the alternate path through L3-SW-Branch1 after the failure of the VLAN 60 path on L3-SW-Branch2.

---

## 9. Current SRX Routing and Security State

The Branch SRX now uses OSPF for dynamic route exchange with both Branch Layer 3 switches.

Current OSPF interfaces:

```text
set protocols ospf area 0.0.0.0 interface ge-0/0/2.0
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0
```

OSPF host-inbound traffic is permitted:

```text
set security zones security-zone internal host-inbound-traffic protocols ospf
```

ICMP is permitted as an inbound system service:

```text
set security zones security-zone internal host-inbound-traffic system-services ping
```

The `internal-internal` policy was added during failover troubleshooting to permit ICMP traffic between the internal interfaces:

```text
set security policies from-zone internal to-zone internal policy internal-internal match source-address any
set security policies from-zone internal to-zone internal policy internal-internal match destination-address any
set security policies from-zone internal to-zone internal policy internal-internal match application junos-icmp-all
set security policies from-zone internal to-zone internal policy internal-internal then permit
```

The Branch routing design therefore evolved from an initial static-routing implementation to a dynamic OSPF-based design with redundant paths through both Layer 3 switches.

---

## Configuration Evolution Summary

| Stage           | Routing / Security Change                                   |
| --------------- | ----------------------------------------------------------- |
| Initial         | Two routed SRX-to-L3 connections                            |
| Initial         | Both interfaces assigned to `internal` zone                 |
| Initial         | ICMP host-inbound service enabled                           |
| Phase 1         | Static `/16` routes configured                              |
| Phase 2         | Static routes removed                                       |
| Phase 2         | OSPF enabled toward both L3 switches                        |
| Troubleshooting | OSPF host-inbound protocol permitted                        |
| Verification    | OSPF neighbors reached `Full` state                         |
| Verification    | Branch VLANs learned through OSPF                           |
| Failover Test   | VLAN 60 shut down on L3-SW-Branch2                          |
| Troubleshooting | ICMP alternate-path traffic identified through `ge-0/0/2.0` |
| Troubleshooting | `internal-internal` ICMP policy added                       |
| Final           | OSPF provides dynamic routing over both SRX-to-L3 paths     |
