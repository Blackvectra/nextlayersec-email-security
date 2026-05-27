# Email Security Incident Response Runbook

Playbooks for the most common email-security incidents in an M365 tenant.
Each playbook is structured as: **detect -> contain -> investigate -> recover -> postmortem**.

> Run sign-in alerts and audit-log alerts to your security mailbox so these
> playbooks are triggered by something other than a user complaint.

---

## Quick Reference

| Incident | First action | See section |
|---|---|---|
| Mailbox compromised | Revoke sessions + reset password | 1 |
| DMARC reports spike of spoof attempts | Verify enforcement, then investigate sender | 2 |
| MTA-STS rejecting legit mail | Check policy `mx:` value vs live MX | 3 |
| DKIM key suspected compromised | Rotate immediately | 4 |
| DNSSEC validation failing | Verify DS at registrar, do not disable | 5 |
| Mass phishing campaign targeting tenant | Hunt + purge via Threat Explorer | 6 |
| Legacy auth attempt against blocked account | Verify CA is blocking; investigate source IP | 7 |

---

## 1. Compromised Mailbox

### Detect
- Entra ID risky sign-in alert
- User reports unexpected mail flow (rules, sent items)
- Outbound spam triggered `NotifyOutboundSpam` to admin
- Unusual forwarding rule appears in audit log

### Contain (in this order, fast)

```powershell
Connect-ExchangeOnline
Connect-MgGraph -Scopes "User.ReadWrite.All"

# 1. Revoke all active sessions
Revoke-MgUserSignInSession -UserId user@<domain>

# 2. Force password reset at next sign-in
# (do in Entra portal: Users -> user -> Reset password)

# 3. Disable inbox rules
Get-InboxRule -Mailbox user@<domain> | Disable-InboxRule -Confirm:$false

# 4. Remove mailbox-level forwarding
Set-Mailbox -Identity user@<domain> -ForwardingAddress $null -ForwardingSmtpAddress $null -DeliverToMailboxAndForward $false

# 5. Block sign-in temporarily while investigating
Update-MgUser -UserId user@<domain> -AccountEnabled:$false
```

### Investigate

```powershell
# Recent sign-ins (last 30 days)
Get-MgAuditLogSignIn -Filter "userPrincipalName eq 'user@<domain>'" -Top 50 |
  Format-Table CreatedDateTime, IpAddress, Location, AppDisplayName, Status

# Mailbox audit log
Search-MailboxAuditLog -Identity user@<domain> -StartDate (Get-Date).AddDays(-30) -EndDate (Get-Date) -ShowDetails

# Sent items in date range -- did the attacker mail others?
Get-MessageTrace -SenderAddress user@<domain> -StartDate (Get-Date).AddDays(-7) -EndDate (Get-Date) |
  Format-Table Received, RecipientAddress, Subject, Status

# Inbox rules created during compromise
Get-InboxRule -Mailbox user@<domain> | Format-List Name, Description, Enabled, WhenChanged
```

Capture findings to the incident ticket. Common artifacts:
- Sign-in from unfamiliar geography immediately before the alert
- Inbox rule auto-deleting incoming mail to hide bounces
- External forwarding rule (despite tenant-level block -- attackers use
  in-message rules with `RedirectTo` instead)

### Recover
- User contacts IT through a verified out-of-band channel before re-enabling
- Re-register MFA from scratch (delete existing methods, require re-enrollment)
- Re-enable account
- Force re-login on all devices
- Notify any recipients of attacker-sent mail

### Postmortem
- Root cause: phishing? credential reuse? token theft?
- Update CA / Defender rules based on the vector used
- Document in changelog

---

## 2. DMARC Reports Show Spoofing Volume

### Detect
- DMARCian dashboard shows new high-volume source from unknown IPs
- Aggregate report XML contains `<source_ip>` you do not recognize with high `<count>`

### Investigate

1. Verify the source is actually unrelated to your sending infrastructure
   (check the IP against ARIN / RIPE, against any vendor sending on your behalf)
2. Confirm `disposition` is `reject` -- if it is, attackers are being blocked,
   this is a monitoring event not an active incident
3. If `disposition` is `none` or `quarantine`, tighten the policy:

```
v=DMARC1; p=reject; rua=mailto:dmarc@<domain>
```

### Contain
- If the volume is targeted at your own users (internal spoof), publish a
  user warning via tenant banner
- Add the source IP to a Defender for Office 365 connection filter block
  list if it shows up across multiple domains

### Recover
- Mail at `p=reject` is already rejected; recovery is verifying users were
  not socially engineered before enforcement
- Search for any messages that landed in junk before enforcement was tight:

```powershell
Get-MessageTrace -StartDate (Get-Date).AddDays(-7) -EndDate (Get-Date) -SenderAddress "*@<domain>" |
  Where-Object { $_.FromIP -notin $authorizedIps } | Format-Table
```

### Postmortem
- If alignment was failing on legitimate mail, fix SPF/DKIM before blaming
  spoofers
- Document in changelog

---

## 3. MTA-STS Rejecting Legitimate Mail

### Detect
- Sending parties report bounces with TLS / STS errors
- TLS-RPT reports show failures from known partner MTAs
- Volume of inbound mail drops suddenly after a DNS change

### Investigate

```bash
# Compare live MX to policy mx
dig +short MX <domain>
curl -fsS https://mta-sts.<domain>/.well-known/mta-sts.txt

# Confirm the policy id is current
dig +short TXT _mta-sts.<domain>

# Check TLS reachability of the MX
echo | openssl s_client -connect <mx-host>:25 -starttls smtp -servername <mx-host> 2>&1 | grep -E "subject=|issuer=|Verify return"
```

Most common cause: MX was updated to the DNSSEC-aware `p-v1.mx.microsoft`
endpoint but the policy file still has the old `mail.protection.outlook.com`
value, or vice versa.

### Contain
- Temporarily flip MTA-STS to `testing` mode while resolving -- this stops
  rejections but still collects TLS-RPT data:

```
version: STSv1
mode: testing
mx: <correct-mx>
max_age: 86400
```

- Bump the `_mta-sts` TXT `id` value so receivers re-fetch within the hour

### Recover
- Correct the `mx:` value to exactly match the live MX
- Update `id` value
- Restore `mode: enforce` after verifying with MXToolbox
- Notify affected senders that the issue is resolved

### Postmortem
- Add MX changes to the change-window checklist:
  "Update DNS MX + policy file `mx:` + `_mta-sts` `id` in the same window"
- Document the timeline in changelog

---

## 4. DKIM Key Compromise

### Detect
- Private key file leaked (laptop seizure, repo accident, vendor breach)
- Outbound spam with valid DKIM signature from your domain that you did not send

### Contain (immediate)

```powershell
Connect-ExchangeOnline

# Rotate to the inactive selector immediately
Rotate-DkimSigningConfig -KeySize 2048 -Identity <domain>

# Verify new selector is active
Get-DkimSigningConfig -Identity <domain> | Format-List Selector*, RotateOnDate
```

This generates a new key for the inactive selector and switches signing.
The previously-compromised selector's key is invalidated.

### Investigate
- How was the key exposed? Was it the M365-managed key (M365 holds it) or
  a custom DKIM setup?
- For M365-managed keys, exposure means the entire signing service was
  compromised -- escalate to Microsoft support immediately
- Search for spoofed messages using the old selector:

```powershell
Get-MessageTrace -StartDate (Get-Date).AddDays(-7) -EndDate (Get-Date) |
  Where-Object { $_.Subject -match "<known-phish-pattern>" }
```

### Recover
- Rotate again after 24 hours so both selectors have fresh keys
- Update changelog
- If a downstream system (third-party sender) used the key directly, rotate
  there too

### Postmortem
- Schedule routine DKIM rotation every 6-12 months as standard
- Never store DKIM private keys in code, repos, or password managers

---

## 5. DNSSEC Validation Failing

### Detect
- Verisign DNSSEC Analyzer shows red for the chain
- `Get-DnssecStatusForVerifiedDomain` returns `Failed`
- Mail delivery failing only from DNSSEC-validating resolvers

### Investigate

```bash
# Check DS record at parent zone
dig +short DS <domain>

# Check DNSKEY at the domain
dig +short DNSKEY <domain>

# Trace the full chain
dig +trace +dnssec <domain>

# Verisign analyzer
# https://dnssec-analyzer.verisignlabs.com/<domain>
```

Common causes:
- DS record removed at registrar (most common -- happens on registrar
  transfers, accidental edits)
- Key roll at the DNS provider not synced with DS at the registrar
- Algorithm mismatch after a provider migration

### Contain

Do **not** disable DNSSEC at the DNS provider as a quick fix -- the parent
zone still has the DS record, and unsigned answers will fail validation
worse than partially-signed ones.

Correct path:

1. Identify which side is wrong (provider DNSKEY vs registrar DS)
2. Update the DS record at the registrar to match current DNSKEY values
3. Wait for DS TTL to expire (commonly 24 hours but can be hours)
4. Re-validate

If you must disable urgently, **remove the DS record at the registrar
first**, wait for TTL, then disable signing at the DNS provider.

### Recover
- Re-validate via Verisign analyzer all-green
- Re-run `Get-DnssecStatusForVerifiedDomain` -- expect `Enabled`
- Re-validate MX delivery from a DNSSEC-validating resolver

### Postmortem
- Add registrar DS record to monitoring (use a DNSSEC chain monitor)
- Document key roll procedure if provider does manual key rolls

---

## 6. Mass Phishing Campaign Targeting the Tenant

### Detect
- Multiple users report the same phish
- Defender Threat Explorer shows campaign clustering
- User-submitted phish queue spikes

### Contain

```powershell
# Search for the campaign
$subject = "<phish-subject-pattern>"
$sender = "<sender-address-or-pattern>"

# Find all instances
Get-MessageTrace -SenderAddress $sender -StartDate (Get-Date).AddHours(-24) -EndDate (Get-Date) |
  Format-Table Received, RecipientAddress, Subject

# Purge from mailboxes (requires Defender for O365 P2 or Search-Mailbox)
# Compliance Center -> Content search -> Purge
```

Or use Defender Threat Explorer:
- Explorer -> All email -> filter by sender / subject
- Actions -> Soft delete from mailboxes
- Submission -> add as user reported phish for ZAP training

Then block at the boundary:

```powershell
# Add sender to Tenant Allow/Block List
New-TenantAllowBlockListItems -ListType Sender -Block -Entries "phisher@example.com" -NoExpiration

# Add URL
New-TenantAllowBlockListItems -ListType Url -Block -Entries "phish-domain.example" -NoExpiration

# Add file hash if attachment-based
New-TenantAllowBlockListItems -ListType FileHash -Block -Entries "<sha256>" -NoExpiration
```

### Investigate
- Did any users click? Use URL click tracking in Defender
- Did any credentials submit? Pivot to Compromised Mailbox playbook (1)
- Was it an impersonation attempt? Update anti-phish impersonation list

### Recover
- Soft-deleted messages can be restored if accidentally over-purged
- Notify affected users
- If credentials were entered: force password reset, revoke sessions, audit

### Postmortem
- Add IOCs (sender, URLs, hashes) to tenant block list with no expiration
- Tune Defender policies if the campaign got past existing rules
- User awareness comms

---

## 7. Legacy Auth Attempt Against Blocked Account

### Detect
- CA "Block Legacy Authentication" policy log entries spike
- Specific user account targeted

### Investigate

```powershell
Connect-MgGraph -Scopes "AuditLog.Read.All"

# Find legacy auth attempts in last 7 days
Get-MgAuditLogSignIn -Filter "clientAppUsed eq 'Other clients' and userPrincipalName eq 'user@<domain>'" -Top 100 |
  Format-Table CreatedDateTime, IpAddress, Location, Status, ErrorCode
```

If volume from a single IP, likely automated credential stuffing. If
distributed, likely password spray.

### Contain
- Verify CA is blocking (Status: Failure, code 53003 or similar)
- If the user's password might already have been guessed, force reset
- Submit the source IP for blocking at the network edge if you have one

### Recover
- No mailbox-side action needed if CA blocked everything
- User informed only if their account was the target

### Postmortem
- Verify the user had MFA enrolled (the password was the only thing standing
  between an attacker and the account, if MFA were missed)
- Tune CA policies if needed

---

## Communication Templates

Keep prewritten templates in a separate runbook for:
- User notification of mailbox compromise
- Recipient notification of attacker-sent mail
- Internal incident summary
- Customer / partner notification (if regulated data involved)

---

## Logging the Incident

Every incident playbook ends with a changelog entry in this format:

```
## YYYY-MM-DD -- Incident: <short title>

- Detected: <how>
- Scope: <users/domains/data affected>
- Actions: <containment + recovery steps>
- Root cause: <known or pending>
- Followups: <tickets / changes still needed>
```

---

## References

- [Microsoft Incident Response Playbooks](https://learn.microsoft.com/security/operations/incident-response-playbooks)
- [Defender for Office 365 Threat Explorer](https://learn.microsoft.com/defender-office-365/threat-explorer-about)
- [Entra ID Risky Sign-Ins](https://learn.microsoft.com/entra/id-protection/concept-identity-protection-risks)
