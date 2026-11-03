Nov 3, 2026

## 1. Document control

This policy defines what network traffic is permitted in the Purple Team Lab and why. Firewall rules are derived from it, not the other way round.

| Field             | Value                                              |
| ----------------- | -------------------------------------------------- |
| Version           | 0.2                                                |
| Status            | Draft                                              |
| Enforcement point | pfSense firewall                                   |
| Related documents | [Firewall Rule Set](firewall-rules.md), [Network Register](network-register.md) |

| Version | Date       | Change        |
| ------- | ---------- | ------------- |
| 0.1     | 3 Nov 2026 | Initial draft |
| 0.2     | 3 Nov 2026 | Web server published to WAN; full tunnel VPN; added P-MGT-04, P-VPN-06, P-VPN-07; P-LAN-04 limited to the DC |

## 2. Purpose and scope

The purpose of this policy is to segment the lab network so that each zone can only reach what it needs to do its job, and a compromise in one zone does not spread freely to another.

**In scope**

- The pfSense firewall and all of its interfaces
- The LAN, DMZ, MGMT and VPN zones and every host in them
- Remote access into the lab
- Administrative access to lab infrastructure

**Out of scope**

- Host-level hardening of individual machines (covered by separate build documents)
- Deliberate vulnerabilities and attack scenarios, which will be introduced later and recorded as exceptions under section 9
- The physical host machine and the home network the lab runs on

This version describes the baseline network: every system working as intended, with no deliberate weaknesses.

## 3. Network zones

The lab is divided into five zones, each on its own firewall interface. Trust level sets how much a zone is allowed to reach, not how important its systems are.

| Zone | Purpose | Key systems | Trust level |
| ---- | ------- | ----------- | ----------- |
| LAN | Internal corporate network for staff workstations and identity services | Domain controller (dc01), workstations | Medium |
| DMZ | Hosts services exposed to less trusted networks | Web server (web01) | Low |
| MGMT | Security operations and monitoring; treated as the SOC | SIEM (siem01) | High |
| VPN | Remote access for administration and remote staff | Remote clients (tunnel subnet) | Medium, authenticated |
| WAN | Everything outside the lab, including the internet | n/a | Untrusted |

## 4. General principles

These principles apply to every zone and every rule.

1. **Default deny.** Any traffic not explicitly permitted by this policy is blocked.
2. **Least privilege.** Permitted traffic is limited to the specific hosts and services required, never whole zones where a single host will do.
3. **Single enforcement point.** All traffic between zones passes through the pfSense firewall. No host may bridge two zones.
4. **Stateful inspection.** Return traffic for a permitted connection is allowed automatically. Rules are written only for the side that starts the connection.
5. **Direction matters.** Permission for zone A to connect to zone B does not grant zone B permission to connect to zone A.
6. **Traceability.** Every firewall rule must trace back to a statement in this policy or an approved exception.

## 5. Inter-zone traffic policy

Only the traffic below crosses the firewall in normal operation. Rows are the zone that starts the connection; columns are the destination.


| From     | Internet             | LAN            | DMZ            | MGMT            | Firewall  |
| -------- | -------------------- | -------------- | -------------- | --------------- | --------- |
| **LAN**  | Permit               | n/a            | Web only       | SIEM agent only | DNS, NTP (DC only) |
| **DMZ**  | Updates only         | Deny           | n/a            | SIEM agent only | DNS, NTP  |
| **MGMT** | Updates only         | Deny           | Deny           | n/a             | DNS, NTP  |
| **VPN**  | Permit (full tunnel) | Admin services | Admin services | Deny            | Admin GUI |
| **WAN**  | n/a                  | Deny           | Web only       | Deny            | VPN only  |

Each statement has an ID so firewall rules can reference it.

**LAN**

- **P-LAN-01** LAN hosts may initiate connections to the internet. LAN may not reach other internal zones under this permission.
- **P-LAN-02** LAN hosts may access the web service on web01. This is treated the same as public access: web01 may only respond, never initiate.
- **P-LAN-03** The domain controller may send agent data to siem01.
- **P-LAN-04** The domain controller may use the firewall for DNS forwarding and time synchronisation. Other LAN hosts use the domain controller for DNS and time.

**DMZ**

- **P-DMZ-01** The DMZ shall not initiate any connection to the LAN.
- **P-DMZ-02** web01 may reach the internet for operating system and package updates only.
- **P-DMZ-03** web01 may send agent data to siem01. No other DMZ host inherits this permission.
- **P-DMZ-04** DMZ hosts may use the firewall for DNS forwarding and time synchronisation.

**MGMT**

- **P-MGT-01** MGMT accepts inbound connections only from approved SIEM agents and from the firewall's log forwarding.
- **P-MGT-02** siem01 may reach the internet for updates and detection content only.
- **P-MGT-03** MGMT shall not initiate connections to the LAN or DMZ.
- **P-MGT-04** siem01 may use the firewall for DNS forwarding and time synchronisation.

**WAN**

- **P-WAN-01** No inbound connections from the WAN are permitted except those listed in this section.
- **P-WAN-02** WAN hosts may access the public web service on web01. No other DMZ service or host is published to the WAN.
- **P-WAN-03** WAN hosts may connect to the firewall's VPN service.

## 6. Remote access

The VPN is the only way into the lab from outside. A single VPN service is used in this version.

- **P-VPN-01** All remote access to any internal zone shall be made through the VPN. The only service published directly to the WAN is the public web service on web01 (P-WAN-02).
- **P-VPN-02** VPN users must authenticate before receiving a tunnel address.
- **P-VPN-03** VPN clients are placed in their own tunnel subnet and are subject to firewall rules like any other zone.
- **P-VPN-04** VPN clients may reach administrative services on LAN and DMZ hosts.
- **P-VPN-05** VPN clients shall not reach the MGMT zone (see section 7).
- **P-VPN-06** VPN clients may use the domain controller for DNS.
- **P-VPN-07** The VPN operates as a full tunnel. All client traffic, including internet access, passes through the firewall and is subject to this policy.

**Open questions**

- When remote staff are added, should they move to a separate VPN with LAN-only access, leaving the current VPN for administrators?

## 7. Administrative access

Administrative interfaces are only reachable from the places listed here.

- **P-ADM-01** The firewall's web management interface shall be reachable only from the VPN. The default LAN anti-lockout permission is removed once VPN administration is confirmed working.
- **P-ADM-02** Administrative services on LAN and DMZ hosts (remote desktop, SSH) shall be reachable only from the VPN.
- **P-ADM-03** siem01 and its dashboard are administered from within the MGMT zone only, equivalent to physical access inside a security operations centre. There is no remote path to the SIEM.
- **P-ADM-04** Administrative services shall not be reachable from the WAN under any circumstances.

## 8. Logging and monitoring

All security-relevant events are collected centrally on siem01.

- **P-LOG-01** The firewall shall forward its logs to siem01.
- **P-LOG-02** Denied traffic between zones shall be logged. Permitted traffic should be logged where it supports investigation, without flooding the SIEM.
- **P-LOG-03** All servers (dc01, web01) shall run a SIEM agent reporting to siem01.
- **P-LOG-04** The SIEM is a logging destination only and does not initiate connections to the hosts it monitors.

## 9. Accepted risks and exceptions

Any traffic outside this policy needs a recorded exception before a rule is created.

**Accepted risks**

| ID   | Risk                                                                                       | Reason accepted                | Mitigation                                                     |
| ---- | ------------------------------------------------------------------------------------------ | ------------------------------ | -------------------------------------------------------------- |
| R-01 | web01 can reach siem01 over the agent channel, so a compromised web01 has a path into MGMT | Monitoring the DMZ requires it | Rule limited to web01 only; agent enrolment closed after setup |

**Exception process**

1. Describe the traffic needed (source, destination, service) and why.
2. Assign an exception ID (E-01, E-02, ...) and record it in the table below.
3. Create the firewall rule with the exception ID in its description.
4. Review the exception at each policy review and remove it when no longer needed.

| ID  | Traffic | Reason | Date added | Review by |
| --- | ------- | ------ | ---------- | --------- |

Deliberate weaknesses introduced for attack scenarios will be recorded here as exceptions, so the hardening phase can show each one being closed.

## 10. Review and change control

The policy is reviewed at the end of each sprint and whenever a zone, host or exception is added.

- **P-CHG-01** Changes to this policy are recorded in the version table in section 1.
- **P-CHG-02** A firewall rule change must reference the policy statement or exception ID it implements.
- **P-CHG-03** Rules that no longer trace to a policy statement or active exception are removed at review.
- **P-CHG-04** The firewall configuration is backed up before and after any rule change.