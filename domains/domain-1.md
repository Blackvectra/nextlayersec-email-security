# domain-1.io -- Email Security Status

Primary domain. Full stack deployed and enforced.

---

## Current Status

| Control | Status | Value |
|---|---|---|
| MX | Active | `domain-1-io.p-v1.mx.microsoft` |
| SPF | Deployed | `v=spf1 include:spf.protection.outlook.com -all` |
| DKIM | Enabled | Selector1 + Selector2 active |
| DMARC | Enforced | `p=reject` |
| MTA-STS | Enforced | `mode: enforce`, `max_age: 604800` |
| DNSSEC | Enabled | ECDSAP256SHA256, `signedDelegation` |
| DNSSEC-aware MX | Active | `p-v1.mx.microsoft` endpoint |
| TLS-RPT | Configured | `rua=mailto:tlsrpt@domain-1.io` |

---

## Deployment Notes

### MTA-STS
- Hosted via GitHub Pages (dedicated MTA-STS repository)
- Policy URL: `https://mta-sts.domain-1.io/.well-known/mta-sts.txt`
- CNAME: `mta-sts.domain-1.io` -> `<github-username>.github.io` (DNS-only)
- Validated: MXToolbox MTA-STS Lookup -- all green
- Deployed: 2026-04-18

### DNSSEC
- Enabled in Cloudflare zone
- DS record: published via registrar
- Chain validated: Verisign DNSSEC Analyzer -- all green
- M365 enabled via: `Enable-DnssecForVerifiedDomain -DomainName domain-1.io`
- MX updated to DNSSEC-aware endpoint: `domain-1-io.p-v1.mx.microsoft`
- Deployed: 2026-04-18

---

## Validation Commands

```powershell
# MTA-STS policy file
curl https://mta-sts.domain-1.io/.well-known/mta-sts.txt

# DNS records
nslookup -type=TXT _mta-sts.domain-1.io
nslookup -type=TXT _dmarc.domain-1.io
nslookup -type=MX domain-1.io

# DNSSEC
Get-DnssecStatusForVerifiedDomain -DomainName domain-1.io | Format-List *
```

---

## Validation Results

- MXToolbox MTA-STS: PASS (2026-04-18)
- Verisign DNSSEC Analyzer: All green (2026-04-18)
- Exchange Online DNSSEC: Enabled (2026-04-18)
- curl policy file: Returns correct content (2026-04-18)
