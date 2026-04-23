# Cloudflare DNS Records Reference

Complete DNS record reference for all domains managed through Cloudflare.
All records are DNS-only (grey cloud) unless noted.

---

## Record Structure Per Domain

Each domain requires the following records for full email security stack enforcement.

### SPF

```
Type: TXT
Name: @
Value: v=spf1 include:spf.protection.outlook.com -all
TTL: Auto
```

### DKIM

```
Type: CNAME
Name: selector1._domainkey
Value: selector1-<tenant>._domainkey.microsoft.com
Proxy: DNS-only

Type: CNAME
Name: selector2._domainkey
Value: selector2-<tenant>._domainkey.microsoft.com
Proxy: DNS-only
```

### DMARC

```
Type: TXT
Name: _dmarc
Value: v=DMARC1; p=reject; rua=mailto:dmarc@domain-1.io
TTL: Auto
```

### MX

```
Type: MX
Name: @
Value: <domain-endpoint>.p-v1.mx.microsoft
Priority: 0
TTL: Auto
```

> Use the DNSSEC-aware p-v1.mx.microsoft endpoint.
> Standard mail.protection.outlook.com endpoint does not support DNSSEC validation.

### MTA-STS

```
Type: CNAME
Name: mta-sts
Value: <github-username>.github.io
Proxy: DNS-only

Type: TXT
Name: _mta-sts
Value: v=STSv1; id=YYYYMMDD
TTL: Auto
```

> Cloudflare proxy must be disabled on the mta-sts CNAME.
> GitHub Pages handles TLS termination — proxying breaks certificate provisioning.

### TLS-RPT

```
Type: TXT
Name: _smtp._tls
Value: v=TLSRPTv1; rua=mailto:tlsrpt@domain-1.io
TTL: Auto
```

### CAA

```
Type: CAA
Name: @
Value: 0 issue "pki.goog"

Type: CAA
Name: @
Value: 0 issue "letsencrypt.org"

Type: CAA
Name: @
Value: 0 issue "digicert.com"

Type: CAA
Name: @
Value: 0 issuewild "pki.goog"

Type: CAA
Name: @
Value: 0 issuewild "letsencrypt.org"

Type: CAA
Name: @
Value: 0 issuewild "digicert.com"

Type: CAA
Name: @
Value: 0 iodef "mailto:admin@domain-1.io"
```

---

## Cloudflare-Specific Notes

**Proxy status:**
- All email-related records must be DNS-only (grey cloud)
- Proxying mail records breaks DKIM alignment and MTA-STS certificate validation
- Only web-facing A/CNAME records should use Cloudflare proxy

**DNSSEC:**
- Enable DNSSEC in Cloudflare under DNS -> Settings -> DNSSEC
- Cloudflare generates the DS record — publish it at your registrar
- After enabling, run `Enable-DnssecForVerifiedDomain` in Exchange Online

**TTL:**
- Use Auto TTL for most records
- Lower TTL to 60 seconds before making changes to MX or MTA-STS records
- Return to Auto TTL after changes propagate

---

*NextLayerSec*
*Last validated: April 2026*
