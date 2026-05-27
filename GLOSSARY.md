# Glossary

Quick reference for the acronyms and terms used throughout this framework.
Where applicable, links point to the relevant RFC or Microsoft Learn page.

---

## Authentication & Anti-Spoofing

| Term | Expansion | What it does |
|---|---|---|
| **SPF** | Sender Policy Framework ([RFC 7208](https://datatracker.ietf.org/doc/html/rfc7208)) | TXT record listing IPs authorized to send mail for a domain |
| **DKIM** | DomainKeys Identified Mail ([RFC 6376](https://datatracker.ietf.org/doc/html/rfc6376)) | Cryptographic signature on outbound mail, published via DNS |
| **DMARC** | Domain-based Message Authentication, Reporting & Conformance ([RFC 7489](https://datatracker.ietf.org/doc/html/rfc7489)) | Policy combining SPF + DKIM alignment + reporting |
| **Alignment** | SPF/DKIM domain alignment | DMARC pass requires the authenticated domain to match the From: header domain |
| **RUA** | Reporting URI for Aggregate reports | Address that receives daily DMARC XML aggregate reports |
| **RUF** | Reporting URI for Forensic reports | Address for per-failure forensic reports (rarely used today) |
| **ARC** | Authenticated Received Chain ([RFC 8617](https://datatracker.ietf.org/doc/html/rfc8617)) | Preserves auth results across intermediaries (mailing lists, forwarders) |
| **BIMI** | Brand Indicators for Message Identification | Publishes a brand logo for inbox display alongside authenticated mail |
| **VMC** | Verified Mark Certificate | BIMI cert backed by a registered trademark |
| **CMC** | Common Mark Certificate | BIMI cert for unregistered marks (Apple Mail only) |

---

## Transport Security

| Term | Expansion | What it does |
|---|---|---|
| **MTA** | Mail Transfer Agent | Any server that relays mail (Outlook.com, Gmail, your own postfix) |
| **STARTTLS** | Opportunistic TLS upgrade | SMTP command to negotiate TLS on a plaintext connection |
| **MTA-STS** | MTA Strict Transport Security ([RFC 8461](https://datatracker.ietf.org/doc/html/rfc8461)) | Policy requiring TLS on inbound SMTP; prevents downgrade attacks |
| **TLS-RPT** | SMTP TLS Reporting ([RFC 8460](https://datatracker.ietf.org/doc/html/rfc8460)) | Daily reports of TLS / MTA-STS failures from sending MTAs |
| **DANE** | DNS-Based Authentication of Named Entities ([RFC 7672](https://datatracker.ietf.org/doc/html/rfc7672)) | TLSA records in DNS for SMTP cert pinning (requires DNSSEC; not used in M365) |

---

## DNS Security

| Term | Expansion | What it does |
|---|---|---|
| **DNSSEC** | DNS Security Extensions ([RFC 4033-4035](https://datatracker.ietf.org/doc/html/rfc4033)) | Cryptographic signatures on DNS records; prevents cache poisoning |
| **DS** | Delegation Signer | Hash of a DNSKEY published in parent zone; anchors chain of trust |
| **DNSKEY** | DNS public key | Public key used to verify DNSSEC signatures |
| **RRSIG** | Resource Record Signature | The signature itself, alongside each signed record set |
| **NSEC / NSEC3** | Next Secure | Proves nonexistence of records without leaking the zone |
| **CAA** | Certification Authority Authorization ([RFC 8659](https://datatracker.ietf.org/doc/html/rfc8659)) | Restricts which CAs can issue certificates for the domain |

---

## Microsoft 365

| Term | Expansion | What it does |
|---|---|---|
| **EXO** | Exchange Online | M365's hosted Exchange service |
| **EOP** | Exchange Online Protection | Built-in mail filtering layer in EXO |
| **MDO** | Microsoft Defender for Office 365 | Premium tier of EOP (Safe Links, Safe Attachments, etc.) |
| **CA** | Conditional Access | Entra ID policy engine that gates sign-ins by user / device / risk |
| **CAE** | Continuous Access Evaluation | Real-time revocation of access tokens on risk events |
| **MFA** | Multi-Factor Authentication | Second factor beyond password (TOTP, push, FIDO2) |
| **FIDO2** | Fast Identity Online v2 | Phishing-resistant MFA using hardware authenticators |
| **WHfB** | Windows Hello for Business | Microsoft's PKI-backed device-bound MFA |
| **ZAP** | Zero-hour Auto Purge | Defender feature that removes messages from inbox after delivery if later classified as malicious |
| **TABL** | Tenant Allow/Block List | Manual override / IOC list for the tenant |
| **OWA** | Outlook Web Access | Browser-based Outlook |
| **TAP** | Temporary Access Pass | Time-limited credential for bootstrapping passwordless auth |

---

## Threats & Concepts

| Term | What it means |
|---|---|
| **Spoofing** | Sending mail with a forged `From:` header |
| **Lookalike domain** | Visually similar domain (e.g. `examp1e.com`) used in phishing |
| **Display name spoofing** | Forging the friendly name only; address remains attacker-controlled |
| **Business Email Compromise (BEC)** | Targeted impersonation of an executive or vendor to trigger payment / data release |
| **Credential stuffing** | Trying leaked username/password pairs against a tenant |
| **Password spray** | Trying a few common passwords against many accounts |
| **SIM swap** | Attacker convinces carrier to port victim's number; defeats SMS MFA |
| **Token theft** | Stealing a session cookie / refresh token to bypass auth |
| **Cache poisoning** | Injecting fake DNS records into a resolver's cache |
| **Downgrade attack** | Forcing a secure connection to fall back to plaintext |

---

## Operations

| Term | What it means |
|---|---|
| **Break-glass account** | Privileged account excluded from CA policies, used only in emergencies |
| **Quarantine** | Held-message review queue for admin / user release |
| **Inbox rule** | User-defined client-side filter (commonly abused after compromise) |
| **Mailbox forwarding** | Server-side automatic forward (commonly abused after compromise) |
| **Connector** | Custom mail-flow path between EXO and another mail system / partner |
| **Mail flow rule** | aka Transport rule -- conditional action on mail in transit |
| **IOC** | Indicator of Compromise -- sender, URL, hash tied to an attack |
