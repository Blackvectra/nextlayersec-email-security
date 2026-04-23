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
