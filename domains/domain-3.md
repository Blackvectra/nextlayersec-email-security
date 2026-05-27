# domain-3.com -- Email Security Status

Personal brand domain. Currently on external provider. Migration to primary M365 tenant pending.

---

## Current Status

| Control | Status | Notes |
|---|---|---|
| MX | External | Migration pending |
| SPF | External | Will be replaced post-migration |
| DKIM | External signing | Provider key -- not portable |
| DMARC | Not configured | Open spoofing surface |
| MTA-STS | Not deployed | Deploy post-migration |
| DNSSEC | Not enabled | Enable post-migration |
| TLS-RPT | Not configured | Deploy alongside MTA-STS |

---

## Migration Plan

### Pre-migration (do now)
- [ ] Lower MX TTL to 300 in DNS (5 minutes) -- do 24-48 hours before cutover
- [ ] Export existing mail to local archive (light volume)

### Migration day
- [ ] Add `domain-3.com` to M365 tenant (Settings -> Domains -> Add)
- [ ] Verify domain ownership via TXT record
- [ ] Enable DKIM in Defender portal -- publish CNAME records in DNS
- [ ] Update SPF to M365: `v=spf1 include:spf.protection.outlook.com -all`
- [ ] Update MX to M365 endpoint
- [ ] Add owner alias on primary mailbox
- [ ] Verify mail flow with test send
- [ ] Publish DMARC at `p=none` with rua reporting

### Post-migration hardening
- [ ] Monitor DMARC aggregate reports (2-4 weeks)
- [ ] Tighten DMARC to `p=reject`
- [ ] Deploy MTA-STS (testing mode first)
- [ ] Monitor TLS-RPT reports
- [ ] Enable DNSSEC in DNS provider
- [ ] Run `Enable-DnssecForVerifiedDomain -DomainName domain-3.com`
- [ ] Update MX to DNSSEC-aware endpoint
- [ ] Flip MTA-STS to enforce

---

## Domain Role

Alias domain -- personal brand. Mail sent to `@domain-3.com` delivers to the
primary tenant inbox. No separate mailbox or license required.

---

## Current DNS (external provider -- pre-migration)

Document current MX/SPF values here before changing anything.

```
# Run before migration and paste output here
nslookup -type=MX domain-3.com
nslookup -type=TXT domain-3.com
```

---

## Validation Commands (post-migration)

```powershell
# Verify domain in tenant
Get-AcceptedDomain | Where-Object { $_.DomainName -eq "domain-3.com" }

# Verify alias on mailbox
Get-Mailbox -Identity admin@domain-1.io | Select -ExpandProperty EmailAddresses

# DKIM status
Get-DkimSigningConfig -Identity domain-3.com

# DNSSEC status
Get-DnssecStatusForVerifiedDomain -DomainName domain-3.com | Format-List *
```
