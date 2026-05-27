# domain-2.dev -- Email Security Status

Secondary brand domain. Added to primary M365 tenant as alias domain. Auth stack in progress.

---

## Current Status

| Control | Status | Notes |
|---|---|---|
| MX | Pending | Not yet set -- in progress |
| SPF | Deployed | `v=spf1 include:spf.protection.outlook.com -all` |
| DKIM | Pending | CNAMEs not yet published |
| DMARC | Deployed | `p=reject` |
| MTA-STS | Pending | GitHub Pages repo not yet created |
| DNSSEC | Pending | Cloudflare zone -- enable after MTA-STS deployed |
| TLS-RPT | Pending | Deploy alongside MTA-STS |

---

## Pending Actions

- [ ] Enable DKIM in Defender portal for `domain-2.dev`
- [ ] Publish DKIM CNAME records in DNS
- [ ] Add MX record pointing to M365
- [ ] Create MTA-STS GitHub Pages repo or subfolder
- [ ] Configure `mta-sts.domain-2.dev` CNAME in DNS
- [ ] Deploy `_mta-sts` and `_smtp._tls` TXT records
- [ ] Enable DNSSEC in DNS provider
- [ ] Run `Enable-DnssecForVerifiedDomain -DomainName domain-2.dev`
- [ ] Update MX to DNSSEC-aware endpoint
- [ ] Add owner alias on primary mailbox

---

## Deployment Notes

### Domain Role
Alias domain -- mail sent to `@domain-2.dev` delivers to the primary inbox.
No separate mailbox or license required.

### DNS Provider
Cloudflare -- proxy must be disabled (grey cloud) on MTA-STS CNAME.

### DNSSEC Status
DNSSEC chain validated as signed via Verisign analyzer.
M365 enablement pending MTA-STS deployment.

---

## Validation Commands

```powershell
# Check domain in tenant
Get-AcceptedDomain | Where-Object { $_.DomainName -eq "domain-2.dev" }

# Check DKIM status
Get-DkimSigningConfig -Identity domain-2.dev

# After MTA-STS deployed
curl https://mta-sts.domain-2.dev/.well-known/mta-sts.txt

# After DNSSEC enabled
Get-DnssecStatusForVerifiedDomain -DomainName domain-2.dev | Format-List *
```
