# nextlayersec.io -- Email Security Status

Primary domain. Full stack deployed and enforced.

---

## Current Status

| Control | Status | Value |
|---|---|---|
| MX | Active | `nextlayersec-io.p-v1.mx.microsoft` |
| SPF | Deployed | `v=spf1 include:spf.protection.outlook.com -all` |
| DKIM | Enabled | Selector1 + Selector2 active |
| DMARC | Enforced | `p=reject` |
| MTA-STS | Enforced | `mode: enforce`, `max_age: 604800` |
| DNSSEC | Enabled | ECDSAP256SHA256, `signedDelegation` |
| DNSSEC-aware MX | Active | `p-v1.mx.microsoft` endpoint |
| TLS-RPT | Configured | `rua=mailto:tlsrpt@nextlayersec.io` |

---

## Deployment Notes

### MTA-STS
- Hosted via GitHub Pages (`nextlayersec-mta-sts` repo)
- Policy URL: `https://mta-sts.nextlayersec.io/.well-known/mta-sts.txt`
- CNAME: `mta-sts.nextlayersec.io` -> `blackvectra.github.io` (DNS-only)
- Validated: MXToolbox MTA-STS Lookup -- all green
- Deployed: 2026-04-18

### DNSSEC
- Enabled in Cloudflare zone
- DS record: published via GoDaddy registrar
- Chain validated: Verisign DNSSEC Analyzer -- all green
- M365 enabled via: `Enable-DnssecForVerifiedDomain -DomainName nextlayersec.io`
- MX updated to DNSSEC-aware endpoint: `nextlayersec-io.p-v1.mx.microsoft`
- Deployed: 2026-04-18

---

## Validation Commands

```powershell
# MTA-STS policy file
curl https://mta-sts.nextlayersec.io/.well-known/mta-sts.txt

# DNS records
nslookup -type=TXT _mta-sts.nextlayersec.io
nslookup -type=TXT _dmarc.nextlayersec.io
nslookup -type=MX nextlayersec.io

# DNSSEC
Get-DnssecStatusForVerifiedDomain -DomainName nextlayersec.io | Format-List *
```

---

## Validation Results

- MXToolbox MTA-STS: PASS (2026-04-18)
- Verisign DNSSEC Analyzer: All green (2026-04-18)
- Exchange Online DNSSEC: Enabled (2026-04-18)
- curl policy file: Returns correct content (2026-04-18)
