<div align="center">

# 🏥 Mediroza General Hospital
### Web Application Penetration Test

**Week 4 Penetration Testing Internship · Batch B082**

*Confidential Security Assessment — Authorized Educational Engagement*

[![Status](https://img.shields.io/badge/Assessment-Complete-brightgreen?style=for-the-badge)](#-milestone-results)
[![Risk](https://img.shields.io/badge/Overall%20Risk-Critical-red?style=for-the-badge)](#-overall-risk-assessment)
[![Type](https://img.shields.io/badge/Type-Black--Box-blue?style=for-the-badge)](#-scope--methodology)
[![Scope](https://img.shields.io/badge/Scope-Authorized-success?style=for-the-badge)](#-assessment-disclaimer)

</div>

---

## 📋 Table of Contents

- [Executive Summary](#-executive-summary)
- [Scope & Methodology](#-scope--methodology)
- [Tools Used](#-tools-used)
- [Attack Surface Discovery](#-attack-surface-discovery)
- [Findings](#-findings)
- [Milestone Results](#-milestone-results)
- [Risk Rating Summary](#-risk-rating-summary)
- [Overall Risk Assessment](#-overall-risk-assessment)
- [Recommendations & Remediation](#-recommendations--remediation)
- [Evidence Handling & Privacy](#-evidence-handling--privacy)
- [Lessons Learned](#-lessons-learned)
- [Final Deliverables](#-final-deliverables)
- [Disclaimer](#-assessment-disclaimer)

---

## 🎯 Executive Summary

A controlled **black-box penetration test** was conducted against Mediroza General Hospital's web infrastructure as part of a Week 4 penetration testing internship assignment.

| | |
|---|---|
| 🌐 **Target** | `https://medirozahospital.com` |
| 🏢 **Client** | Mediroza General Hospital |
| 🧪 **Assessment Type** | Black-box Web Application Penetration Test |
| ⏱️ **Duration** | 3 days |
| ✅ **Authorization** | Explicitly authorized, educational scope only |
| 🚫 **Excluded** | Social engineering, denial-of-service |

### Objectives

1. Gain access to the restricted patient portal
2. Retrieve three confidential patient laboratory reports
3. Crack the encryption protecting all three reports
4. Identify exposed employee salary information
5. Identify exposed shareholder information
6. Document vulnerabilities, evidence, impact, and remediation

### Key Findings

| ID | Finding | Severity |
|---|---|:---:|
| MED-01 | SQL Injection → Patient Portal Authentication Bypass | 🔴 **Critical** |
| MED-02 | Publicly Accessible Database Backup (HR & Shareholder Data) | 🔴 **Critical** |
| MED-03 | Weak PDF Passwords / Recoverable Encryption | 🟠 **High** |
| MED-04 | SQL Error Disclosure / Unsafe SQL Construction | 🟡 **Medium** |
| MED-05 | Username Enumeration via Auth Responses | 🟢 **Low** |
| MED-06 | Directory Listing / Information Disclosure | 🟢 **Low** |

> **Bottom line:** A username-field SQL injection bypassed patient login entirely, exposing three encrypted lab reports. All three were cracked offline with Hashcat. Separately, a forgotten backup at `/old/` leaked internal staff salary and shareholder records to the open internet — no authentication required.

---

## 🔍 Scope & Methodology

### Scope

**In scope:** public web app · authentication mechanisms · patient portal · staff portal · publicly accessible files/directories · input handling · accessible backups · retrieved PDFs · offline document analysis

**Out of scope:** social engineering · denial-of-service · systems outside the target domain · unrelated third-party infrastructure

### Testing Phases

```
Phase 1  Reconnaissance            → headers, sitemap, robots.txt, directory discovery
Phase 2  Authentication Analysis   → input validation, SQLi indicators, enumeration
Phase 3  Restricted Area Access    → SQL injection auth bypass → patient portal
Phase 4  Data Extraction           → 3 encrypted lab report PDFs retrieved
Phase 5  Offline Password Recovery → pdf2john + hashcat (mode 10500) → all 3 cracked
Phase 6  Further Exposure Analysis → /old/ backup discovery → HR & shareholder leak
```

---

## 🛠 Tools Used

| Tool | Purpose |
|---|---|
| `curl` | HTTP requests, header & endpoint analysis |
| Browser | Authentication testing, visual verification |
| `ffuf` | Directory / file discovery |
| `sqlmap` | SQL injection analysis |
| `pdf2john` | PDF password-hash extraction |
| `hashcat` | Offline password recovery |
| `pdftotext` | Verification & extraction of decrypted PDF contents |
| Linux CLI utilities | Evidence collection & file analysis |

---

## 🕵️ Attack Surface Discovery

**HTTP fingerprinting** revealed the server stack and CMS version (`LiteSpeed`, `PHP/8.2.33`, `Mediroza CMS 1.4.2`), providing reconnaissance value to an attacker.

**`robots.txt`** disallowed — and thereby advertised — three sensitive paths:

```
Disallow: /patient/
Disallow: /staff/
Disallow: /old/
```

`/old/` turned out to contain a fully exposed database backup — a reminder that `robots.txt` is a *suggestion to crawlers*, not an access control.

---

## 🐛 Findings

<details open>
<summary><strong>MED-01 · SQL Injection Authentication Bypass</strong> — 🔴 Critical</summary>

**Component:** `/patient/login.php`

The login form processed user input in a way that allowed SQL syntax manipulation. A crafted username using SQL comment syntax (`admin'--`) caused the application to bypass the password check entirely and redirect into the restricted patient portal at `/patient/portal.php`, exposing three lab report entries.

**Likely vulnerable construction:**
```sql
SELECT * FROM patients
WHERE username = '$username' AND password = '$password';
```

**Remediation:** parameterized queries, secure password hashing (Argon2id/bcrypt), generic auth errors, rate limiting/lockout, suspicious-activity logging.

```php
$stmt = $db->prepare("SELECT id, password_hash FROM patients WHERE username = ?");
$stmt->bind_param("s", $username);
$stmt->execute();
```
</details>

<details open>
<summary><strong>MED-02 · Publicly Accessible Database Backup</strong> — 🔴 Critical</summary>

**Path:** `/old/mediroza_db_backup_2019.sql` — downloadable with no authentication.

Contained a `staff` table (name, title, department, email, phone, **national ID, monthly salary**, date joined) and a `shareholders` table (name, share %, shares held, share class).

**Remediation:** remove immediately, disable directory indexing, rotate any exposed credentials, store backups outside the web root and encrypted at rest, restrict access, add monitoring for exposed backup files.
</details>

<details open>
<summary><strong>MED-03 · Weak PDF Password Protection</strong> — 🟠 High</summary>

All three retrieved lab reports (`patient_report_1/2/3.pdf`) were password-protected but crackable offline:

```
pdf2john patient_report_N.pdf > reportN.hash
hashcat -m 10500 reportN.hash rockyou.txt
```

All three passwords were recovered and independently verified via `pdftotext -upw`. Passwords, hashes, and file contents are **not published** in this repo (see [Evidence Handling](#-evidence-handling--privacy)).

**Remediation:** strong random secrets, application-level authorization instead of static PDF passwords, modern encryption with proper key management.
</details>

<details>
<summary><strong>MED-04 · SQL Error Disclosure</strong> — 🟡 Medium</summary>

Malformed login input reflected raw MySQL/MariaDB error text, confirming unsafe SQL construction and assisting further exploitation.

**Remediation:** generic errors to users, verbose errors logged server-side only, disable debug/error display in production.
</details>

<details>
<summary><strong>MED-05 · Username Enumeration</strong> — 🟢 Low</summary>

Distinct "username not found" vs. "incorrect password" responses allowed valid-account discovery.

**Remediation:** a single generic message — *"Invalid username or password."*
</details>

<details>
<summary><strong>MED-06 · Directory Listing</strong> — 🟢 Low</summary>

Indexing enabled on `/patient/`, `/staff/`, `/patient/reports/`, `/old/` — the latter directly exposing the leaked database backup.

**Remediation:** `Options -Indexes` (Apache) / equivalent LiteSpeed config; enforce authorization on sensitive directories.
</details>

---

## ✅ Milestone Results

| Milestone | Description | Status |
|---|---|:---:|
| **M1** | Initial access — auth bypass & 3 lab reports retrieved | ✅ Complete |
| **M2** | Data extraction — encryption cracked on all 3 PDFs, decryption verified | ✅ Complete |
| **M3** | Critical data exposure — staff salary & shareholder data identified | ✅ Complete |
| **M4** | Professional penetration-testing report delivered | ✅ Complete |

```
M1 ████████████████████ 100% ✅
M2 ████████████████████ 100% ✅
M3 ████████████████████ 100% ✅
M4 ████████████████████ 100% ✅
```

---

## 📊 Risk Rating Summary

| Finding | Severity | Primary Impact |
|---|:---:|---|
| SQL Injection Auth Bypass | 🔴 Critical | Unauthorized access to restricted patient data |
| Public Database Backup | 🔴 Critical | Exposure of confidential HR/shareholder information |
| Weak PDF Password Protection | 🟠 High | Offline recovery of protected medical reports |
| SQL Error Disclosure | 🟡 Medium | Reveals database/query information |
| Username Enumeration | 🟢 Low | Enables account discovery |
| Directory Listing | 🟢 Low | Reveals application/server structure |

---

## ⚠️ Overall Risk Assessment

<div align="center">

### **Overall Rating: CRITICAL**

</div>

Multiple vulnerabilities chain together into two serious compromise paths:

**Path A — Patient Data**
```
Internet → Public Web App → Patient Login → SQL Injection
   → Authentication Bypass → Restricted Patient Portal
   → Confidential Lab Reports → Offline PDF Password Recovery
   → Medical Information Disclosure
```

**Path B — Internal Records**
```
Internet → /old/ → Public Directory Listing
   → mediroza_db_backup_2019.sql → Internal Database Records
   → Employee Info · Salaries · National IDs · Shareholder Data
```

---

## 🔧 Recommendations & Remediation

### Priority 1 — Immediate
- Fix SQL injection with parameterized queries throughout
- Remove `/old/mediroza_db_backup_2019.sql` from the public web root
- Disable directory indexing site-wide
- Investigate whether exposed data was accessed by unauthorized parties

### Priority 2 — High
- Strengthen patient authentication (secure hashing, MFA, rate limiting, lockout, CSRF protection)
- Replace static PDF passwords with strong secrets + application-level authorization
- Remove verbose database error disclosure from production

### Priority 3 — Medium / Low
- Generic authentication error messages (prevent enumeration)
- Reduce technology/version disclosure (`X-Powered-By`, `Server`, CMS version)
- Implement security monitoring (failed logins, SQLi patterns, backup access)
- Store backups outside the web root, encrypted, access-restricted, periodically audited

---

## 🔐 Evidence Handling & Privacy

Because this engagement involved real healthcare and HR data, evidence has been **sanitized for public release**. The following are intentionally excluded from this repository and retained only in a private submission:

```
patient_report_1.pdf / 2.pdf / 3.pdf
report1.txt / 2.txt / 3.txt
report1.hash / 2.hash / 3.hash
week4_db_backup.sql
Raw screenshots containing patient names, medical results, patient IDs,
passwords, session cookies, tokens, national IDs, or personal contact info
```

Recommended repo structure:

```
mediroza-week4-pentest/
├── README.md
├── evidence/
│   ├── m1-patient-portal-access.png
│   ├── m1-report-list.png
│   ├── m2-hashcat-report[1-3].png
│   ├── m2-report[1-3]-decryption.png
│   ├── m3-public-old-directory.png
│   ├── m3-database-structure.png
│   ├── m3-staff-exposure.png
│   └── m3-shareholder-exposure.png
└── methodology/
    └── testing-notes.md
```

---

## 💡 Lessons Learned

| # | Takeaway |
|---|---|
| 1 | Authentication must never trust client-supplied input |
| 2 | Verbose error messages are attacker intelligence, not just noise |
| 3 | `robots.txt` is not an access-control mechanism |
| 4 | Backups and legacy files are part of the attack surface |
| 5 | Encryption doesn't compensate for a weak password |
| 6 | Low-severity findings can chain into critical impact |

---

## 📦 Final Deliverables

| Deliverable | Status |
|---|:---:|
| Target reconnaissance | ✅ |
| Authentication analysis & bypass | ✅ |
| Patient portal access | ✅ |
| Three patient PDFs retrieved | ✅ |
| PDF hashes extracted & passwords recovered | ✅ |
| Three PDFs decrypted & verified | ✅ |
| Staff salary exposure identified | ✅ |
| Shareholder exposure identified | ✅ |
| Evidence collected (private) | ✅ |
| Risk ratings assigned | ✅ |
| Remediation recommendations | ✅ |
| Professional report | ✅ |

---

## 📄 Assessment Disclaimer

This penetration test was conducted as part of an **authorized educational security assessment**. The target was explicitly authorized for testing, and all activity was restricted to the agreed scope. No social engineering or denial-of-service testing was performed. All sensitive information has been redacted from this public report. This methodology and evidence are provided for authorized security assessment and educational purposes only.

---

<div align="center">

**Tags:** `penetration-testing` `web-security` `cybersecurity` `ethical-hacking` `black-box-pentest` `sql-injection` `authentication-bypass` `information-disclosure` `password-cracking` `hashcat` `pdf-security` `security-assessment` `vulnerability-assessment`

*Week 4 Penetration Testing Internship — Batch B082*

</div>
