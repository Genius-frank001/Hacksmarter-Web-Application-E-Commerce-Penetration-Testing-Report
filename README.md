# HackSmarter Web Application — Penetration Test Report

> **Training/lab engagement — HackSmarter Labs environment (`hacksmarter.hsm`).**
> All testing was conducted against a deliberately vulnerable training application within an isolated lab environment.

---

## Overview

This repository contains the penetration test report and supporting artefacts produced during a web application security assessment of the HackSmarter e-commerce lab application. The engagement was conducted as a **grey box** assessment — a standard authenticated user account was available, no source code or infrastructure documentation was supplied up front.

| | |
|---|---|
| **Target** | `hacksmarter.hsm` / `dev.hacksmarter.hsm` — PHP/MySQL (Apache 2.4.56, Debian, PHP 8.0.30) e-commerce application |
| **Assessment Type** | Grey Box Web Application Penetration Test (lab environment) |
| **Tester** | Genius |
| **Report status** | In progress — findings below to be compiled into full report separately |

---

## Findings Summary

**9 findings identified to date: 1 Critical (unconfirmed), 2 High, 4 Medium, 2 Low/Informational.**

| Severity | Finding | Status |
|---|---|---|
| Critical | SQL Injection — UNION-based, full database extraction (`store.php?search=`) | Confirmed |
| High | Broken Access Control — IDOR allowing purchase of unreleased product (`product.php?id=6`) | Confirmed |
| High | Stored HTML Injection via Feedback Form (site-wide persistent defacement) | Confirmed |
| Medium | Broken Access Control — Client-Side-Only Enforcement of Region/Availability Restriction | Confirmed |
| Medium | Sensitive Information Disclosure — Unauthenticated API Exposure of Restricted/Unreleased Product Data (`/api/products/all.php`) | Confirmed |
| Medium | Sensitive File/Directory Exposure — Exposed `/config/` directory and `db.php` | Confirmed (listing only; source not retrieved) |
| Medium | Information Disclosure — Leaked CI/CD Build Log Revealing Internal Paths and Subdomain (`/test/build_log.txt`) | Confirmed |
| Low | Development Environment Exposure — Debug Dashboard, Live `phpinfo()`, Debug Mode/Display Errors Enabled (`dev.hacksmarter.hsm`) | Confirmed |
| Unconfirmed | Unrestricted File Upload — Potential RCE via Profile/Avatar Upload | Attempted, not completed — see Methodology notes |

### Attack Chain Highlights

- **Reconnaissance → Internal Disclosure** — Directory enumeration (`dirsearch`/`ffuf`/`feroxbuster`) uncovered an exposed `/config/` directory listing `db.php`, and a `/test/` directory leaking a GitHub Actions CI/CD build log. The build log revealed the production deploy path (`/var/www/hacksmarter.hsm/`) and the existence of an undocumented `dev.hacksmarter.hsm` subdomain.
- **Subdomain discovery → Debug Exposure** — `dev.hacksmarter.hsm` was reachable via virtual-host (Host header) routing on the same IP/port as production. The staging dashboard exposed a live `phpinfo()` page and confirmed `Debug Mode: ON / Display Errors: ON`, both of which increase the server's information-disclosure surface.
- **Unauthenticated search parameter → Full DB enumeration** — UNION-based SQL injection on `store.php?search=` (confirmed via `sqlmap`) returned the back-end DBMS (MySQL ≥ 5.6), all 5 databases on the server, with table/column enumeration staged as the next step.
- **Client-side trust → Broken Access Control (x2)** — A "region availability" restriction on products, and an "unreleased" product flag (`is_released`), were both enforced only in client-side JavaScript/UI state and/or only hidden from a product listing API. In both cases, direct navigation and DOM/button-state manipulation allowed the restricted/unreleased item to be added to cart and successfully purchased through to order confirmation with no server-side check.
- **Unsanitized input → Site-wide persistent defacement** — The Customer Feedback form (`feedback.php`) stores and renders submitted HTML without encoding, allowing a full-page overlay payload to persist and render for every subsequent visitor to the page — confirmed stored injection, with a clear escalation path to stored XSS (session hijacking) using the same injection point.

---

## Repository Contents

```
hacksmarter-pentest/
├── README.md
├── methodology.md
├── report/
│   └── (full report — to be added)
└── evidence/
    └── (screenshots referenced in findings — to be added)
```

---

## Skills Demonstrated

- Web application reconnaissance and content discovery (`dirsearch`, `ffuf`, `feroxbuster`)
- Virtual host / subdomain discovery and access via Host-header manipulation
- Manual and automated SQL injection exploitation (`sqlmap`, UNION-based technique)
- Client-side control analysis via browser DevTools (DOM/attribute manipulation, JavaScript disabling)
- Identification of Broken Access Control / IDOR patterns across multiple application features
- Stored HTML injection / defacement testing and escalation-path analysis toward stored XSS
- Session cookie handling review (`HttpOnly` implications for cookie theft) and session fixation methodology
- Structured documentation of findings mapped to OWASP Top 10 (2021) and CWE

---

## Tools Used

| Tool | Purpose |
|---|---|
| dirsearch | Web content/directory discovery |
| ffuf | Directory, file-extension, and vhost fuzzing |
| feroxbuster | Recursive content discovery |
| sqlmap | SQL injection confirmation and database enumeration |
| Browser DevTools (Chromium) | Client-side control inspection and bypass (DOM editing, JS disabling) |
| Burp Suite Community Edition | Request inspection/interception (proxy connectivity issues noted in methodology) |
| curl | Manual HTTP request crafting and endpoint verification |
| Python `http.server` | Lightweight listener for payload hosting and out-of-band testing |

---

## Disclaimer

This assessment was performed exclusively within the HackSmarter Labs lab environment against a deliberately vulnerable training application. No real systems, networks, or production data were accessed. Full exploitation steps involving remote code execution tooling (webshells, reverse shells) were intentionally not pursued/completed in this engagement; the file-upload vector is documented as an open/unconfirmed finding requiring a non-destructive proof-of-concept (e.g. a benign marker-file execution test) before being escalated to Critical severity.
