# Validation and troubleshooting

Use this guide with a local copy of [Final Project.pkt](../packet-tracer/Final%20Project.pkt) open in Cisco Packet Tracer. The tables describe expected behavior derived from the topology and [source configuration notes](reference/ios-configuration-notes.txt). Record observations from your own run. The project artifacts were preserved byte for byte.

## Routed links

| Link | Transit network | Endpoint A | Endpoint B |
| --- | --- | --- | --- |
| R1–R2 | `1.1.1.0/30` | R1 `Gi0/0/0` · `1.1.1.1` | R2 `Gi0/0/0` · `1.1.1.2` |
| R1–R3 | `2.2.2.0/30` | R1 `Gi0/0/1` · `2.2.2.1` | R3 `Gi0/0/1` · `2.2.2.2` |
| R2–R3 | `3.3.3.0/30` | R2 `Gi0/0/2` · `3.3.3.1` | R3 `Gi0/0/2` · `3.3.3.2` |
| R1–R5 | `4.4.4.0/30` | R1 `Se0/1/1` · `4.4.4.1` | R5 `Se0/1/1` · `4.4.4.2` |
| R1–R4 | `5.5.5.0/30` | R1 `Se0/1/0` · `5.5.5.1` | R4 `Se0/1/0` · `5.5.5.2` |
| R4–R5 | `6.6.6.0/30` | R4 `Se0/1/1` · `6.6.6.2` | R5 `Se0/1/0` · `6.6.6.1` |

## Verification sequence

| Check | Where and how | Expected observation |
| --- | --- | --- |
| Physical and Layer 3 links | `show ip interface brief` on R1–R5 | Enabled interfaces for the routed links are up/up and use the planned addresses. |
| Access VLANs and trunks | `show vlan brief` and `show interfaces trunk` on the departmental switches | Endpoint ports are assigned to their expected VLANs. Relevant trunks carry VLANs 10, 20, and 30. |
| EtherChannel and Rapid PVST+ | `show etherchannel summary` and `show spanning-tree vlan 10` on both sides. Repeat STP checks for VLANs 20 and 30. | Active member links are bundled under a port-channel with the intended negotiation protocol. Each VLAN has a consistent root and a loop-free forwarding topology. |
| Access-port controls | `show port-security interface fastethernet0/20` on a documented endpoint switch port | Port security settings, learned MAC addresses, and any recorded violations match the endpoint design. |
| DHCP and local routing | On each side, inspect client settings with `ipconfig /all`, ping the local gateway, and check `show ip dhcp binding` on R2 or R3. Test between two documented VLANs. | A DHCP client receives the expected subnet and gateway. Inter-VLAN traffic follows the local router when allowed. |
| OSPF and cross-site routing | Use `show ip ospf neighbor` and `show ip route ospf`. Ping and trace between known departmental clients on opposite sides. | The point-to-point neighbors reach FULL on the intended links. Remote client routes are present in both directions. |
| HSRP baseline | Use `show standby brief` on R4 and R5. Confirm the PC12/PC13 gateway and ping the virtual IP. | R4 is expected to be active and R5 standby for group 1. Clients use `192.168.1.254`. |
| HSRP failover | In a disposable copy, disconnect the active router's LAN-facing link and recheck both roles. Reconnect it and observe preemption. | R5 assumes the active role after convergence, the virtual gateway remains the client gateway, and R4 resumes the active role if preemption is present. Check routed reachability separately. |
| R2 ACL and R1 NAT/PAT | Use the service checks below and compare `show access-lists 100` on R2 before and after. For permitted traffic, inspect `show ip nat translations` and `show ip nat statistics` on R1. | Matching deny counters increase where the platform exposes them. Permitted traffic that crosses the NAT boundary creates translations if configured. For PAT, verify multiple concurrent inside sessions sharing the outside address. |
| SSH management | Run `show ip ssh` and inspect the VTY section of `show running-config`. Test an SSH session from a reachable lab endpoint with the simulation's local credentials. | SSHv2 is enabled and a valid local login works when the target is reachable. VTY transport is configured for SSH. |

### R2 ACL test cases

Verify Server0's configured IP address, gateway, and enabled services before interpreting application behavior. The intended ACL target is `209.165.200.100`. Use the relevant Packet Tracer client or Simulation Mode. Compare a deny case with an allowed case for the same service.

| Source | Expected deny | Control traffic permitted by ACL 100 |
| --- | --- | --- |
| IT · `192.168.10.0/24` | TCP 80 and 443 to the ACL target | ICMP to that target |
| HR · `192.168.20.0/24` | FTP control on TCP 21 to the ACL target | HTTP to that target |
| Sales · `192.168.30.0/24` | ICMP to the ACL target | HTTP to that target |

A permitted ACL decision still depends on routing, translation, and server service availability. The FTP rule addresses the control connection on TCP 21.

## Troubleshooting by symptom

| Symptom | Diagnostic path |
| --- | --- |
| Client receives no DHCP lease | Check the client link and access VLAN, then confirm the VLAN is carried on the trunk and its router subinterface is up. On R2 or R3, check `show ip dhcp pool` and `show ip dhcp binding`. |
| Switch uplink fails to bundle | Compare the two ends for the same negotiation protocol, compatible modes, matching trunk settings, and expected member ports. Check `show etherchannel summary` before interpreting STP port state. |
| Remote LAN is unreachable | Confirm local gateway reachability, the transit link, OSPF neighbor state, and forward and return routes. For a serial link that is down, inspect the DCE side and clocking. |
| Server service fails | Verify the target IP and service. Compare ACL 100 counters on R2, then check R1's NAT ACL, inside/outside roles, and translations. Check the server's gateway or return route. |
| HSRP changes roles but remote connectivity fails | Verify client reachability to the virtual IP, then check the new active router's OSPF path, R1 edge translation where relevant, and the return path. |
| SSH login fails | Check IP reachability, the hostname and domain used to generate RSA keys, SSH status, and local authentication on the VTY lines. |

### Source configuration details to verify

- **NAT/PAT.** The source NAT statement omits `overload`. Confirm the saved R1 configuration before testing shared-address translation. [Cisco IOS XE NAT configuration guide](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)
- **Default route.** The R1 notes use an interface-only Ethernet default route. Verify the installed route and upstream next hop before treating default-route advertisement as evidence of external reachability.
- **Switching templates.** The notes reuse interface ranges for LACP and PAgP and list port security on `Fa0/1`. Confirm actual switch roles and port assignments first, including the STP root per VLAN.
- **SSH examples.** The reference material contains lab-only example credentials. Keep any production credentials out of simulation files and public notes.

When collecting evidence, record the simulation filename, device or client, command or stimulus, expected outcome, observed outcome, and any changes made in the disposable copy. The [original study notebook](reference/ccna-project-notebook.pdf) contains additional verification examples and troubleshooting scenarios.
