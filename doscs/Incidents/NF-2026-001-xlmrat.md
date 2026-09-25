
# Incident Report: AsyncRAT Infection & Process Injection

| Metadata | Details |
| :--- | :--- |
| **Incident ID** | NF-2026-002 |
| **Date** | 2026-09-24 |
| **Platform / Lab** | CyberDefenders |
| **Severity** | High / Critical |
| **Status** | Closed (Investigated) |
| **Tools Used** | Wireshark, CyberChef, VirusTotal, VS Code |

---

## Executive Summary
An infection chain was identified originating from a malicious web request to `madmrx.duckdns.com`. A VBScript loader (`xlm.txt`) was delivered to the endpoint, which spawned a hidden PowerShell session to fetch a second-stage payload (`mdm.jpg`). The payload contained an obfuscated `.NET` executable that was reflectively loaded and injected into a legitimate Windows process (`RegSvcs.exe`) using Process Hollowing, establishing persistence via dropped batch and VBS scripts.

---

## The 5 Ws Analysis

* **Who:** Threat actor controlling infrastructure at IP `45.126.209.4` (`madmrx.duckdns.com`). Target: Internal Windows Host at IP `10.1.9.101`.
* **What:** Malware downloader delivering **AsyncRAT** via VBScript loader and Process Hollowing injection into `RegSvcs.exe`.
* **When:** Infection initiated during simulated PCAP capture window (Jan 9, 2024 at 17:27:27).
* **Where:** Initial download from port `222/HTTP`, executing locally under `%TEMP%` and injecting into `C:\Windows\Microsoft.NET\Framework\v4.0.30319\RegSvcs.exe`.
* **Why:** To gain unauthorized Remote Access (RAT), evade detection via LOLBins, and maintain persistent access.

---

## Incident Timeline

```timeline
2024-01-09 17:27:27 UTC : First web request sent to download the 'xlm.txt' file.
2024-01-09 17:27:28 UTC : The 'xlm.txt' file runs PowerShell quietly in the background.
2024-01-09 17:27:29 UTC : PowerShell downloads 'mdm.jpg' from the same remote server.
2024-01-09 17:27:30 UTC : The code converts hexadecimal text into an executable binary and injects it into RegSvcs.exe.
2024-01-09 17:27:32 UTC : 'Conted.ps1', 'Conted.bat', and 'Conted.vbs' files are created to guarantee persistence.
2024-01-09 17:27:34 UTC : A Scheduled Task named 'Update Edge' is created to run the malware every 2 minutes.
```

---

## Technical Analysis

### 1. Delivery & Stager (`xlm.txt`)

The initial vector involved downloading a VBScript file containing obfuscated array elements. When executed, the script concatenates strings to construct an encoded PowerShell command:

```vbscript
' Extracted VBScript snippet
Set objShell = CreateObject("WScript.Shell")
objShell.Run "Cmd.exe /c POWeRSHeLL.eXe -NOP -WIND HIDDeN -eXeC BYPASS -NONI " & OodjR, 0, True

```

### 2. First Stage Payload (`mdm.jpg`)

The PowerShell command downloaded `mdm.jpg`. Although the file extension suggests an image, it actually contains raw hexadecimal code. The script reads this data, converts it back into a binary executable directly in memory, and runs it.

### 3. Evasion & Injection (LOLBin)

To avoid antivirus detection, the malware does not run under its own file name. Instead, it launches a legitimate Windows application (`RegSvcs.exe`) and replaces its internal code with the malicious payload.

### 4. Persistence Mechanisms

The payload called `WriteAllText` to write three persistence scripts to the filesystem:

* `Conted.vbs`
* `Conted.ps1`
* `Conted.bat`

---

## Indicators of Compromise (IoCs)

| Type | Indicator / Value | Description |
| --- | --- | --- |
| **Domain** | `madmrx.duckdns.com` | Domain used by the attacker |
| **IP / Port** | `45.126.209.4:222` | Host server storing the malicious files |
| **URL** | `http://45.126.209.4:222/xlm.txt` | Download link for the initial stager script |
| **URL** | `http://45.126.209.4:222/mdm.jpg` | Download link for the main malware payload |
| **SHA-256 Hash** | `1eb7b02e18f67420f42b1d94e74f3b6289d92672a0fb1786c30c03d68e81d798` | Unique file hash for the AsyncRAT executable |
| **Target Process** | `C:\Windows\Microsoft.NET\Framework\v4.0.30319\RegSvcs.exe` | Legitimate system process used as cover |
| **Dropped Files** | `C:\Users\Public\Conted.ps1`, `Conted.bat`, `Conted.vbs` | Persistence files dropped onto disk |
| **Scheduled Task** | `Update Edge` | Windows task created to re-trigger the virus |

---

---

## Verdict & Remediation

* **Verdict:** True Positive — Confirmed **AsyncRAT** Infection.
* **Remediation Steps:**
1. **Isolate Endpoint:** Disconnect host from the network immediately.
2. **Process Termination:** Kill the spawned `RegSvcs.exe` process instance.
3. **Network Blocking:** Block IP `45.126.209.4` and domain `madmrx.duckdns.com` at the perimeter firewall/DNS sinkhole.
4. **Credential Reset:** Force password resets for all user accounts logged into the compromised host.
