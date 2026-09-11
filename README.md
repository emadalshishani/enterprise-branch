# Enterprise Branch Network Lab

A redundant enterprise Branch network built to demonstrate Layer 2, Layer 3, gateway redundancy, and firewall connectivity using Cisco switching technologies and a Juniper SRX firewall.

## Project Objectives

* Design a redundant Branch network architecture
* Implement VLAN segmentation
* Implement Layer 3 gateway redundancy using VRRP
* Provide redundant connectivity between the access and distribution layers
* Connect the Branch network to a Juniper SRX firewall
* Implement and verify static routing between the SRX and Branch Layer 3 switches
* Test gateway failover and end-to-end connectivity
* Identify and document routing and failover limitations through practical testing

## Network Components

* 1 × Juniper SRX firewall
* 2 × Layer 3 switches
* 2 × Layer 2 access switches
* 4 × end devices

## VLANs

| VLAN | Network         | VRRP Virtual IP |
| ---- | --------------- | --------------- |
| 50   | 192.168.50.0/29 | 192.168.50.1    |
| 60   | 192.168.60.0/29 | 192.168.60.1    |

## Gateway Redundancy

VRRP is implemented between the two Layer 3 switches.

* VLAN 50 → L3-SW-Branch1 preferred Master
* VLAN 60 → L3-SW-Branch2 preferred Master

This provides gateway redundancy while distributing the preferred gateway role between both Layer 3 switches.

## SRX Connectivity

The SRX connects to both Layer 3 switches using separate point-to-point links.

Static routes are configured on the SRX toward the Branch internal networks through both Layer 3 switches.

## Verification

The Branch design was tested using:

* VRRP gateway failover
* Continuous ICMP testing
* Branch PC to VRRP gateway connectivity
* Branch PC to SRX connectivity
* VLAN failure scenarios
* Recovery after restoring the affected interfaces

The testing also identified a limitation in the current SRX static-routing design: VRRP can successfully fail over the Branch gateway, but the SRX does not dynamically detect the downstream VLAN path failure and automatically select the alternate Layer 3 path.

This limitation is documented as part of the lab and can be addressed in a future phase using a more robust routing and failover mechanism.

## Documentation

Detailed architecture and verification results are available in:

`docs/branch-architecture.md`
