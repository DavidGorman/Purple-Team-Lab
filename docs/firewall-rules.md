## 1. Addressing

Servers use static addresses below each subnet's DHCP range (.100 to .200).

| Zone | Subnet                         | fw01 interface        | Hosts                                               |
| ---- | ------------------------------ | --------------------- | --------------------------------------------------- |
| WAN  | 192.168.1.0/24 (VMnet8 NAT)    | 192.168.1.10 (static) | VMware NAT gateway 192.168.1.2, attacker VM (later) |
| DMZ  | 10.10.10.0/24                  | 10.10.10.1            | web01 10.10.10.10                                   |
| LAN  | 10.10.20.0/24                  | 10.10.20.1            | dc01 10.10.20.10, workstations via DHCP             |
| MGMT | 10.10.30.0/24                  | 10.10.30.1            | siem01 10.10.30.10                                  |
| VPN  | 10.10.40.0/24 (OpenVPN tunnel) | 10.10.40.1            | Remote clients                                      |

## 2. Aliases

Rules refer to these aliases by name.

| Alias       | Type    | Value                                     | Used for                            |
| ----------- | ------- | ----------------------------------------- | ----------------------------------- |
| web01       | Host    | 10.10.10.10                               | Web server                          |
| dc01        | Host    | 10.10.20.10                               | Domain controller                   |
| siem01      | Host    | 10.10.30.10                               | SIEM (Wazuh manager)                |
| RFC1918     | Network | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 | "Not internal" destinations         |
| Web_Ports   | Port    | 80, 443                                   | HTTP and HTTPS                      |
| Wazuh_Agent | Port    | 1514                                      | Agent event traffic                 |
| Wazuh_Enrol | Port    | 1515                                      | Agent enrolment (temporary)         |
| Win_Admin   | Port    | 3389, 5985, 5986                          | RDP and PowerShell remoting (WinRM) |

Built-in names also used: _WAN address_, _LAN address_, _DMZ address_, _MGMT address_ (fw01's own IP on that interface) and _LAN net_, _DMZ net_ (the whole subnet).

## 3. Interface settings

Non-default settings the rule set depends on.

| Setting                     | Value                                    | Reason                                  |
| --------------------------- | ---------------------------------------- | --------------------------------------- |
| WAN: block private networks | Off                                      | WAN is a private range (192.168.1.0/24) |
| WAN: block bogon networks   | On                                       | No legitimate bogon traffic expected    |
| WAN IPv4                    | Static 192.168.1.10, gateway 192.168.1.2 | Fixed target for port forward and VPN   |
| LAN default allow rules     | Removed                                  | Replaced by section 6                   |
| Anti-lockout rule           | Disabled once VPN admin works            | P-ADM-01                                |
| Default block rule logging  | On                                       | P-LOG-02                                |
| IPv6                        | Not used; no IPv6 pass rules             | Reduces attack surface                  |

## 4. NAT

| Type           | Interface | Protocol | Original destination                        | Port      | Translated to     | Policy   |
| -------------- | --------- | -------- | ------------------------------------------- | --------- | ----------------- | -------- |
| Port forward   | WAN       | TCP      | WAN address                                 | Web_Ports | web01 : Web_Ports | P-WAN-02 |
| NAT reflection | All       | n/a      | Mode: Pure NAT                              | n/a       | n/a               | P-LAN-02 |
| Outbound NAT   | WAN       | Any      | Automatic (internal subnets to WAN address) | n/a       | n/a               | P-LAN-01 |

Port forwards are applied before filter rules, so reflected LAN traffic is matched by LAN rule 4 against web01, not the WAN address.

## 5. WAN rules

Rules are listed in evaluation order; first match wins. Anything unmatched hits the default deny, which is logged.

| #   | Action | Proto | Source | Destination | Port      | Log | Policy   | Description                       |
| --- | ------ | ----- | ------ | ----------- | --------- | --- | -------- | --------------------------------- |
| 1   | Pass   | TCP   | Any    | web01       | Web_Ports | Yes | P-WAN-02 | Public web service (port forward) |
| 2   | Pass   | UDP   | Any    | WAN address | 1194      | Yes | P-WAN-03 | OpenVPN                           |

## 6. LAN rules

Rule 7 blocks all internal destinations, so rule 8 only reaches the internet.

| #   | Action | Proto   | Source  | Destination | Port        | Log | Policy   | Description                           |
| --- | ------ | ------- | ------- | ----------- | ----------- | --- | -------- | ------------------------------------- |
| 1   | Pass   | TCP     | LAN net | LAN address | 443         | No  | P-ADM-01 | Anti-lockout (temporary, system rule) |
| 2   | Pass   | TCP/UDP | dc01    | LAN address | 53          | No  | P-LAN-04 | DNS forwarding from DC                |
| 3   | Pass   | UDP     | dc01    | LAN address | 123         | No  | P-LAN-04 | NTP from DC                           |
| 4   | Pass   | TCP     | LAN net | web01       | Web_Ports   | No  | P-LAN-02 | Company website (via NAT reflection)  |
| 5   | Pass   | TCP     | dc01    | siem01      | Wazuh_Agent | No  | P-LAN-03 | SIEM agent                            |
| 6   | Pass   | TCP     | dc01    | siem01      | Wazuh_Enrol | No  | P-LAN-03 | Agent enrolment (disable after)       |
| 7   | Block  | Any     | LAN net | RFC1918     | Any         | Yes | P-LAN-01 | Block all other internal traffic      |
| 8   | Pass   | Any     | LAN net | Any         | Any         | No  | P-LAN-01 | Internet access                       |

## 7. DMZ rules

| #   | Action | Proto   | Source  | Destination | Port        | Log | Policy   | Description                                     |
| --- | ------ | ------- | ------- | ----------- | ----------- | --- | -------- | ----------------------------------------------- |
| 1   | Pass   | TCP/UDP | web01   | DMZ address | 53          | No  | P-DMZ-04 | DNS                                             |
| 2   | Pass   | UDP     | web01   | DMZ address | 123         | No  | P-DMZ-04 | NTP                                             |
| 3   | Pass   | TCP     | web01   | siem01      | Wazuh_Agent | No  | P-DMZ-03 | SIEM agent (R-01)                               |
| 4   | Pass   | TCP     | web01   | siem01      | Wazuh_Enrol | No  | P-DMZ-03 | Agent enrolment (disable after)                 |
| 5   | Block  | Any     | DMZ net | RFC1918     | Any         | Yes | P-DMZ-01 | Block all other internal traffic, including LAN |
| 6   | Pass   | TCP     | web01   | Any         | Web_Ports   | No  | P-DMZ-02 | Updates                                         |

## 8. MGMT rules

Inbound agent traffic is permitted on the LAN and DMZ interfaces, where it enters the firewall.

| #   | Action | Proto   | Source   | Destination  | Port      | Log | Policy           | Description                      |
| --- | ------ | ------- | -------- | ------------ | --------- | --- | ---------------- | -------------------------------- |
| 1   | Pass   | TCP/UDP | siem01   | MGMT address | 53        | No  | P-MGT-04         | DNS                              |
| 2   | Pass   | UDP     | siem01   | MGMT address | 123       | No  | P-MGT-04         | NTP                              |
| 3   | Block  | Any     | MGMT net | RFC1918      | Any       | Yes | P-MGT-03         | Block all other internal traffic |
| 4   | Pass   | TCP     | siem01   | Any          | Web_Ports | No  | P-MGT-02         | Updates and detection content    |

## 9. OpenVPN rules

VPN clients come from 10.10.40.0/24. The VPN is full tunnel: all client traffic, including internet, passes through fw01. Automatic outbound NAT covers the tunnel subnet. Rule 5 blocks all internal destinations, so rule 6 only reaches the internet.

| #   | Action | Proto   | Source        | Destination   | Port      | Log | Policy   | Description                                      |
| --- | ------ | ------- | ------------- | ------------- | --------- | --- | -------- | ------------------------------------------------ |
| 1   | Pass   | TCP     | 10.10.40.0/24 | This Firewall | 443       | Yes | P-ADM-01 | pfSense web GUI                                  |
| 2   | Pass   | TCP     | 10.10.40.0/24 | dc01          | Win_Admin | Yes | P-ADM-02 | RDP and PowerShell remoting to DC                |
| 3   | Pass   | TCP     | 10.10.40.0/24 | web01         | 22        | Yes | P-ADM-02 | SSH to web server                                |
| 4   | Pass   | TCP/UDP | 10.10.40.0/24 | dc01          | 53        | No  | P-VPN-06 | DNS for VPN clients (internal and internet)      |
| 5   | Block  | Any     | 10.10.40.0/24 | RFC1918       | Any       | Yes | P-VPN-05 | Block all other internal traffic, including MGMT |
| 6   | Pass   | Any     | 10.10.40.0/24 | Any           | Any       | No  | P-VPN-07 | Internet access through the tunnel               |

## 10. Firewall-originated traffic

Traffic sent by fw01 itself is not filtered by interface rules.

| Source                         | Destination | Proto   | Port         | Policy   | Description                           |
| ------------------------------ | ----------- | ------- | ------------ | -------- | ------------------------------------- |
| fw01 (MGMT address 10.10.30.1) | siem01      | UDP     | 514          | P-LOG-01 | Firewall logs to SIEM (remote syslog) |
| fw01 (WAN address)             | Internet    | TCP/UDP | 53, 123, 443 | n/a      | Upstream DNS, NTP and pfSense updates |

## 11. Gaps and open items

Items still open after policy v0.2.

| Item                                      | Detail                                                                                   | Proposed resolution                                             |
| ----------------------------------------- | ---------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| "Updates only" is not enforceable by port | DMZ rule 6 and MGMT rule 4 allow 80/443 to any external address, not just update servers | Accept for baseline; tighten later with FQDN aliases or a proxy |
| Wazuh dashboard access                    | P-ADM-03 needs a browser inside MGMT                                                     | SOC workstation VM in MGMT, or a desktop on siem01              |
| Enrolment rules                           | LAN rule 6 and DMZ rule 4                                                                | Disable after agents enrol (R-01 mitigation)                    |
| LAN to 192.168.1.0/24                     | LAN rule 7 blocks the simulated internet segment because it is a private range           | Accept; revisit when the attacker VM is added                   |