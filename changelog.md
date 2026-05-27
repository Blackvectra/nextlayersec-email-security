# Changelog

All changes to the NextLayerSec email security infrastructure are documented here.
This file serves as the audit trail for domain security deployments and configuration changes.

---

## 2026-05-27 -- Framework expansion: BIMI, CAA, CA, IR, Defender, ARC

### New chapters
- `bimi/deployment.md` -- BIMI deployment, VMC vs CMC, SVG P/S requirements, validation
- `dns/caa-records.md` -- CAA record deployment, recommended baseline, CA cheat sheet
- `conditional-access/policies.md` -- seven-policy Entra ID CA baseline (block legacy auth, require MFA, mobile compliance, country block, risky sign-in, unmanaged browser, admin phishing-resistant MFA)
- `incident-response/runbook.md` -- seven playbooks (compromised mailbox, DMARC spoof spike, MTA-STS false positive, DKIM key compromise, DNSSEC failure, mass phish, legacy auth attempts)
- `defender/tuning.md` -- Safe Attachments / Safe Links / Anti-phish / Anti-spam / Outbound spam / Quarantine tuning
- `defender/tenant-allow-block.md` -- TABL management with quarterly audit pattern
- `defender/user-reporting.md` -- end-user phish reporting workflow and triage
- `dmarc/arc.md` -- ARC overview, trusted-sealer config, verification
- `domains/alias-quickstart.md` -- 30-min alias domain playbook
- `GLOSSARY.md` -- acronym + concept reference
- `CONTRIBUTING.md` -- contribution + redaction guidance

### README updates
- Quickstart section: 10-minute path through the stack
- Mermaid mail-flow diagram
- Stack table expanded with ARC, BIMI, CAA, Conditional Access, Defender for O365
- Documentation Map for navigation
- Repo structure tree refreshed

### Pending status
- CAA records now documented; deployment still pending per-domain
- BIMI now documented; deployment pending VMC acquisition decision

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

- [ ] All domains -- CAA record deployment (docs ready: `dns/caa-records.md`)
- [ ] Primary domain -- BIMI deployment (docs ready: `bimi/deployment.md`; pending VMC acquisition)
- [ ] Conditional Access policies -- staged rollout in Report-only (docs ready: `conditional-access/policies.md`)
- [ ] Defender tuning -- adopt baseline from `defender/tuning.md`
- [ ] Domain-2.dev -- complete the auth stack per `domains/domain-2.md`
- [ ] Domain-3.com -- migration to M365 tenant

---

*Format: entries logged by date, domain, and change description.*
*This file doubles as CPE evidence for ISC² continuing education requirements.*
