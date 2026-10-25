# Build guide

This guide shows how the lab was built, in the order that was followed. It is a condensed version, meant to make the project reproducible. 

## 1. VMware networks and base machines

Create one isolated VMware segment per zone (`SEG-LAN`, `SEG-SRV`, `SEG-DMZ`, `SEG-DEMO`, `SEG-MGMT`, `SEG-BR`). For the simulated Internet, use the NAT network `VMnet8` on `203.0.113.0/24` with the VMware DHCP disabled. Clone a base Debian machine for the servers and the workstations.

## 2. HQ-FW

Install OPNsense and assign the six interfaces (WAN plus one per zone), each firewall interface carrying the `.1` of its zone. Set NTP and DNS (Unbound), then a first rule that allows the LAN to reach the Internet. This is the backbone that everything else relies on.

## 3. PKI

In System, Trust, create the internal authority `MediConnect-CA`. It issues the server certificate of the portal and the doctor area (with both names as SAN), the OpenVPN server certificate, the firewall GUI certificate, and one client certificate per VPN user and per test doctor. Import the CA into the clients so they trust the server certificates.

## 4. BR-FW and the IPsec tunnel

Install the branch firewall (WAN plus one LAN). Build the site-to-site tunnel: IKEv2, AES-256, SHA-256, MODP-2048, authentication by a strong pre-shared key. Phase 2 links `10.10.0.0/16` and `10.20.10.0/24`. Validate with a ping between the two sites and check the tunnel is installed.

## 5. LDAP and application servers

On SRV-LDAP, install OpenLDAP with the base `dc=mediconnect,dc=lab`, the units `users` and `groups`, six groups and one test user per group. On SRV-APPS, add the alias addresses `.21` to `.25` and one nginx instance per resource. Then declare the LDAP server in OPNsense (System, Access, Servers), using LDAPS on port 636 and a read-only bind account.

## 6. DMZ and demo

On DMZ-WEB, install nginx with the two sites : the public portal, and the doctor area in mutual TLS (`ssl_verify_client on`) plus HTTP basic auth. Use `htpasswd` to create the doctor password (`<password>`). On the firewall, add the DNAT: `203.0.113.10:443` to the DMZ host, and `:80` only for the redirection to HTTPS. Harden both hosts with a local nftables firewall in default deny (see `configs/nftables/`). Build DEMO-WEB the same way, with fictitious data and no link to production.

The nginx configurations are in `configs/nginx/`.

## 7. OpenVPN and the identity rules

Create the OpenVPN server : UDP 1194, `tls-crypt`, client certificate required, LDAP authentication, username as common name, tunnel `10.8.0.0/24`. Add one Client Specific Override per user so each one keeps a fixed address inside the range of his group. Group those addresses into OPNsense aliases, then write the per-identity rules (see the architecture document). Export the client profiles with the OPNsense client export.

## 8. Management and administration hardening

Set the OPNsense GUI to listen only on the management interface and remove the LAN anti-lockout rule. On every Linux host, apply the SSH hardening (`configs/ssh/sshd_hardening.conf`) : key-only, no root, one named account. Check the key test : the LAN workstation must not reach the firewall GUI, nor SSH the servers.

## 9. Suricata

Enable Suricata in IPS mode (Netmap) on HQ-FW. Download a curated subset of ET Open (scan, web server, exploit, attack response, denial of service) and a sample of OPNsense-App-detect. Add a policy that turns the default action from alert to drop for the chosen categories, which moves the system from detection to actual blocking.

## 10. Wazuh

Deploy the Wazuh appliance in the management zone. Enroll one agent per host (the OPNsense plugin on the firewalls, the Debian agent elsewhere), allowed by the firewall only toward the SIEM on 1514 and 1515. Add the custom decoder and rules (`configs/wazuh/`): firewall blocks, scans, out-of-profile VPN access, and VPN authentication failures with a brute-force alert.

## 11. Final filtering

Set every interface to deny by default with logging, then keep only the required flows.