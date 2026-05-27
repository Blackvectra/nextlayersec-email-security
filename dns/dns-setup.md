# DNS Setup -- Provider-Agnostic Guide

How to deploy the full email security stack at any DNS provider. For
Cloudflare-specific notes (proxy behavior, dashboard paths) see
[`cloudflare-records.md`](cloudflare-records.md).

This guide assumes:
- M365 Business Premium tenant (or equivalent Exchange Online plan)
- Domain verified in M365 as an accepted domain
- Access to your DNS provider's zone management

---

## Records to Publish

The full stack needs seven record types. Add them in this order — later
records depend on earlier ones being live.

| Order | Record | Type | Why this order |
|---|---|---|---|
| 1 | MX | MX | Mail flow must work before authentication signals matter |
| 2 | SPF | TXT | Authorizes M365 as a sender; cheap to add early |
| 3 | DKIM | CNAME x2 | Enable signing in Defender after CNAMEs resolve |
| 4 | DMARC | TXT | Start at `p=none` to monitor before enforcing |
| 5 | DNSSEC | (zone signing + DS) | Enable at provider, publish DS at registrar |
| 6 | MTA-STS | CNAME + TXT | Requires GitHub Pages or equivalent HTTPS host |
| 7 | TLS-RPT | TXT | Deploy alongside MTA-STS |

> CAA records are optional but recommended — deploy after the stack is stable.

---

## Provider Capability Matrix

Confirm your provider supports the record types and DNSSEC before starting.

| Provider | DNSSEC | CAA | TXT (long) | Notes |
|---|:---:|:---:|:---:|---|
| Cloudflare | Yes | Yes | Yes | Zone signing one-click; DS auto-published when Cloudflare is registrar |
| GoDaddy | Yes | Yes | Yes | One-click DNSSEC when GoDaddy is both registrar and DNS provider |
| Namecheap | Yes | Yes | Yes | DNSSEC requires switching to Premium DNS or external host |
| Google Domains / Squarespace Domains | Yes | Yes | Yes | DNSSEC one-click on managed zones |
| Route 53 | Yes | Yes | Yes | DNSSEC supported; key management is manual |
| GoDaddy basic DNS | Limited | Yes | Yes | DNSSEC only when GoDaddy is also the registrar |
| Wix / Squarespace site-bundled DNS | No | Sometimes | Yes | Move zone to a real DNS provider before deploying MTA-STS / DNSSEC |

If your provider does not support DNSSEC, delegate the zone to a provider that
does (Cloudflare is free) before continuing.

---

## Step 1 -- MX

```
Type:     MX
Name:     @
Value:    <domain-with-hyphens>.mail.protection.outlook.com
Priority: 0
```

> Use the standard endpoint at this stage. Switch to
> `<domain-with-hyphens>.p-v1.mx.microsoft` only after DNSSEC is enabled
> in Exchange Online (Step 5).

Verify with `nslookup -type=MX <domain>` before continuing.

---

## Step 2 -- SPF

```
Type:  TXT
Name:  @
Value: v=spf1 include:spf.protection.outlook.com -all
```

Common pitfalls:
- Duplicate SPF records — RFC 7208 allows only one TXT starting with `v=spf1`. Merge if there are multiple senders.
- 10-lookup limit — each `include:` counts. Use SPF flattening if you exceed it.
- Soft fail (`~all`) vs hard fail (`-all`) — use `-all` once you are confident in the included senders.

---

## Step 3 -- DKIM

CNAMEs must be live in DNS **before** enabling signing in the Defender portal,
otherwise enablement fails with a DNS lookup error.

```
Type:  CNAME
Name:  selector1._domainkey
Value: selector1-<domain-with-hyphens>._domainkey.<tenant>.onmicrosoft.com

Type:  CNAME
Name:  selector2._domainkey
Value: selector2-<domain-with-hyphens>._domainkey.<tenant>.onmicrosoft.com
```

After CNAMEs resolve, enable signing:

```powershell
Connect-ExchangeOnline
Enable-DkimSigningConfig -Identity <domain>
```

Verify by checking message headers for `DKIM-Signature` and `dkim=pass`.

---

## Step 4 -- DMARC

```
Type:  TXT
Name:  _dmarc
Value: v=DMARC1; p=none; rua=mailto:dmarc@<domain>
```

Progression:
- Start at `p=none`
- Review aggregate reports for 2-4 weeks
- Advance to `p=quarantine` when no legitimate sources fail
- Advance to `p=reject` after another 2-4 weeks at quarantine

Alias domains can skip the progression and deploy directly at `p=reject` if
they share sending infrastructure with the primary domain.

---

## Step 5 -- DNSSEC

This step depends on your provider. See the capability matrix above.

### Generic flow

1. Enable zone signing at the DNS provider — this publishes DNSKEY records.
2. Copy the DS record values from the provider.
3. Publish the DS record at your **registrar** (often a different vendor than DNS).
4. Wait for DS propagation — usually 5-60 minutes.
5. Verify the chain: `https://dnssec-analyzer.verisignlabs.com/<domain>` — all green.
6. Enable in Exchange Online:

```powershell
Enable-DnssecForVerifiedDomain -DomainName <domain>
```

7. Note the `MxValue` from the output. Update MX to the new endpoint
   (`<domain-with-hyphens>.p-v1.mx.microsoft`) **and** update the MTA-STS
   policy file `mx:` value in the same change window.

See [`dnssec-deployment.md`](dnssec-deployment.md) for full guide.

---

## Step 6 -- MTA-STS

MTA-STS requires an HTTPS host serving the policy file. GitHub Pages is the
simplest free option. See [`/mta-sts/deployment-guide.md`](../mta-sts/deployment-guide.md).

DNS side:

```
Type:   CNAME
Name:   mta-sts
Value:  <github-username>.github.io
Proxy:  off (Cloudflare-specific — set DNS-only / grey cloud)

Type:   TXT
Name:   _mta-sts
Value:  v=STSv1; id=YYYYMMDD
```

The `mta-sts` CNAME **must not be proxied** behind any CDN that does its own
TLS termination — GitHub Pages handles the cert and proxying breaks
provisioning. This is Cloudflare-specific behavior worth flagging
explicitly.

---

## Step 7 -- TLS-RPT

```
Type:  TXT
Name:  _smtp._tls
Value: v=TLSRPTv1; rua=mailto:tlsrpt@<domain>
```

Reports arrive as JSON attachments daily from sending MTAs that encountered
TLS failures delivering to your domain. Review for 2+ weeks before flipping
MTA-STS from `testing` to `enforce`.

---

## Optional -- CAA

```
Type:  CAA
Name:  @
Value: 0 issue "pki.goog"

Type:  CAA
Name:  @
Value: 0 issue "letsencrypt.org"

Type:  CAA
Name:  @
Value: 0 iodef "mailto:admin@<domain>"
```

CAA restricts which certificate authorities can issue certificates for your
domain. Pair with DNSSEC to close two separate attack paths against domain
identity (DNS tampering and unauthorized cert issuance).

---

## Verification Cheat Sheet

```bash
# MX
dig +short MX <domain>

# SPF
dig +short TXT <domain> | grep spf1

# DKIM
dig +short CNAME selector1._domainkey.<domain>

# DMARC
dig +short TXT _dmarc.<domain>

# MTA-STS
dig +short TXT _mta-sts.<domain>
curl -fsS https://mta-sts.<domain>/.well-known/mta-sts.txt

# TLS-RPT
dig +short TXT _smtp._tls.<domain>

# DNSSEC
dig +short DS <domain>
dig +dnssec +short MX <domain>
```

External validators:
- MTA-STS: `https://mxtoolbox.com/SuperTool.aspx` (MTA-STS Lookup)
- DNSSEC: `https://dnssec-analyzer.verisignlabs.com/<domain>`
- Full stack: `https://internet.nl/mail/`

---

## Provider-Specific Notes

| Provider | See |
|---|---|
| Cloudflare | [`cloudflare-records.md`](cloudflare-records.md) |
| DNSSEC deep dive | [`dnssec-deployment.md`](dnssec-deployment.md) |
| Copy-paste templates | [`record-templates.md`](record-templates.md) |
