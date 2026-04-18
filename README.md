NextLayerSec Email Security Framework
> A production-validated email security hardening framework for M365 environments.
> Built and deployed across multiple domains. Designed as a repeatable MSP delivery framework.
![Domains](https://img.shields.io/badge/Domains-3_Hardened-00c853?style=flat-square)
![MTA-STS](https://img.shields.io/badge/MTA--STS-Enforced-00c853?style=flat-square)
![DNSSEC](https://img.shields.io/badge/DNSSEC-Enabled-00c853?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-blue?style=flat-square)
---
Overview
Most organizations deploy DMARC and call their email security done.
DMARC does not protect SMTP transit.
This framework documents the complete email security stack — from sender authentication
through transport-layer enforcement — deployed in production across multiple domains under
a single Microsoft 365 Business Premium tenant.
Every control documented here has been implemented, validated, and verified against live DNS
and M365 tenant configuration. Screenshots of all validation outputs are available in the
`/validation` directory.
---
The Stack
Layer	Control	Purpose
Sender Authentication	SPF	Authorizes sending sources for the domain
Message Integrity	DKIM	Cryptographic signature on every outbound message
Policy Enforcement	DMARC	Enforces SPF/DKIM alignment, enables aggregate reporting
Transport Security	MTA-STS	Forces TLS on inbound SMTP, prevents downgrade attacks
DNS Integrity	DNSSEC	Cryptographically signs DNS records, prevents cache poisoning
Receive Hardening	DNSSEC-aware MX	M365 DNSSEC-validated inbound endpoint
Failure Visibility	TLS-RPT	Reports TLS failures on inbound mail delivery
Why all seven matter
Each control closes a gap the others cannot:
SPF alone — doesn't prevent header spoofing, no cryptographic proof
DKIM alone — doesn't prevent replay attacks without DMARC alignment
DMARC alone — doesn't protect SMTP transit after DNS resolution
MTA-STS alone — cached policy can be stale, doesn't protect the DNS layer
DNSSEC alone — doesn't enforce TLS on delivery, doesn't authenticate sender
The full stack is required for complete inbound mail path hardening.
---
Domain Coverage
Domain	SPF	DKIM	DMARC	MTA-STS	DNSSEC	TLS-RPT
`nextlayersec.io`	PASS	PASS	`p=reject`	`enforce`	Enabled	Configured
`nextlayersec.dev`	`-all`	N/A	`p=reject`	N/A (parked)	Pending	N/A
`mattlevorson.com`	Pending	Pending	Pending	Pending	Pending	Pending
> `nextlayersec.dev` is a parked domain with no mail flow.
> SPF `-all` and DMARC `p=reject` close the spoofing surface with no MX record present.
---
Repository Structure
```
nextlayersec-email-security/
|
|-- mta-sts/
|   |-- nextlayersec-io.md          # MTA-STS deployment - nextlayersec.io
|   |-- nextlayersec-dev.md         # Parked domain lockdown
|   `-- deployment-guide.md         # Repeatable deployment framework
|
|-- exchange-online/
|   |-- hardening-runbook.md        # Full PowerShell hardening session
|   `-- baseline-checklist.md       # Verification checklist
|
|-- dns/
|   |-- dnssec-deployment.md        # DNSSEC enablement via M365 PowerShell
|   |-- cloudflare-records.md       # Full DNS record reference
|   `-- record-templates.md         # Copy-paste DNS record templates
|
|-- dmarc/
|   |-- monitoring-setup.md         # DMARC aggregate reporting setup
|   `-- report-analysis.md          # Reading and acting on aggregate reports
|
|-- validation/
|   |-- mxtoolbox-mta-sts.png       # MXToolbox MTA-STS validation
|   |-- verisign-dnssec.png         # Verisign DNSSEC chain validation
|   `-- powershell-dnssec.png       # Exchange Online PowerShell output
|
`-- README.md
```
---
MTA-STS Deployment
MTA-STS (RFC 8461) allows domain owners to publish a policy requiring sending MTAs to
validate TLS certificates before delivering mail. Policy enforcement prevents SMTP downgrade
attacks where an attacker strips TLS from the delivery path.
How it works
```
Sending MTA                DNS                    Policy Host             Receiving MTA
     |                      |                          |                       |
     |-- MX lookup -------->|                          |                       |
     |<- MX record ---------|                          |                       |
     |                      |                          |                       |
     |-- _mta-sts TXT ----->|                          |                       |
     |<- "v=STSv1; id=X" ---|                          |                       |
     |                      |                          |                       |
     |-- HTTPS policy fetch --------------------------->|                       |
     |<- mta-sts.txt ----------------------------------|                       |
     |                      |                          |                       |
     |-- Validate MX matches policy                                            |
     |-- Validate TLS cert on MX host                                          |
     |-- Deliver over verified TLS ------------------------------------------>|
```
Policy file structure
```
version: STSv1
mode: enforce
mx: nextlayersec-io.p-v1.mx.microsoft
max_age: 604800
```
Field	Value	Notes
`version`	`STSv1`	Only supported version
`mode`	`enforce`	Reject delivery if TLS validation fails
`mx`	MX hostname	Must exactly match live MX record
`max_age`	`604800`	Policy cache TTL in seconds (7 days)
Hosting on GitHub Pages
MTA-STS policy files must be served over HTTPS at:
`https://mta-sts.<domain>/.well-known/mta-sts.txt`
GitHub Pages provides free HTTPS hosting. Two configuration requirements:
1. Disable Jekyll processing
Jekyll silently blocks dotfolders including `.well-known/`.
Add an empty `.nojekyll` file to the repo root to serve static files as-is.
2. Disable Cloudflare proxy on the CNAME
GitHub Pages handles TLS termination for custom domains.
Cloudflare proxying (orange cloud) breaks certificate provisioning.
The `mta-sts` CNAME must be DNS-only (grey cloud).
Required DNS records
```
# Policy host
mta-sts.<domain>    CNAME    <user>.github.io      (DNS-only)

# Policy signal
_mta-sts.<domain>   TXT      "v=STSv1; id=YYYYMMDD"

# TLS failure reporting
_smtp._tls.<domain> TXT      "v=TLSRPTv1; rua=mailto:tlsrpt@<domain>"
```
> Update the `id` value in `_mta-sts` whenever the policy file changes.
> Receiving MTAs cache the policy — a changed `id` signals them to re-fetch.
---
DNSSEC + DNSSEC-Aware MX Endpoint
What DNSSEC protects against
Without DNSSEC, an attacker who poisons a DNS resolver cache can redirect MX lookups
to a server they control — intercepting inbound business email silently with no delivery
failure visible on either end.
DNSSEC cryptographically signs DNS records. Resolvers validate signatures against the
chain of trust from root → TLD → domain. A forged or tampered record fails validation
and is rejected before reaching the sending MTA.
DNSSEC-aware MX endpoint
When DNSSEC is enabled in Exchange Online, the MX endpoint changes from:
```
# Standard endpoint
domain-com.mail.protection.outlook.com

# DNSSEC-aware endpoint
domain-com.p-v1.mx.microsoft
```
The `p-v1.mx.microsoft` endpoint only accepts connections from resolvers that validated
DNSSEC — adding verification on the receiving side in addition to the sending side.
Enabling via Exchange Online PowerShell
```powershell
# Connect
Connect-ExchangeOnline -UserPrincipalName admin@yourdomain.com

# Enable - note the new MX value in output
Enable-DnssecForVerifiedDomain -DomainName yourdomain.com

# Verify
Get-DnssecStatusForVerifiedDomain -DomainName yourdomain.com | Format-List *
```
> After enabling, update the MX record in DNS and the `mx:` value in `mta-sts.txt`
> to match the new `p-v1.mx.microsoft` endpoint before mail flow is affected.
Validation
```powershell
# Check DNSSEC signing chain
# https://dnssec-analyzer.verisignlabs.com/<domain>

# Check M365 tenant recognition
Get-DnssecStatusForVerifiedDomain -DomainName yourdomain.com | Format-List *

# Check DNS record
nslookup -type=DS yourdomain.com
```
---
Parked Domain Lockdown
Domains that do not send or receive email are active spoofing surfaces if left unprotected.
A domain with no SPF or DMARC record can be used to send email that appears to originate
from your organization.
Required records for parked domains
```
# SPF - hard fail, nothing authorized to send
v=spf1 -all

# DMARC - immediate reject, no monitoring phase needed
v=DMARC1; p=reject;

# No MX record
```
> DKIM is not required on a parked domain as there is no signing infrastructure.
> SPF `-all` and DMARC `p=reject` are sufficient to close the spoofing surface.
---
Exchange Online Hardening
The full PowerShell hardening runbook is in `/exchange-online/hardening-runbook.md`.
Controls covered
Control	Command	Impact
Legacy auth lockdown	`Set-AuthenticationPolicy`	Blocks basic auth on all protocols
SMTP client auth	`Set-TransportConfig`	Disables legacy SMTP relay tenant-wide
External auto-forward	`Set-RemoteDomain`	Prevents exfiltration via mail forwarding rules
Protocol hardening	`Set-CasMailbox`	Disables POP, IMAP, ActiveSync at mailbox level
Mailbox auditing	`Set-Mailbox`	180-day audit log retention, all actions logged
Outbound spam notify	`Set-HostedOutboundSpamFilterPolicy`	Alert on compromised account sending spam
DNSSEC enablement	`Enable-DnssecForVerifiedDomain`	Switches to DNSSEC-aware MX endpoint
---
Validation Tools
Tool	Purpose	URL
MXToolbox MTA-STS	Full MTA-STS policy validation	mxtoolbox.com/SuperTool
Verisign DNSSEC Analyzer	Full DNSSEC chain validation	dnssec-analyzer.verisignlabs.com
DMARCian	DMARC aggregate report analysis	dmarcian.com
curl	Direct policy file verification	`curl https://mta-sts.<domain>/.well-known/mta-sts.txt`
nslookup	DNS record verification	`nslookup -type=TXT _mta-sts.<domain>`
Exchange Online PS	Tenant-side DNSSEC validation	`Get-DnssecStatusForVerifiedDomain`
---
Related Tools
Invoke-M365SecurityBaseline — PowerShell script that runs a full security baseline check against any M365 tenant with color-coded pass/fail output
Invoke-EmailSecurityAssessment — Domain assessment script that checks the full email security stack via public DNS lookups with no tenant access required
---
References
RFC	Title
RFC 8461	SMTP MTA Strict Transport Security (MTA-STS)
RFC 8460	SMTP TLS Reporting (TLS-RPT)
RFC 7208	Sender Policy Framework (SPF)
RFC 6376	DomainKeys Identified Mail (DKIM)
RFC 7489	Domain-based Message Authentication, Reporting, and Conformance (DMARC)
RFC 4033	DNS Security Introduction and Requirements (DNSSEC)
---
License
MIT License. See LICENSE for details.
---
<div align="center">
NextLayerSec  | 
LinkedIn  | 
GitHub
Cybersecurity consulting for organizations that take security seriously.
</div>
