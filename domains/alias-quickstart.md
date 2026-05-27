# Alias Domain Quickstart

Condensed playbook for adding a new alias domain to an already-hardened
primary tenant. Target: 30 minutes of active work, plus DNS propagation
wait time.

This skips the DMARC monitoring progression -- alias domains share sending
infrastructure with the primary, so they can deploy directly at `p=reject`.

For new standalone domains (not an alias), use [`_template.md`](_template.md)
and the full DMARC progression instead.

---

## Prerequisites

- Primary domain fully hardened (SPF, DKIM, DMARC enforced, DNSSEC, MTA-STS)
- Primary tenant `*.onmicrosoft.com` identifier known
- Reporting addresses live on primary (`dmarc@`, `tlsrpt@`)
- DNS provider supports CAA, TXT, CNAME records and DNSSEC

---

## Step 1 -- Add to Tenant (5 min)

1. M365 Admin Center -> Settings -> Domains -> Add domain
2. Enter alias domain name
3. Verify ownership via TXT record at registrar / DNS provider
4. Choose "Don't use Exchange / set DNS records later" -- you will manage
   records manually

---

## Step 2 -- Publish DNS Records (10 min)

Add all of these at once. Replace `<domain>`, `<tenant>`, and
`<github-username>` with your values.

```
# MX -- standard endpoint initially (switch to DNSSEC-aware after Step 5)
@                       MX     <domain-hyphens>.mail.protection.outlook.com  pri 0

# SPF
@                       TXT    "v=spf1 include:spf.protection.outlook.com -all"

# DKIM
selector1._domainkey    CNAME  selector1-<domain-hyphens>._domainkey.<tenant>.onmicrosoft.com
selector2._domainkey    CNAME  selector2-<domain-hyphens>._domainkey.<tenant>.onmicrosoft.com

# DMARC -- direct to reject, reporting consolidated to primary
_dmarc                  TXT    "v=DMARC1; p=reject; rua=mailto:dmarc@<primary-domain>"

# MTA-STS
mta-sts                 CNAME  <github-username>.github.io  (DNS-only)
_mta-sts                TXT    "v=STSv1; id=YYYYMMDD"
_smtp._tls              TXT    "v=TLSRPTv1; rua=mailto:tlsrpt@<primary-domain>"

# Autodiscover (optional)
autodiscover            CNAME  autodiscover.outlook.com
```

> Use `<domain-hyphens>` = the alias domain with `.` replaced by `-`.

---

## Step 3 -- Enable DKIM Signing (2 min)

After DKIM CNAMEs resolve (verify with `dig +short CNAME selector1._domainkey.<domain>`):

```powershell
Connect-ExchangeOnline
Enable-DkimSigningConfig -Identity <domain>

# Verify
Get-DkimSigningConfig -Identity <domain> | Format-List Enabled, Selector1KeySize
```

---

## Step 4 -- MTA-STS Policy File (5 min)

Add to the existing MTA-STS GitHub Pages repo:

1. Add a new branch or directory for `<domain>` (most setups host one
   policy per subdomain CNAME -- one repo can serve multiple)
2. Verify `mta-sts.<domain>` CNAME resolves to the Pages host
3. Confirm the Pages site serves `https://mta-sts.<domain>/.well-known/mta-sts.txt`

```bash
curl -fsS https://mta-sts.<domain>/.well-known/mta-sts.txt
```

Initial policy file (will update `mx:` in Step 5):

```
version: STSv1
mode: enforce
mx: <domain-hyphens>.mail.protection.outlook.com
max_age: 604800
```

---

## Step 5 -- DNSSEC + DNSSEC-aware MX (10 min + propagation)

```powershell
# Enable in M365 -- copy the MxValue from output
Enable-DnssecForVerifiedDomain -DomainName <domain>
```

Then in the same change window:

1. Update MX in DNS to the new `<domain-hyphens>.p-v1.mx.microsoft` value
2. Update `mta-sts.txt` `mx:` value to match
3. Bump `_mta-sts` TXT `id` value

Also enable DNSSEC at the DNS provider if not already (and publish DS at
the registrar -- see [`dns/dnssec-deployment.md`](../dns/dnssec-deployment.md)).

---

## Step 6 -- Add Mailbox Alias (1 min)

If the alias should receive mail to an existing mailbox:

```powershell
Set-Mailbox -Identity primary@<primary-domain> `
  -EmailAddresses @{add="alias@<domain>"}
```

---

## Step 7 -- CAA (2 min)

```
@    CAA    0 issue "letsencrypt.org"
@    CAA    0 issue "pki.goog"
@    CAA    0 iodef "mailto:admin@<primary-domain>"
```

Add others if applicable -- see [`dns/caa-records.md`](../dns/caa-records.md).

---

## Step 8 -- Validate (5 min)

```bash
# All authentication records
dig +short MX <domain>
dig +short TXT <domain> | grep spf1
dig +short CNAME selector1._domainkey.<domain>
dig +short TXT _dmarc.<domain>
dig +short TXT _mta-sts.<domain>
dig +short TXT _smtp._tls.<domain>
dig +short CAA <domain>

# MTA-STS policy
curl -fsS https://mta-sts.<domain>/.well-known/mta-sts.txt

# DNSSEC
dig +short DS <domain>
```

External validation:
- MXToolbox SuperTool: MX + SPF + DMARC + MTA-STS lookups
- Verisign DNSSEC Analyzer
- internet.nl mail test for end-to-end check

---

## Step 9 -- Document (2 min)

1. Copy `_template.md` to `domain-<n>.md`
2. Fill in the status table
3. Add changelog entry

---

## Common Issues

| Issue | Fix |
|---|---|
| DKIM enable fails with DNS error | CNAMEs not propagated; wait 5-10 min, retry |
| MTA-STS validation fails | Verify HTTPS host serves the policy file; check `mx:` matches live MX |
| DMARC reports never arrive | Check `rua=` address mailbox is receiving; some providers send weekly not daily |
| DNSSEC validation fails | Verify DS record at registrar matches DNSKEY at DNS provider |
| Mail bouncing after MX change | Old MX cached at sender; flip MTA-STS to `testing` temporarily |

See [`/incident-response/runbook.md`](../incident-response/runbook.md) for
deeper troubleshooting.
