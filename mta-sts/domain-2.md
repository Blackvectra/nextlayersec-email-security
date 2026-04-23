# MTA-STS Deployment — domain-2.dev

MTA-STS configuration and validation record for the secondary domain.

---

## Policy File

Hosted at: `https://mta-sts.domain-2.dev/.well-known/mta-sts.txt`

```
version: STSv1
mode: enforce
mx: domain-2-dev.p-v1.mx.microsoft
max_age: 604800
```

---

## DNS Records

```
mta-sts.domain-2.dev        CNAME    blackvectra.github.io      (DNS-only)
_mta-sts.domain-2.dev       TXT      "v=STSv1; id=20260423"
_smtp._tls.domain-2.dev     TXT      "v=TLSRPTv1; rua=mailto:tlsrpt@domain-1.io"
```

---

## GitHub Pages Configuration

- Repository: `nextlayersec-mta-sts`
- Branch: `main`
- Custom domain: `mta-sts.domain-2.dev`
- `.nojekyll` file: present
- Cloudflare proxy on CNAME: disabled (DNS-only)

---

## Validation

| Check | Tool | Result | Date |
|---|---|---|---|
| MTA-STS policy fetch | MXToolbox | Pass | 2026-04-23 |
| TLS certificate validation | MXToolbox | Pass | 2026-04-23 |
| MX match in policy | Manual | Pass | 2026-04-23 |
| TLS-RPT reporting | Inbox check | Receiving | 2026-04-23 |

---

## Change Log

| Date | Change | Notes |
|---|---|---|
| 2026-04-23 | Initial deployment | Deployed directly to enforce |

---

*NextLayerSec*
