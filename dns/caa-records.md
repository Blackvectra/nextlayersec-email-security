# CAA Records

Certification Authority Authorization (RFC 8659). Restricts which CAs are
permitted to issue certificates for your domain. Without CAA, any public CA
can issue a cert for any domain -- a single rogue or compromised CA is enough
to attack TLS.

CAA complements DNSSEC: DNSSEC prevents DNS tampering, CAA prevents
fraudulent certificate issuance. Both should be deployed together.

---

## How CAA Works

When a CA receives a certificate request, RFC 8659 requires it to query the
target domain's CAA records. If a CAA record exists and the CA is not listed,
issuance is refused. If no CAA record exists, any CA can issue.

All major public CAs honor CAA. The check happens at issuance time, not at
TLS handshake -- existing certs are unaffected by CAA changes.

---

## Record Format

```
<domain>    CAA    <flags> <tag> "<value>"
```

| Field | Meaning |
|---|---|
| `flags` | `0` (default) or `128` (critical -- reject if tag unknown) |
| `tag` | `issue`, `issuewild`, or `iodef` |
| `value` | CA domain name or, for `iodef`, a `mailto:` URI |

### Tags

- `issue` -- authorize a CA to issue regular certs
- `issuewild` -- authorize a CA to issue wildcard certs (separate from `issue`)
- `iodef` -- where to report CAA failures (helps detect issuance attempts)

---

## Recommended Baseline

```
Type:  CAA
Name:  @
Value: 0 issue "pki.goog"

Type:  CAA
Name:  @
Value: 0 issue "letsencrypt.org"

Type:  CAA
Name:  @
Value: 0 issue "digicert.com"

Type:  CAA
Name:  @
Value: 0 issuewild "pki.goog"

Type:  CAA
Name:  @
Value: 0 issuewild "letsencrypt.org"

Type:  CAA
Name:  @
Value: 0 issuewild "digicert.com"

Type:  CAA
Name:  @
Value: 0 iodef "mailto:admin@<domain>"
```

Why these three:
- **Let's Encrypt** -- default for most automated TLS (GitHub Pages, Caddy, certbot)
- **Google Trust Services (pki.goog)** -- backs Google Cloud, often used by managed platforms
- **DigiCert** -- common for paid certs, VMCs, code signing

Drop any you do not actually use. The smaller the list, the smaller the attack
surface.

---

## Adding Additional CAs

Only add CAs you currently issue certificates from. If you later move to a
different CA, update CAA first, wait for the cache TTL (often 12-24 hours
in CA infrastructure), then request the new cert.

| Use Case | CA to Add |
|---|---|
| AWS Certificate Manager (public) | `amazon.com`, `amazontrust.com`, `awstrust.com`, `amazonaws.com` |
| Azure Front Door / App Service managed certs | `digicert.com` |
| Cloudflare Universal SSL | `letsencrypt.org`, `pki.goog`, `comodoca.com`, `digicert.com` |
| Sectigo (Comodo) | `sectigo.com`, `comodoca.com` |
| Entrust | `entrust.net` |
| GlobalSign | `globalsign.com` |

---

## Validation

```bash
# Verify CAA records
dig +short CAA <domain>

# Should return entries like:
# 0 issue "letsencrypt.org"
# 0 issue "pki.goog"
# ...
```

External validators:
- <https://caatest.co.uk/>
- <https://sslmate.com/caa/>
- MXToolbox CAA Lookup: <https://mxtoolbox.com/SuperTool.aspx>

---

## Common Pitfalls

| Pitfall | Effect | Fix |
|---|---|---|
| Forgetting `issuewild` for wildcard certs | Wildcard issuance refused | Add `issuewild` for each CA that issues wildcards |
| CAA on subdomain only | Parent zone CAA still applies | Test from the exact name the CA will query |
| Strict CAA + automated renewal change | Renewal silently fails until cert expires | Stage CAA changes ahead of CA changes |
| Missing `iodef` | No visibility into failed issuance attempts | Always include `iodef` -- it costs nothing |
| `flags=128` (critical) with broad tags | Some CAs refuse issuance entirely | Use `flags=0` unless you have a specific reason |

---

## CAA + DNSSEC

CAA records without DNSSEC are still useful but weaker -- an attacker who can
poison DNS could strip CAA from the answer and impersonate an "any CA"
result. Deploy DNSSEC first (see [`dnssec-deployment.md`](dnssec-deployment.md))
so that CAA itself cannot be spoofed.

---

## Deployment Checklist

- [ ] Inventory of every CA currently issuing certs for the domain
- [ ] CAA records published for each CA in `issue` and `issuewild` as appropriate
- [ ] `iodef` mailto record published
- [ ] Validated via dig + external checker
- [ ] DNSSEC enabled on the zone
- [ ] Documented in repository

---

## References

- RFC 8659 -- DNS Certification Authority Authorization (CAA) Resource Record
- RFC 8657 -- CAA Record Extensions for Account URI and ACME Method Binding
- [CA/Browser Forum CAA requirements](https://cabforum.org/)
