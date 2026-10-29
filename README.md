# Self-Healing Network: Automated Compliance Auditing and Drift Remediation

A Cisco Packet Tracer 9.0.1 campus + branch network that serves as a hardened, verified **golden baseline**. Router and switch configs are later audited against this baseline to detect configuration drift and restore compliance.

> **Status:** Phase 1 (secure baseline network) is complete in this repo. The auditing and remediation tooling (Phases 2–4) is planned and not yet included.

## Problem Statement

Network devices drift away from their intended secure configuration over time. Causes include manual changes, quick troubleshooting fixes that are never reverted, misconfiguration and malicious tampering. A single changed line can open a hole: a removed ACL, disabled port security, Telnet re-enabled, or an unused port turned back on. In most small organisations:

- Nobody holds a formal, verified record of what each device's configuration *should* be.
- Drift is found only after an outage or security incident, often weeks later.
- Checking compliance means reading `show running-config` on every device by hand, which is slow and error-prone.
- Fixing drift is manual, so the network stays non-compliant until someone notices and acts.

This project addresses the problem in two parts:

1. **Define "compliant".** Build a segmented, hardened network for one campus site (staff, guests, servers, IT admins) and one branch office. Hardening uses VLAN segmentation, inter-VLAN routing, ACLs, port security, SSH-only management, authenticated OSPF and central Syslog/NTP. Every expected behaviour is verified, then the result is saved as the golden baseline (`1_golden.pkt` + running configs).
2. **Detect and heal drift.** Compare current device configs against the golden baseline, report every deviation as a compliance violation, and push corrective commands to restore the baseline.

Without the hardening baseline, the network itself would have these weaknesses:

- Guest devices can reach staff PCs and internal servers.
- Anyone on the LAN can try to log in to routers and switches, often over cleartext Telnet.
- Unused switch ports are live, so anyone can plug in a rogue device.
- Devices have no central logging or common time, so incidents cannot be traced.
- Branch routes are configured by hand and routing updates are not authenticated.

## Topology

```
                 [R2] Gi0/1 ── PC7 (Branch, 172.16.1.0/24)
                  │ Gi0/0
                  │ 10.0.12.0/30 (OSPF area 0, MD5 auth)
                  │ Gi0/1
                 [R1] Gi0/0 (router-on-a-stick: .10 .20 .30 .99)
                  │
                 [SW1] ── Fa0/4 ── SRV1 (VLAN 20)
                 /    \
           Fa0/2      Fa0/3
             /          \
         [SW2]          [SW3]
     PC1 PC2 PC3     PC4 PC5 PC6
```

| Device | Model | Role |
|---|---|---|
| R1 | 2911 | Campus router: inter-VLAN routing, DHCP, ACLs, OSPF |
| R2 | 2911 | Branch router: OSPF, branch LAN gateway |
| SW1 | 2960 | Distribution switch: trunks to R1/SW2/SW3, server port |
| SW2, SW3 | 2960 | Access switches |
| SRV1 | Server-PT | Syslog, NTP, TFTP, DNS address for clients |
| PC1–PC7 | PC-PT | End hosts |

## Addressing and VLANs

| VLAN | Name | Subnet | Gateway | Hosts | Addressing |
|---|---|---|---|---|---|
| 10 | Staff | 192.168.10.0/24 | 192.168.10.1 | PC1, PC2, PC4 | DHCP pool `STAFF` |
| 20 | Servers | 192.168.20.0/24 | 192.168.20.1 | SRV1 (192.168.20.10) | Static |
| 30 | Guest | 192.168.30.0/24 | 192.168.30.1 | PC3, PC5 | DHCP pool `GUEST` |
| 99 | Management (native) | 192.168.99.0/24 | 192.168.99.1 | PC6 (192.168.99.100), SW1 .11, SW2 .12, SW3 .13 | Static |
| 999 | UNUSED | none | none | All unused ports (shut down) | none |
| none | R1–R2 link | 10.0.12.0/30 | none | R1 .1, R2 .2 | Static |
| none | Branch LAN | 172.16.1.0/24 | 172.16.1.1 | PC7 (172.16.1.10) | Static |

DHCP excludes `.1`–`.10` in the Staff and Guest pools.

## Security Features

| Feature | Where | Purpose |
|---|---|---|
| VLAN segmentation | SW1–SW3 | Separate Staff, Servers, Guest and Management traffic |
| Router-on-a-stick | R1 `Gi0/0.x` subinterfaces | Inter-VLAN routing |
| `GUEST_IN` extended ACL | R1 `Gi0/0.30` inbound | Guests are denied access to Staff, Servers and Management, but can still reach other networks (branch/internet) |
| `MGMT_ONLY` standard ACL on VTY | All devices | Only VLAN 99 may open remote sessions |
| SSH v2 only, local login | All devices | No Telnet, per-user accounts |
| `enable secret`, `service password-encryption`, MOTD banner | All devices | Credential hygiene, legal notice |
| Port security (max 2, sticky, shutdown) + PortFast | All access ports | Stop MAC flooding and rogue devices |
| Unused ports in VLAN 999 and shut down | All switches | Shrink attack surface |
| Trunk allowed list `10,20,30,99`, native VLAN 99 | Trunks | No VLAN 1 on trunks, limited VLAN hopping |
| OSPF MD5 authentication, passive LAN interfaces | R1–R2 | Block rogue routing updates |
| Syslog + NTP to 192.168.20.10, log timestamps | All devices | Central, time-correlated audit trail |

## Use Cases

1. **Staff work.** Staff PCs get an address by DHCP and reach the server VLAN and the branch office.
2. **Guest Wi-Fi/visitor access.** Guests get an address and can reach the branch/outside, but are blocked from Staff, Servers and Management.
3. **Secure device administration.** IT admins on VLAN 99 (PC6) manage every router and switch over SSH. SSH attempts from any other VLAN are refused.
4. **Branch connectivity.** The branch LAN is learned dynamically through authenticated OSPF, with no static routes.
5. **Rogue device protection.** An unknown device plugged into an access port, or more than 2 MACs on one port, puts the port in err-disabled state. Unused ports are dead.
6. **Centralised logging and time sync.** All devices send timestamped logs to SRV1 and sync time from it, for incident investigation.
7. **Compliance auditing (planned).** Current running configs are compared against the golden configs. Each deviation is reported as a violation: an ACL removed, port security disabled, Telnet re-enabled, an unused port no-shut, or OSPF authentication removed.
8. **Drift remediation (planned).** For each violation, corrective commands are generated to restore the baseline. Example: re-apply `ip access-group GUEST_IN in` on `Gi0/0.30`.
9. **Tamper evidence.** Syslog entries on SRV1, with NTP-synced timestamps, show when a drifting change was made and from where.
10. **Teaching/lab exercise.** Demonstrates VLANs, trunking, inter-VLAN routing, DHCP, ACLs, OSPF, device hardening and configuration compliance in one topology.

## Roadmap

| Phase | Scope | Status |
|---|---|---|
| 1. Secure baseline | Topology, hardened configs, verification tests, golden configs | Done |
| 2. Compliance audit | Parse running configs, compare to golden, report violations | Planned |
| 3. Drift simulation | Inject misconfigurations (e.g. remove `GUEST_IN`, enable Telnet) to test detection | Planned |
| 4. Remediation | Generate and apply corrective commands, then re-verify compliance | Planned |

## Files

| File | Contents |
|---|---|
| `project.pkt` | Packet Tracer 9.0.1 project file |
| `BUILD_GUIDE.md` | Step-by-step build: devices, cabling, paste order, end-device settings, verification tests |
| `R1.txt`, `R2.txt` | Router configs (paste into CLI) |
| `SW1.txt`, `SW2.txt`, `SW3.txt` | Switch configs (paste into CLI) |

## Quick Start

1. Build and cable the topology as described in `BUILD_GUIDE.md` sections 1–2.
2. Paste the configs into each device's CLI in this order: SW1, SW2, SW3, R1, R2.
3. Set the end devices as in `BUILD_GUIDE.md` section 4. Then turn on Syslog, NTP and TFTP on SRV1.
4. Run every test in `BUILD_GUIDE.md` section 5. All tests must give the expected result.
5. Save as `1_golden.pkt`. Then export `show running-config` from each device.

## Verification Summary

| Test | Expected |
|---|---|
| R1 `show ip ospf neighbor` | R2 FULL |
| R1 `show ip route ospf` | 172.16.1.0/24 learned |
| PC1 ping SRV1 / PC7 | Success |
| PC3 (Guest) ping PC1 / SRV1 | Fail (`GUEST_IN`) |
| PC3 (Guest) ping PC7 | Success |
| PC6 `ssh -l admin 192.168.99.1` | Login works |
| PC1 `ssh -l admin 192.168.99.1` | Refused (`MGMT_ONLY`) |
| SW2 `show port-security` | Enabled on Fa0/2–0/4 |
| SW2 `show vlan brief` | VLANs 10, 20, 30, 99, 999 present |

## Known Limitations

- **Lab credentials only.** The configs hold plaintext passwords (`Admin@123`, `Cisco@123`, OSPF key `OSPFKey1`). Change them before any real use, and never commit real credentials.
- The DHCP pools give `192.168.20.10` as DNS, but `GUEST_IN` blocks Guest access to `192.168.20.0/24`. Guests therefore get no DNS resolution.
- RSA keys are 1024-bit, the Packet Tracer default. Real deployments should use 2048-bit or larger.
- `BUILD_GUIDE.md` states the configs are not yet tested in Packet Tracer.
