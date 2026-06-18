# bug-bounty-scanner

Automated reconnaissance and vulnerability triage scanner for bug bounty programs.

---

## Overview

This tool automates the first two phases of a bug bounty workflow — scope enumeration and surface-level vulnerability triage — across HackerOne programs. It is not a fire-and-forget attack tool; it is a triage accelerator. Every signal it produces is treated as a candidate for manual review, not a submission. The philosophy is that a false positive submitted to a program is worse than a missed finding, so the system is engineered to be skeptical of its own output at every stage.

The scanner was built iteratively, shaped by real false-positive patterns encountered across dozens of programs, and has accumulated an explicit false-positive reduction layer that continues to grow with each scan run.

---

## Pipeline Architecture

```
HackerOne Scope API
        │
        ▼
┌──────────────────────┐
│  Scope Discovery &   │  Fetches in-scope assets per program handle,
│  Target Ranking      │  deduplicates, and queues targets by estimated
│                      │  signal-to-noise ratio.
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  Passive Recon       │  Subdomain enumeration, Wayback Machine
│                      │  historical URL mining, JS parameter extraction,
│                      │  and authenticated crawling for endpoint discovery.
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  Automated Scanning  │  57 distinct vulnerability check categories
│                      │  run concurrently across each target, covering
│                      │  injection classes, configuration weaknesses,
│                      │  authentication/authorization issues, and
│                      │  cloud/infrastructure exposure. Optional
│                      │  template-based detection via Nuclei.
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  False-Positive      │  Multi-layer suppression: baseline-stability
│  Reduction           │  checks, WAF/CDN interception-page detection,
│                      │  cross-host deduplication to catch shared-
│                      │  infrastructure noise, and content-presence
│                      │  pre-checks before flagging keyword-class
│                      │  findings.
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  Manual Verification │  Every surviving candidate is reproduced by
│  (mandatory)         │  hand with curl or a browser before anything
│                      │  is written up. The scanner output is a
│                      │  starting point, not a report.
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  Report Generation   │  Structured markdown reports for triaged,
│                      │  manually-confirmed findings only.
└──────────────────────┘
```

---

## Engineering Story: Iterative False-Positive Reduction

The scanner started as a straightforward HTTP client that sent known-bad inputs and checked responses for error signatures. The first real-world scan runs immediately revealed the gap between "this check fires" and "this is actually a finding."

Several concrete problems emerged and were solved in sequence:

**Baseline instability.** Checking response size or status code against a single prior request fails on any page with dynamic content — session tokens, timestamps, A/B test variants. The fix was a two-sample stability check: if the same URL returns meaningfully different body lengths on back-to-back unauthenticated requests, behavioral diffs (length, timing, status) are suppressed for that URL. Error-signature matches, which are content-specific rather than size-specific, remain valid.

**WAF/CDN interception pages.** Many targets sit behind perimeter defenses that return a uniform "blocked" page for any request containing an injection payload. That page can match error signatures, differ from the baseline, and even set unusual headers — all the signals we look for — while never touching the application. The fix was a library of WAF/CDN page signatures (Cloudflare, Akamai, Imperva, Envoy, F5) checked on both the baseline and the payload response. If either side is an interception page, the check is suppressed.

**Shared-infrastructure noise.** On programs with many subdomains behind a common load balancer or WAF, the same payload can produce the same response body across dozens of unrelated hosts. That body then matches signatures and generates findings on all of them simultaneously. The fix was a cross-host deduplication registry: if the same payload response hash appears on three or more distinct hosts within a scan session, it is reclassified as infrastructure noise and suppressed regardless of content.

**SPA catch-all routing.** Single-page apps commonly return HTTP 200 with the index shell for any unrecognised route, including probe paths like `/.env` or `/admin`. The fix was an SPA-fingerprinting step that captures the homepage's title, root element ID, and script bundle URLs, then rejects any probe response that matches all three — the SPA's router is swallowing the request, not the application serving real content.

**Keyword false positives.** Some check categories (XSS, open redirect) use marker strings that appear in minified JavaScript naturally — a reflection check that fires on any page containing the marker word, regardless of whether the input was echoed, is useless. The fix was an explicit reflection pre-check: inject a random marker value first, and only proceed with payloads if the parameter actually echoes it.

Each of these was added in response to specific observed false-positive patterns, not speculatively. The goal at each step was not a more exhaustive scanner, but a more trustworthy one.

---

## Vulnerability Check Categories

The scanner covers 57 distinct finding types across these broad categories:

| Category | Examples |
|---|---|
| **Injection** | SQL injection, NoSQL injection, command injection, SSTI, XXE, CRLF injection, path traversal, prototype pollution |
| **Authentication & Authorization** | CSRF, IDOR, privilege escalation, forced browsing, mass assignment, OAuth missing state parameter, JWT algorithm confusion, 403 bypass, HTTP verb tunneling |
| **Infrastructure & Configuration** | CORS misconfiguration, host header injection, cache poisoning, web cache deception, dangerous HTTP methods, Unicode path bypass |
| **Secrets & Exposure** | Exposed secrets in HTML/JS, source map disclosure, S3/GCS/Azure bucket exposure, sensitive file paths, sensitive error disclosure |
| **Client-Side** | XSS (reflected), open redirect, clickjacking, JSONP, postMessage origin validation |
| **Recon / Informational** | Subdomain takeover, security header gaps, insecure cookie attributes, GraphQL introspection, API version enumeration, user enumeration, server-timing disclosure |
| **Out-of-Band** | Blind SSRF (header injection), Log4Shell (OOB callback via JNDI), command injection (DNS/HTTP callback confirmation) |

OOB checks require [interactsh-client](https://github.com/projectdiscovery/interactsh) to be installed; they are skipped gracefully when it is unavailable.

---

## Tech Stack

- **Language:** Python 3.12
- **HTTP / networking:** `requests` with a persistent session, `urllib3`, a custom split-timeout wrapper using `concurrent.futures.ThreadPoolExecutor` (40 workers) for wall-clock deadline enforcement on DNS-stalling requests
- **Terminal UI:** `rich` — progress bars, live finding tables, color-coded severity output
- **Browser automation:** `playwright` (authenticated crawling and session management)
- **Optional integrations:** [Nuclei](https://github.com/projectdiscovery/nuclei) (template-based detection), [interactsh-client](https://github.com/projectdiscovery/interactsh) (OOB callback confirmation for blind SSRF, XXE, Log4Shell, command injection)
- **Finding persistence:** SQLite via Python's standard `sqlite3` module
- **Historical URL discovery:** Wayback Machine CDX API
- **Scope management:** HackerOne public scope API

---

## Scope

Targets span dozens of HackerOne programs across fintech, SaaS, e-commerce, travel, and communications verticals. Scope is managed per-program and pulled dynamically from the HackerOne API, not hardcoded.

---

## Responsible Disclosure / Ethics

- Every finding is manually reproduced — curl request, raw response, behavioral diff confirmed — before any writeup is drafted. The scanner output is a candidate list, not a report.
- The scanner is never mentioned in actual HackerOne submissions. Several programs explicitly prohibit tool-generated submissions, and even where they do not, an unverified scanner finding is not a vulnerability report.
- No credentials, session tokens, or user data from target applications are stored or transmitted beyond the local machine.
- Scans are rate-limited and scoped strictly to the assets listed in each program's in-scope policy. Out-of-scope assets are never probed.
