# DNSSEC Deployment Guide

Enabling DNSSEC and the DNSSEC-aware MX endpoint in Microsoft 365 via Exchange Online PowerShell.

---

## What DNSSEC Protects Against

Without DNSSEC, an attacker who poisons a DNS resolver cache can redirect MX lookups
to a server they control -- intercepting inbound business email silently with no
delivery failure visible on either end.

DNSSEC cryptographically signs DNS records. Resolvers validate signatures against the
chain of trust from root -> TLD -> domain. A forged or tampered record fails validation
and is rejected before reaching the sending MTA.

### Attack scenario without DNSSEC

```
Client sends email to user@yourdomain.com
    |
    v
Sending MTA queries DNS: "What is the MX for yourdomain.com?"
    |
    v
Poisoned resolver returns: "mail.attacker.com" (fraudulent)
No signature to validate -- resolver accepts the answer
    |
    v
Sending MTA delivers email to attacker server
Attacker reads email, optionally forwards to real server
No delivery failure -- attack is silent
```

### With DNSSEC enabled

```
Client sends email to user@yourdomain.com
    |
    v
Sending MTA queries DNS: "What is the MX for yourdomain.com?"
    |
    v
Poisoned resolver returns fraudulent record + forged signature
DNSSEC-validating resolver checks signature against published DNSKEY
Signature validation FAILS -- record rejected
    |
    v
Resolver returns SERVFAIL
Sending MTA cannot deliver to attacker server
Attack fails at DNS validation layer
```

---

## Prerequisites

- Domain DNS managed in Cloudflare (or GoDaddy if both registrar and DNS provider)
- Exchange Online PowerShell access
- Domain verified in M365 tenant
- MTA-STS deployed and validated (update `mx:` value after DNSSEC changes MX endpoint)

---

## Step 1 -- Enable DNSSEC at DNS Provider

### Cloudflare (recommended)

Cloudflare --> domain --> DNS --> DNSSEC tab --> Enable

Cloudflare handles zone signing automatically. If Cloudflare is also the registrar,
the DS record is published automatically -- no further action needed.

If Cloudflare is the DNS provider but another registrar holds the domain, Cloudflare
generates DS record values that must be added at the registrar.

### GoDaddy (registrar and DNS provider)

GoDaddy --> My Products --> Domains --> domain --> DNS --> DNSSEC --> Enable

GoDaddy handles both zone signing and DS record publication automatically.

---

## Step 2 -- Verify DNSSEC Chain

Before enabling in M365, verify the full chain of trust is valid:

```
https://dnssec-analyzer.verisignlabs.com/<yourdomain>
```

All entries should be green. Common failure points:
- DS record not yet propagated at registrar
- Zone signing enabled but DS record not published
- Algorithm mismatch between zone and DS record

Also verify via nslookup:
```
nslookup -type=DS yourdomain.com
```

Should return DS record values. If empty, DNSSEC is not yet fully propagated.

---

## Step 3 -- Enable in Exchange Online

```powershell
# Connect to Exchange Online
Connect-ExchangeOnline -UserPrincipalName admin@yourdomain.com

# Check current status
Get-DnssecStatusForVerifiedDomain -DomainName yourdomain.com

# Enable DNSSEC -- note the MxValue in output
Enable-DnssecForVerifiedDomain -DomainName yourdomain.com
```

Expected output:
```
Domain    : yourdomain.com
MxValue   : yourdomain-com.p-v1.mx.microsoft
Region    :
Result    : Success
ErrorData :
```

The `MxValue` field contains the new DNSSEC-aware MX endpoint.
**Copy this value -- you need it for the next step.**

---

## Step 4 -- Update DNS and MTA-STS

After enabling, three records must be updated simultaneously:

### 1. MX record in Cloudflare/GoDaddy

```
Type:     MX
Name:     @
Value:    <MxValue from PowerShell output>
Priority: 0
```

### 2. MTA-STS policy file (`mta-sts.txt`)

Update the `mx:` value to match the new endpoint:

```
version: STSv1
mode: enforce
mx: <MxValue from PowerShell output>
max_age: 604800
```

### 3. `_mta-sts` TXT record -- bump the id

```
Type:  TXT
Name:  _mta-sts
Value: v=STSv1; id=<new date YYYYMMDD>
```

---

## Step 5 -- Verify

```powershell
# Expand full validation output
Get-DnssecStatusForVerifiedDomain -DomainName yourdomain.com | Format-List *

# Check individual validation objects
Get-DnssecStatusForVerifiedDomain -DomainName yourdomain.com | Select-Object -ExpandProperty DnsValidation
Get-DnssecStatusForVerifiedDomain -DomainName yourdomain.com | Select-Object -ExpandProperty MxValidation
Get-DnssecStatusForVerifiedDomain -DomainName yourdomain.com | Select-Object -ExpandProperty MtaStsValidation
```

Also validate via MXToolbox and Verisign after DNS propagates (allow 5-10 minutes).

---

## DNSSEC Endpoint Reference

| Domain Format | Standard MX | DNSSEC-Aware MX |
|---|---|---|
| `nextlayersec.io` | `nextlayersec-io.mail.protection.outlook.com` | `nextlayersec-io.p-v1.mx.microsoft` |
| `nextlayersec.dev` | `nextlayersec-dev.mail.protection.outlook.com` | `nextlayersec-dev.p-v1.mx.microsoft` |
| `mattlevorson.com` | `mattlevorson-com.mail.protection.outlook.com` | `mattlevorson-com.p-v1.mx.microsoft` |

---

## Important Notes

- Do not run `Enable-DnssecForVerifiedDomain` before DNSSEC is signed and verified
  at the DNS provider. M365 will switch the MX endpoint but validation will fail.
- MX change affects live mail flow. Have DNS changes ready to execute immediately
  after running the PowerShell command.
- If MTA-STS is in `enforce` mode, an MX mismatch will cause delivery failures.
  Update `mta-sts.txt` and the `_mta-sts` id value as part of the same change window.

---

## References

- RFC 4033 -- DNS Security Introduction and Requirements
- RFC 4034 -- Resource Records for DNS Security Extensions
- RFC 4035 -- Protocol Modifications for DNS Security Extensions
- [Microsoft DNSSEC for Exchange Online](https://learn.microsoft.com/en-us/microsoft-365/security/office-365-security/email-authentication-dnssec)
