# MTA-STS Deployment — domain-1.io

MTA-STS configuration and validation record for the primary domain.

---

## Policy File

Hosted at: `https://mta-sts.domain-1.io/.well-known/mta-sts.txt`

```
version: STSv1
mode: enforce
mx: domain-1-io.p-v1.mx.microsoft
max_age: 604800
```

---

## DNS Records

```
mta-sts.domain-1.io        CNAME    blackvectra.github.io      (DNS-only)
_mta-sts.domain-1.io       TXT      "v=STSv1; id=20260416"
_smtp._tls.domain-1.io     TXT      "v=TLSRPTv1; rua=mailto:tlsrpt@domain-1.io"
```

---

## GitHub Pages Configuration

- Repository: `nextlayersec-mta-sts`
- Branch: `main`
- Custom domain: `mta-sts.domain-1.io`
- `.nojekyll` file: present
- Cloudflare proxy on CNAME: disabled (DNS-only)

---

## Validation

| Check | Tool | Result | Date |
|---|---|---|---|
| MTA-STS policy fetch | MXToolbox | Pass | 2026-04-18 |
| TLS certificate validation | MXToolbox | Pass | 2026-04-18 |
| MX match in policy | Manual | Pass | 2026-04-18 |
| TLS-RPT reporting | Inbox check | Receiving | 2026-04-18 |

---

## Change Log

| Date | Change | Notes |
|---|---|---|
| 2026-04-16 | Initial deployment | Mode: testing |
| 2026-04-18 | Flipped to enforce | Zero TLS failures in testing period |

---

*NextLayerSec*
