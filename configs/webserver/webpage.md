# Web Server & Vulnerable Application Deployment (Miloscompany)

## 1. Overview & Service Scope

The Web & Database Server operates on host `10.10.1.7` within the DMZ subnet (`10.10.1.0/24`). It hosts **Miloscompany**, a deliberately vulnerable web application built with Node.js, Express, and SQLite for security testing and monitoring purposes.

The server is responsible for running the application and its local database. External traffic handling is delegated to the security and proxy components of the architecture:

* **Cloudflare Tunnel:** Provides public access to the application without requiring inbound HTTP/HTTPS port forwarding on the WAN interface.
* **NGINX Reverse Proxy & OpenAppSec WAF (`10.10.1.5`):** Receives requests from the Cloudflare Tunnel, inspects traffic through the WAF, and forwards permitted requests to `10.10.1.7:3000`.

The application listens on TCP port `3000` and is not directly exposed to the Internet.

---

## 2. Application Structure & Deployment

The application is deployed under `/opt/devhub/` and consists of a Node.js backend, a SQLite database, and a static frontend.

```text
devhub/
├── package.json
├── src/
│   └── server.js
├── db/
│   ├── init.js
│   └── devhub.db
└── public/
    └── index.html
```

### Installation

```bash
# Install Node.js using NVM
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc

nvm install 20
nvm use 20

# Install SQLite CLI
sudo apt update
sudo apt install -y sqlite3

# Install application dependencies
cd /opt/devhub
npm install

# Initialize the database
node db/init.js

# Start the application
npm start
```

The application can be accessed locally through:

```text
http://localhost:3000
```

---

## 3. Database & Test Accounts

Miloscompany uses SQLite as its local database. The database file is created during application initialization:

```text
/opt/devhub/db/devhub.db
```

The application includes intentionally insecure test accounts for controlled security testing.

| Username     | Password    | Role       |
| ------------ | ----------- | ---------- |
| `admin`      | `admin123`  | admin      |
| `maria`      | `maria2024` | user       |
| `carlos`     | `carlos123` | user       |
| `superadmin` | `super999`  | superadmin |

These credentials are intentionally weak and are used exclusively for security testing within the laboratory environment.

---

## 4. Vulnerability Scope

Miloscompany is intentionally deployed as an insecure application to provide a controlled target for penetration testing and security monitoring.

The application includes the following vulnerability categories:

* **SQL Injection (SQLi):** Vulnerable login and search functionality.
* **Cross-Site Scripting (XSS):** Reflected and stored XSS functionality.
* **Broken Access Control / IDOR:** API endpoints that allow access to other users' information without proper authorization checks.
* **Mass Assignment:** User-controlled fields can modify privileged attributes such as the application role.
* **Debug SQL Endpoint:** A deliberately exposed endpoint capable of executing SQL queries against the application database.

These vulnerabilities are intentionally included for controlled testing and are not representative of a production deployment.

---

## 5. Security Monitoring Scope

The application provides a controlled target for validating defensive monitoring capabilities within the laboratory.

The server can generate web application activity that can be used for:

* HTTP request and application activity monitoring.
* Detection of suspicious SQL injection patterns.
* Detection of potentially malicious web requests.
* File Integrity Monitoring of application files and the SQLite database.
* Correlation of application activity with other security events collected by Wazuh.

Specific detection rules and monitoring configurations are documented separately under the Wazuh configuration documentation.

---

## 6. Validation

The deployment was validated by confirming that:

1. The miloscompany application starts successfully on port `3000`.
2. The SQLite database is created and populated correctly.
3. The application is reachable through the internal NGINX reverse proxy.
4. Public access is provided through the Cloudflare Tunnel.
5. Direct Internet access to the application server is not required.
6. The intentionally vulnerable functionality can be used as a controlled target for subsequent penetration testing and security monitoring exercises.