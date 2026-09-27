# Incident Report: JetBrains 

| Metadata | Details |
| --- | --- |
| **Incident ID** | NF-2026-002 |
| **Date** | 2026-09-26 |
| **Platform / Lab** | CyberDefenders |
| **Severity** | Critical |
| **Status** | Closed (Investigated) |
| **Tools Used** | Wireshark |

---

## Executive Summary

A critical incident was identified involving an unauthenticated remote code execution (RCE) attack against an internal **JetBrains TeamCity** server (version `2023.11.3`). The attacker leveraged an Authentication Bypass vulnerability (**CVE-2024-27198**) to forcefully create an unauthorized administrator account (`c91oyemw`). Using these elevated credentials, the threat actor generated a persistent API token, uploaded a malicious ZIP archive containing a Java-based webshell (`NSt8bHTg.zip`), and established command execution capabilities. The attacker subsequently tampered with local credential files (`Creds.txt`), conducted host discovery, and successfully escaped the web application container to achieve full host-level root access by mounting the host root filesystem via Docker.

---

## The 5 Ws Analysis

* **Who:** Threat Actor operating from external IP address `23.158.56.196`. Target: Internal TeamCity Server running at `3.71.79.4:8111`.
* **What:** Exploitation of CVE-2024-27198 (TeamCity Authentication Bypass) leading to webshell deployment (`NSt8bHTg.zip`), local credential manipulation, and container breakout via Docker socket/privileges.
* **When:** Attack initiated on June 30, 2024. Initial webshell command execution recorded at `2024-06-30 08:03 UTC`.
* **Where:** Web Application layer targeting TeamCity REST API endpoints (`/app/rest/users`), expanding to container execution and chroot host filesystem access (`/host`).
* **Why:** To achieve initial access, establish persistence via webshell deployment, modify local administrative credentials, and obtain full host-level control.

---

## Incident Timeline

```timeline
2024-06-30 08:02:49 UTC : Attacker sends crafted HTTP POST request exploiting CVE-2024-27198 to create admin user 'c91oyemw'.
2024-06-30 08:02:50 UTC : Attacker requests and receives a Bearer authentication token ('mD5r0yemB0') via REST API path traversal.
2024-06-30 08:02:51 UTC : Authentication confirmed via Bearer token test request to '/authenticationTest.html'.
2024-06-30 08:02:55 UTC : Malicious archive 'NSt8bHTg.zip' containing a JSP webshell is uploaded to the application server.
2024-06-30 08:03:00 UTC : Attacker executes first command via the webshell to verify web server execution privileges.
2024-06-30 08:03:15 UTC : Attacker tampers with local 'Creds.txt' file, replacing genuine admin credentials with 'a1l4m:youarecompromised'.
2024-06-30 08:03:30 UTC : Attacker executes Docker breakout command, mounting host root directory and executing chroot to gain full host access.

```

---

## Technical Analysis

### 1. Initial Access & Auth Bypass (CVE-2024-27198)

The adversary exploited a URL parsing flaw in JetBrains TeamCity versions prior to `2023.11.4`. By appending `;.jsp` path matrix parameters to arbitrary HTTP requests targeting the REST API, the attacker bypassed authentication checks and created an administrative user:

```http
POST /hax?jsp=/app/rest/users;.jsp HTTP/1.1
Host: 3.71.79.4:8111
Content-Type: application/json

{"username": "c91oyemw", "password": "CL5vzdwLuK", "email": "c91oyemw@example.com", "roles": {"role": [{"roleId": "SYSTEM_ADMIN", "scope": "g"}]}}

```

### 2. Persistence & Webshell Deployment

Following successful user creation, the attacker generated an access token and uploaded a compressed webshell payload named `NSt8bHTg.zip`. Once uncompressed by the application, the webshell provided interactive HTTP-based remote command execution (`cmd=`).

### 3. Data Manipulation & Defense Evasion

The attacker located `Creds.txt`, a local file containing administrator credentials for the web application, and altered its contents. The original credentials were overwritten with false values (`a1l4m:youarecompromised`) to disrupt administrative recovery efforts and manipulate system data.

### 4. Container Escape & Host Takeover

Upon discovering access to the Docker daemon socket from within the web application container, the attacker issued a command mounting the underlying host's root directory (`/`) into a temporary container and invoked `chroot`:

```bash
docker run --rm -it -v /:/host ubuntu chroot /host

```

This operation completely bypassed container boundaries, granting the attacker interactive root access directly over the host operating system.

---

## Indicators of Compromise (IoCs)

| Type | Indicator / Value | Description |
| --- | --- | --- |
| **Attacker IP** | `23.158.56.196` | Source IP address utilized during the attack |
| **Target URL / Host** | `3.71.79.4:8111` | Compromised JetBrains TeamCity Server instance |
| **CVE Reference** | `CVE-2024-27198` | JetBrains TeamCity Authentication Bypass Vulnerability |
| **Created Credentials** | `c91oyemw:CL5vzdwLuK` | Rogue Administrator account created via API exploitation |
| **Uploaded Payload** | `NSt8bHTg.zip` | Archive file containing the JSP Webshell deployment |
| **Tampered File Data** | `a1l4m:youarecompromised` | Unauthorized data written into local `Creds.txt` file |
| **MITRE Sub-technique** | `T1565.001` | Data Manipulation: Stored Data Manipulation |
| **Breakout Command** | `docker run --rm -it -v /:/host ubuntu chroot /host` | Command used to achieve host filesystem takeover |

---

## Verdict & Remediation

* **Verdict:** True Positive — Critical system compromise resulting in full host takeover.
* **Remediation Steps:**

1. **Immediate Isolation:** Disconnect host `3.71.79.4` from the internal network segment.
2. **Software Patching:** Upgrade JetBrains TeamCity to version `2023.11.4` or later immediately to patch CVE-2024-27198.
3. **Account Auditing & Revocation:** Invalidate all active TeamCity tokens, delete rogue user account `c91oyemw`, and rotate all internal application credentials.
4. **Perimeter Defense:** Block external IP `23.158.56.196` at the perimeter firewall/WAF.