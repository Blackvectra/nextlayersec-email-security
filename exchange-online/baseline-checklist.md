# Exchange Online Hardening Baseline Checklist

Verification checklist for M365 Business Premium tenant hardening.
Run after initial deployment and quarterly thereafter.

---

## Authentication Controls

- [ ] Authentication policy created and named
- [ ] All `AllowBasicAuth*` parameters set to `$false`
- [ ] Authentication policy assigned to all users
- [ ] Organization default authentication policy set
- [ ] Break glass account excluded from authentication policy
- [ ] SMTP client authentication disabled tenant-wide

---

## Mail Flow Controls

- [ ] External auto-forwarding disabled on Default remote domain
- [ ] No mailboxes with active forwarding addresses
- [ ] No inbox rules forwarding or redirecting to external addresses
- [ ] Outbound spam notification enabled with admin recipient configured

---

## Protocol Controls

- [ ] POP disabled on all mailboxes
- [ ] IMAP disabled on all mailboxes
- [ ] ActiveSync scoped to compliant devices via Conditional Access
- [ ] OWA text messaging disabled

---

## Auditing Controls

- [ ] Unified audit logging enabled
- [ ] Mailbox auditing enabled on all mailboxes
- [ ] Audit log retention set to 180 days
- [ ] Full audit actions enabled for Owner, Delegate, and Admin

---

## Defender for Office 365

- [ ] Safe Attachments policy enabled with Block action
- [ ] Safe Links policy enabled with URL scanning on all messages
- [ ] Safe Links enabled for internal senders
- [ ] Anti-phishing policy enabled with mailbox intelligence
- [ ] Anti-phishing policy enabled with spoof intelligence
- [ ] ZAP enabled for spam
- [ ] ZAP enabled for phishing
- [ ] ATP enabled for SharePoint, Teams, and OneDrive
- [ ] Malware filter file filtering enabled

---

## DKIM

- [ ] DKIM signing enabled on all accepted domains
- [ ] Selector CNAME records validated in DNS
- [ ] DKIM pass confirmed in message headers

---

## DNSSEC

- [ ] DNSSEC enabled on all domains via Exchange Online PowerShell
- [ ] MX records updated to DNSSEC-aware endpoints
- [ ] DNSSEC chain validated via Verisign DNSSEC Analyzer

---

## Break Glass Account

- [ ] No authentication policy assigned
- [ ] Excluded from all Conditional Access policies
- [ ] No MFA registered
- [ ] Password stored offline
- [ ] Sign-in alert active in Entra ID
- [ ] Reviewed this quarter

---

## Known Accepted Risks Review

- [ ] OWA LinkedIn Integration — reviewed and accepted
- [ ] ActiveSync scoping — reviewed and accepted

---

## Quarterly Review Sign-Off

| Date | Reviewer | Notes |
|---|---|---|
| 2026-04-18 | NextLayerSec | Initial deployment baseline |

---

*NextLayerSec*
*Last validated: April 2026*
