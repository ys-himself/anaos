# ANAOS — Automated Network & Analysis Operations System

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![Python](https://img.shields.io/badge/python-3.x-blue)
![Wazuh](https://img.shields.io/badge/SIEM-Wazuh%204.9-00bbff)
![Ansible](https://img.shields.io/badge/IaC-Ansible-EE0000)
![Status](https://img.shields.io/badge/status-research%20prototype-orange)

**ANAOS** (Automated Network & Analysis Operations System) is a fully automated, open-source Security Operations Centre built on **Wazuh**, **Suricata/pfSense**, **Sysmon/Auditd**, and **Ansible**. It addresses three well-documented barriers to SOC adoption in small and medium organisations: deployment complexity, detection coverage opacity, and analyst triage friction.

This repository accompanies a cybersecurity research chapter produced at ENSA Khouribga (2025-2026). The system was deployed end-to-end in a four-zone virtual network and evaluated against four MITRE ATT&CK-mapped attack scenarios, achieving 100% detection recall, 0% false-positive rate, and full provisioning in under 15 minutes via Ansible.

Full write-up: [`docs/paper/ANAOS_Research_Chapter.pdf`](docs/paper/ANAOS_Research_Chapter.pdf)

---

## Table of Contents

1. [Motivation](#motivation)
2. [Results Summary](#results-summary)
3. [System Architecture](#system-architecture)
4. [Data Pipeline](#data-pipeline)
5. [Detection Engineering](#detection-engineering)
6. [ANAOS Dashboard](#anaos-dashboard)
7. [ATT&CK Coverage Matrix](#attck-coverage-matrix)
8. [Repository Layout](#repository-layout)
9. [Getting Started](#getting-started)
10. [Experimental Environment](#experimental-environment)
11. [Metrics — Definitions](#metrics--definitions)
12. [Limitations](#limitations)
13. [Future Work](#future-work)
14. [Authors](#authors)
15. [Citation](#citation)
16. [License](#license)

---

## Motivation

Commercial SIEM licensing, professional tuning services, and round-the-clock analyst coverage put a fully staffed SOC out of reach for most organisations. Open-source platforms such as Wazuh close the cost gap, but adoption is still impeded by two structural problems:

- **Deployment reproducibility** — installing Wazuh, configuring agents across Windows and Linux hosts, enabling Suricata on pfSense, and distributing rule files involves dozens of manual steps, any one of which can silently break telemetry ingestion.
- **Detection coverage opacity** — Wazuh ships with thousands of default rules covering a wide but shallow surface, and operators rarely know which ATT&CK techniques those rules actually cover, at what confidence level, or which techniques are blind spots.

ANAOS addresses the first problem through Ansible-driven infrastructure-as-code, and the second through an explicit ATT&CK-aligned rule set validated against live attack simulation.

## Results Summary

| Metric | Measured | Target | Result |
|---|---|---|---|
| True Positives | 4 / 4 | >= 3 | Pass |
| False Positives | 0 | 0 | Pass |
| Detection Rate (Recall) | 100.0% | >= 80% | Pass |
| False Positive Rate | 0.0% | <= 5% | Pass |
| MTTD (network-layer attacks) | ~0 s | <= 60 s | Pass |
| ATT&CK Coverage Score | 100% (4/4) | >= 75% | Pass |
| Deployment Time | ~12 min | <= 15 min | Pass |

Both guiding research questions were answered affirmatively:

- **RQ1 (Automation):** Ansible deployment completed in approximately 12 minutes across five endpoints, against a 15-minute target.
- **RQ2 (Detection Quality):** False-positive rate of 0.0% and recall of 100%, against targets of <= 5% FPR and >= 80% recall.

## System Architecture

ANAOS spans four virtualised network zones, with all inter-zone traffic routed through a pfSense firewall running Suricata as an inline IDS:

| Zone | Contents | Role |
|---|---|---|
| WAN | Kali Linux attacker host | Adversary simulation (Nmap, Hydra, SQLmap, Squiblydoo) |
| DMZ | OWASP Juice Shop (Ubuntu, Auditd, Wazuh agent) | Intentionally vulnerable web application |
| LAN1 | Windows 10 endpoint (Sysmon v14), Ubuntu endpoint (Auditd) | Monitored internal endpoints |
| LAN2 | Wazuh Manager v4.9, Ansible control node, ANAOS dashboard | SOC server infrastructure |

![Network Topology](docs/images/topology.png)

## Data Pipeline

Telemetry reaches the Wazuh Manager through two paths: Wazuh agents on endpoints (Sysmon and Auditd events, forwarded over encrypted TCP/1514) and agentless syslog ingestion from pfSense (Suricata EVE-JSON alerts, forwarded over UDP/514). The manager correlates incoming events against `local_rules.xml`, generates ATT&CK-tagged alerts, and writes them to `alerts.json`. The ANAOS dashboard (`anaos_gui.py`) tails that file in real time, parses it, computes SOC metrics, and renders the analyst-facing visualisations.

![Data Pipeline](docs/images/pipeline.png)

## Detection Engineering

Rule development followed a Threat-Informed Defense workflow: scope relevant ATT&CK techniques, map each to a reliable telemetry source, author Wazuh rules that chain on existing parent rule IDs rather than re-parsing raw logs, validate with a synthetic payload, then run the full live attack scenario.

| Technique | Data Source | Detection Signal |
|---|---|---|
| T1595.002 | Suricata IDS (EVE-JSON) | Nmap user-agent / NSE HTTP header |
| T1190 | Suricata IDS (EVE-JSON) | SQL metacharacter pattern in POST body |
| T1110 | Suricata IDS (EVE-JSON) | HTTP POST threshold (>= 10 requests / 5 s) |
| T1218.010 | Sysmon Event ID 1 | `parentImage = regsvr32.exe` |

Each custom Wazuh rule deliberately chains on an existing Suricata or Sysmon parent rule ID rather than matching raw events directly. This two-layer design — for example, Sysmon -> Wazuh's default process-creation rule -> the ANAOS custom rule — restricts pattern evaluation to events already classified by an upstream rule, which was the single largest contributor to achieving a 0% false-positive rate in testing. The complete rule set is in [`wazuh-rules/local_rules.xml`](wazuh-rules/local_rules.xml).

## ANAOS Dashboard

`anaos_gui.py` is a dependency-free Python HTTP server — no external web framework — implementing four subsystems:

1. **Alert ingestion engine** — tails `alerts.json`, parses JSON lines, filters to a rule-ID whitelist, and enriches each event with Suricata metadata and normalised ATT&CK labels.
2. **t0 tracker** — records the earliest observed timestamp per source IP across *all* events (not just whitelisted ones), enabling MTTD computation even when the initial probe precedes the first rule match.
3. **Triage persistence** — analyst TP/FP verdicts are written to `soc_database.json` and survive server restarts.
4. **Single-page frontend** — vanilla JavaScript polling the REST API every 10 seconds, rendering the KPI row, active-agent indicators, and a sortable/filterable triage table, with MTTD/FPR computed client-side.

![ANAOS Dashboard](docs/images/Dashboard.png)

## ATT&CK Coverage Matrix

| ID | Technique | Tactic | Data Source | Detected |
|---|---|---|---|---|
| T1595.002 | Active Scanning | Reconnaissance | Suricata / Web | Yes |
| T1190 | Exploit Public-Facing Application | Initial Access | Suricata / HTTP | Yes |
| T1110 | Brute Force | Credential Access | Suricata / HTTP | Yes |
| T1218.010 | Signed Binary Proxy Execution: Regsvr32 | Defense Evasion | Sysmon EID 1 | Yes |

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
├── LICENSE
└── README.md
```

## Getting Started

### Prerequisites

- A Wazuh Manager (v4.9 or later) already installed on the SOC server (LAN2)
- An Ansible control node with network access to target endpoints (WinRM for Windows, SSH for Linux)
- pfSense with Suricata configured to forward EVE-JSON alerts via syslog to the Wazuh Manager's remote listener

### 1. Deploy the rule set

```bash
sudo cp wazuh-rules/local_rules.xml /var/ossec/etc/rules/local_rules.xml
sudo systemctl restart wazuh-manager
```

### 2. Provision endpoints with Ansible

```bash
cd ansible
cp inventory.example.ini inventory.ini   # fill in real hosts and credentials; gitignored by default
ansible-playbook -i inventory.ini playbooks/deploy_wazuh_agent_linux.yml
ansible-playbook -i inventory.ini playbooks/deploy_windows_endpoint.yml
ansible-playbook -i inventory.ini playbooks/configure_auditd.yml
```

Each playbook is idempotent — re-running it against an already-configured host is safe and will not duplicate configuration.

### 3. Run the dashboard

On the Wazuh Manager host, where `alerts.json` resides:

```bash
python3 anaos_gui.py
```

Before running, edit the `CONFIG` dictionary at the top of `anaos_gui.py`:

| Key | Purpose |
|---|---|
| `log_file` | Path to Wazuh's `alerts.json` (default: `/var/ossec/logs/alerts/alerts.json`) |
| `dashboard_ip` | IP address the dashboard will be reachable at |
| `port` | Listening port (default: 8080) |
| `target_rule_ids` | Whitelist of rule IDs surfaced in the triage console |

Then open `http://<dashboard_ip>:8080/` in a browser.

## Experimental Environment

All virtual machines for the reference deployment ran on VMware Workstation Pro 17, one VMnet switch per zone, with inter-zone routing enforced exclusively through pfSense.

| Host | Zone | OS | vCPU | RAM | Role |
|---|---|---|---|---|---|
| pfSense Gateway | — | pfSense 2.7 | 2 | 1 GB | Firewall + Suricata IDS |
| Kali Linux | WAN | Kali 2024.2 | 2 | 4 GB | Attacker (Nmap, Hydra, SQLmap) |
| Juice Shop | DMZ | Ubuntu 22.04 | 2 | 2 GB | Vulnerable web app + Wazuh agent + Auditd |
| Windows Endpoint | LAN1 | Windows 10 22H2 | 2 | 4 GB | Sysmon v14 + Wazuh agent |
| Ubuntu Endpoint | LAN1 | Ubuntu 22.04 | 2 | 2 GB | Auditd + Wazuh agent |
| Wazuh Server | LAN2 | Ubuntu 22.04 | 4 | 8 GB | SIEM + `anaos_gui.py` |
| Ansible Server | LAN2 | Ubuntu 22.04 | 2 | 2 GB | IaC deployment controller |

## Metrics — Definitions

- **Detection Rate (Recall)** = TP / (TP + FN), where an attack scenario counts as a true positive if it produced at least one analyst-confirmed alert.
- **False Positive Rate** = FP / (TP + FP) — the fraction of *triggered* alerts that are noise, which is the operationally relevant quantity for a triage-oriented SOC (as opposed to the standard statistical FP / (FP + TN) definition).
- **Mean Time to Detect (MTTD)** = average, over all true positives, of (timestamp of first correlated alert − timestamp of the first observed event from the same source IP). The latter timestamp is tracked across the *entire* unfiltered event stream by the dashboard's t0 tracker, not just whitelisted rule matches.
- **ATT&CK Coverage Score** = |D| / |A| x 100%, where A is the set of techniques present in the simulated attack chain and D is the subset for which a correlated alert was generated.

## Limitations

- **Controlled environment** — all experiments ran in an isolated virtual network with no background traffic. Real environments introduce legitimate noise that would likely raise the false-positive rate, particularly for the threshold-based brute-force rule, which could fire on legitimate monitoring tools or CI/CD pipelines making rapid authentication requests.
- **Limited attack breadth** — four scenarios cover a narrow slice of the ATT&CK Enterprise matrix. Lateral movement, persistence, command-and-control, and exfiltration are not represented.
- **Signature-only detection** — ANAOS relies exclusively on signature and threshold-based rules. Novel or obfuscated variants of the tested techniques (e.g. a Regsvr32 payload proxied through a legitimate CDN, or time-based blind SQL injection with minimal metacharacter exposure) may evade current rules entirely.
- **Single-analyst triage** — TP/FP verdicts were assigned by one analyst; no inter-rater reliability scoring (e.g. Cohen's kappa) was performed.
- **MTTD proxy validity** — using the first event from a source IP as the attack-onset proxy can underestimate true onset when reconnaissance originates from a different IP, or when the initial foothold was established through a non-network vector such as a phishing email delivered hours before the observed exploitation phase.

A full discussion is provided in the research chapter (`docs/paper/ANAOS_Research_Chapter.pdf`, Section 9).

## Future Work

- **Expanded coverage** — extend the custom rule set to at least 20 ATT&CK techniques, prioritising post-exploitation phases (T1055, T1003, T1059, T1083) using Sysmon Event IDs 8 and 10 plus Auditd correlation rules.
- **SOAR integration** — implement an automated response module that triggers pfSense firewall block rules on confirmed true-positive triage decisions, closing the detect-to-contain loop for high-confidence alerts without analyst intervention.
- **Behavioural anomaly layer** — integrate Wazuh's built-in anomaly detection or an external ML-based outlier model to catch technique variants that bypass signature-based rules, and measure the resulting change in recall and false-positive rate.

## Authors

Ismail Bajjou, Ousmane Issa Adam, Othmane Nechchadi, Yassine Sarih, Akram Zerbane

Supervised by Dr. Yassine Maleh — ENSA Khouribga, 2025-2026

## Citation

If referencing this work, please cite the accompanying research chapter:

```
Bajjou, I., Issa Adam, O., Nechchadi, O., Sarih, Y., Zerbane, A. (2026).
ANAOS — Automated SOC Deployment & Detection Coverage.
ENSA Khouribga, Cybersecurity Research, 2025-2026.
```

## License
