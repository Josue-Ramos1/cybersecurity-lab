# Incident Report: Phishing Mail & Malicious Attachment Delivery

| Metadata | Details |
| :--- | :--- |
| **Incident ID** | Event-82 (SOC140) |
| **Alert Date (Simulated)** | 2021-03-21 |
| **Investigation Date** | 2026-10-06 |
| **Platform / Lab** | LetsDefend |
| **Severity** | High |
| **Status** | Closed (True Positive) |
| **Tools Used** | Mailbox Analyzer, Log Management,  Sandbox, VirusTotal |

---

## Executive Summary
A phishing alert (**SOC140**) was triggered following the delivery of a suspicious email spoofing a COVID-19 vaccine topic. The email originated from sender `aaronluo@cmail.carleton.ca` via SMTP server `189.162.189.159`, targeting internal recipient `mark@letsdefend.io` (`172.16.20.3`). The email contained a malicious link pointing to an external AWS infrastructure (`dunamis-ethos508-prod-va6-856defacfb833db1.elb.us-east-1.amazonaws.com`), which dropped an executable file disguised as a PDF named `Material.pdf`. Dynamic execution in a sandbox environment and threat intelligence checks confirmed the payload as malicious. Containment protocols were executed: the email was purged and the endpoint was isolated.

---

## The 5 Ws Analysis

* **Who:** Threat Actor using external SMTP IP `189.162.189.159` and sender address `aaronluo@cmail.carleton.ca`. Target: Internal user Mark (`mark@letsdefend.io` / Host `172.16.20.3`).
* **What:** Phishing attempt delivering a malicious payload (`Material.pdf`) hosted on AWS ELB infrastructure, designed to establish persistence via Task Scheduler.
* **When:** Email sent on `2021-03-21 03:26:57 UTC`. Alert triggered at `2021-03-21 12:26:57 UTC`.
* **Where:** Delivery via SMTP to Endpoint `172.16.20.3`. Download origin: `dunamis-ethos508-prod-va6-856defacfb833db1.elb.us-east-1.amazonaws.com`.
* **Why:** To trick internal users into executing a malicious payload under the guise of COVID-19 vaccine documentation and achieve persistence via Scheduled Tasks.

---

## Incident Timeline

```timeline
2021-03-21 03:26:57 UTC : Phishing email with subject 'COVID19 Vaccine' sent from 'aaronluo@cmail.carleton.ca'.
2021-03-21 12:26:57 UTC : SIEM triggers Alert Event 82 (SOC140 - Phishing Mail Detected).
2021-03-21 12:30:00 UTC : Analyst inspects user mailbox and confirms presence of suspicious email and external download URL.
2021-03-21 12:35:00 UTC : Log Management logs queried to confirm network interaction and endpoint connection attempt.
2021-03-21 12:40:00 UTC : Download URL analyzed in Sandbox environment; malicious binary disguised as 'Material.pdf' extracted.
2021-03-21 12:45:00 UTC : MD5 hash '72c812cf21909a48eb9cceb9e04b865d' submitted to VirusTotal, confirming malware classification.
2021-03-21 12:50:00 UTC : Remediation triggered: Phishing email purged, endpoint '172.16.20.3' isolated, case closed as True Positive.
```

---

## Technical Analysis

### 1. Delivery & Phishing Vector
The attack utilized social engineering surrounding COVID-19 vaccines to entice the recipient to interact with the message. The email body included a direct link to an AWS ELB domain hosting the payload rather than a standard attachment.

### 2. Payload Extraction & Sandbox Analysis
The download URL was safely submitted to an isolated sandbox environment. The downloaded file presented itself as `Material.pdf`, but static inspection revealed it to be a executable payload.

* **File Name:** `Material.pdf`
* **File Creation Timestamp:** `2006-04-10 15:22:21 UTC`
* **MD5 Hash:** `72c812cf21909a48eb9cceb9e04b865d`

### 3. Intelligence Verification (VirusTotal)
Submitting the extracted file hash `72c812cf21909a48eb9cceb9e04b865d` to VirusTotal confirmed multiple detections for malicious behavior associated with unauthorized persistence creation via Windows Task Scheduler.

---

## Indicators of Compromise (IoCs)

| Type | Indicator / Value | Description |
| --- | --- | --- |
| **Sender Email** | `aaronluo@cmail.carleton.ca` | Source address used in spear-phishing email |
| **SMTP Server IP**| `189.162.189.159` | External relay IP sending the message |
| **Target Host IP** | `172.16.20.3` | Internal destination host address (`mark@letsdefend.io`) |
| **Malicious Domain**| `dunamis-ethos508-prod-va6-856defacfb833db1.elb.us-east-1.amazonaws.com` | AWS endpoint hosting the payload |
| **Payload Name** | `Material.pdf` | Malicious executable disguised as a PDF document |
| **File Hash (MD5)**| `72c812cf21909a48eb9cceb9e04b865d` | Unique file hash of the extracted payload |

---

## Verdict & Remediation

* **Verdict:** True Positive — Confirmed Phishing attack delivering malicious payload.
* **Remediation Steps:**
1. **Email Purge:** Permanently remove the phishing message from the target user's inbox.
2. **Host Isolation:** Isolate host `172.16.20.3` from the network to prevent potential lateral movement or C2 communication.
3. **Network Blocking:** Block IP `189.162.189.159` and domain `dunamis-ethos508-prod-va6-856defacfb833db1.elb.us-east-1.amazonaws.com` at the secure email gateway and firewall level.
