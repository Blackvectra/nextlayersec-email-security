# Domain Security Template

Use this template when onboarding a new domain to the NextLayerSec email security framework.
Complete each section in order. Do not advance to enforcement until prerequisites are validated.

---

## Domain Information

| Field | Value |
|---|---|
| Domain | |
| Registrar | |
| DNS Provider | |
| Mail Platform | |
| Tenant | |
| Onboarded | |
| Last Reviewed | |

---

## Pre-Deployment Checklist

- [ ] Domain added to M365 tenant as accepted domain
- [ ] MX record pointing to M365 endpoint
- [ ] Autodiscover configured
- [ ] Domain verified in Microsoft 365 admin center

---

## SPF

```
v=spf1 include:spf.protection.outlook.com -all
```

- [ ] SPF record published in DNS
- [ ] Validated via MXToolbox
- [ ] No duplicate SPF records
- [ ] Under 10 DNS lookup limit

---

## DKIM

```
selector1._domainkey.<domain>    CNAME    selector1-<tenant>._domainkey.microsoft.com
selector2._domainkey.<domain>    CNAME    selector2-<tenant>._domainkey.microsoft.com
```

- [ ] CNAME records published in DNS
- [ ] DKIM signing enabled via `Enable-DkimSigningConfig`
- [ ] Validated via `Get-DkimSigningConfig`
- [ ] DKIM pass confirmed in message headers

---

## DMARC

```
v=DMARC1; p=none; rua=mailto:dmarc@domain-1.io
```

- [ ] DMARC record published at p=none
- [ ] RUA reporting address confirmed receiving reports
- [ ] Aggregate reports reviewed for 2-4 weeks
- [ ] No legitimate sources failing authentication
- [ ] Advanced to p=quarantine
- [ ] Advanced to p=reject

---

## MTA-STS

```
version: STSv1
mode: enforce
mx: <domain-endpoint>.p-v1.mx.microsoft
max_age: 604800
```

- [ ] Policy file hosted at `https://mta-sts.<domain>/.well-known/mta-sts.txt`
- [ ] `_mta-sts` TXT record published
- [ ] `_smtp._tls` TXT record published for TLS-RPT
- [ ] Cloudflare proxy disabled on mta-sts CNAME
- [ ] `.nojekyll` file present in GitHub Pages repo
- [ ] Validated via MXToolbox MTA-STS checker
- [ ] Mode set to `enforce`

---

## DNSSEC

- [ ] DNSSEC enabled via registrar or Cloudflare
- [ ] `Enable-DnssecForVerifiedDomain` run in Exchange Online
- [ ] MX record updated to `p-v1.mx.microsoft` endpoint
- [ ] `mta-sts.txt` mx: value updated to new endpoint
- [ ] Validated via Verisign DNSSEC Analyzer
- [ ] Validated via `Get-DnssecStatusForVerifiedDomain`

---

## CAA Records

```
<domain>    CAA    0 issue "pki.goog"
<domain>    CAA    0 issue "letsencrypt.org"
<domain>    CAA    0 issue "digicert.com"
<domain>    CAA    0 issuewild "pki.goog"
<domain>    CAA    0 issuewild "letsencrypt.org"
<domain>    CAA    0 issuewild "digicert.com"
<domain>    CAA    0 iodef "mailto:admin@domain-1.io"
```

- [ ] CAA records published in DNS
- [ ] Validated via MXToolbox CAA checker

---

## Final Validation

| Control | Status | Validated |
|---|:---:|:---:|
| SPF | | |
| DKIM | | |
| DMARC | | |
| MTA-STS | | |
| DNSSEC | | |
| DNSSEC-aware MX | | |
| TLS-RPT | | |
| CAA | | |

---

## Notes

---

*NextLayerSec Domain Security Template*
