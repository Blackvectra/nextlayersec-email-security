# DKIM Selector Management

Configuration and management reference for DKIM selectors across all domains
in the NextLayerSec M365 tenant.

---

## Overview

DKIM selectors allow multiple DKIM keys to exist for a single domain simultaneously.
Each selector is a unique identifier that points to a specific public key in DNS.
M365 uses two selectors by default: `selector1` and `selector2`.

Only one selector is active at a time. Key rotation switches between them.

---

## M365 Default Selectors

| Selector | DNS Record | Status |
|---|---|---|
| `selector1` | `selector1._domainkey.<domain>` | Active or inactive depending on rotation state |
| `selector2` | `selector2._domainkey.<domain>` | Active or inactive depending on rotation state |

---

## Checking Active Selector

```powershell
# Connect to Exchange Online
Connect-ExchangeOnline -UserPrincipalName admin@yourdomain.com

# Check DKIM signing config for a domain
Get-DkimSigningConfig -Identity yourdomain.com | Format-List *

# Key fields to review
# Enabled: True/False
# Selector1KeySize, Selector2KeySize
# Selector1PublicKey, Selector2PublicKey
# RotateOnDate -- when next rotation is scheduled
# SelectorBeforeRotateOnDate -- which selector is currently active
```

---

## Enabling DKIM Signing

```powershell
# Enable DKIM signing for a domain
Enable-DkimSigningConfig -Identity yourdomain.com

# Verify
Get-DkimSigningConfig -Identity yourdomain.com | Format-List Enabled, Selector1KeySize, Selector2KeySize
```

> M365 generates the DKIM keys automatically.
> Two CNAME records must be published in DNS before enabling.
> M365 provides the exact CNAME values in the Microsoft 365 Defender portal under Email Authentication Settings.

---

## Required DNS Records

```
selector1._domainkey.<domain>    CNAME    selector1-<domain-onmicrosoft>._domainkey.microsoft.com
selector2._domainkey.<domain>    CNAME    selector2-<domain-onmicrosoft>._domainkey.microsoft.com
```

> Replace `<domain-onmicrosoft>` with your tenant's onmicrosoft.com subdomain with hyphens replacing dots.
> Example: `contoso-com._domainkey.microsoft.com`

---

## Selector Validation

```powershell
# Check DNS publication of selector
nslookup -type=TXT selector1._domainkey.yourdomain.com
nslookup -type=TXT selector2._domainkey.yourdomain.com
```

Expected output contains: `v=DKIM1; k=rsa; p=<public key>`

---

## Domain Coverage

| Domain | DKIM Enabled | Active Selector | Key Size |
|---|:---:|:---:|:---:|
| `domain-1.io` | Yes | selector1 | 2048 |
| `domain-2.dev` | Yes | selector1 | 2048 |
| `domain-3.com` | Yes | selector1 | 2048 |

---

*NextLayerSec*
*Last validated: April 2026*
