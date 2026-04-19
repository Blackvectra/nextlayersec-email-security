# MTA-STS Deployment Guide

Step-by-step deployment of MTA-STS (RFC 8461) using GitHub Pages as the policy host.
Validated deployment process based on production deployments across multiple domains.

---

## Prerequisites

- Domain DNS managed in Cloudflare
- GitHub account with Pages enabled
- Exchange Online PowerShell access
- Domain verified in M365 tenant

---

## Step 1 -- Create GitHub Repository

Create a new public repository. Recommended naming convention:

```
<domain>-mta-sts
```

Example: `nextlayersec-mta-sts`

---

## Step 2 -- Repository File Structure

```
repo-root/
|-- .nojekyll              <- REQUIRED - empty file, no content
|-- .well-known/
|   `-- mta-sts.txt        <- policy file
|-- CNAME                  <- custom domain
`-- README.md
```

### `.nojekyll`

Must be an **empty file** with no content. GitHub Pages uses Jekyll by default which
silently blocks all dotfolders including `.well-known/`. This file disables Jekyll
and allows static file serving.

Common mistake: creating this file with content (e.g. the shell command `touch .nojekyll`
as literal text). The file must be empty.

### `.well-known/mta-sts.txt`

```
version: STSv1
mode: testing
mx: <your-mx-record>
max_age: 86400
```

Start with `mode: testing` and `max_age: 86400` (24 hours). Switch to enforce only
after validating with TLS-RPT reports. See [Enforce Mode](#enforce-mode) below.

### `CNAME`

```
mta-sts.<yourdomain>
```

Example: `mta-sts.nextlayersec.io`

---

## Step 3 -- Configure GitHub Pages

Repository --> Settings --> Pages --> Source --> Deploy from branch --> main --> / (root)

Wait for the deployment to go green before proceeding.

---

## Step 4 -- DNS Records in Cloudflare

Add the following records. **All must be DNS-only (grey cloud) -- no Cloudflare proxy.**

Cloudflare proxying breaks GitHub Pages TLS certificate provisioning. Orange cloud
on the CNAME will cause certificate errors and the policy file will not be served.

```
# Policy host CNAME
Type:   CNAME
Name:   mta-sts
Target: <github-username>.github.io
Proxy:  DNS-only (grey cloud)

# Policy signal TXT record
Type:   TXT
Name:   _mta-sts
Value:  v=STSv1; id=20260418

# TLS failure reporting (recommended)
Type:   TXT
Name:   _smtp._tls
Value:  v=TLSRPTv1; rua=mailto:tlsrpt@<yourdomain>
```

---

## Step 5 -- Add Custom Domain in GitHub Pages

Repository --> Settings --> Pages --> Custom domain --> enter `mta-sts.<yourdomain>`

**Add the Cloudflare CNAME before saving here.** GitHub Pages verifies DNS before
accepting the custom domain. If the CNAME doesn't resolve, the save will fail.

Wait 2-3 minutes after adding the CNAME before attempting to save the custom domain.

---

## Step 6 -- Verify Policy File

```bash
curl https://mta-sts.<yourdomain>/.well-known/mta-sts.txt
```

Expected output:
```
version: STSv1
mode: testing
mx: <your-mx-record>
max_age: 86400
```

If you get a 404, check:
1. `.nojekyll` file exists and is empty
2. `.well-known/` folder exists with `mta-sts.txt` inside
3. GitHub Pages deployment completed (green checkmark in Actions tab)
4. Custom domain is saved correctly in Pages settings

---

## Step 7 -- Verify DNS Records

```
# Verify policy signal
nslookup -type=TXT _mta-sts.<yourdomain>

# Verify TLS-RPT
nslookup -type=TXT _smtp._tls.<yourdomain>
```

---

## Step 8 -- Full Validation

Use MXToolbox MTA-STS Lookup for complete validation:
`https://mxtoolbox.com/SuperTool.aspx`

Select MTA-STS Lookup, enter your domain. Confirm:
- Record resolves correctly
- Policy file fetched from correct URL
- MX record matches policy file `mx:` value
- TLS certificate valid

---

## Enforce Mode

Before switching to `mode: enforce`, confirm:

1. TLS-RPT reports show zero failures over 2+ weeks
2. MX record in policy file exactly matches live MX record
3. DNSSEC is enabled if using DNSSEC-aware MX endpoint

To enforce:

1. Update `mta-sts.txt`:
```
version: STSv1
mode: enforce
mx: <your-mx-record>
max_age: 604800
```

2. Update `_mta-sts` TXT record -- bump the `id` value:
```
v=STSv1; id=20260418
```

The `id` value signals receiving MTAs to re-fetch the policy. Any change works --
using the current date in `YYYYMMDD` format is a simple convention.

3. Commit and push. GitHub Pages redeploys automatically.

---

## DNSSEC-Aware MX Endpoint

If DNSSEC is enabled in Exchange Online, the MX endpoint changes format:

```
# Standard endpoint
<domain-tld>.mail.protection.outlook.com

# DNSSEC-aware endpoint
<domain-tld>.p-v1.mx.microsoft
```

When switching to the DNSSEC-aware endpoint, update both:
- MX record in Cloudflare DNS
- `mx:` value in `mta-sts.txt`
- `id` value in `_mta-sts` TXT record

Do all three simultaneously or in rapid succession to avoid MX mismatch in enforce mode.

---

## Common Issues

| Issue | Cause | Fix |
|---|---|---|
| 404 on policy file | Jekyll blocking `.well-known/` | Add empty `.nojekyll` to repo root |
| TLS cert error | Cloudflare proxy enabled | Set CNAME to DNS-only (grey cloud) |
| Custom domain fails to save | CNAME not propagated | Wait 2-3 min after adding CNAME |
| MX mismatch in validation | Policy file has wrong MX value | Match `mx:` exactly to live MX record |
| Policy not re-fetched after change | `id` not updated | Bump `id` value in `_mta-sts` TXT record |
| `.nojekyll` not working | File has content | Must be completely empty |

---

## References

- RFC 8461 -- SMTP MTA Strict Transport Security
- RFC 8460 -- SMTP TLS Reporting
- [GitHub Pages Custom Domains](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site)
