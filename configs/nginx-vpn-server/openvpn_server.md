# OpenVPN Server Configuration & CCD Management

## 1. Overview & Setup Prerequisites

OpenVPN runs on host `10.10.1.5` to provide secure, encrypted remote access into the lab infrastructure.

For package installation, standard OpenVPN documentation should be followed. As a security best practice, all generated PKI components (CA certificates, server keys, Diffie-Hellman parameters, and TLS auth keys) are stored in a dedicated directory (`/etc/openvpn/server/`) rather than the default path to maintain clear file organization and access permissions.

System IP forwarding is enabled on the host to allow VPN client traffic routing:

```bash
# Enable IPv4 forwarding across reboots
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/50-enable-ipv4-forwarding.conf

# Apply configuration without restarting
sudo sysctl -p /etc/sysctl.d/50-enable-ipv4-forwarding.conf

```

---

## 2. Server Configuration (`/etc/openvpn/server/server.conf`)

The OpenVPN server configuration uses a standard TUN routing topology, UDP transmission for low overhead, and `tls-crypt` for additional channel encryption and protection against port scanning.

```ini
# Network Interface & Protocol Settings
port 1194
proto udp
dev tun

# Cryptographic Keys & Certificates
ca /etc/openvpn/server/ca.crt
cert /etc/openvpn/server/miloscompany.crt
key /etc/openvpn/server/miloscompany.key 
dh /etc/openvpn/server/dh.pem
tls-crypt /etc/openvpn/server/ta.key

# Network & Addressing Topology
topology subnet
server 10.8.0.0 255.255.255.0

# Client Configuration Directory
client-config-dir /etc/openvpn/ccd

# Routing Policies
client-to-client

# Persistence Options
persist-key
persist-tun

```

### Key Directives Rationale

* **`port 1194` / `proto udp`:** Runs on standard OpenVPN UDP port `1194`, forwarded through the pfSense WAN interface for external connectivity.
* **`topology subnet`:** Configures the VPN pool (`10.8.0.0/24`) as a single IP subnet where the server interface acts as `10.8.0.1`.
* **`tls-crypt`:** Encrypts and authenticates OpenVPN control-channel traffic, making it more difficult for unauthorized parties to inspect or identify OpenVPN handshake traffic.
* **`client-config-dir /etc/openvpn/ccd`:** Enables per-client custom configurations, allowing specific static IP assignments based on client certificate Common Names (CN).
* **`client-to-client`:** Allows authenticated VPN clients within the `10.8.0.0/24` tunnel network to communicate directly with each other.

---

## 3. Static IP Assignment via Client Configuration Directory (CCD)

Static IP addresses are assigned to specific VPN clients using the CCD mechanism. These predictable addresses can then be referenced by firewall rules to apply client-specific access controls.

Inside `/etc/openvpn/ccd/`, configuration files are named to match the exact **Common Name (CN)** of the client’s SSL/TLS certificate.

### Configured Clients

```bash
# List configured client profile overrides
root@nginxwaf:/etc/openvpn/ccd# ls
josuelap  josuephone  webserver

```

### Operator Laptop Configuration Override (`/etc/openvpn/ccd/josuelap`)

```ini
ifconfig-push 10.8.0.10 255.255.255.0

```

* **`josuelap`:** Assigns the fixed IP address `10.8.0.10` to the operator's primary workstation upon successful authentication. This predictable address allows pfSense NAT/WAN rules to permit destination access to the Wazuh Manager dashboard (`10.10.1.10:443`).

---

## 4. Service Management

Start and enable the OpenVPN systemd unit tied to the server configuration:

```bash
# Start OpenVPN server using the specific configuration instance
sudo systemctl start openvpn-server@server

# Enable automatic start on boot
sudo systemctl enable openvpn-server@server

```