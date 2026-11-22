# Switch Port Security Configuration and Troubleshooting

Cisco Packet Tracer lab by Michael Bessard.

## Overview

Configured sticky MAC port security on SW1 FastEthernet0/1 to permit one learned source MAC address. Connecting a different PC triggered a shutdown violation and placed the interface in an err-disabled state. Reconnecting the original PC and manually recovering the interface restored connectivity while retaining the permitted sticky MAC.

**Lab file:** [switch-port-security-troubleshooting.pkt](switch-port-security-troubleshooting.pkt)

## Lab context

This exercise builds on an existing two-switch VLAN topology with DHCP snooping and router-on-a-stick routing. The port security change was limited to SW1 Fa0/1.

| Connection | Purpose |
|---|---|
| PC1 to SW1 Fa0/1 | Protected access port in VLAN10 |
| PC5 temporarily to SW1 Fa0/1 | Different device used to trigger the violation |
| R1 G0/0 to SW1 G0/2 | Trunk providing VLAN gateways |
| SW1 G0/1 to SW2 G0/1 | Existing inter-switch trunk |

| Device | Addressing | Source MAC |
|---|---|---|
| PC1 | Existing DHCP client in 192.168.10.0/24; gateway 192.168.10.1 | 0003.E485.EBAA |
| PC5 | Test settings: 192.168.10.50/24; gateway 192.168.10.1 | 00E0.F740.9AB1, observed as last source during violation |
| R1 VLAN10 gateway | 192.168.10.1/24 | Not part of the port's learned client entry |

PC5's test address was within the existing DHCP allocation range. No address conflict was observed or investigated; for future repetitions, reserve the test address or exclude it from DHCP before use. The port security violation was based on the source MAC address.

## Port security configuration

On SW1:

```text
enable
configure terminal
interface fastEthernet0/1
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
end
```

The existing access VLAN remained VLAN10. Sticky learning records the learned secure MAC in the running configuration. Saving the running configuration preserves that entry in startup configuration.

Generated traffic from PC1:

```text
ping 192.168.10.1
```

Checked SW1:

```text
show port-security interface fastEthernet0/1
show port-security address
```

## Baseline evidence

The switch showed:

- Port security: Enabled.
- Port status: Secure-up.
- Violation mode: Shutdown.
- Maximum MAC addresses: 1.
- Total MAC addresses: 1.
- Sticky MAC addresses: 1.
- Security violation count: 0.
- SecureSticky entry: 0003.E485.EBAA, VLAN10, Fa0/1.

The baseline screenshot confirms the port state and learned MAC. A separate baseline ping result was not captured in the evidence set.

## Controlled violation

Disconnected PC1 from SW1 Fa0/1 and connected temporary PC5 to the same port using a copper straight-through cable. Configured PC5's test IP settings and generated traffic:

```text
ping 192.168.10.1
```

Inspected the switch:

```text
show port-security interface fastEthernet0/1
show interfaces fastEthernet0/1
```

Observed:

| Field | Result |
|---|---|
| Port status | Secure-shutdown |
| Violation mode | Shutdown |
| Security violation count | 1 |
| Last source address/VLAN | 00E0.F740.9AB1:10 |
| Interface state | Down, line protocol down, err-disabled |
| Sticky MAC address count | 1 |

PC1's learned MAC had already filled the one-address allowance. Traffic from PC5's different source MAC triggered the configured shutdown action. The interface output identified err-disabled as the cause of the down state. No separate PC5 ping output was captured.

## Recovery

Disconnected PC5 and reconnected PC1 to SW1 Fa0/1. Retained PC1's permitted sticky MAC and manually recovered the port:

```text
configure terminal
interface fastEthernet0/1
 shutdown
 no shutdown
end
```

After the link came up, tested from PC1:

```text
ping 192.168.10.1
```

Verified SW1:

```text
show port-security interface fastEthernet0/1
show port-security address
```

The port returned to **Secure-up**, and **0003.E485.EBAA** remained a **SecureSticky** entry on VLAN10, Fa0/1. The captured recovery output displayed a violation count of 0. PC1 received **four replies with 0% packet loss** from 192.168.10.1.

The different device was removed before recovery so it would not immediately trigger another violation. No sticky entry was cleared or replaced, and port security remained enabled.

Saved SW1's configuration:

```text
copy running-config startup-config
```

Saved the Packet Tracer file with PC1 reconnected and PC5 disconnected. Restart persistence was not tested.

## Observed results

| Stage | Port state | Secure MAC evidence | Connectivity evidence |
|---|---|---|---|
| Baseline | Secure-up; 0 violations | PC1 learned as SecureSticky | Separate ping output not captured |
| Different device | Secure-shutdown; err-disabled; 1 violation | Last source was PC5's different MAC | Switch shutdown captured; PC5 ping result not captured |
| Recovery | Secure-up | PC1 sticky MAC retained | PC1 to gateway: 4/4 replies, 0% loss |

## Screenshots

| Screenshot | Evidence |
|---|---|
| [01-port-security-baseline.png](01-port-security-baseline.png) | Enabled security, Secure-up state, one sticky MAC, zero violations |
| [02-port-security-violation.png](02-port-security-violation.png) | Secure-shutdown, different source MAC, one violation, err-disabled interface |
| [03-port-security-restored.png](03-port-security-restored.png) | Secure-up with the original sticky MAC retained |
| [04-pc1-connectivity-restored.png](04-pc1-connectivity-restored.png) | Four successful replies from the VLAN10 gateway |

## Lessons learned

- Port security limits permitted source MAC addresses on a switch port.
- Sticky learning requires care about which device is connected when learning occurs.
- Shutdown mode can place the entire port into an err-disabled state after a violation.
- Inspect the port security status, last source MAC, violation count, and interface state together.
- Remove the violating device before recovering the port.
- Restoring the authorized device does not require clearing its existing sticky entry.
- Saving configuration preserves learned sticky entries; restart behavior should be verified separately when required.

This exercise tested one access port in shutdown mode with a different physical client. Protect/restrict modes, automatic err-disable recovery, device replacement authorization, and MAC spoofing were not tested. MAC-based port security does not authenticate a user's identity or provide the same controls as 802.1X.
