# ANAOS — Automated Network & Analysis Operations System

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![Python](https://img.shields.io/badge/python-3.x-blue)
![Wazuh](https://img.shields.io/badge/SIEM-Wazuh%204.9-00bbff)

ANAOS is a fully automated, open-source Security Operations Centre (SOC) built on Wazuh, Suricata/pfSense, Sysmon/Auditd, and Ansible. It was designed and evaluated as part of a cybersecurity research project at ENSA Khouribga (2025-2026).

The project demonstrates that a reproducible, infrastructure-as-code SOC can be deployed in under 15 minutes while achieving strong detection performance — 100% recall and 0% false-positive rate — across four MITRE ATT&CK-mapped attack scenarios.

Full write-up: `docs/paper/ANAOS_Research_Chapter.pdf`

## Results Summary

| Metric | Measured | Target | Result |
|---|---|---|---|
| True Positives | 4 / 4 | >= 3 | Pass |
| False Positives | 0 | 0 | Pass |
| Detection Rate (Recall) | 100.0% | >= 80% | Pass |
| False Positive Rate | 0.0% | <= 5% | Pass |
| MTTD (network layer) | ~0s | <= 60s | Pass |
| ATT&CK Coverage | 100% (4/4) | >= 75% | Pass |
| Deployment Time | ~12 min | <= 15 min | Pass |

## Architecture

ANAOS spans four virtualised network zones — WAN (attacker), DMZ (vulnerable web app), LAN1 (monitored endpoints), and LAN2 (SOC servers) — with all inter-zone traffic inspected by pfSense/Suricata.

![Network Topology](docs/images/topology.png)

Telemetry flows from endpoints and the firewall into the Wazuh Manager, which correlates events against `local_rules.xml` and writes alerts to `alerts.json`. The custom dashboard (`anaos_gui.py`) tails that file in real time.

![Data Pipeline](docs/images/pipeline.png)

## Dashboard

`anaos_gui.py` is a dependency-free Python HTTP server exposing a single-page analyst dashboard: live KPIs (TP/FP count, FPR, MTTD), active-agent indicators, and a sortable/filterable triage console with ATT&CK-tagged alert context.

![ANAOS Dashboard](docs/images/Dashboard.png)

## Detected Techniques

| Technique ID | Name | Tactic | Data Source |
|---|---|---|---|
| T1595.002 | Active Scanning | Reconnaissance | Suricata / Web |
| T1190 | Exploit Public-Facing Application (SQLi) | Initial Access | Suricata / HTTP |
| T1110 | Brute Force | Credential Access | Suricata / HTTP |
| T1218.010 | Regsvr32 (Squiblydoo) | Defense Evasion | Sysmon EID 1 |

## Repository Layout

```
anaos/
├── anaos_gui.py                 Analyst dashboard / alert triage server
├── wazuh-rules/
│   └── local_rules.xml          Custom ATT&CK-mapped Wazuh detection rules
├── ansible/
│   ├── inventory.example.ini    Sample inventory (copy to inventory.ini)
│   └── playbooks/
│       ├── deploy_wazuh_agent_linux.yml
│       ├── deploy_windows_endpoint.yml
│       └── configure_auditd.yml
├── docs/
│   ├── images/                  Architecture and dashboard screenshots
│   └── paper/                   Full research chapter (PDF)
└── README.md
```

## Getting Started

### Prerequisites

- A Wazuh Manager (v4.9 or later) already installed on the SOC server (LAN2)
- An Ansible control node with access to target endpoints (WinRM for Windows, SSH for Linux)
- pfSense with Suricata configured to forward EVE-JSON alerts via syslog to the Wazuh Manager

### 1. Deploy the rule set

Copy `wazuh-rules/local_rules.xml` to `/var/ossec/etc/rules/local_rules.xml` on the Wazuh Manager and restart the manager service:

```bash
sudo cp wazuh-rules/local_rules.xml /var/ossec/etc/rules/local_rules.xml
sudo systemctl restart wazuh-manager
```

### 2. Provision endpoints with Ansible

```bash
cd ansible
cp inventory.example.ini inventory.ini   # fill in real hosts and credentials
ansible-playbook -i inventory.ini playbooks/deploy_wazuh_agent_linux.yml
ansible-playbook -i inventory.ini playbooks/deploy_windows_endpoint.yml
ansible-playbook -i inventory.ini playbooks/configure_auditd.yml
```

### 3. Run the dashboard

On the Wazuh Manager host, where `alerts.json` resides:

```bash
python3 anaos_gui.py
```

Edit the `CONFIG` dictionary at the top of `anaos_gui.py` to set `dashboard_ip`, `port`, and `log_file` for your environment, then open `http://<dashboard_ip>:8080/`.

## Limitations

- Evaluated in a noise-free lab; real-world traffic would likely raise the false-positive rate, particularly for the threshold-based brute-force rule.
- Only 4 of several hundred ATT&CK techniques are covered. Lateral movement, persistence, command-and-control, and exfiltration are not yet represented.
- Detection is signature/threshold-based only; no behavioural anomaly detection.
- Triage verdicts were assigned by a single analyst, with no inter-rater reliability scoring.
- MTTD uses "first packet observed from source IP" as an attack-onset proxy, which can underestimate true onset in multi-vector intrusions.

A full discussion of these limitations is provided in the research chapter (`docs/paper/ANAOS_Research_Chapter.pdf`).

## Future Work

- Expand rule coverage to 20 or more ATT&CK techniques, prioritising post-exploitation phases (T1055, T1003, T1059, T1083)
- SOAR integration to automatically block confirmed true positives via pfSense
- A behavioural or ML-based anomaly detection layer to catch signature-evading variants

## Authors

Ismail Bajjou, Ousmane Issa Adam, Othmane Nechchadi, Yassine Sarih, Akram Zerbane

Supervised by Dr. Yassine Maleh — ENSA Khouribga, 2025-2026

## License

This project is released under the MIT License. See `LICENSE` for details.
