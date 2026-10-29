# Build guide: baseline network in Packet Tracer 9.0.1 (no .pkt file needed)

Configs in this folder were generated from one template and have NOT been tested in Packet Tracer. Paste, then run the checks at the bottom. If a command is rejected, note it and tell me, and we will adapt it.

## 1. Add devices
| Name | Device |
|---|---|
| R1, R2 | Routers > 2911 |
| SW1, SW2, SW3 | Switches > 2960 |
| SRV1 | End Devices > Server-PT |
| PC1 to PC7 | End Devices > PC-PT |

## 2. Cable them (Connections > Copper Straight-Through, or Crossover for like devices)
| From | To |
|---|---|
| R1 Gi0/0 | SW1 Fa0/1 |
| SW1 Fa0/2 | SW2 Fa0/1 |
| SW1 Fa0/3 | SW3 Fa0/1 |
| SW1 Fa0/4 | SRV1 Fa0 |
| SW2 Fa0/2 | PC1 |
| SW2 Fa0/3 | PC2 |
| SW2 Fa0/4 | PC3 |
| SW3 Fa0/2 | PC4 |
| SW3 Fa0/3 | PC5 |
| SW3 Fa0/4 | PC6 |
| R1 Gi0/1 | R2 Gi0/0 (crossover) |
| R2 Gi0/1 | PC7 (crossover) |

PC1, PC2, PC4 are Staff (VLAN 10). PC3, PC5 are Guest (VLAN 30). PC6 is Admin (VLAN 99). PC7 is the branch PC.

## 3. Paste configs (CLI tab of each device)
Order: SW1, SW2, SW3, R1, R2. Open the file, copy everything, paste into the CLI. Wait for it to finish; the RSA key step can take a few seconds. If the CLI asks anything, answer and paste the rest again.

## 4. End devices
| Device | Setting |
|---|---|
| PC1, PC2, PC3, PC4, PC5 | Desktop > IP Configuration > DHCP |
| PC6 | Static 192.168.99.100 / 255.255.255.0, gateway 192.168.99.1 |
| PC7 | Static 172.16.1.10 / 255.255.255.0, gateway 172.16.1.1 |
| SRV1 | Static 192.168.20.10 / 255.255.255.0, gateway 192.168.20.1 |

SRV1 > Services: turn ON Syslog, NTP and TFTP.

## 5. Verify (all must pass before you save the golden copy)
| Test | Expected |
|---|---|
| R1: `show ip ospf neighbor` | R2 in FULL state |
| R1: `show ip route ospf` | 172.16.1.0/24 learned |
| PC1 pings SRV1 (192.168.20.10) | Success |
| PC1 pings PC7 (172.16.1.10) | Success |
| PC3 (Guest) pings PC1 | FAIL (blocked by GUEST_IN) |
| PC3 (Guest) pings SRV1 | FAIL |
| PC3 (Guest) pings PC7 | Success |
| PC6 `ssh -l admin 192.168.99.1` | Login works |
| PC1 `ssh -l admin 192.168.99.1` | Blocked (not in VLAN 99) |
| SW2: `show port-security` | Enabled on Fa0/2 to 0/4 |
| SW2: `show vlan brief` | VLANs 10, 20, 30, 99, 999 exist |

Note: PC1 may show a first-ping timeout while ARP resolves. Ping again.

## 6. Then
Save as `1_golden.pkt`. Copy `show running-config` from each device into text files. These are the golden configs for the audit tool.
