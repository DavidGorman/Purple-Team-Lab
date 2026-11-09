Single source of truth for every VM, address and segment in the lab. Update this whenever something changes, and record the change in [changelog.md](changelog.md).

## Segments

| Segment | VMware network   | Subnet         | Gateway                  | Purpose                                                    |
| ------- | ---------------- | -------------- | ------------------------ | ---------------------------------------------------------- |
| WAN     | WAN LAN segment  | 203.0.113.0/24 | None (no upstream)       | Simulated internet, attacker side                          |
| DMZ     | DMZ LAN segment  | 10.10.10.0/24  | 10.10.10.1 (fw01)        | Public-facing services                                     |
| LAN     | LAN LAN segment  | 10.10.20.0/24  | 10.10.20.1 (fw01)        | Internal corporate network                                 |
| MGMT    | MGMT LAN segment | 10.10.30.0/24  | 10.10.30.1 (fw01)        | Administration and monitoring                              |
| VPN     | OpenVPN tunnel   | 10.10.40.0/24  | 10.10.40.1 (fw01)        | Remote access (planned)                                    |
| UPLINK  | VMnet8 (NAT)     | 192.168.1.0/24 | 192.168.1.2 (VMware NAT) | fw01's real internet connection, maintenance only |

WAN, DMZ, LAN and MGMT are VMware LAN segments: no VMware DHCP and no host adapter, so the lab cannot reach the host PC. fw01 is the only route between segments. The only adapters on VMnet8 are fw01's UPLINK and kali's maintenance adapter, both disconnected except for updates. On VMnet8, 192.168.1.1 is the host PC and 192.168.1.2 is VMware's NAT gateway.

## Hosts

| Hostname | Role                    | OS                                         | Segment | IP                                   | RAM       | Status         | Notes                                 |
| -------- | ----------------------- | ------------------------------------------ | ------- | ------------------------------------ | --------- | -------------- | ------------------------------------- |
| fw01     | Firewall / router / VPN | pfSense CE 2.9.0                           | All     | INTERNET 203.0.113.1, UPLINK 192.168.1.10, .1 on each segment | 1 GB      | Configured     | em0 UPLINK (pfSense WAN, VMnet8), em1 LAN, em2 DMZ, em3 MGMT, OPT3 INTERNET (WAN segment) |
| kali     | Attacker                | Kali Linux                                 | WAN     | 203.0.113.50 static (eth2)           | 2 GB      | Configured     | Outside the firewall; no internet unless a NAT adapter is connected for updates |
| web01    | Web server              | Ubuntu Server 24.04 LTS (minimized)        | DMZ     | 10.10.10.10 (planned)                | 2 GB      | OS installed   | Dunder Mifflin web app                |
| dc01     | Domain Controller, DNS  | Windows Server 2025 Standard (Server Core) | LAN     | 10.10.20.10 (planned)                | 2 GB      | Hostname set   | Managed with PowerShell               |
| ws01     | Workstation             | Windows 11 Enterprise (eval)               | LAN     | DHCP (seen as 10.10.20.101, 9 Nov; to confirm) | 2 to 4 GB | Network tested | Domain join pending                   |
| siem01   | SIEM                    | Ubuntu Server 24.04 LTS (minimized)        | MGMT    | 10.10.30.10 (planned)                | 6 to 8 GB | OS installed   | Manual Wazuh install, static IP first |

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

## Firewall

Rules are defined in [firewall-rules.md](firewall-rules.md) and justified by [network-security-policy.md](network-security-policy.md). Implementation status:

| Area | Status |
| --- | --- |
| Aliases | Done |
| WAN port forward to web01 | Done, tested from kali (3 Nov 2026) |
| LAN rules, default allow rules disabled | In progress (9 Nov 2026) |
| DMZ and MGMT rules | Not started |
| Remote syslog to siem01 | Waiting on siem01 |
| OpenVPN | Not started |
| Simulated internet and UPLINK | Done, tested 9 Nov 2026 (isolation confirmed with UPLINK disconnected) |

## Evaluation licence clock

| Host | Installed  | Activate by | Expires    | Rearms used |
| ---- | ---------- | ----------- | ---------- | ----------- |
| dc01 | 25/10/2026 | 04/11/2026  | 23/04/2027 | 0 of 5      |
| ws01 | 25/10/2026 | n/a         | 23/01/2027 | TBC         |