# Methodology — HackSmarter Web Application Pentest

This document records the phases, tools, and reasoning used during the assessment, in the order they were performed.

---

## 1. Reconnaissance & Content Discovery

**Objective:** Map the application's exposed surface before any exploitation.

- Enumerated directories/files using `dirsearch`, `ffuf`, and `feroxbuster` against `hacksmarter.hsm`.
- Representative commands used:
  ```bash
  dirsearch -u http://hacksmarter.hsm -e php,html,js,txt,bak,zip,config

  ffuf -u http://hacksmarter.hsm/FUZZ -w /usr/share/wordlists/dirb/common.txt -fc 404

  feroxbuster -u http://hacksmarter.hsm -w /usr/share/wordlists/dirb/common.txt -C 404
  ```
- **Result:** Identified an exposed `/config/` directory with Apache directory listing enabled, containing `db.php`. Direct browser access to `db.php` returned a blank page (expected — PHP executes server-side; the file produces no output), confirming the file exists and runs without error but not yielding its source directly.
- **Result:** Identified a `/test/` directory containing `build_log.txt`, a leaked GitHub Actions CI/CD production-deploy log. This disclosed:
  - Production deploy path: `/var/www/hacksmarter.hsm/`
  - An undocumented subdomain: `dev.hacksmarter.hsm` (explicitly noted in the log as *not* synced with the latest production deploy)
  - A commit reference (`5b2a9f`) tying the deploy to a version-controlled source repository

## 2. Subdomain / Virtual Host Discovery

- Confirmed `dev.hacksmarter.hsm` resolves via **name-based virtual hosting** on the same IP and port as production (not a separate port).
- Added a manual `/etc/hosts` entry to resolve the subdomain locally, and cross-verified using an explicit Host header:
  ```bash
  curl -H "Host: dev.hacksmarter.hsm" http://<target-ip>/
  ```
- **Result:** `dev.hacksmarter.hsm` served a staging dashboard explicitly labelled "STAGING ENVIRONMENT — INTERNAL USE ONLY." The page disclosed:
  - A live, unauthenticated `phpinfo()` debug page
  - `Debug Mode: ON` / `Display Errors: ON` footer notice
  - A note referencing a "Region Locking" feature and advising users experiencing checkout lockouts to clear cookies — flagged as a hint toward a client-side/cookie-based access control mechanism, later confirmed (see §4).

## 3. SQL Injection Testing

**Target:** `store.php?search=` (product search parameter)

- Manual syntax-breaking test (`'`) used to confirm injectability before automated tooling.
- Escalated to `sqlmap` for confirmation and enumeration:
  ```bash
  sqlmap -u "http://hacksmarter.hsm/store.php?search=test" --cookie="PHPSESSID=<session>" --batch --dbs
  ```
- **Result:** Confirmed **UNION-based SQL injection**, 7-column result set, back-end DBMS identified as MySQL ≥ 5.6 on Linux Debian / Apache 2.4.56 / PHP 8.0.30. Five databases enumerated: `hacksmarter`, `information_schema`, `mysql`, `performance_schema`, `sys`.
- **Next steps staged (not yet completed):** table enumeration and credential dump against the `hacksmarter` database via:
  ```bash
  sqlmap -u "..." -D hacksmarter --tables --batch
  sqlmap -u "..." -D hacksmarter -T <table> --columns --batch
  sqlmap -u "..." -D hacksmarter -T <table> --dump --batch
  ```

## 4. Client-Side Access Control Analysis

**Target:** Region-availability restriction on product listings (`store.php`, `assets/js/store.js`)

- Reviewed `store.js` directly and identified a self-flagged vulnerability in a code comment: the `checkRegionAvailability()` function reads a `data-region-available` HTML attribute and conditionally disables/hides the "Buy" button **client-side only**, with no corresponding server-side check.
- **Bypass method 1:** Edited the `data-region-available` attribute directly in browser DevTools (Elements panel).
- **Bypass method 2:** Disabled JavaScript execution entirely via DevTools (Command Menu → "Disable JavaScript"), preventing `checkRegionAvailability()` from running and leaving the Buy/Add-to-Cart control in its default (enabled, visible) state.
- **Result:** Restricted product ("Flipper Zero (Restricted)") was successfully added to cart and carried through a full checkout to order confirmation (`Order #31882697`), proving the restriction has no server-side enforcement at any stage of the purchase flow.

## 5. IDOR / Unreleased Resource Access

**Target:** `/api/products/all.php`, `product.php?id=`

- Queried the product listing API directly and observed it returned **all** products, including one flagged `"is_released":"0"` ("Quantum Decryptor (Unreleased)", `id: 6`), with no filtering applied before the response was sent to an unprivileged client.
- Navigated directly to `product.php?id=6` (an ID not linked anywhere in the normal UI) and completed a full purchase (`Order #46942953`, $9,999.00), confirming no server-side check against the `is_released` flag at any point in the add-to-cart or checkout flow.
- **Classification:** CWE-639 (Authorization Bypass Through User-Controlled Key) / CWE-200 (Exposure of Sensitive Information), mapped to OWASP A01:2021 – Broken Access Control.

## 6. Stored HTML Injection / Defacement Testing

**Target:** Customer Feedback form (`feedback.php`)

- Submitted a crafted HTML/CSS payload (full-viewport fixed-position overlay) into the "Message" field.
- **Result:** Payload persisted server-side and rendered unescaped on every subsequent page load of `feedback.php`, confirming **stored** (not reflected) injection affecting all visitors, not just the submitting session.
- Noted as a direct escalation path to stored XSS (session/cookie theft) using the identical injection point, differing only in payload content (`<script>`/event-handler payloads vs. static HTML/CSS).

## 7. Cross-Site Scripting (XSS) Testing

- Tested reflected XSS payloads against the search parameter (baseline `<script>alert(document.domain)</script>` and event-handler/filter-bypass variants).
- Tested stored XSS via SVG file upload (profile/avatar feature):
  - Crafted SVG payloads using both `<script>` blocks and `onload`-based event handler vectors.
  - Noted the key distinction that `<img>`-embedded SVGs do not execute inline scripts in modern browsers, and that direct navigation to the uploaded file (not the profile page) is required to test true execution.
  - **Result:** Not conclusively confirmed working within this engagement session — flagged as open, pending verification of (a) served `Content-Type` header on the uploaded file, (b) whether upload processing re-encodes/strips the file, and (c) direct-navigation testing.

## 8. Unrestricted File Upload (RCE Vector)

**Target:** Profile/avatar upload functionality

- Confirmed the upload form accepts a `Content-Type: image/jpeg` override on an uploaded file while the underlying content was PHP source — i.e., client-supplied `Content-Type` is not independently verified against actual file content.
- **Status:** A safe, non-destructive proof-of-concept was proposed (uploading an inert PHP snippet that echoes a hash, to confirm server-side code execution without establishing an interactive shell) but was **not completed/confirmed** within this engagement. Full webshell/reverse-shell tooling was intentionally excluded from this testing approach.
- **Recommendation for follow-up:** Complete the inert-marker PoC (`<?php echo "RCE-CONFIRMED-" . md5("test"); ?>`) and verify via direct request to the uploaded file path whether the server executes or merely serves the content, before assigning final severity to this finding.

## 9. CSRF, Clickjacking, and Session Attack Methodology (Reference Notes)

The following attack classes were discussed and payload/methodology prepared during the engagement for structured testing against relevant endpoints (e.g. `update-email`, `transfer`):

- **CSRF:** Auto-submitting cross-origin HTML form technique for state-changing GET/POST endpoints; dependent on absence of anti-CSRF tokens and permissive `SameSite` cookie configuration.
- **Clickjacking:** Transparent (`opacity: 0`) iframe overlay technique, layering a legitimate "Transfer Money"-style button beneath an attacker-controlled decoy; dependent on absence of `X-Frame-Options` / `frame-ancestors` CSP.
- **Session Fixation:** Pre-login vs. post-login session identifier comparison methodology (`PHPSESSID` captured before and after authentication) to determine whether the application regenerates session tokens on privilege change.

These were documented as methodology/technique references; confirmed execution against specific HackSmarter endpoints should be logged separately as each is completed.

---

## Notes on Tooling Issues Encountered

- Burp Suite Community Edition intermittently failed to proxy requests to the target (`Failed to connect to <target-ip>:80`), traced to a lab environment outage rather than a Burp configuration fault — confirmed via `ping` packet loss and curl connection hangs against the target IP. Resolved by waiting for lab environment restart; `curl` was used as the primary interception-independent fallback throughout.
