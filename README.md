# 🛡️ Enterprise Web Application Security & ModSecurity WAF Defense
### Technical Internship Deliverable: E-Commerce Hardening, Vulnerability Assessment, and Perimeter WAF Engineering

<p align="center">
  <img src="https://img.shields.io/badge/Target-PrestaShop_1.7.8.9-blue.svg?style=for-the-badge&logo=prestashop" />
  <img src="https://img.shields.io/badge/Gateway-Apache_2.4_ModSecurity-red.svg?style=for-the-badge&logo=apache" />
  <img src="https://img.shields.io/badge/Ruleset-OWASP_CRS_v3.3.8_(823_Rules)-orange.svg?style=for-the-badge&logo=owasp" />
  <img src="https://img.shields.io/badge/Platform-Docker_&_Ubuntu_22.04-239120.svg?style=for-the-badge&logo=docker" />
  <img src="https://img.shields.io/badge/Status-Completed_&_Audited-success.svg?style=for-the-badge" />
</p>

---

## 📌 Project Overview & Metadata

* **Author / Lead Analyst:** Qazeem Samsudeen Temitope ([@qazeemsamshudeen-web](https://github.com/qazeemsamshudeen-web) | *BlaqqSec*)
* **Specialization:** Security Operations Center (SOC), Application Security & Threat Defense
* **Target Application:** PrestaShop E-Commerce v1.7.8.9 (Containerized) & DVWA (vulnerable baseline)
* **Target Host IP:** `10.0.0.5` (Ubuntu Server 22.04 LTS)
* **Attacker Host IP:** `10.0.0.3` (Kali Linux 2026.2 VM)
* **Network Topology:** VirtualBox Isolated Host-Only Subnet (`10.0.0.0/24`)

In modern enterprise architectures, e-commerce applications are prime targets for automated scanners and threat actors seeking credentials, customer databases, and payment tokens. Relying solely on internal application code for input sanitization violates the core security principle of **Defense-in-Depth**.

This repository contains the complete documentation, forensic telemetry, custom WAF rules, and operational runbook for a dual-host attack simulation and defensive engineering engagement. We demonstrate how an unshielded application processes malicious inputs, deploy an **Apache ModSecurity reverse proxy WAF** backed by 823 active **OWASP Core Rule Set (CRS v3.3.8)** rules and custom virtual patches, and prove that attacks are proactively blocked with `HTTP 403 Forbidden` before reaching application code.

---

## 🏗️ Architectural Topology

```text
+-------------------------------------------------------------------------------+
|                             LAB ENVIRONMENT TOPOLOGY                          |
+-------------------------------------------------------------------------------+

   [ ATTACKER MACHINE ]                               [ TARGET DEFENSE SERVER ]
       Kali Linux VM                                      Ubuntu Server VM
       IP: 10.0.0.3                                         IP: 10.0.0.5
  +--------------------+                             +------------------------+
  |  Security Auditing |                             | Apache 2 Web Server    |
  |  Browser / CLI     |                             | Port 80 (WAF Gateway)  |
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
├── PROJECT_RUNBOOK.pdf             # Publication-grade step-by-step operational runbook (PDF)
├── PROJECT_RUNBOOK.html            # Pre-styled runbook source document
├── LAB_REPORT.md                   # Full comprehensive technical lab report
├── LAB_REPORT.html                 # Formatted rich-text report ready for Google Docs import
├── prestashop_custom_rules.conf    # Custom ModSecurity ruleset (Rules 1000001 - 1000004)
├── push_to_github.bat              # Automation script to stage, commit, and push to remote
├── screenshots/                    # Complete empirical evidence & forensic captures
│   ├── 00_burp_baseline_and_waf_traffic.jpg   # Burp history showing 200 OK vs 403 Forbidden
│   ├── 01_customer_vs_admin_login.png         # Account separation audit
│   ├── 02_prestashop_admin_rename_prompt.png  # PrestaShop mandatory directory rename guard
│   ├── 03_prestashop_admin_dashboard.jpg      # Secure back-office administrative access
│   ├── 04_burp_ssl_handshake_error.png        # SSL handshake failure on origin container
│   ├── 05_annotated_modsec_evidence.png       # Annotated forensic error log analysis
│   ├── 05_prestashop_system_info_disclosure.jpg# System information leakage audit
│   ├── 06_owasp_crs_823_rules.jpg             # Terminal verification of 823 OWASP CRS rules
│   ├── 07_prestashop_xss_search_baseline.jpg  # Baseline search with script injection (200 OK)
│   ├── 08_apache_admin_ip_restricted_403.png  # Negative testing: 403 Forbidden on admin path
│   ├── 09_prestashop_outdated_modules.jpg     # Audit of 32 outdated modules needing patches
│   ├── 10_robots_txt_directory_leak.jpg       # Information disclosure leaking 101 directories
│   ├── 11_config_dir_403_forbidden.png        # Directory listing disabled verification (403)
│   ├── 12_nikto_scan_headers.jpg              # Nikto automated scan highlighting missing headers
│   ├── 13_cookie_security_httponly.jpg        # F12 Storage Inspector: HttpOnly & SameSite flags
│   ├── 14_prestashop_db_backup.jpg            # PrestaShop compressed .sql.gz database backups
│   ├── 15_wafw00f_detection_verification.png  # Automated WAF fingerprinting via wafw00f
│   └── annotated_modsec_evidence.png          # High-contrast color-coded forensic log breakdown
└── README.md                       # Repository documentation and architecture guide
```

---

## 🎯 Core Deliverables Summary

### Deliverable 1: DVWA Exploitation & Defense
* **SQL Injection (SQLi):** Exploited `' OR '1'='1` on Port 8080 (`HTTP 200 OK`, full database user dump). Re-tested through Port 80 WAF proxy $\rightarrow$ dropped with **`HTTP 403 Forbidden`**.
* **Cross-Site Scripting (XSS):** Executed `<script>alert('XSS-DVWA-POC')</script>` on Port 8080 (`HTTP 200 OK`, alert box rendered). Re-tested through Port 80 WAF proxy $\rightarrow$ dropped with **`HTTP 403 Forbidden`**.

### Deliverable 2: Custom ModSecurity Ruleset (`prestashop_custom_rules.conf`)
* **Rule 1000001:** Reflected & Stored XSS Prevention (inspects `ARGS`, `REQUEST_COOKIES`, `REQUEST_HEADERS`).
* **Rule 1000002:** SQL Injection Prevention (`UNION SELECT`, `' OR 1=1`, comment truncation).
* **Rule 1000003:** Directory Traversal & Critical File Protection (`../../`, `/etc/passwd`, `settings.inc.php`, `.env`).
* **Rule 1000004:** Hardened Path Obfuscation & Installer Directory Restriction (`/install`).

### Deliverable 3: Defense Validation & Anomaly Scoring
* **Automated Verification:** `wafw00f http://10.0.0.5/` confirmed site is behind an active WAF (`Normal: 200, XSS: 403`).
* **Anomaly Scoring Model:** ModSecurity operates under collaborative scoring (`REQUEST-949-BLOCKING-EVALUATION.conf`). When cumulative attack scores exceed **5**, the engine issues a disruptive `403 Forbidden` block in Phase 2.

### Deliverable 4: PrestaShop 10-Point Security Checklist Audit
1. **Application & Module Updates:** Audited 32 outdated modules; deployed virtual patching; scheduled PrestaShop 8.x upgrade.
2. **Default/Demo Accounts & Data:** Toggled Demo Mode OFF on dashboard; audited customer accounts list.
3. **Admin Password & MFA:** Enforced 16-character complex passphrase; recommended TOTP 2FA module.
4. **Permissions & Directory Listing:** Confirmed `//config/` returns 403 Forbidden; deleted `/var/www/html/install`.
5. **Session Cookies & Headers:** Storage Inspector confirmed `HttpOnly: true` and `SameSite: Lax` on session cookies.
6. **Disable Unnecessary Modules:** Disabled unneeded onboarding and marketing plugins to achieve Attack Surface Reduction.
7. **API & Payment Endpoints:** Verified Webservice API is set to `NO`; no live payment merchant keys configured.
8. **TLS/SSL Enforcement:** Identified container handshake failure; documented reverse proxy Port 443 TLS termination requirement.
9. **Admin Path Obfuscation & Negative Testing:** Core `/admin` path returns 404; active on secret `/admin47140aj33/`. Apache `<Location>` block tested with `Require ip 10.0.0.99` (**Negative Testing**), successfully blocking test machine `10.0.0.3` with **`HTTP 403 Forbidden`**.
10. **Backup & Recovery Verification:** Successfully generated two compressed `.sql.gz` database backup archives.

### Deliverable 5: PrestaShop Attack Simulation & Forensic Telemetry
* **Baseline (Port 8080):** Injected `<script>alert('XSS-PRESTASHOP')</script>` into catalog search parameter `s`. PrestaShop accepted the input and responded with `HTTP 200 OK` (*"No matches found"*).
* **Protected (Port 80):** Injected identical payload through the WAF. ModSecurity dropped the connection with `HTTP 403 Forbidden`.
* **Forensic Analysis (`/var/log/apache2/error.log`):**
  * **Attacker Attribution:** `[client 10.0.0.3:58494]` directly identifies Kali Linux.
  * **Rule Matching:** Libinjection and CRS Rule `941110` matched `<script>` in `ARGS:s`.
  * **Action:** `Access denied with code 403 (phase 2)`.
  * **Anomaly Score:** Inbound Anomaly Score reached **18** (limit: 5).

---

## 🔒 Production Architecture & Hardening Blueprint

In enterprise production deployments, **Port 8080 is completely invisible to the outside world**:
1. **Loopback Binding:** The container is bound strictly to `127.0.0.1:8080`, preventing outside interfaces from reaching it.
2. **Host Firewall Enforcement:**
   ```bash
   sudo ufw default deny incoming
   sudo ufw default allow outgoing
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   sudo ufw deny 8080
   sudo ufw enable
   ```
3. **Privileged Access:** Authorized engineers access origin containers via encrypted SSH tunnels:
   ```bash
   ssh -L 8080:127.0.0.1:8080 blaqqsec03@10.0.0.5
   ```

---

## 🚀 How to Clone and Push to GitHub

To push this entire project to your personal GitHub account:
1. Create a new empty repository on [GitHub](https://github.com/new) named:
   ```text
   Web-Application-Security-PrestaShop-WAF-Defense
   ```
2. Double-click **`push_to_github.bat`** inside this project directory (or run it from the command line). It will automatically stage all files, commit the changes, link your remote URL, and push to branch `main`.

---

<p align="center">
  <b>&copy; 2026 Qazeem Samsudeen Temitope (BlaqqSec). Certified Cyber Security Internship Portfolio.</b>
</p>
