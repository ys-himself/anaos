# ANAOS — Automated Network & Analysis Operations System

![Python](https://img.shields.io/badge/python-3.x-blue)
![Wazuh](https://img.shields.io/badge/SIEM-Wazuh%204.9-00bbff)

ANAOS is a fully automated, open-source Security Operations Centre (SOC) built on Wazuh, Suricata/pfSense, Sysmon/Auditd, and Ansible. Designed and evaluated as part of a cybersecurity research project at ENSA Khouribga (2025-2026).

Across four MITRE ATT&CK-mapped attack scenarios, the system achieved 100% detection recall, 0% false-positive rate, and deployed end-to-end in under 15 minutes via Ansible.

Full write-up: `docs/paper/ANAOS_Research_Chapter.pdf`

## Results

| Metric | Result |
|---|---|
| Detection Rate (Recall) | 100% |
| False Positive Rate | 0% |
| MTTD (network layer) | ~0s |
| ATT&CK Coverage | 4/4 techniques |
| Deployment Time | ~12 min |

## Architecture

![Network Topology](docs/images/topology.png)

Telemetry flows from endpoints and the firewall into the Wazuh Manager, which correlates events against `wazuh-rules/local_rules.xml` and writes alerts to `alerts.json`. The dashboard (`anaos_gui.py`) tails that file in real time.

![Data Pipeline](docs/images/pipeline.png)

## Dashboard

`anaos_gui.py` is a dependency-free Python HTTP server providing live KPIs (TP/FP, FPR, MTTD), active-agent indicators, and a sortable triage console with ATT&CK-tagged alerts.

![ANAOS Dashboard](docs/images/dashboard.png)

## Detected Techniques

| ID | Name | Tactic |
|---|---|---|
| T1595.002 | Active Scanning | Reconnaissance |
| T1190 | Exploit Public-Facing App (SQLi) | Initial Access |
| T1110 | Brute Force | Credential Access |
| T1218.010 | Regsvr32 (Squiblydoo) | Defense Evasion |

## Limitations

Evaluated in a noise-free lab with only 4 ATT&CK techniques covered, signature-based detection only, and single-analyst triage. Full discussion in the research chapter.
