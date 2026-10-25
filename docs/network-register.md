# Network Register

Single source of truth for every VM, address and segment in the lab. Update this whenever something changes, and record the change in [changelog.md](changelog.md).

## Segments

| Segment | VMware network | Subnet | Gateway (pfSense) | Purpose |
| --- | --- | --- | --- | --- |
| WAN | VMnet8 (NAT) | Assigned by VMware | DHCP from VMware | Simulated internet, attacker side |
| DMZ | VMnet2 (host-only, isolated) | 10.10.10.0/24 | 10.10.10.1 | Public-facing services |
| LAN | VMnet3 (host-only, isolated) | 10.10.20.0/24 | 10.10.20.1 | Internal corporate network |
| MGMT | VMnet4 (host-only, isolated) | 10.10.30.0/24 | 10.10.30.1 | Administration and monitoring |

VMnet2 to VMnet4: local DHCP disabled, host virtual adapter disconnected, so the lab cannot reach the host PC.

## Hosts

| Hostname | Role | OS | Segment | IP | RAM | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| fw01 | Firewall / router / VPN | pfSense CE | All | WAN DHCP, .1 on each segment | 1 GB | One adapter per segment |
| kali | Attacker | Kali Linux | WAN | DHCP | 2 GB | Outside the firewall |
| web01 | Web server | Ubuntu Server 24.04 LTS | DMZ | 10.10.10.10 | 2 GB | Vulnerable PHP app (CVE TBC) |
| dc01 | Domain Controller, DNS | Windows Server 2025 (eval) | LAN | 10.10.20.10 | 2 to 4 GB | Activate within 10 days of install |
| ws01 | Workstation | Windows 11 Enterprise (eval) | LAN | DHCP or 10.10.20.50 | 2 to 4 GB | Domain-joined |
| siem01 | SIEM | Ubuntu Server 24.04 LTS + Wazuh | MGMT | 10.10.30.10 | 6 to 8 GB | Manual install, static IP before install |

## Active Directory

| Item | Value |
| --- | --- |
| Domain | TBC (e.g. corp.lab.local) |
| DNS | dc01, with pfSense forwarding external queries |

## Firewall policy summary

Default deny between segments. Allowed flows are listed here and justified in the change log.

| # | Source | Destination | Port / protocol | Purpose |
| --- | --- | --- | --- | --- |
| 1 | WAN | web01 | TCP 80/443 (port forward) | Public web access |
| 2 | All segments | siem01 | Wazuh agent ports | Log forwarding |
| | | | | |

## Evaluation licence clock

| Host | Installed | Expires | Rearms used |
| --- | --- | --- | --- |
| dc01 | | | 0 of 5 |
| ws01 | | | |
