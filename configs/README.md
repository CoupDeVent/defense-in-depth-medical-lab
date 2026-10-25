# Configuration files

Sanitized configuration files taken from the lab. They are here as a reference, to show how each control is set up.

| File | Host | Role |
|---|---|---|
| `nginx/portal-and-pro.conf` | DMZ web | Public portal and mutual-TLS doctor area |
| `nginx/demo.conf` | Demo web | Isolated demo copy |
| `nftables/dmz-web.nft` | DMZ web | Local firewall, web open, SSH from management only |
| `nftables/app-servers.nft` | App servers | Local firewall, HTTPS and scoped SSH |
| `nftables/ldap-server.nft` | LDAP server | Local firewall, LDAPS and scoped SSH |
| `ssh/sshd_hardening.conf` | all Linux hosts | Key-only SSH, one named account, no root, no forwarding |
| `wazuh/local_decoder.xml` | SIEM | Decoder for OPNsense filterlog messages |
| `wazuh/local_rules.xml` | SIEM | Custom rules (blocks, scans, VPN failures) |