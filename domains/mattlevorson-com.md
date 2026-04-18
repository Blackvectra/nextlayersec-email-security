# mattlevorson.com -- Email Security Status

Personal brand domain. Currently on iCloud. Migration to M365 tenant pending.

---

## Current Status

| Control | Status | Notes |
|---|---|---|
| MX | iCloud | Migration pending |
| SPF | iCloud | Will be replaced post-migration |
| DKIM | iCloud signing | iCloud key -- not portable |
| DMARC | Not configured | Open spoofing surface |
| MTA-STS | Not deployed | Deploy post-migration |
| DNSSEC | Not enabled | Enable post-migration |
| TLS-RPT | Not configured | Deploy alongside MTA-STS |

---

## Migration Plan

### Pre-migration (do now)
- [ ] Lower MX TTL to 300 in DNS (5 minutes) -- do 24-48 hours before cutover
- [ ] Export iCloud mail via Apple Mail -> drag to Outlook (light volume)

### Migration day
- [ ] Add `mattlevorson.com` to M365 tenant (Settings -> Domains -> Add)
- [ ] Verify domain ownership via TXT record
- [ ] Enable DKIM in Defender portal -- publish CNAME records in DNS
- [ ] Update SPF to M365: `v=spf1 include:spf.protection.outlook.com -all`
- [ ] Update MX to M365 endpoint
- [ ] Add `matthew@mattlevorson.com` as alias on primary mailbox
- [ ] Verify mail flow with test send
- [ ] Publish DMARC at `p=none` with rua reporting

### Post-migration hardening
- [ ] Monitor DMARC aggregate reports (2-4 weeks)
- [ ] Tighten DMARC to `p=reject`
- [ ] Deploy MTA-STS (testing mode first)
- [ ] Monitor TLS-RPT reports
- [ ] Enable DNSSEC in Cloudflare
- [ ] Run `Enable-DnssecForVerifiedDomain -DomainName mattlevorson.com`
- [ ] Update MX to DNSSEC-aware endpoint
- [ ] Flip MTA-STS to enforce

---

## Domain Role

Alias domain -- personal brand. Mail sent to `@mattlevorson.com` delivers to
`mlevorson@nextlayersec.io` inbox. No separate mailbox or license required.

---

## Current DNS (iCloud -- pre-migration)

Document current MX/SPF values here before changing anything.

```
# Run before migration and paste output here
nslookup -type=MX mattlevorson.com
nslookup -type=TXT mattlevorson.com
```

---

## Validation Commands (post-migration)

```powershell
# Verify domain in tenant
Get-AcceptedDomain | Where-Object { $_.DomainName -eq "mattlevorson.com" }

# Verify alias on mailbox
Get-Mailbox -Identity mlevorson@nextlayersec.io | Select -ExpandProperty EmailAddresses

# DKIM status
Get-DkimSigningConfig -Identity mattlevorson.com

# DNSSEC status
Get-DnssecStatusForVerifiedDomain -DomainName mattlevorson.com | Format-List *
```
