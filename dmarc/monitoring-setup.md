# DMARC Monitoring Setup

Aggregate reporting configuration and analysis guidance for all domains
under the primary M365 tenant.

---

## Reporting Configuration

All three domains send DMARC aggregate reports to a single destination:

```
rua=mailto:dmarc@domain-1.io
```

This consolidates reporting across all domains into one inbox for unified visibility.

---

## DMARC Records

### domain-1.io
```
v=DMARC1; p=reject; rua=mailto:dmarc@domain-1.io
```

### domain-2.dev
```
v=DMARC1; p=reject; rua=mailto:dmarc@domain-1.io
```

### domain-3.com
```
v=DMARC1; p=reject; rua=mailto:dmarc@domain-1.io
```

> All three domains are at full enforcement.
> For new domain migrations start at p=none and tighten after 2-4 weeks of clean reports.

---

## Reading Aggregate Reports

DMARC aggregate reports arrive as XML files attached to emails from receiving mail servers.
Use DMARCian or similar tool to parse and visualize.

### Key fields to review

```xml
<record>
  <row>
    <source_ip>...</source_ip>          <!-- Where the mail came from -->
    <count>...</count>                  <!-- Number of messages from this source -->
    <policy_evaluated>
      <disposition>none|quarantine|reject</disposition>
      <dkim>pass|fail</dkim>
      <spf>pass|fail</spf>
    </policy_evaluated>
  </row>
  <identifiers>
    <header_from>yourdomain.com</header_from>
  </identifiers>
</record>
```

### What to look for

**Legitimate sources failing DMARC:**
- SPF fail + DKIM fail from your own sending IPs = SPF misconfiguration
- SPF pass + DKIM fail = forwarding issue or DKIM key problem
- SPF fail + DKIM pass = authorized sender not in SPF record

**Spoofing attempts:**
- Unknown source IPs with high message counts
- Sources you don't recognize passing or failing

**Expected sources:**
- M365 outbound IPs (Microsoft) -- should always pass
- Any third-party senders (newsletters, CRMs) -- verify they're authorized

---

## Policy Progression

For new domains or after significant changes:

```
Week 1-2:  p=none       -- monitoring only, no enforcement
Week 3-4:  p=none       -- continue monitoring, verify no legitimate failures
Month 2:   p=quarantine -- failed mail goes to spam
Month 3+:  p=reject     -- failed mail rejected outright
```

Only move to next level when aggregate reports show zero legitimate source failures.

---

## Tools

| Tool | Purpose | URL |
|---|---|---|
| DMARCian | Aggregate report parsing and visualization | dmarcian.com |
| MXToolbox DMARC | Quick DMARC record validation | mxtoolbox.com/DMARC.aspx |
| Google Admin Toolbox | Check DMARC record syntax | toolbox.googleapps.com |

---

## TLS-RPT Monitoring

TLS-RPT reports (`_smtp._tls` records) provide visibility into TLS failures on
inbound mail delivery. Relevant when MTA-STS is in `testing` mode -- reports
show failures that would cause delivery rejection in `enforce` mode.

All three domains send TLS-RPT reports to:

```
rua=mailto:tlsrpt@domain-1.io
```

### What triggers a TLS-RPT report
- Sending MTA cannot establish STARTTLS to your MX
- TLS certificate validation failure on your MX
- MTA-STS policy mismatch (wrong MX in policy file)

### Before flipping MTA-STS to enforce
- Review TLS-RPT reports for 2+ weeks in testing mode
- Zero failures = safe to enforce
- Any failures = investigate before enforcing

---

## CAA Records

CAA records restrict which Certificate Authorities are authorized to issue
SSL/TLS certificates for your domain. Without CAA any CA can issue a cert
for your domain. With CAA only authorized CAs can issue.

### Records deployed on all three domains

Together they close two separate attack paths against your domain identity:
- Attacker cannot poison your DNS records — DNSSEC
- Attacker cannot obtain a fraudulent cert for your domain — CAA
### Why CAA matters alongside DNSSEC
DNSSEC prevents DNS record tampering. CAA prevents unauthorized certificate issuance.
Together they close two separate attack paths against your domain identity:
- Attacker cannot poison your DNS records — DNSSEC
- Attacker cannot obtain a fraudulent cert for your domain — CAA

---

## S/MIME

S/MIME signing is enabled on the primary domain mailbox.
S/MIME operates independently of CAA records — CAA governs SSL/TLS issuance only
and does not affect S/MIME certificate provisioning or signing behavior.

---

*NextLayerSec -- nextlayersec.io*
