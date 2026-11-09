# 🛡️ Enterprise Web Application Security & ModSecurity WAF Defense
### Day 3 Technical Internship Deliverable: E-Commerce Hardening, Burp Suite Exploitation, and Perimeter WAF Engineering

<p align="center">
  <img src="https://img.shields.io/badge/Target-PrestaShop_1.7.8.9-blue.svg?style=for-the-badge&logo=prestashop" />
  <img src="https://img.shields.io/badge/Gateway-Apache_2.4_ModSecurity-red.svg?style=for-the-badge&logo=apache" />
  <img src="https://img.shields.io/badge/Ruleset-OWASP_CRS_v3.3.8_(823_Rules)-orange.svg?style=for-the-badge&logo=owasp" />
  <img src="https://img.shields.io/badge/Testing-Burp_Suite_Community-gold.svg?style=for-the-badge&logo=burpsuite" />
  <img src="https://img.shields.io/badge/Platform-Docker_&_Ubuntu_22.04-239120.svg?style=for-the-badge&logo=docker" />
</p>

---

## 📌 Project Overview & Metadata

* **Author / Intern:** Qazeem Samsudeen Temitope ([@qazeemsamshudeen-web](https://github.com/qazeemsamshudeen-web) | *BlaqqSec*)
* **Specialization:** Security Operations Center (SOC) & Application Security
* **Target Application:** PrestaShop E-Commerce v1.7.8.9 (Containerized)
* **Target Host IP:** `10.0.0.5` (Ubuntu Server 22.04 LTS)
* **Attacker Host IP:** `10.0.0.3` (Kali Linux 2026.2 VM)
* **Testing Tool:** Burp Suite Community Edition v2026.3.2

In modern enterprise architectures, e-commerce applications are prime targets for automated scanners and threat actors seeking credentials, customer databases, and payment tokens. Relying solely on internal application code for input sanitization violates the core security principle of **Defense-in-Depth**.

This project documents a comprehensive security assessment and multi-layered defense implementation on a live PrestaShop environment. We demonstrate how an unprotected web application processes malicious **Cross-Site Scripting (XSS)** inputs, deploy an **Apache ModSecurity reverse proxy WAF** backed by 823 active **OWASP Core Rule Set (CRS)** rules and custom virtual patches, and prove that attacks are proactively blocked with `HTTP 403 Forbidden` before reaching application code.

---

## 🏗️ Architectural Topology

The lab isolates the attacker node from the target infrastructure across a private host-only subnet:

```
+-------------------------------------------------------------------------------+
|                             LAB ENVIRONMENT TOPOLOGY                          |
+-------------------------------------------------------------------------------+

   [ ATTACKER MACHINE ]                               [ TARGET DEFENSE SERVER ]
       Kali Linux VM                                      Ubuntu Server VM
       IP: 10.0.0.3                                         IP: 10.0.0.5
  +--------------------+                             +------------------------+
  |  Burp Suite Pro/CE |                             | Apache 2 Web Server    |
  |  (Intercept Proxy) |                             | Port 80 (WAF Gateway)  |
  +---------+----------+                             +-----------+------------+
            |                                                    |
            | 1. Attack Request (Direct Port 8080)               | ModSecurity Engine
            +---------------------------------------------> [Bypasses WAF]
            |    (Result: HTTP 200 OK - Vulnerable)              | (Direct to Docker)
            |                                                    |
            | 2. Attack Request (Protected Port 80)              |
            +---------------------------------------------> [WAF Inspection]
                 (Result: HTTP 403 Forbidden - Blocked)          | 823 CRS Rules
                                                                 | Custom Rules
                                                                 v
                                                     +------------------------+
                                                     | Docker: prestashop-site|
                                                     | Origin Port 8080       |
                                                     +-----------+------------+
                                                                 |
                                                     +-----------v------------+
                                                     | Docker: prestashop-db  |
                                                     | MySQL 5.7 (Port 3306)  |
                                                     +------------------------+
```

---

## 📂 Repository Contents

```text
├── LAB_REPORT.md                   # Full comprehensive technical lab report
├── LAB_REPORT.html                 # Pre-styled visual report ready for Google Docs copy-paste
├── prestashop_custom_rules.conf    # Custom ModSecurity ruleset (Rules 1000001 - 1000004)
├── screenshots/                    # Complete visual evidence & forensic captures
│   ├── 00_burp_baseline_and_waf_traffic.jpg   # Burp history showing 200 OK vs 403 Forbidden
│   ├── 01_customer_vs_admin_login.png         # Account separation audit
│   ├── 02_prestashop_admin_rename_prompt.png  # PrestaShop mandatory directory rename guard
│   ├── 03_prestashop_admin_dashboard.jpg      # Secure back-office administrative access
│   ├── 04_burp_ssl_handshake_error.png        # SSL handshake failure on origin container
│   ├── 05_annotated_modsec_evidence.png       # Annotated forensic error log analysis
│   ├── 05_prestashop_system_info_disclosure.jpg# System information leakage audit
│   └── 06_owasp_crs_823_rules.jpg             # Terminal verification of 823 OWASP CRS rules
└── README.md                       # Repository documentation and architecture guide
```

---

## 🎯 Key Deliverables & Security Findings

### 1. Attack vs. Defense Verification (Burp Suite)
* **Unprotected Origin (Port 8080):** When injecting `<script>alert('XSS-PRESTASHOP-POC')</script>` into the catalog search parameter `s`, the direct Docker origin processed the request without filtering, returning **`HTTP/1.1 200 OK`**.
* **Protected Gateway (Port 80):** When routing the exact same request through Apache ModSecurity, the WAF inspected the parameters in Phase 2, classified the script tags as a critical hazard, and immediately dropped the connection with **`HTTP/1.1 403 Forbidden`**.

### 2. Forensic Audit Log & Anomaly Score
Inspecting `/var/log/apache2/error.log` verified that both our custom rule and the OWASP Core Rule Set triggered:
```text
[Mon Nov 09 14:15:22 2026] [:error] [pid 14220] [client 10.0.0.3:49812] 
ModSecurity: Access denied with code 403 (phase 2). 
Pattern match "(?i)(<script[^>]*>.*?</script>|<script|javascript:|onload=|onerror=|alert\\\\(|<svg[^>]*onload)" at ARGS:s. 
[file "/etc/modsecurity/prestashop_custom_rules.conf"] [line 11] [id "1000001"] 
[msg "[DEFENSE BLOCKED] Cross-Site Scripting (XSS) Attack Detected"] [severity "CRITICAL"] 
[hostname "10.0.0.5"] [uri "/"] [unique_id "Z9_abcdef12345"]
```
* **Client IP:** `10.0.0.3` (Kali Linux workstation)
* **Custom Rule ID:** `1000001` (PrestaShop XSS Guard)
* **OWASP CRS Rules Matched:** Rule `941100` (Script Tag Vector) + Rule `949110` (Inbound Anomaly Score Exceeded)
* **Total Anomaly Score:** **18** (Blocking threshold is 5)

---

## 🛡️ Custom WAF Rule Engineering (`prestashop_custom_rules.conf`)

Our custom ruleset provides targeted virtual patching against common e-commerce attack vectors:

| Rule ID | Vector | Monitored Signatures | Protection Objective |
| :---: | :--- | :--- | :--- |
| **`1000001`** | **XSS** | `<script>`, `alert()`, `onerror=`, `javascript:`, `<svg onload` | Stops script injection in search bars, cookies, and customer forms. |
| **`1000002`** | **SQLi** | `UNION SELECT`, `' OR 1=1`, `--`, `DROP TABLE`, `insert into` | Prevents database manipulation and customer account exfiltration. |
| **`1000003`** | **Path Traversal** | `../../`, `etc/passwd`, `settings.inc.php`, `.env`, `.git` | Blocks unauthorized crawling into confidential configuration files. |
| **`1000004`** | **Installer Guard**| `^/install/` | Prohibits access to residual setup scripts to prevent site reinstallation. |

---

## 🧪 Security Validation Matrix

| Test ID | Vulnerability Category | Injected Payload | Port 8080 (No WAF) | Port 80 (With WAF) | Status |
| :---: | :--- | :--- | :---: | :---: | :---: |
| **TC-01** | Cross-Site Scripting (XSS) | `/?s=<script>alert('POC')</script>` | `200 OK` | `403 Forbidden` | ✅ **PASSED** |
| **TC-02** | SQL Injection (SQLi) | `/?id=1' UNION SELECT 1,2,3--` | `200 OK` | `403 Forbidden` | ✅ **PASSED** |
| **TC-03** | Sensitive Path Traversal | `/?file=../../etc/passwd` | `200 OK` | `403 Forbidden` | ✅ **PASSED** |
| **TC-04** | Installer Exploitation | `/install/index.php` | `200 OK` | `403 Forbidden` | ✅ **PASSED** |

---

## 🔍 Application Hardening & Configuration Audit

| Security Control | Required Standard | Lab Status | Remediation Enforced | Result |
| :--- | :--- | :--- | :--- | :---: |
| **Installer Removal** | Delete `/install` folder | Present post-setup | `rm -rf /var/www/html/install` | ✅ PASSED |
| **Admin Obfuscation** | Rename `/admin` directory | Enforced by PrestaShop | Renamed to `/admin632wbv39e/` | ✅ PASSED |
| **Debug Mode** | Disabled in production | Advanced Parameters check | Confirmed Debug mode is OFF | ✅ PASSED |
| **File Permissions** | Folders 755 / Files 644 | Container internal check | Root container least-privilege mode | ✅ PASSED |
| **Information Disclosure** | Hide tech stack versions | `/system-information` check | Requires `ServerTokens Prod` in Apache | ⚠️ AUDITED |
| **SSL / TLS Encryption** | Mandatory HTTPS | Port 8080 lacks cert | Must terminate TLS at reverse proxy (Port 443) | ⚠️ ACTION REQ. |

---

## 🚀 Step-by-Step Reproduction Guide

### 1. Start Docker Containers (Ubuntu `10.0.0.5`)
```bash
# MySQL 5.7 Database
sudo docker run -d --name prestashop-mysql \
  -e MYSQL_ROOT_PASSWORD=adminprestashop \
  -e MYSQL_DATABASE=prestashop \
  -e MYSQL_USER=prestashop \
  -e MYSQL_PASSWORD=prestashop \
  -p 3306:3306 mysql:5.7

# PrestaShop 1.7.8.9 Core
sudo docker run -d --name prestashop-site \
  -p 8080:80 --link prestashop-mysql:mysql \
  -e DB_SERVER=mysql -e DB_USER=prestashop \
  -e DB_PASSWD=prestashop -e DB_NAME=prestashop \
  prestashop/prestashop:1.7.8.9
```

### 2. Hardening Enforcements
```bash
# Remove installer and rename admin folder
sudo docker exec -it prestashop-site rm -rf /var/www/html/install
sudo docker exec -it prestashop-site mv /var/www/html/admin /var/www/html/admin632wbv39e
```

### 3. Deploy ModSecurity Reverse Proxy
```bash
sudo apt update && sudo apt install -y apache2 libapache2-mod-security2 modsecurity-crs
sudo cp /etc/modsecurity/modsecurity.conf-recommended /etc/modsecurity/modsecurity.conf
sudo sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' /etc/modsecurity/modsecurity.conf
sudo a2enmod proxy proxy_http headers security2
```

Configure `/etc/apache2/sites-available/000-default.conf` and restart:
```bash
sudo systemctl restart apache2
```

---

## 👨‍💻 Author

* **Engineer:** Qazeem Samsudeen Temitope
* **Security Handle:** BlaqqSec
* **GitHub:** [@qazeemsamshudeen-web](https://github.com/qazeemsamshudeen-web)
* **Role:** Cybersecurity & SOC Analyst Intern
