# Defender for Office 365 -- Tuning

Beyond the on/off baseline in the hardening runbook, Defender for Office 365
has several policies that need tuning per-tenant. Defaults are conservative
and miss real threats; over-tuning generates false positives.

> All commands below require Exchange Online PowerShell connected as a Global
> Admin or Security Admin.

---

## Safe Attachments

```powershell
# Recommended baseline
New-SafeAttachmentPolicy -Name "Strict-Block" `
  -Action Block `
  -Enable $true `
  -Redirect $false `
  -ActionOnError $true

# Apply to all recipients in the tenant
New-SafeAttachmentRule -Name "Strict-Block-All" `
  -SafeAttachmentPolicy "Strict-Block" `
  -RecipientDomainIs "<domain>" `
  -Priority 0
```

Key settings:
- **Action: Block** -- detected malware blocks delivery entirely
- **Redirect: false** -- do not redirect to admin (just block; admin can recover from quarantine if needed)
- **ActionOnError: true** -- treat scan failure as malicious (fail closed)

Avoid **Dynamic Delivery** unless you have heavy attachment workflows --
it delivers the body immediately and the attachment later, which can confuse
users.

### Verify

```powershell
Get-SafeAttachmentPolicy | Format-List Name, Enable, Action, Redirect, ActionOnError
Get-SafeAttachmentRule | Format-List Name, State, SafeAttachmentPolicy, RecipientDomainIs, Priority
```

---

## Safe Links

```powershell
New-SafeLinksPolicy -Name "Strict-Scan" `
  -EnableSafeLinksForEmail $true `
  -EnableSafeLinksForTeams $true `
  -EnableSafeLinksForOffice $true `
  -ScanUrls $true `
  -DeliverMessageAfterScan $true `
  -DisableUrlRewrite $false `
  -EnableForInternalSenders $true `
  -TrackClicks $true `
  -AllowClickThrough $false

New-SafeLinksRule -Name "Strict-Scan-All" `
  -SafeLinksPolicy "Strict-Scan" `
  -RecipientDomainIs "<domain>" `
  -Priority 0
```

Key settings:
- **ScanUrls + DeliverMessageAfterScan: true** -- wait for scan, fail closed
- **EnableForInternalSenders: true** -- attackers pivot inside the tenant
- **AllowClickThrough: false** -- prevents user from bypassing the warning page
- **DisableUrlRewrite: false** -- URLs must be rewritten for click tracking to work

Common false positives:
- Internal apps generating one-time login URLs that Safe Links pre-clicks and consumes -- add those domains to `DoNotRewriteUrls`

```powershell
Set-SafeLinksPolicy -Identity "Strict-Scan" -DoNotRewriteUrls @("internal-app.<domain>", "vendor-portal.example.com")
```

---

## Anti-Phish (Mailbox Intelligence + Impersonation)

```powershell
New-AntiPhishPolicy -Name "Strict-AntiPhish" `
  -Enabled $true `
  -EnableMailboxIntelligence $true `
  -EnableMailboxIntelligenceProtection $true `
  -MailboxIntelligenceProtectionAction Quarantine `
  -EnableSpoofIntelligence $true `
  -AuthenticationFailAction Quarantine `
  -EnableFirstContactSafetyTips $true `
  -EnableUnauthenticatedSender $true `
  -EnableViaTag $true `
  -TargetedUsersToProtect @("CEO Name;ceo@<domain>", "CFO Name;cfo@<domain>") `
  -TargetedUsersProtectionAction Quarantine `
  -EnableTargetedUserProtection $true `
  -EnableOrganizationDomainsProtection $true `
  -TargetedDomainProtectionAction Quarantine

New-AntiPhishRule -Name "Strict-AntiPhish-All" `
  -AntiPhishPolicy "Strict-AntiPhish" `
  -RecipientDomainIs "<domain>" `
  -Priority 0
```

Key settings:
- **MailboxIntelligence + Protection** -- ML-based "this sender does not normally email this user" signal
- **TargetedUsersToProtect** -- name + email of high-value users (executives, finance, IT admins). Format is `"Display Name;email@domain"`
- **EnableOrganizationDomainsProtection** -- catches "yourdomain.co" vs "yourdomain.com" lookalikes
- **FirstContactSafetyTips** -- banner in Outlook when a sender is new to the recipient

### Add or update targeted users

```powershell
$users = @(
  "Jane Smith;jane@<domain>",
  "John Doe;john@<domain>",
  "Finance Group;finance@<domain>"
)

Set-AntiPhishPolicy -Identity "Strict-AntiPhish" -TargetedUsersToProtect $users
```

Maximum 350 protected users per policy.

---

## Anti-Spam (Hosted Content Filter)

```powershell
Set-HostedContentFilterPolicy -Identity Default `
  -BulkThreshold 6 `
  -SpamAction Quarantine `
  -HighConfidenceSpamAction Quarantine `
  -PhishSpamAction Quarantine `
  -HighConfidencePhishAction Quarantine `
  -BulkSpamAction Quarantine `
  -SpamZapEnabled $true `
  -PhishZapEnabled $true `
  -InlineSafetyTipsEnabled $true `
  -EnableRegionBlockList $true `
  -RegionBlockList @("CN", "RU", "KP", "IR") `
  -EnableLanguageBlockList $false
```

Key settings:
- **BulkThreshold: 6** -- middle of the road; 4 is aggressive, 9 is lenient
- **All actions: Quarantine** -- nothing goes to junk folder, all goes to a reviewable quarantine
- **ZAP enabled** -- retroactively moves messages out of inbox if later classified as malicious
- **RegionBlockList** -- block by sender country IP geo. Adjust for your business reality

> Quarantine vs Junk: prefer Quarantine for everything. Junk is invisible to
> admins and harder to recover from. Quarantine has a 30-day retention and a
> review queue.

---

## Outbound Spam

```powershell
Set-HostedOutboundSpamFilterPolicy -Identity Default `
  -NotifyOutboundSpam $true `
  -NotifyOutboundSpamRecipients "secops@<domain>" `
  -RecipientLimitExternalPerHour 100 `
  -RecipientLimitInternalPerHour 200 `
  -RecipientLimitPerDay 500 `
  -ActionWhenThresholdReached BlockUser
```

Outbound limits should match real usage. **BlockUser** when limits exceeded
contains the blast radius of a compromised mailbox sending spam.

---

## Quarantine Policy

By default, end users can self-release some quarantine items. For strict
environments, force admin review:

```powershell
# Restrict user release for high-confidence phish
Set-QuarantinePolicy -Identity DefaultFullAccessPolicy `
  -EndUserQuarantinePermissionsValue 0
```

Quarantine notifications to users:

```powershell
Set-QuarantinePolicy -Identity DefaultFullAccessPolicy `
  -EndUserQuarantinePermissionsValue 13 `
  -ESNEnabled $true
```

`13` = preview + release request + delete. Users can request release; admin
approves.

---

## Tuning Loop

1. Set policies to **recommended baseline** above
2. Monitor for 7 days via the **Threat Explorer** dashboard
3. Identify false positives -- review with affected users
4. Adjust by domain / sender exception, not by lowering the policy
5. Re-validate every quarter

False positives are usually handled by:
- Adding the sender to `Allowed senders` in the anti-phish policy
- Adding a tenant allow entry (see [`tenant-allow-block.md`](tenant-allow-block.md))
- Adjusting `DoNotRewriteUrls` for legitimate internal services

---

## Validation

```powershell
# All policies
Get-AntiPhishPolicy | Format-List Name, Enabled
Get-SafeAttachmentPolicy | Format-List Name, Enable, Action
Get-SafeLinksPolicy | Format-List Name, EnableSafeLinksForEmail, AllowClickThrough
Get-HostedContentFilterPolicy | Format-List Name, BulkThreshold, *Action, *ZapEnabled

# All rules and priorities
Get-AntiPhishRule | Format-Table Name, Priority, State, AntiPhishPolicy
Get-SafeAttachmentRule | Format-Table Name, Priority, State
Get-SafeLinksRule | Format-Table Name, Priority, State
```

---

## References

- [Defender for Office 365 recommended settings](https://learn.microsoft.com/defender-office-365/recommended-settings-for-eop-and-office365)
- [Configuration analyzer](https://learn.microsoft.com/defender-office-365/configuration-analyzer-for-security-policies)
