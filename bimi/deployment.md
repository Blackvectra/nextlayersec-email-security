# BIMI Deployment

Brand Indicators for Message Identification (BIMI) -- RFC 9258 / RFC draft-blank-ietf-bimi.
Publishes a brand logo that participating mailbox providers display alongside
authenticated mail from your domain.

> BIMI requires `p=quarantine` or `p=reject` DMARC. Without enforcement the BIMI
> record is ignored by every major provider. Complete the DMARC chapter first.

---

## Why BIMI

- Visual confirmation to recipients that mail is authenticated and from the brand
- Cannot be spoofed unless an attacker breaks DMARC alignment
- Brand consistency in inbox view across providers that render BIMI

BIMI does not improve deliverability directly -- mail still has to pass SPF,
DKIM and DMARC alignment. It is a presentation-layer signal.

---

## Provider Support

| Provider | Renders BIMI | Requires VMC | Notes |
|---|:---:|:---:|---|
| Gmail | Yes | Yes | Requires Verified Mark Certificate |
| Apple Mail (iOS 16+, macOS 13+) | Yes | Yes | Requires VMC or CMC |
| Yahoo / AOL | Yes | No | Renders self-asserted logos |
| Fastmail | Yes | No | Renders self-asserted logos |
| Microsoft 365 / Outlook.com | Limited / Preview | Yes | Rolling rollout; check current state |

---

## Certificate Options

| Type | Issuer | Use Case |
|---|---|---|
| VMC (Verified Mark Certificate) | DigiCert, Entrust | Registered trademark logos |
| CMC (Common Mark Certificate) | DigiCert | Unregistered marks (Apple Mail only) |

VMC requires a registered trademark in one of the supported jurisdictions
(USPTO, EUIPO, UKIPO, JPO, IPO India, IP Australia, CIPO).

CMC is cheaper and faster but currently only renders in Apple Mail.

---

## Logo Requirements

- **Format:** SVG Tiny Portable / Secure (SVG P/S, profile based on SVG 1.2 Tiny)
- **Shape:** Must render correctly in a square crop -- no transparent backgrounds
- **Size:** Under 32 KB
- **Color:** Solid background recommended; ensure visible against light and dark UI
- **No JavaScript, no external references, no animation**

Validation tools:
- BIMI Group SVG validator: <https://bimigroup.org/svg-validator/>
- Common pitfalls: embedded fonts, raster images, viewBox missing

---

## DNS Record

```
Type:  TXT
Name:  default._bimi
Value: v=BIMI1; l=https://bimi.<domain>/logo.svg; a=https://bimi.<domain>/vmc.pem
```

| Tag | Required | Meaning |
|---|:---:|---|
| `v` | Yes | Version -- always `BIMI1` |
| `l` | Yes | HTTPS URL to the SVG P/S logo |
| `a` | When using VMC/CMC | HTTPS URL to the certificate |

The `default` selector applies the same logo across all mail. Custom selectors
(e.g. `marketing._bimi`) can scope logos by sending stream.

---

## Hosting

The `l=` and `a=` URLs must be served over HTTPS with a valid certificate.
GitHub Pages works well, same pattern as MTA-STS hosting:

```
bimi.<domain>    CNAME    <github-username>.github.io      (DNS-only)
```

Repository layout:

```
repo-root/
|-- .nojekyll
|-- logo.svg          <- the validated SVG P/S file
|-- vmc.pem           <- VMC certificate from issuer
`-- CNAME             <- bimi.<domain>
```

The certificate file is the full PEM chain from the issuer -- do not edit it.

---

## Validation

```bash
# Verify the DNS record
dig +short TXT default._bimi.<domain>

# Verify the SVG is reachable
curl -fsSI https://bimi.<domain>/logo.svg

# Verify the VMC is reachable
curl -fsSI https://bimi.<domain>/vmc.pem
```

External validators:
- <https://bimigroup.org/bimi-generator/> (combined check)
- <https://mxtoolbox.com/SuperTool.aspx> -> BIMI Lookup

End-to-end test: send a message from the domain to a Gmail account that has
Gmail Web open. The logo appears in the avatar circle after the first
DMARC-aligned message is processed (can take 24-48 hours).

---

## Deployment Checklist

- [ ] DMARC at `p=quarantine` or `p=reject` for 30+ days
- [ ] SVG P/S logo prepared and validated
- [ ] VMC or CMC acquired (skip if Yahoo / Apple-only is acceptable)
- [ ] HTTPS host configured (`bimi.<domain>`)
- [ ] Logo and certificate uploaded
- [ ] `_bimi` TXT record published
- [ ] Validated via BIMI Group generator
- [ ] End-to-end test send to Gmail

---

## Common Issues

| Symptom | Cause | Fix |
|---|---|---|
| Logo not rendering in Gmail | DMARC not at quarantine/reject | Tighten DMARC enforcement |
| Logo not rendering anywhere | SVG fails P/S validation | Re-export from a P/S-aware tool, validate before upload |
| Cert chain error | Partial PEM uploaded | Use the complete chain from the issuer, no edits |
| Works in Yahoo but not Gmail | No VMC | Acquire VMC from DigiCert or Entrust |
| Logo appears stretched | SVG viewBox / aspect ratio wrong | Make it true 1:1 square, no padding |

---

## References

- [BIMI Group](https://bimigroup.org/)
- RFC 9258 -- Updates to ECH
- draft-blank-ietf-bimi-04 -- BIMI specification
- [Microsoft BIMI status](https://learn.microsoft.com/microsoft-365/security/office-365-security/)
- [Google BIMI support](https://support.google.com/a/answer/10911321)
