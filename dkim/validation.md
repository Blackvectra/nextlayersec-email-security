# DKIM Validation

Validation procedures and current status for DKIM signing across all domains
in the NextLayerSec M365 tenant.

---

## Validation Methods

### PowerShell — Tenant Side

```powershell
# Connect to Exchange Online
Connect-ExchangeOnline -UserPrincipalName admin@yourdomain.com

# Full DKIM config for a domain
Get-DkimSigningConfig -Identity yourdomain.com | Format-List *

# Quick status check across all domains
Get-DkimSigningConfig | Format-Table Domain, Enabled, Status
```

### DNS — Public Key Lookup

```powershell
# Check selector1 public key
nslookup -type=TXT selector1._domainkey.yourdomain.com

# Check selector2 public key
nslookup -type=TXT selector2._domainkey.yourdomain.com
```

Expected output: `v=DKIM1; k=rsa; p=<base64 public key>`

### Message Header Inspection

Send a test message to a Gmail or Outlook.com address.
In Gmail: open message -> three dots -> Show original.
Look for:

```
Authentication-Results: mx.google.com;
  dkim=pass header.i=@yourdomain.com header.s=selector1
```

`dkim=pass` confirms signing is working end to end.

---

## External Validation Tools

| Tool | Purpose | URL |
|---|---|---|
| MXToolbox DKIM | Public key lookup and validation | mxtoolbox.com/dkim.aspx |
| Mail-tester | Full authentication check on sent message | mail-tester.com |
| Google Admin Toolbox | Check DKIM record syntax | toolbox.googleapps.com |

---

## Current Validation Status

| Domain | DNS Records Published | Signing Enabled | Last Validated |
|---|:---:|:---:|:---:|
| `domain-1.io` | Yes | Yes | 2026-04-18 |
| `domain-2.dev` | Yes | Yes | 2026-04-18 |
| `domain-3.com` | Yes | Yes | 2026-04-23 |

---

## Common Failure Modes

| Symptom | Likely Cause | Fix |
|---|---|---|
| `dkim=fail` in headers | CNAME not published or wrong value | Verify DNS CNAME matches M365 portal value |
| `dkim=none` in headers | DKIM signing not enabled | Run `Enable-DkimSigningConfig` |
| `dkim=permerror` | DNS record syntax error | Check CNAME target for typos |
| DKIM pass but DMARC fail | Alignment mismatch | Verify `d=` in DKIM signature matches header From domain |

---

*NextLayerSec*
*Last validated: April 2026*
