# Changelog

All changes to the NextLayerSec email security infrastructure are documented here.
This file serves as the audit trail for domain security deployments and configuration changes.

---

## 2026-11-09 -- CI repair and best-practice pass

The `docs` workflow added on 2026-05-27 never passed: all three of its runs
failed the same day and nothing was committed afterwards.

### Why it was red

- **Markdown lint**: 41 violations under the action's markdownlint (blank lines
  around headings, lists and fences; 73 fences with no language; two ordered
  lists split by a code fence so the numbering restarted).
- **Link check**: two root-relative links in the README (`/validation`,
  `/exchange-online/hardening-runbook.md`) that GitHub renders but lychee
  resolves against the filesystem root, and a Microsoft Learn URL that now
  returns 404 (the standalone DNSSEC article was folded into *How SMTP DANE
  works*).
- **Secret scan**: `base` was the default branch and `head` was `HEAD`, which
  on a push to `main` is the same commit; the TruffleHog action exits non-zero
  when the two match, so the push run could never pass.

### Fixes

- Every fence tagged (`text` for DNS records, policy files, headers and
  diagrams; `powershell` for sessions); the split ordered lists re-joined by
  indenting the fence under its step. `MD060` (table column style, new in
  markdownlint v0.42) disabled with a reason, as the other rules are.
- README links made relative; DNSSEC reference re-pointed to Microsoft's
  current article and the `Enable-DnssecForVerifiedDomain` cmdlet page.
- TruffleHog scans the whole tree and history (`--results=verified,unknown`)
  with no `base`/`head`.

### Content corrections found on the way

- `domains/_template.md` still carried the DKIM CNAME form the 2026-05-27 pass
  corrected elsewhere (`selector1-<tenant>._domainkey.microsoft.com`); now
  `selector1-<domain-with-hyphens>._domainkey.<tenant>.onmicrosoft.com`.
- README's MTA-STS policy example named the live MX host; it now uses the
  `<domain-with-hyphens>` placeholder SECURITY.md documents.

### Workflow hardening

- All actions pinned to commit SHAs with the version in a comment;
  `.github/dependabot.yml` (github-actions ecosystem, weekly, grouped) keeps
  the pins current.
- `persist-credentials: false` on every checkout; `timeout-minutes` on every
  job; `concurrency` cancels a superseded pull-request run and never a run on
  `main`.
- `workflow_dispatch` for manual runs and a weekly schedule for the link check
  alone, since external pages move without a commit here.
- `--root-dir` passed to lychee so a future root-relative link resolves the
  way GitHub renders it.

---

## 2026-05-27 -- Documentation hardening pass

### Security guidance corrections

- Break-glass account guidance updated: replaced "No MFA registered" with
  phishing-resistant MFA (FIDO2 key stored offline) and narrowed the
  Conditional Access exclusion to policies that could cause lockout
  (`exchange-online/hardening-runbook.md`, `baseline-checklist.md`).
- DKIM CNAME format error corrected: `microsoft.com` -> `<tenant>.onmicrosoft.com`
  in `dns/cloudflare-records.md` and `dkim/selector-management.md`.

### OSINT redaction

- Real domain names, tenant identifiers and GitHub usernames removed from all
  operational docs. Domain files renamed from `<real-name>.md` to
  `domain-1/2/3.md`. Redaction policy documented in new `SECURITY.md`.
- Stale status tables in DKIM and DMARC docs reconciled with per-domain
  reality (only primary domain fully deployed).

### Repo cleanup

- Duplicated CAA section in `dmarc/monitoring-setup.md` removed.
- README repo-structure tree updated to match on-disk filenames.
- README domain-coverage table updated to reflect actual deployment status.

### New content

- `dns/dns-setup.md` -- provider-agnostic DNS setup guide. Cloudflare-specific
  notes consolidated into `dns/cloudflare-records.md`.
- `SECURITY.md` -- responsible disclosure and OSINT redaction policy.
- CI workflow (`.github/workflows/docs.yml`) -- markdownlint, lychee link
  check, TruffleHog secret scan, and a custom OSINT-redaction check that
  fails the build if real operational identifiers reappear in docs.

---

## 2026-04-23 -- Primary Domain Full Stack Complete

### domain-1.io (primary)

- Full email security stack validated end-to-end
- SPF — PASS
- DKIM — configured and signing
- DMARC — `p=reject` enforced
- MTA-STS — `enforce` mode, validated
- DNSSEC — enabled and validated
- DNSSEC-aware MX — `p-v1.mx.microsoft` endpoint active
- TLS-RPT — configured and reporting
- S/MIME — signing enabled on primary mailbox

### domain-2.dev (secondary)

- SPF deployed, DMARC at `p=reject` with reporting consolidated to primary
- MX, DKIM, MTA-STS, DNSSEC, TLS-RPT still pending -- see `/domains/domain-2.md`

### domain-3.com (personal brand)

- Still on iCloud -- migration to M365 tenant pending
- See `/domains/domain-3.md` for migration plan

---

## 2026-04-18 -- Initial Deployment

### domain-1.io

- Deployed MTA-STS policy via GitHub Pages
- Configured `mta-sts.domain-1.io` CNAME in Cloudflare (DNS-only)
- Published `_mta-sts` TXT record (`id=20260416`)
- Published `_smtp._tls` TXT record for TLS-RPT reporting
- Enabled DNSSEC via Exchange Online PowerShell (`Enable-DnssecForVerifiedDomain`)
- Updated MX to DNSSEC-aware endpoint: `domain-1-io.p-v1.mx.microsoft`
- Validated full chain: MXToolbox, Verisign DNSSEC Analyzer, Exchange Online PowerShell
- MTA-STS mode: `enforce`
- DNSSEC status: `Enabled`

### domain-2.dev

- Deployed SPF: `v=spf1 include:spf.protection.outlook.com -all`
- Deployed DMARC: `v=DMARC1; p=reject; rua=mailto:dmarc@domain-1.io`
- Domain added to primary M365 tenant as accepted domain

### M365 Tenant Hardening

- Blocked all legacy authentication protocols via `Set-AuthenticationPolicy`
- Assigned authentication policy to all tenant users
- Set organization default authentication policy
- Excluded break glass account from authentication policy
- Disabled SMTP client authentication tenant-wide (`Set-TransportConfig`)
- Disabled external auto-forwarding (`Set-RemoteDomain`)
- Disabled POP and IMAP on all mailboxes (`Set-CasMailbox`)
- Extended mailbox audit log retention to 180 days
- Enabled full audit actions on all mailboxes (Owner, Delegate, Admin)
- Confirmed unified audit logging enabled
- Configured outbound spam notification to tenant admin mailbox
- Disabled OWA text messaging
- Confirmed Safe Attachments block mode on all policies
- Confirmed Safe Links URL scanning on all policies
- Confirmed ZAP enabled on all spam and malware filter policies
- Confirmed ATP enabled for SharePoint, Teams, OneDrive
- Enabled DNSSEC recognition in Exchange Online tenant

---

## Pending

- [ ] All domains -- CAA record deployment
- [ ] All domains -- BIMI configuration (post-CAA deployment)

---

*Format: entries logged by date, domain, and change description.*
*This file doubles as CPE evidence for ISC² continuing education requirements.*
