# OpenVPN Client Configuration & Profile Generation

## 1. Overview & Directory Architecture

Once the OpenVPN server configuration and Certificate Authority (CA) are deployed, client configuration files and certificates are managed in a separate workspace.

To maintain strict operational hygiene and prevent accidental exposure of server-side private keys, Client configuration templates, certificates, private keys, and generated profiles are organized under `/etc/openvpn/client/` on the VPN server.

### Directory Layout

```bash
root@nginxwaf:/etc/openvpn/client# ls
client.conf  files  keys  make_config.sh

# Organized keys by target user or service
root@nginxwaf:/etc/openvpn/client# ls keys/
ca.crt  dh.pem  josue  ta.key  webserver

# User-specific certificates and generated profiles
root@nginxwaf:/etc/openvpn/client# ls keys/josue/
josuelap.crt  josuelap.key  josuephone.crt  josuephone.key  josuephone.ovpn

# Service-specific profiles
root@nginxwaf:/etc/openvpn/client# ls keys/webserver/
webserver.crt  webserver.key  webserver.ovpn

```

### Rationale Behind the Structure

* **Configuration Separation:** Keeps server runtime configuration separate from client profile generation
* **Granular Organization:** Organizes credentials into subdirectories by user (`josue/`) or role (`webserver/`), making certificate tracking and future revocations straightforward.
* **Shared Trust Material:** Maintains shared trust anchors (`ca.crt` and `ta.key`) in `/etc/openvpn/client/keys/` for easy access during profile compilation.

---

## 2. Base Client Configuration (`client.conf`)

The base file `/etc/openvpn/client/client.conf` acts as a generic template defining network options, remote endpoints, and protocol settings.

```ini
# Specify client mode
client

# Network Interface & Protocol Configuration
dev tun
proto udp

# Remote Server Endpoint (pfSense WAN IP / Forwarded Port)
remote 192.168.1.53 1194

# Connection Persistence
persist-key
persist-tun

# SSL/TLS Directives (Commented - Inlined via automation script)
;ca ca.crt
;cert client.crt
;key client.key
;tls-crypt ta.key

```

### Why Key Directives Are Commented Out

Standard OpenVPN clients usually load certificates from separate external files (`ca.crt`, `client.crt`, `client.key`, `ta.key`).

In this setup, those line items are commented out (using `;`) in the base template because all cryptographic material is appended directly inside the final `.ovpn` file using XML-style tags (`<ca>`, `<cert>`, `<key>`, `<tls-crypt>`). This approach converts the configuration into a single, unified profile that is easy to import on mobile devices or remote workstations.

---

## 3. Automated Profile Compilation (`make_config.sh`)

To automate the bundling process, the script `make_config.sh` merges the base configuration template with the required certificates into a single `.ovpn` output file.

### Script Source Code

```bash
#!/bin/bash

# Arguments:
# $1: Directory name under keys/ (e.g., josue, webserver)
# $2: Certificate and key identifier (e.g., josuelap, josuephone)

KEY_DIR=/etc/openvpn/client/keys/
OUTPUT_DIR=/etc/openvpn/client/keys/${1}
BASE_CONFIG=/etc/openvpn/client/client.conf

cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/${1}/${2}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/${1}/${2}.key \
    <(echo -e '</key>\n<tls-crypt>') \
    ${KEY_DIR}/ta.key \
    <(echo -e '</tls-crypt>') \
    > ${OUTPUT_DIR}/${2}.ovpn

```

### How the Profile Creation Works

1. **Input Parameters:**
* `$1`: Target user or service folder inside `keys/` (e.g., `josue`).
* `$2`: Specific certificate/key file prefix (e.g., `josuelap`).


2. **Execution Process:** The script reads `client.conf` and appends inline blocks for `<ca>`, `<cert>`, `<key>`, and `<tls-crypt>`, extracting content directly from the respective files.
3. **Output:** Writes the unified configuration to `/etc/openvpn/client/keys/$1/$2.ovpn`.

### Usage Example

```bash
# Generate unified profile for the operator laptop
./make_config.sh josue josuelap

```

---
