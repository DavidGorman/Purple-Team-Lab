# Purple Team Home Lab

A segmented small-business network built in VMware, then attacked, monitored, hardened and documented end to end.

The scenario: an attacker breaches a public-facing web server in the DMZ, then tries to pivot into the internal Active Directory network. The firewall rules decide whether that pivot succeeds. The hardening phase fixes both the web vulnerability and the internal weaknesses, and every change is validated by re-attacking.

> **Status:** Planning. Build starts December 2026.

## Skills demonstrated

| Area | What this project covers |
| --- | --- |
| Networking | pfSense firewall, network segmentation (DMZ / LAN / management), VPN, firewall policy |
| Systems | Ubuntu Server, Windows Server Active Directory, Windows 11 domain workstation |
| Monitoring | Wazuh SIEM, installed and configured manually, log onboarding, detection rules |
| Offensive | Scoped penetration test: recon, exploitation, post-exploitation, pivoting |
| Hardening | ASD Essential Eight, ASD ISM, CIS Benchmarks, before/after validation |
| Reporting | Penetration test report and project write-up |

## Lab architecture

| Segment | Subnet | Hosts |
| --- | --- | --- |
| WAN ("internet") | VMware NAT | Kali Linux attacker, pfSense WAN |
| DMZ | 10.10.10.0/24 | Ubuntu web server (vulnerable PHP app) |
| LAN | 10.10.20.0/24 | Windows Server DC, Windows 11 workstation |
| MGMT | 10.10.30.0/24 | Wazuh SIEM |

Full details: [docs/network-register.md](docs/network-register.md)

## Method

Each attack technique goes through the same loop:

1. **Attack** from Kali using a named technique
2. **Map** it to MITRE ATT&CK
3. **Hunt** for it in the SIEM
4. **Detect** with a rule that fires on it
5. **Harden** the target
6. **Re-attack** to prove the fix and the detection both work

## Repository layout

```
docs/          Network register, change log, references, architecture
journal/       Dated build log, one file per session
configs/       Exported and sanitised configs (pfSense, Wazuh, hardening)
detections/    Detection rules written for each technique
evidence/      Screenshots and log extracts, named by date and topic
report/        Penetration test report and final write-up
```

## Standards and references

Hardening targets ASD Essential Eight Maturity Level One across all eight strategies, with network design guided by the ASD Information Security Manual. See [docs/references.md](docs/references.md).

## Disclaimer

Everything in this repository was performed in an isolated lab I built and own. No techniques here were used against systems I do not have permission to test. Credentials shown are lab placeholders.
