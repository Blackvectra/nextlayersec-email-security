# Changelog

All changes to the NextLayerSec email security infrastructure are documented here.
This file serves as the audit trail for domain security deployments and configuration changes.

---

## 2026-04-18 -- Initial Deployment

### nextlayersec.io

- Deployed MTA-STS policy via GitHub Pages (`nextlayersec-mta-sts` repo)
- Configured `mta-sts.nextlayersec.io` CNAME in Cloudflare (DNS-only)
- Published `_mta-sts` TXT record (`id=20260416`)
- Published `_smtp._tls` TXT record for TLS-RPT reporting
- Enabled DNSSEC via Exchange Online PowerShell (`Enable-DnssecForVerifiedDomain`)
- Updated MX to DNSSEC-aware endpoint: `nextlayersec-io.p-v1.mx.microsoft`
- Validated full chain: MXToolbox, Verisign DNSSEC Analyzer, Exchange Online PowerShell
- MTA-STS mode: `enforce`
- DNSSEC status: `Enabled`

### nextlayersec.dev

- Deployed SPF: `v=spf1 include:spf.protection.outlook.com -all`
- Deployed DMARC: `v=DMARC1; p=reject; rua=mailto:dmarc@nextlayersec.io`
- Domain added to nextlayersec.io M365 tenant as accepted domain
- MTA-STS and DNSSEC pending

### M365 Tenant Hardening (nextlayersec.io)

- Blocked all legacy authentication protocols via `Set-AuthenticationPolicy`
- Assigned authentication policy to all tenant users
- Set organization default authentication policy
- Excluded break glass account from authentication policy
- Disabled SMTP client authentication tenant-wide (`Set-TransportConfig`)
- Disabled external auto-forwarding (`Set-RemoteDomain`)
- Disabled POP, IMAP, ActiveSync on all mailboxes (`Set-CasMailbox`)
- Extended mailbox audit log retention to 180 days
- Enabled full audit actions on all mailboxes (Owner, Delegate, Admin)
- Confirmed unified audit logging enabled
- Configured outbound spam notification to `mlevorson@nextlayersec.io`
- Disabled OWA text messaging
- Confirmed Safe Attachments block mode on all policies
- Confirmed Safe Links URL scanning on all policies
- Confirmed ZAP enabled on all spam and malware filter policies
- Confirmed ATP enabled for SharePoint, Teams, OneDrive
- Enabled DNSSEC recognition in Exchange Online tenant

---

## Pending

- [ ] `nextlayersec.dev` -- MTA-STS deployment
- [ ] `nextlayersec.dev` -- DNSSEC enablement
- [ ] `nextlayersec.dev` -- DKIM configuration
- [ ] `mattlevorson.com` -- iCloud to M365 migration
- [ ] `mattlevorson.com` -- Full auth stack deployment
- [ ] `mattlevorson.com` -- MTA-STS deployment
- [ ] `mattlevorson.com` -- DNSSEC enablement
- [ ] All domains -- BIMI configuration (post-DMARC enforce validation)
- [ ] All domains -- CAA record deployment
- [ ] `nrgtechservices.com` -- DNSSEC enablement (requires NRG authorization)
- [ ] `nrgtechservices.com` -- MTA-STS flip to enforce mode

---

*Format: entries logged by date, domain, and change description.*
*This file doubles as CPE evidence for ISC2 SSCP/CySA+ continuing education requirements.*
