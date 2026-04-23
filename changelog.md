# Changelog

All changes to the NextLayerSec email security infrastructure are documented here.
This file serves as the audit trail for domain security deployments and configuration changes.

---

## 2026-04-23 -- Infrastructure Complete

### domain-3.com
- Completed M365 migration
- Deployed full authentication stack — SPF, DKIM, DMARC p=reject
- MTA-STS deployed and enforced
- DNSSEC enabled and validated

### domain-2.dev
- DKIM configured and validated
- MTA-STS deployed and enforced
- DNSSEC enabled and validated

### All Domains
- S/MIME signing configured on primary domain (domain-1.io)
- Full email security stack validated across all three domains

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

- [ ] All domains -- BIMI configuration (post-DMARC enforce validation)
- [ ] All domains -- CAA record deployment

---

*Format: entries logged by date, domain, and change description.*
*This file doubles as CPE evidence for ISC² continuing education requirements.*
