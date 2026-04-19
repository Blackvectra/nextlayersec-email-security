# Exchange Online Hardening Runbook

Full PowerShell hardening baseline for M365 Business Premium tenants.
Validated on nextlayersec.io tenant -- April 2026.

---

## Prerequisites

```powershell
# Connect to Exchange Online
Connect-ExchangeOnline -UserPrincipalName admin@yourdomain.com

# Verify connection
Get-OrganizationConfig | Select DisplayName
```

---

## 1. DNSSEC

```powershell
# Check current status
Get-DnssecStatusForVerifiedDomain -DomainName yourdomain.com

# Enable -- note MxValue in output, update DNS immediately after
Enable-DnssecForVerifiedDomain -DomainName yourdomain.com

# Verify full output
Get-DnssecStatusForVerifiedDomain -DomainName yourdomain.com | Format-List *
```

After enabling, update in DNS and mta-sts.txt:
- MX record -> new `p-v1.mx.microsoft` endpoint from MxValue output
- `mta-sts.txt` mx: value -> new endpoint
- `_mta-sts` TXT id value -> bump to current date

---

## 2. Legacy Authentication

```powershell
# Block all legacy auth protocols (use colon syntax for $false)
Set-AuthenticationPolicy -Identity "YourPolicyName" `
  -AllowBasicAuthActiveSync:$false `
  -AllowBasicAuthAutodiscover:$false `
  -AllowBasicAuthImap:$false `
  -AllowBasicAuthMapi:$false `
  -AllowBasicAuthOfflineAddressBook:$false `
  -AllowBasicAuthOutlookService:$false `
  -AllowBasicAuthPop:$false `
  -AllowBasicAuthReportingWebServices:$false `
  -AllowBasicAuthRpc:$false `
  -AllowBasicAuthWebServices:$false `
  -AllowBasicAuthPowershell:$false

# Set as org default for new users
Set-OrganizationConfig -DefaultAuthenticationPolicy "YourPolicyName"

# Assign to all users
Get-User -ResultSize Unlimited | Set-User -AuthenticationPolicy "YourPolicyName"

# Exclude break glass account
Set-User -Identity breakglass@yourdomain.com -AuthenticationPolicy $null

# Verify policy config
Get-AuthenticationPolicy -Identity "YourPolicyName" | Format-List Name, AllowBasicAuth*

# Verify user assignments
Get-User -ResultSize Unlimited | Format-List DisplayName, AuthenticationPolicy, UserPrincipalName
```

Note: If backtick line continuation fails, run as single line with colon syntax on $false.

---

## 3. SMTP Client Authentication

```powershell
# Disable legacy SMTP relay tenant-wide
Set-TransportConfig -SmtpClientAuthenticationDisabled $true

# Verify
Get-TransportConfig | Format-List SmtpClientAuthenticationDisabled
```

---

## 4. External Mail Forwarding

```powershell
# Disable auto-forward
Set-RemoteDomain Default -AutoForwardEnabled $false

# Verify
Get-RemoteDomain Default | Format-List AutoForwardEnabled

# Check for existing forwarding rules on all mailboxes
Get-InboxRule -Mailbox user@yourdomain.com | Where-Object {
    $_.ForwardTo -ne $null -or $_.RedirectTo -ne $null
} | Format-List Name, ForwardTo, RedirectTo

# Check mailbox-level forwarding
Get-Mailbox -ResultSize Unlimited | Where-Object {
    $_.ForwardingAddress -ne $null -or $_.ForwardingSmtpAddress -ne $null
} | Format-List DisplayName, ForwardingAddress, ForwardingSmtpAddress
```

---

## 5. Mailbox Protocol Hardening

```powershell
# Disable POP, IMAP, ActiveSync at mailbox level (belt and suspenders on top of auth policy)
Get-CasMailbox -ResultSize Unlimited | Set-CasMailbox `
  -PopEnabled $false `
  -ImapEnabled $false `
  -ActiveSyncEnabled $false

# Verify
Get-CasMailbox -Identity user@yourdomain.com | Format-List `
  PopEnabled, ImapEnabled, OWAEnabled, ActiveSyncEnabled, MAPIEnabled
```

---

## 6. Mailbox Auditing

```powershell
# Enable auditing and extend retention to 180 days
Set-Mailbox -Identity user@yourdomain.com -AuditEnabled $true -AuditLogAgeLimit 180

# Enable all audit actions
Set-Mailbox -Identity user@yourdomain.com `
  -AuditOwner MailboxLogin,HardDelete,SoftDelete,Update,Move,MoveToDeletedItems,UpdateFolderPermissions,UpdateCalendarDelegation,UpdateInboxRules `
  -AuditDelegate HardDelete,SoftDelete,Update,Move,MoveToDeletedItems,SendAs,SendOnBehalf,Create `
  -AuditAdmin HardDelete,MoveToDeletedItems,SendAs,SendOnBehalf,Create

# Verify unified audit logging
Get-AdminAuditLogConfig | Format-List UnifiedAuditLogIngestionEnabled

# Verify mailbox audit config
Get-Mailbox -Identity user@yourdomain.com | Format-List AuditEnabled, AuditLogAgeLimit
```

---

## 7. Outbound Spam Protection

```powershell
# Configure notification for compromised account detection
Set-HostedOutboundSpamFilterPolicy -Identity Default `
  -NotifyOutboundSpam $true `
  -NotifyOutboundSpamRecipients admin@yourdomain.com

# Verify
Get-HostedOutboundSpamFilterPolicy | Format-List `
  Name, ActionWhenThresholdReached, NotifyOutboundSpam, NotifyOutboundSpamRecipients
```

---

## 8. OWA Policy

```powershell
# Disable unnecessary features
Set-OwaMailboxPolicy -Identity OwaMailboxPolicy-Default -TextMessagingEnabled $false

# Verify
Get-OwaMailboxPolicy | Format-List Name, InstantMessagingEnabled, TextMessagingEnabled, LinkedInEnabled
```

---

## 9. Verification Checks

```powershell
# DKIM signing per domain
Get-DkimSigningConfig -Identity yourdomain.com

# All accepted domains in tenant
Get-AcceptedDomain

# Anti-phishing policies
Get-AntiPhishPolicy | Format-List Name, Enabled, EnableMailboxIntelligence, EnableSpoofIntelligence

# Safe Attachments
Get-SafeAttachmentPolicy | Format-List Name, Enable, Action

# Safe Links
Get-SafeLinksPolicy | Format-List Name, Enabled, ScanUrls, EnableForInternalSenders

# ATP policy
Get-AtpPolicyForO365 | Format-List Name, EnableATPForSPOTeamsODB, EnableSafeDocs, AllowSafeDocsOpen

# Malware filter + ZAP
Get-MalwareFilterPolicy | Format-List Name, EnableFileFilter, ZapEnabled
Get-HostedContentFilterPolicy | Format-List Name, SpamZapEnabled, PhishZapEnabled

# Tenant allow/block list
Get-TenantAllowBlockListItems -ListType Sender
Get-TenantAllowBlockListItems -ListType Url
Get-TenantAllowBlockListItems -ListType FileHash

# Inbound connectors
Get-InboundConnector | Format-List Name, Enabled, TlsSenderCertificateName, RequireTls
```

---

## Hardening Baseline Verification Table

| Control | Command | Expected Result |
|---|---|---|
| DNSSEC | `Get-DnssecStatusForVerifiedDomain` | `DnssecFeatureStatus: Enabled` |
| Legacy auth | `Get-AuthenticationPolicy` | All `AllowBasicAuth*: False` |
| SMTP auth | `Get-TransportConfig` | `SmtpClientAuthenticationDisabled: True` |
| Auto-forward | `Get-RemoteDomain Default` | `AutoForwardEnabled: False` |
| POP/IMAP/ActiveSync | `Get-CasMailbox` | All `False` |
| Mailbox audit | `Get-Mailbox` | `AuditEnabled: True` |
| Unified audit log | `Get-AdminAuditLogConfig` | `UnifiedAuditLogIngestionEnabled: True` |
| Outbound spam notify | `Get-HostedOutboundSpamFilterPolicy` | `NotifyOutboundSpam: True` |
| ZAP | `Get-HostedContentFilterPolicy` | `SpamZapEnabled: True, PhishZapEnabled: True` |
| Safe Attachments | `Get-SafeAttachmentPolicy` | `Action: Block` |
| ATP SharePoint/Teams | `Get-AtpPolicyForO365` | `EnableATPForSPOTeamsODB: True` |

---

## Break Glass Account Configuration

```powershell
# Remove auth policy from break glass
Set-User -Identity breakglass@yourdomain.com -AuthenticationPolicy $null

# Verify
Get-User -Identity breakglass@yourdomain.com | Format-List DisplayName, AuthenticationPolicy
```

Break glass account requirements:
- No authentication policy assigned
- Excluded from all Conditional Access policies in Entra ID
- No MFA registered
- Long complex password stored offline (not in password manager)
- Sign-in alert configured in Entra ID monitoring

---

## Known Accepted Risks

| Control | Status | Justification | Reviewed |
|---|---|---|---|
| OWA LinkedIn Integration | Enabled | Actively used for NextLayerSec business development. OAuth-based, low risk. | 2026-04-18 |

---

*NextLayerSec -- nextlayersec.io*
*Last validated: April 2026*
