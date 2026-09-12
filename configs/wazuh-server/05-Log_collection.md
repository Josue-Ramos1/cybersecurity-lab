# Log Collection & Centralization: pfSense & Custom Decoders

## 1. Overview & Scope

By default, the Wazuh agent automatically collects basic system logs (such as **`/var/log/syslog`** or **`Windows Event Logs`**). However, to achieve broader monitoring visibility, we also need to ingest logs from network security devices that cannot run a Wazuh agent.

While web server logs (like *Nginx*) are easily collected directly on the agent side with simple file monitoring paths, network firewalls require a dedicated remote ingestion strategy.

This document focuses on the log centralization setup for the *pfSense Firewall,* which sends its events remotely using syslog over *UDP* *514* directly to the Wazuh Manager.

To process pfSense logs cleanly, we configured the Wazuh Manager to accept remote syslog traffic, enabled event archiving, configured Filebeat to ingest archived events, created custom XML decoders for filterlog entries, and configured an index pattern in the Wazuh Dashboard.

---

## 2. Wazuh Manager Configuration (`/var/ossec/etc/ossec.conf`)

To receive syslog events from pfSense and archive all incoming logs, we updated `/var/ossec/etc/ossec.conf` on the Wazuh Manager (`10.10.1.10`).

### Step 1: Enable Log Archiving & Remote Syslog Listener

```xml
<ossec_config>
  <!-- Global log archiving options -->
  <jsonout_output>yes</jsonout_output>
  <alerts_log>yes</alerts_log>
  <logall>yes</logall>
  <logall_json>yes</logall_json>

  <!-- Remote Syslog Listener -->
  <remote>
    <connection>syslog</connection>
    <port>514</port>
    <protocol>udp</protocol>
    <allowed-ips>10.10.1.1/24</allowed-ips>
  </remote>
</ossec_config>

```

#### Configuration Breakdown:

* **`<logall>yes` & `<logall_json>yes`:** Instructs Wazuh to store **all** incoming events into `/var/ossec/logs/archives/archives.json`, even if they do not trigger a specific alert rule. This is essential for troubleshooting and custom decoder development.
* **`<remote>` block:** Listens for remote UDP syslog traffic on port `514` from the firewall subnet (`10.10.1.1/24`).

*After saving the configuration, restart the Wazuh Manager service:*

```bash
sudo systemctl restart wazuh-manager

```

---

## 3. pfSense Remote Logging Setup

On the pfSense firewall WebGUI (`10.10.1.1`), navigate to **Status > System Logs > Settings** and configure the **Remote Logging Options**:

### Remote Logging Options Settings

| Setting | Value | Description |
| --- | --- | --- |
| **Enable Remote Logging** | `Checked` | Enables sending log messages to a remote syslog server. |
| **Source Address** | `Default (any)` | Binds the logging daemon to the default interface IP. |
| **IP Protocol** | `IPv4` | Uses IPv4 addressing for syslog UDP datagrams. |
| **Remote Log Servers** | `10.10.1.10:514` | Points directly to the Wazuh Manager IP and port. |
| **Remote Syslog Contents** | `Everything` | Forwards Firewall, System, OpenVPN, DHCP, and Auth events. |

---

## 4. Filebeat Archive Configuration (`/etc/filebeat/filebeat.yml`)

By default, Filebeat only indexes alerts (`wazuh-alerts-*`). To visualize all raw ingested logs—including unindexed firewall events—we must enable the **archives** module in Filebeat:

```yaml
# /etc/filebeat/filebeat.yml
filebeat.modules:
  - module: wazuh
    alerts:
      enabled: true
    archives:
      enabled: true

```

### Why Enable Archives?

Enabling `archives: enabled: true` allows Filebeat to forward raw events from `/var/ossec/logs/archives/archives.json` to OpenSearch/Elasticsearch under the `wazuh-archives-*` index pattern. This ensures that pfSense logs are searchable in real time before rule triggers are created.

*Apply the changes by restarting Filebeat:*

```bash
sudo systemctl restart filebeat

```

---

## 5. Custom pfSense Decoder Configuration

Standard pfSense `filterlog` outputs are comma-separated values (CSV) that default Wazuh decoders do not parse cleanly. To extract key network fields (interface, action, protocol, IP addresses, and ports), we created a custom decoder file at `/var/ossec/etc/decoders/pfsense_decoders.xml`:

```xml
<!-- Base Header Decoder -->
<decoder name="pfsense-custom-header">
    <prematch>^filterlog[\d+]: </prematch>
</decoder>

<!-- IPv4 Traffic Decoder (ICMP / General) -->
<decoder name="pfsense-custom-ipv4">
    <parent>pfsense-custom-header</parent>
    <regex type="pcre2" offset="after_parent">^.*?,(\w+),(\w+),(\w+),(\w+),4,.*?,(\w+),\d+,(\d+\.\d+\.\d+\.\d+),(\d+\.\d+\.\d+\.\d+)</regex>
    <order>interface, reason, action, direction, protocol, srcip, dstip</order>
</decoder>

<!-- IPv4 Port Decoder (TCP / UDP) -->
<decoder name="pfsense-custom-ports">
    <parent>pfsense-custom-header</parent>
    <regex type="pcre2" offset="after_parent">^.*?,(\w+),(\w+),(\w+),(\w+),4,.*?,(tcp|udp),\d+,(\d+\.\d+\.\d+\.\d+),(\d+\.\d+\.\d+\.\d+),(\d+),(\d+)</regex>
    <order>interface, reason, action, direction, protocol, srcip, dstip, srcport, dstport</order>
</decoder>

```

### Decoder Working Principle:

1. **`pfsense-custom-header`:** Matches log entries starting with `filterlog[<PID>]:`.
2. **`pfsense-custom-ipv4`:** Uses PCRE2 regex to extract `interface`, `reason`, `action` (e.g., `block` or `pass`), `direction` (`in`/`out`), `protocol`, `srcip`, and `dstip`.
3. **`pfsense-custom-ports`:** Extends parsing for TCP/UDP traffic to capture `srcport` and `dstport`.

*Restart the Wazuh Manager to load the new decoders:*

```bash
sudo systemctl restart wazuh-manager

```

---

## 6. Dashboard Index Pattern Creation & Verification

To view the parsed pfSense logs in the Wazuh Dashboard:

1. Go to **Wazuh Menu > Dashboard Management > Index Patterns**.
2. Click **Create index pattern**.
3. Type `wazuh-archives-4.x-*` (e.g., `wazuh-archives-4.x-2026.09.12`) and select `@timestamp` as the primary time field.
4. Open **Discover**, switch to the `wazuh-archives-*` index pattern, and filter by `location: 10.10.1.1`.

### ICMP Block Verification Capture

An ICMP ping request from host `192.168.1.14` targeting `192.168.1.53` was dropped by pfSense. The log was successfully ingested, parsed by our custom decoder, and indexed in the dashboard:

#### Extracted Event Fields:

* **`@timestamp`:** `Sep 11, 2026 @ 20:44:52.099`
* **`location`:** `10.10.1.1` (pfSense Firewall IP)
* **`decoder.name`:** `pfsense-custom-header`
* **`data.interface`:** `em0`
* **`data.action`:** `block`
* **`data.direction`:** `in`
* **`data.protocol`:** `icmp`
* **`data.srcip`:** `192.168.1.14`
* **`data.dstip`:** `192.168.1.53`
* **`full_log`:** `Sep 11 21:44:52 filterlog[8179]: 4,,,1000000103,em0,match,block,in,4,0x0,,128,13950,0,none,1,icmp,60,192.168.1.14,192.168.1.53,request,1,2240`