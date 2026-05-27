# Security Policy

## Scope

This repository documents an email security hardening framework. It does not
host live services, secrets, or executable code. Disclosures here cover:

- Factual errors in security guidance (wrong DMARC/SPF/DKIM syntax, misleading
  PowerShell, incorrect threat model claims)
- Documentation that could lead readers to a less secure configuration than
  intended
- Accidental disclosure of real operational details (tenant identifiers,
  internal email addresses, selector hostnames)

## Reporting a vulnerability or documentation defect

Open a GitHub issue, or for sensitive disclosures email
[security@nextlayersec.io](mailto:security@nextlayersec.io). PGP available on
request.

Please include:

1. The file and line range affected
2. What the current text says
3. Why it is wrong or misleading
4. The corrected guidance, with a citation if possible (RFC, Microsoft Learn,
   etc.)

We aim to acknowledge within 5 business days.

## Redaction policy (OSINT)

Real domain names, tenant identifiers, internal email addresses, GitHub
usernames hosting MTA-STS policy repos, and DKIM selector hostnames are
**not committed** to this repository. All examples use the anonymized
placeholders:

| Placeholder | Stands for |
|---|---|
| `domain-1.io` | Primary brand domain |
| `domain-2.dev` | Secondary brand / dev domain |
| `domain-3.com` | Personal-brand / migration domain |
| `<tenant>` | M365 `*.onmicrosoft.com` tenant prefix |
| `<github-username>` | GitHub user or org hosting the MTA-STS Pages repo |
| `<domain-with-hyphens>` | Domain with `.` replaced by `-` for M365 endpoint construction |

The only real public identifier intentionally retained is the
`nextlayersec.io` brand URL in the README header badges and footer, since
that is already the public brand of the framework.

If you find a real value that should have been redacted, please report it as
above — we will rotate or restructure as needed.

## Supported guidance

The framework targets:

- Microsoft 365 Business Premium and equivalent Exchange Online plans
- Cloudflare and provider-agnostic DNS setups (see `/dns/`)
- Modern RFCs: 7208 (SPF), 6376 (DKIM), 7489 (DMARC), 8461 (MTA-STS),
  8460 (TLS-RPT), 4033-4035 (DNSSEC)

Guidance for retired Exchange Online cmdlets, legacy MFA, or deprecated
Defender policies should be flagged for removal.
