# Network Register

Single source of truth for every VM, address and segment in the lab. Update this whenever something changes, and record the change in [changelog.md](changelog.md).

## Segments

| Segment | VMware network   | Subnet         | Gateway               | Purpose                           |
| ------- | ---------------- | -------------- | --------------------- | --------------------------------- |
| WAN     | VMnet8 (NAT)     | 192.168.1.0/24 | 192.168.1.2 (VMware NAT) | Simulated internet, attacker side |
| DMZ     | DMZ LAN segment  | 10.10.10.0/24  | 10.10.10.1 (fw01)     | Public-facing services            |
| LAN     | LAN LAN segment  | 10.10.20.0/24  | 10.10.20.1 (fw01)     | Internal corporate network        |
| MGMT    | MGMT LAN segment | 10.10.30.0/24  | 10.10.30.1 (fw01)     | Administration and monitoring     |

DMZ, LAN and MGMT are VMware LAN segments: no VMware DHCP and no host adapter, so the lab cannot reach the host PC. fw01 is the only route between segments. On VMnet8, 192.168.1.1 is the host PC and 192.168.1.2 is VMware's NAT gateway.

## Hosts

| Hostname | Role                    | OS                                        | Segment | IP                                   | RAM       | Status         | Notes                                     |
| -------- | ----------------------- | ----------------------------------------- | ------- | ------------------------------------ | --------- | -------------- | ----------------------------------------- |
| fw01     | Firewall / router / VPN | pfSense CE 2.9.0                          | All     | WAN 192.168.1.10, .1 on each segment | 1 GB      | Configured     | em0 WAN, em1 LAN, em2 DMZ, em3 MGMT       |
| kali     | Attacker                | Kali Linux                                | WAN     | DHCP (192.168.1.x)                   | 2 GB      | Configured     | Outside the firewall                      |
| web01    | Web server              | Ubuntu Server 24.04 LTS (minimized)       | DMZ     | 10.10.10.10 (planned)                | 2 GB      | OS installed   | Dunder Mifflin web app                    |
| dc01     | Domain Controller, DNS  | Windows Server 2025 Standard (Server Core) | LAN    | 10.10.20.10 (planned)                | 2 GB      | Hostname set   | Managed with PowerShell                   |
| ws01     | Workstation             | Windows 11 Enterprise (eval)              | LAN     | 10.10.20.50 (static, temporary)      | 2 to 4 GB | Network tested | Domain join pending                       |
| siem01   | SIEM                    | Ubuntu Server 24.04 LTS (minimized)       | MGMT    | 10.10.30.10 (planned)                | 6 to 8 GB | OS installed   | Manual Wazuh install, static IP first     |

## Active Directory

| Item          | Value                                          |
| ------------- | ---------------------------------------------- |
| Domain        | dundermifflin.local                            |
| NetBIOS name  | DUNDERMIFFLIN                                  |
| DNS           | dc01, with fw01 forwarding external queries    |
| Status        | Not yet promoted                               |

## Accounts

Passwords are stored in KeePass only, never in this repository.

| Account | System | Purpose | Deliberately weak |
| ------- | ------ | ------- | ----------------- |
| admin | fw01 | pfSense web UI administrator (default password changed) | No |
| Nick | ws01 | Local account created during install | No |
| TBC | dc01 | Local Administrator, becomes domain Administrator on promotion | No |
| TBC | web01, siem01 | Sudo admin accounts | No |
| TBC | kali | Attacker account (default kali/kali changed) | No |

## Firewall policy summary

Current state is pfSense defaults. Target is default deny between segments, with allowed flows listed here and justified in the change log.

| # | Interface | Source | Destination | Port / protocol | Purpose | Status |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | LAN | LAN net | Any | Any | pfSense default allow rule | Active, to be tightened |
| 2 | DMZ | n/a | n/a | n/a | No rules, all traffic blocked | Active |
| 3 | MGMT | n/a | n/a | n/a | No rules, all traffic blocked | Active |
| 4 | WAN | Any | web01 | TCP 80/443 (port forward) | Public web access | Planned |
| 5 | All | All segments | siem01 | Wazuh agent ports | Log forwarding | Planned |

## Evaluation licence clock

| Host | Installed  | Activate by | Expires    | Rearms used |
| ---- | ---------- | ----------- | ---------- | ----------- |
| dc01 | 25/10/2026 | 04/11/2026  | 23/04/2027 | 0 of 5      |
| ws01 | 25/10/2026 | n/a         | 23/01/2027 | TBC         |