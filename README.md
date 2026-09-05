# Security Lab: SOC Operations & Offensive Testing

## Overview

This repository documents the architecture, configuration, and practical execution of an integrated security lab. The environment is designed to bridge offensive security (Red Team) and defensive monitoring (Blue Team/SOC), demonstrating the full lifecycle of threat simulation, detection engineering, incident analysis, and automated response.

---

## Lab Architecture & Topology

The laboratory models a standard enterprise infrastructure exposed to simulated external traffic via a secure tunnel, monitored by a centralized Security Information and Event Management (SIEM) platform.

* **Detailed Topology:** [docs/lab-architecture.md](docs/lab-architecture.md)

### Core Components

* **SIEM / Central Monitoring:** Wazuh Manager.
* **Perimeter & WAF:** pfSense Firewall, OpenAppSec, OpenVPN, Nginx Reverse Proxy, and Cloudflare Tunnels (external routing).
* **Protected Endpoints & Servers:**
* Windows (172.168.1.10).
* Web & Database Server (10.10.1.7) running Node.js and web services.


* **Offensive Testing Host:** Kali Linux (Nmap, Nessus, Burp Suite, Hydra).

---

## Repository Structure

```text
cybersecurity-lab/
├── README.md
├── docs/
│   ├── README.md
│   ├── lab-architecture.md
│   └── incidents/
│       └── IR-2026-001-ssh-bruteforce.md
├── configs/
│   ├── README.md
│   ├── wazuh-server/
│   ├── nginx-vpn-server/    
│   ├── webserver/
│   ├── pfsense/
│   └── LAN/
├── pentesting/
│   ├── README.md
│   └── notes/
│       ├── 01-reconnaissance.md
│       ├── 02-web-vulnerabilities.md
│       └── 03-recommendations.md
└── screenshots/
    ├── architecture/
    ├── wazuh-server/
    ├── nginx-vpn-server/
    ├── pfsense/
    ├── webserver/
    └── incidents/

```

---

## Implemented Use Cases & Scenarios

### 1. Offensive Security Testing (Red Team)

Comprehensive documentation of offensive procedures, research, investigation, and exploitation methodologies executed against the lab environment:

* **Reconnaissance:** Port scanning, service enumeration, and network mapping using Nmap.
* **Web Application Assessment:** Vulnerability testing and exploitation (SQL Injection, Path Traversal) against local Node.js applications using Burp Suite and PortSwigger methodologies.
* **Research & Findings:** Documented investigation notes detailing attack vectors and security recommendations.
* *Detailed notes and documentation:* pentesting/README.md

### 2. Defensive Monitoring & Response (Blue Team / SOC)

Implementation of comprehensive detection engineering, monitoring, and automated hardening strategies:

* **Log Data Collection & Analysis:** Real-time collection, parsing, and correlation of system, firewall, and application logs through Wazuh.
* **File Integrity Monitoring (FIM):** Tracking unauthorized modifications to critical files and web application directories.
* **Malware Detection:** Integration with the VirusTotal API for automated file hash reputation checking on suspicious binaries.
* **Security Configuration Assessment (SCA):** Automated compliance and hardening checks on endpoint operating systems.
* **Command Monitoring:** Tracking the execution of critical or sensitive administrative commands across endpoints.
* **Custom Detection Rules:** Development of custom Wazuh XML rules and decoders tailored to detect specific threat signatures against internal services.

---