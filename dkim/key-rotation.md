# DKIM Key Rotation

Key rotation procedure for DKIM selectors across all domains
in the NextLayerSec M365 tenant.

---

## Overview

DKIM key rotation switches the active signing selector from the current one
to the alternate. M365 supports manual and scheduled rotation between
`selector1` and `selector2`.

Rotating keys limits the exposure window if a private key is ever compromised.
Recommended rotation cadence: every 6-12 months minimum.

---

## How M365 Key Rotation Works

M365 maintains two selectors per domain. At any point one is active and one is
inactive. Rotation:

1. Generates a new key pair for the inactive selector
2. Updates the inactive selector's DNS CNAME to point to the new public key
3. Switches signing to the newly generated selector
4. The previously active selector becomes inactive

Both selectors remain published in DNS. Receiving servers that cached the old
selector can still validate messages signed before rotation.

---

## Manual Rotation

```powershell
# Connect to Exchange Online
Connect-ExchangeOnline -UserPrincipalName admin@yourdomain.com

# Rotate DKIM key for a domain
Rotate-DkimSigningConfig -KeySize 2048 -Identity yourdomain.com

# Verify rotation
Get-DkimSigningConfig -Identity yourdomain.com | Format-List `
  Enabled, `
  SelectorBeforeRotateOnDate, `
  RotateOnDate, `
  Selector1KeySize, `
  Selector2KeySize
```

---

## Scheduled Rotation

M365 does not enforce automatic rotation on a defined cadence.
Manual rotation should be performed on a documented schedule.

Recommended process:
1. Note current active selector before rotation
2. Run `Rotate-DkimSigningConfig`
3. Verify new selector is active via `Get-DkimSigningConfig`
4. Validate DNS propagation via nslookup
5. Send a test message and verify DKIM pass in message headers
6. Document rotation date in changelog

---

## Verifying Rotation in Message Headers

After rotation send a test message and inspect headers.
Look for:

```
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed; d=yourdomain.com;
  s=selector2; ...
```

The `s=` field confirms which selector signed the message.
Verify it matches the selector you rotated to.

---

## Rotation Log

| Domain | Date | Previous Selector | New Selector | Performed By |
|---|---|---|---|---|
| domain-1.io | 2026-04-18 | selector2 | selector1 | Initial deployment |
| domain-2.dev | 2026-04-18 | selector2 | selector1 | Initial deployment |
| domain-3.com | 2026-04-23 | selector2 | selector1 | Initial deployment |

---

*NextLayerSec*
*Last validated: April 2026*
