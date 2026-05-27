# Tenant Allow / Block List

The Tenant Allow/Block List (TABL) is the manually-curated overlay on top of
Defender's automated classification. Use it for known-bad indicators
(phishing campaigns, malware hashes) and known-good exceptions (verified
senders being false-positived).

> Every entry in this list is technical debt -- it bypasses or overrides
> automated decisions. Review quarterly and remove anything no longer
> needed.

---

## When to Use TABL

### Block
- IOCs from a confirmed phishing campaign (sender, URL, hash)
- Lookalike domain spoofing your brand
- Known commodity malware hash that Defender did not catch

### Allow
- Vendor sending alerts that get flagged as phish (rare -- prefer DKIM/SPF fix)
- Internal test infrastructure sending mail that triggers detections
- Bulk sender (newsletter) below the bulk threshold but flagged anyway

### Do not use TABL for
- Filtering personal preferences (use user-level junk filter)
- Whitelisting an entire domain you do not control (broadens trust)
- Permanent fixes for things that should be handled by SPF / DKIM / DMARC

---

## Adding Entries

### Block sender

```powershell
Connect-ExchangeOnline

New-TenantAllowBlockListItems -ListType Sender `
  -Block `
  -Entries "phisher@example.com", "*@evil-domain.example" `
  -NoExpiration `
  -Notes "Confirmed phishing campaign 2026-05-27 -- IR ticket 1234"
```

### Block URL

```powershell
New-TenantAllowBlockListItems -ListType Url `
  -Block `
  -Entries "phish-domain.example", "evil-cdn.example/path*" `
  -NoExpiration `
  -Notes "Click destination from phish campaign 2026-05-27"
```

### Block file hash (SHA256)

```powershell
New-TenantAllowBlockListItems -ListType FileHash `
  -Block `
  -Entries "a1b2c3d4..." `
  -NoExpiration `
  -Notes "Malware sample from IR ticket 1234"
```

### Allow sender (with expiration -- recommended)

```powershell
New-TenantAllowBlockListItems -ListType Sender `
  -Allow `
  -Entries "alerts@vendor.example" `
  -ExpirationDate (Get-Date).AddDays(30) `
  -Notes "Vendor alerts being false-positived; review by 2026-06-26"
```

> Allow entries should default to 30-day expiration. Force quarterly review
> rather than permanent exceptions.

---

## Listing Current Entries

```powershell
# By list type
Get-TenantAllowBlockListItems -ListType Sender | Format-Table Value, Action, Notes, ExpirationDate
Get-TenantAllowBlockListItems -ListType Url | Format-Table Value, Action, Notes, ExpirationDate
Get-TenantAllowBlockListItems -ListType FileHash | Format-Table Value, Action, Notes, ExpirationDate

# Items expiring soon
Get-TenantAllowBlockListItems -ListType Sender |
  Where-Object { $_.ExpirationDate -lt (Get-Date).AddDays(7) -and $_.ExpirationDate -gt (Get-Date) } |
  Format-Table Value, Action, ExpirationDate
```

---

## Removing Entries

```powershell
# By value
Remove-TenantAllowBlockListItems -ListType Sender -Entries "old-entry@example.com"

# By IDs (when an entry has multiple matches)
$ids = (Get-TenantAllowBlockListItems -ListType Sender -Entries "old@example.com").Identity
Remove-TenantAllowBlockListItems -ListType Sender -Ids $ids
```

---

## Spoofed Senders

Spoofed senders are managed separately from the main TABL:

```powershell
# View detected spoofs
Get-TenantAllowBlockListSpoofItems | Format-Table SpoofedUser, SendingInfrastructure, Action

# Allow a legitimate spoof (e.g. a service that sends on your behalf)
New-TenantAllowBlockListSpoofItems -Identity "user@<domain>" `
  -Action Allow `
  -SpoofType External `
  -SendingInfrastructure "mailer.vendor.example"
```

---

## Limits

| List | Max entries | Max entry length |
|---|---|---|
| Sender | 500 | 100 chars |
| URL | 500 | 250 chars |
| FileHash | 500 | 64 chars (SHA256) |

If you hit a limit, you have either an incident response queue you have not
cleared, or you are using TABL where Defender policy tuning should handle it.

---

## Audit and Cleanup

Run quarterly:

```powershell
# All allow entries -- justify each one
Get-TenantAllowBlockListItems -ListType Sender |
  Where-Object { $_.Action -eq "Allow" } |
  Format-Table Value, Notes, LastModifiedDateTime, ExpirationDate

# Block entries older than 1 year -- consider if IOC is still relevant
Get-TenantAllowBlockListItems -ListType Sender |
  Where-Object { $_.Action -eq "Block" -and $_.LastModifiedDateTime -lt (Get-Date).AddDays(-365) } |
  Format-Table Value, Notes, LastModifiedDateTime
```

Document the quarterly cleanup pass in the changelog.

---

## References

- [Manage the Tenant Allow/Block List](https://learn.microsoft.com/defender-office-365/tenant-allow-block-list-about)
- [Spoofed sender entries](https://learn.microsoft.com/defender-office-365/tenant-allow-block-list-email-spoof-configure)
