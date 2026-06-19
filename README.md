# ANAOS — Automated Network & Analysis Operations System

![Python](https://img.shields.io/badge/python-3.x-blue)
![Wazuh](https://img.shields.io/badge/SIEM-Wazuh%204.9-00bbff)

ANAOS is an automated, open-source Security Operations Centre (SOC) combining Wazuh, Suricata/pfSense, Sysmon/Auditd, and Ansible. It was built and evaluated as part of a cybersecurity research project at ENSA Khouribga (2025-2026), demonstrating that a fully reproducible SOC can be deployed via infrastructure-as-code while still detecting real attacks with high accuracy.

Full research write-up: `docs/paper/ANAOS_Research_Chapter.pdf`

## Key Results

- 100% detection recall across four tested ATT&CK techniques
- 0% false-positive rate
- Near-instant detection (MTTD ~0s) for network-layer attacks
- Full stack deployed in under 15 minutes via Ansible

## How It Works

ANAOS spans four network zones — attacker-facing WAN, a DMZ web app, internal endpoints, and SOC servers — with all traffic inspected by pfSense/Suricata.

![Network Topology](docs/images/topology.png)

Events flow from endpoints and the firewall into the Wazuh Manager, which applies the custom rules in `wazuh-rules/local_rules.xml` and writes matches to `alerts.json`. The dashboard (`anaos_gui.py`) reads that file in real time and renders it as a triage console.

![Data Pipeline](docs/images/pipeline.png)

![ANAOS Dashboard](docs/images/dashboard.png)

## ATT&CK Coverage

| Technique | Tactic |
|---|---|
| T1595.002 — Active Scanning | Reconnaissance |
| T1190 — Exploit Public-Facing App (SQLi) | Initial Access |
| T1110 — Brute Force | Credential Access |
| T1218.010 — Regsvr32 (Squiblydoo) | Defense Evasion |

## Limitations

Tested in a controlled lab against only four techniques, using signature/threshold-based detection with single-analyst triage. See the research chapter for a full discussion of scope and validity.
