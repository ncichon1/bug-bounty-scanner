# Bounty Assistant Program

An automated bug bounty reconnaissance and vulnerability scanning toolkit.

## Scanner Overview

- **72 checks** spanning web, API, and Web3 attack surface
- **Async batch scanning** with concurrency limiting and automatic scope pull
- **Multi-platform support**: Bugcrowd, Intigriti, YesWeHack (HackerOne deprioritized)
- **Intelligent target scoring and ranking** with payout-based prioritization
- **Overnight autonomous agent** that proposes new checks nightly

## Check Categories

The scanner groups its checks into the following categories. Detection logic and
payloads are intentionally omitted.

- **Injection** — SQLi, NoSQLi, SSTI, XXE, Log4Shell, Prototype Pollution, CRLF
- **Authentication** — JWT algorithm confusion, weak secrets, RS256→HS256 downgrade,
  OAuth state / PKCE / redirect handling
- **Access Control** — IDOR, CORS, 403 bypass, verb tunneling, Unicode bypass,
  mass assignment
- **Information Disclosure** — source maps, exposed files, leaked secrets,
  S3 / GCS / Azure dangling assets, metrics exposure
- **Business Logic** — payment amount tampering, rate limiting, GraphQL batch abuse,
  SAML bypass
- **Network** — HTTP Request Smuggling (with CDN guards for
  CloudFront / Cloudflare / Envoy / Akamai / Imperva), SSRF
  (params / headers / redirect / file upload / webhook), cache poisoning
- **Web3** — Ethereum RPC, IPFS / Arweave scanning, private key patterns
- **Recon** — subdomain takeover, subdomain enumeration, port scanning

## False Positive Engineering

Reducing noise is a first-class concern. The scanner invests heavily in confirming
findings before reporting them:

- **CDN-aware smuggling guards** — CloudFront, AWS ALB, Cloudflare, Envoy, Akamai,
  and Imperva are detected and handled to suppress false HTTP Request Smuggling hits
- **Behavioral diff confirmation** — SSTI and path traversal require an observed
  content difference before a finding is raised
- **Source map severity downgrade** — severity is lowered when fewer than three
  non-vendor sources are present
- **Secret-pattern filtering** — Stripe publishable keys are suppressed, an elliptic
  curve constant blocklist is applied, and CSS variable patterns are filtered out
- **Live credential validation** — Google API keys are validated against the live
  endpoint before being flagged
- **WAF-aware injection logic** — WAF detection gates the injection checks

## Architecture

- **Python CLI** exposed via the `bh` alias
- **Playwright integration** for JavaScript analysis and deep crawling
- **Interactsh OOB** for blind SSRF, XXE, Log4Shell, and command injection
- **Per-job async logging**, state files, and batch queue management
- **Overnight Claude Code agent** for autonomous check proposals

## ⚠️ Legal

Only use on programs and targets you are explicitly authorized to test. Always stay
within the defined scope of the program.
