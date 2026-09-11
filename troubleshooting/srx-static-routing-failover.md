# SRX Static Routing Failover Troubleshooting

## Issue

During Branch failover testing, VRRP successfully moved the VLAN 50 gateway from L3-SW-Branch1 to L3-SW-Branch2.

However, connectivity from Branch-PC3 to the SRX was temporarily interrupted.

## Test

A continuous ping was initiated from Branch-PC3 toward the SRX:

```text
ping 10.10.20.1 -t
```

The VLAN 50 SVI on L3-SW-Branch1 was then shut down:

```text
interface vlan 50
shutdown
```

VRRP correctly changed state and L3-SW-Branch2 assumed the Master role for VLAN 50.

However, the ping toward the SRX produced timeouts:

```text
10.10.20.1 icmp_seq=57 timeout
10.10.20.1 icmp_seq=58 timeout
...
10.10.20.1 icmp_seq=67 timeout
```

## SRX Routing Configuration

The SRX uses two static next-hops:

```text
set routing-options static route 192.168.0.0/16 next-hop 10.10.20.2
set routing-options static route 192.168.0.0/16 next-hop 10.10.20.6
```

* `10.10.20.2` → L3-SW-Branch1
* `10.10.20.6` → L3-SW-Branch2

## Root Cause / Observation

The VRRP gateway failover itself worked correctly.

The issue was related to the SRX return path. The static routing configuration does not dynamically detect the failure of the downstream VLAN path associated with L3-SW-Branch1.

Therefore, the SRX did not automatically select the alternate Layer 3 path when the VLAN 50 path through L3-SW-Branch1 became unavailable.

## Recovery

The VLAN 50 SVI on L3-SW-Branch1 was restored:

```text
no shutdown
```

VRRP reconverged and the ICMP replies to the SRX resumed.

## Conclusion

The test confirmed that the Branch LAN has working VRRP gateway redundancy, while the current SRX static-routing design does not yet provide complete end-to-end path failover.

This limitation is intentionally documented and can be addressed in a future phase using a more robust routing and failover mechanism.
