# 2026-10-25: Project planning

**Phase:** Planning
**Time spent:**

## Goal

Scope the lab, choose the architecture and tooling, and set up this repository.

## What I did

- Defined the scenario: DMZ web server breach, attempted pivot into internal AD
- Chose the hypervisor: VMware Workstation on the main PC, VMs capped at 24 GB of 32 GB RAM, powered on selectively
- Designed four segments behind pfSense: WAN (attacker), DMZ, LAN, MGMT
- Decided to install Wazuh manually on Ubuntu Server rather than use the OVA, for hands-on experience
- Picked hardening frameworks: ASD Essential Eight (ML1), ASD ISM, CIS Benchmarks
- Created the repository structure, network register, change log and references

## Decisions and reasoning

| Decision | Why |
| --- | --- |
| Windows Server and Windows 11 evaluation editions | Free and legal; Server eval runs 180 days, client 90 days |
| Windows 11 rather than Windows 10 | Windows 10 is past end of support |
| Linux PHP web app in the DMZ, specific CVE chosen later | Pipeline matters more than the exact vulnerability |
| Kali on the WAN side | Forces every attack through the firewall, like a real external attacker |

## Next step

Build starts December 2026: create VMnets, install pfSense, assign interfaces.
