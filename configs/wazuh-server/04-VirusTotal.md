# Malware Detection: VirusTotal API Integration

## 1. Overview & Architecture

To enhance threat detection across endpoints, Wazuh was integrated with the VirusTotal API. This setup automatically checks the reputation of new or modified files in real time.

### How it Works:

1. **Dependency on FIM:** This integration relies on the **File Integrity Monitoring (FIM)** module. When FIM detects a file creation or modification event on an agent, it generates a `syscheck` alert containing the file's cryptographic hash (MD5, SHA1, SHA256).
2. **VirusTotal Lookup:** The Wazuh Manager intercepts the `syscheck` alert and sends the file hash to the VirusTotal API.
3. **Alerting:** If VirusTotal flags the hash as malicious, Wazuh triggers a custom high-severity alert and dispatches an immediate email notification to the security analyst.

---

## 2. Wazuh Manager Integration Setup

### Step 1: VirusTotal API Key Configuration

After obtaining an API key from VirusTotal, the integration block was added to the Wazuh Manager's global configuration file (`/var/ossec/etc/ossec.conf`):

```xml
<integration>
  <name>virustotal</name>
  <api_key>YOUR_VIRUSTOTAL_API_KEY</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>

```

* **`<name>virustotal`:** Enables the built-in VirusTotal integration script.
* **`<api_key>`:** Authenticates requests sent to the VirusTotal API.
* **`<group>syscheck`:** Directs Wazuh to trigger VirusTotal checks only when FIM (`syscheck`) events occur.
* **`<alert_format>json`:** Formats alert data passed to the integration script as JSON.

---

### Step 2: Custom Alert Rule (`/var/ossec/etc/rules/fim_rules.xml`)

When VirusTotal responds with a detection, Wazuh processes the result using parent rule `87105`. To escalate malicious detections and send instant email notifications, custom rule `100030` was created:

```xml
<group name="virustotal,">
  <rule id="100030" level="16">
    <if_sid>87105</if_sid>
    <options>alert_by_email</options>
    <description>MALICIOUS FILE DETECTED: $(virustotal.source.file) - VirusTotal: $(virustotal.positives)/$(virustotal.total) detections.</description>
  </rule>
</group>

```

#### Rule Parameters:

* **`id="100030"` & `level="16"`:** Assigns a custom rule ID and escalates severity to **Level 16** (Critical Threat).
* **`<if_sid>87105</if_sid>`:** Intercepts default VirusTotal positive detection events.
* **`<options>alert_by_email</options>`:** Forces an immediate email alert without delay.
* **Dynamic Variables:** Uses `$(virustotal.source.file)`, `$(virustotal.positives)`, and `$(virustotal.total)` to dynamically display the file path and engine detection ratio in the alert title.

*Note: After applying these configurations, the Wazuh Manager service was restarted to apply the changes:*

```bash
sudo systemctl restart wazuh-manager

```

---

## 3. Malware Simulation & Detection Verification

To test the integration end-to-end, the standard **EICAR anti-malware test file** was downloaded onto the monitored endpoint (`carrucel`) inside the web server directory (`/home/josh/devhub/`):

```bash
josh@carrucel:~/devhub$ sudo curl https://secure.eicar.org/eicar.com -o /home/josh/devhub/eicar
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    68  100    68    0     0     59      0  00:01  00:01 --:--:--    61

```

---

## 4. Real-Time Alert & Email Verification

Within seconds of file creation, the following workflow executed automatically:

1. FIM detected the new file `/home/josh/devhub/eicar` and generated a hash.
2. Wazuh sent the hash to VirusTotal.
3. VirusTotal identified the hash as malicious (**53/55 engine detections**).
4. Rule `100030` fired and dispatched an email notification.

### Triggered Email Alert Notification

```text
Wazuh Notification.
2026 Sep 10 21:21:26

Received From: (carrucel) 10.10.1.7->virustotal
Rule: 100030 fired (level 16) -> " MALICIOUS FILE DETECTED: /home/josh/devhub/eicar - VirusTotal: 53/55 detections."
Portion of the log(s):

{
  "virustotal": {
    "found": 1,
    "malicious": 1,
    "source": {
      "alert_id": "1789075280.1288377",
      "file": "/home/josh/devhub/eicar",
      "md5": "44d88612fea8a8f36de82e1278abb02f",
      "sha1": "3395856ce81f2b7382dee72602f798b642f14140"
    },
    "scan_date": "2026-09-10 21:14:58",
    "positives": 53,
    "total": 55,
    "permalink": "https://www.virustotal.com/gui/file/275a021bbfb6489e54d471899f7db9d1663fc695ec2fe2a2c4538aabf651fd0f/detection"
  },
  "integration": "virustotal"
}

virustotal.found: 1
virustotal.malicious: 1
virustotal.source.file: /home/josh/devhub/eicar
virustotal.source.md5: 44d88612fea8a8f36de82e1278abb02f
virustotal.source.sha1: 3395856ce81f2b7382dee72602f798b642f14140
virustotal.positives: 53
virustotal.total: 55
virustotal.permalink: https://www.virustotal.com/gui/file/275a021bbfb6489e54d471899f7db9d1663fc695ec2fe2a2c4538aabf651fd0f/detection

 --END OF NOTIFICATION

```

---