# Architecture (detailed)

This document gives the full addressing, the zones and the filtering policy of the lab. It completes the overview in the main README.

## Virtual machines

Twelve machines run across the headquarters, the branch site and a simulated Internet.

| Machine | System | Address | Role |
|---|---|---|---|
| HQ-FW | OPNsense | .10 WAN, .1 on each zone | Central firewall, IPsec, OpenVPN, Suricata, CA |
| BR-FW | OPNsense | .20 WAN, 10.20.10.1 | Branch firewall, IPsec |
| SRV-LDAP | Debian 12 | 10.10.20.10 | OpenLDAP directory |
| SRV-APPS | Debian 12 | 10.10.20.20 (.21 to .25) | Simulated business resources |
| DMZ-WEB | Debian 12 | 10.10.30.10 | Nginx : portal and doctor area |
| DEMO-WEB | Debian 12 | 10.10.40.10 | Demo copy |
| SIEM-WAZUH | appliance | 10.10.99.10 | Collection and alerts |
| ADMIN-WS | Debian | 10.10.99.50 | Administration workstation |
| HQ-PC | Debian | 10.10.10.50 | LAN user |
| BR-PC | Debian | 10.20.10.50 | Branch user |
| EXT-PATIENT | Debian | 203.0.113.101 | Patient and doctor |
| EXT-VPN | Debian | 203.0.113.102 | VPN client and tests |

Each zone is an isolated VMware segment. Consequently, only the firewall connects the zones, and the isolation does not depend on any VLAN.

## Addressing

| Zone | Network | Gateway | Main hosts |
|---|---|---|---|
| Simulated Internet | 203.0.113.0/24 | NAT .2 | HQ-FW .10, BR-FW .20, EXT-PATIENT .101, EXT-VPN .102 |
| HQ LAN | 10.10.10.0/24 | .1 | HQ-PC .50 |
| Servers | 10.10.20.0/24 | .1 | SRV-LDAP .10, SRV-APPS .20 (.21 to .25) |
| DMZ | 10.10.30.0/24 | .1 | DMZ-WEB .10 |
| Demo | 10.10.40.0/24 | .1 | DEMO-WEB .10 |
| Management | 10.10.99.0/24 | .1 | SIEM-WAZUH .10, ADMIN-WS .50 |
| Branch LAN | 10.20.10.0/24 | .1 | BR-PC .50 |
| OpenVPN tunnel | 10.8.0.0/24 | | one /28 per group |

Business resources are simulated on SRV-APPS. Each one has its own address and its own nginx instance, which allows precise rules (one resource per allowed group).

| Address | Resource | Allowed group |
|---|---|---|
| 10.10.20.21 | CRM | sales |
| 10.10.20.22 | Support apps | support |
| 10.10.20.23 | Accounting server | accounting |
| 10.10.20.24 | Office resources | employees |
| 10.10.20.25 | Technical resources (web + SSH) | IT |

Each remote group gets a /28 of the 10.8.0.0/24 tunnel, and each user a fixed address inside that range. That fixed address is the key of the identity policy, because the firewall recognizes the user by his address.

| Group | Tunnel range | Test user (fixed IP) |
|---|---|---|
| support | 10.8.0.16/28 | u.support (10.8.0.17) |
| accounting | 10.8.0.32/28 | u.compta (10.8.0.33) |
| employees | 10.8.0.48/28 | u.employe (10.8.0.49) |
| IT | 10.8.0.64/28 | u.it (10.8.0.65) |
| admins | 10.8.0.80/28 | u.admin (10.8.0.81) |
| sales | 10.8.0.96/28 | u.commercial (10.8.0.97) |

## Zones and trust levels

| Zone | Trust level | General rule |
|---|---|---|
| Internet | 0, untrusted | Nothing inbound except 443 toward the DMZ and the VPN |
| DMZ | 1, exposed | Cannot initiate toward the inside, except its logs to the SIEM |
| Demo | 1, isolated | No link with production, nor with real data |
| Branch LAN | 3, remote internal | Limited access to business resources through the tunnel |
| HQ LAN | 3, internal | Internet egress and targeted resources, no administration |
| Servers | 4, sensitive | Reachable only from authorized flows |
| Management | 5, critical | Reachable only by the administrators |

The default policy on each interface is deny, with logging. Consequently, a missing rule becomes a visible block and never an accidental allow.

## Filtering matrix

Read from the source row toward the destination column. `x` means denied and logged.

| Source / Dest | Internet | DMZ | Demo | LAN | Servers | MGMT |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| **Internet** | | 443 | x | x | x | x |
| **DMZ** | updates | | x | x | x | logs |
| **Demo** | x | x | | x | x | logs |
| **LAN** | web | x | x | | identity | x |
| **Servers** | updates | x | x | x | | logs |
| **MGMT** | updates | 22 | 22 | x | 22 | |
| **Branch LAN** | via BR | x | x | x | identity | x |

Here "logs" means 1514/1515 toward the SIEM only, and "identity" means the per-user rules below.

## Identity-based rules

| Group (VPN subnet) | Allowed | Denied |
|---|---|---|
| support (10.8.0.16/28) | support app (.22) | everything else |
| accounting (10.8.0.32/28) | accounting server (.23) | everything else |
| employees (10.8.0.48/28) | office resources (.24) | everything else |
| IT (10.8.0.64/28) | technical resource (.25 on 22 and 443), SSH to servers | firewalls, management |
| admins (10.8.0.80/28) | full management zone (GUI, SIEM, SSH) | |
| sales (10.8.0.96/28) | CRM (.21) and demo zone | production, management |
| HQ LAN | office, support, CRM, DNS | management, DMZ, demo, LDAP, accounting, technical |
| Branch LAN | office, support, CRM, DNS | management, DMZ, demo, LDAP |

## Shared services

- **NTP.** HQ-FW is the time source for the headquarters, BR-FW synchronizes on a public pool through its own WAN, and BR-PC on BR-FW. Certificates and logs depend on time, so this matters.
- **DNS.** The Unbound resolver of OPNsense handles external resolution. The lab names are set through `/etc/hosts` on the clients.
- **Addressing.** Static everywhere, no DHCP, which keeps the lab reproducible.
- **Backup.** Export of the OPNsense `config.xml` on both firewalls, plus VMware snapshots.
