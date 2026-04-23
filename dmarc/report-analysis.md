# DMARC Report Analysis

Guide for reading, interpreting, and acting on DMARC aggregate reports
across all domains in the NextLayerSec M365 tenant.

---

## Overview

DMARC aggregate reports (RUA) are XML files sent by receiving mail servers
to your reporting address after processing mail from your domain.
They arrive as email attachments — typically compressed .gz or .zip files.

Reports consolidate authentication results across all messages received
during a reporting interval — usually 24 hours.

Use DMARCian or similar tool to parse and visualize. Raw XML is readable
but impractical at volume.

---

## Report Structure

```xml
<feedback>
  <report_metadata>
    <org_name>google.com</org_name>
    <email>noreply-dmarc-support@google.com</email>
    <report_id>1234567890</report_id>
    <date_range>
      <begin>1713398400</begin>
      <end>1713484800</end>
    </date_range>
  </report_metadata>

  <policy_published>
    <domain>yourdomain.com</domain>
    <p>reject</p>
    <sp>reject</sp>
    <pct>100</pct>
  </policy_published>

  <record>
    <row>
      <source_ip>209.85.220.41</source_ip>
      <count>42</count>
      <policy_evaluated>
        <disposition>none</disposition>
        <dkim>pass</dkim>
        <spf>pass</spf>
      </policy_evaluated>
    </row>
    <identifiers>
      <header_from>yourdomain.com</header_from>
    </identifiers>
    <auth_results>
      <dkim>
        <domain>yourdomain.com</domain>
        <result>pass</result>
        <selector>selector1</selector>
      </dkim>
      <spf>
        <domain>yourdomain.com</domain>
        <result>pass</result>
      </spf>
    </auth_results>
  </record>
</feedback>
```

---

## Key Fields

| Field | Meaning |
|---|---|
| `org_name` | Receiving mail server that sent the report |
| `source_ip` | IP address the mail came from |
| `count` | Number of messages from that IP in the reporting period |
| `disposition` | What DMARC policy was applied — none, quarantine, or reject |
| `dkim` | DKIM alignment result — pass or fail |
| `spf` | SPF alignment result — pass or fail |
| `header_from` | The domain in the From header of the message |
| `selector` | Which DKIM selector signed the message |

---

## Interpreting Results

### Healthy report
- Known sending IPs — M365 outbound, authorized third parties
- DKIM pass + SPF pass on all legitimate sources
- Disposition reflects your current policy — none, quarantine, or reject
- Zero unknown sources with high message counts

### SPF fail + DKIM fail
- Unknown source sending as your domain
- Could be spoofing attempt or misconfigured third-party sender
- At p=reject these are blocked — verify disposition shows reject

### SPF pass + DKIM fail
- Message forwarded through another server — SPF breaks on forward
- DKIM should still pass on forwarded messages if signing is correct
- Persistent DKIM fail on known sources = key or selector issue

### SPF fail + DKIM pass
- Sender not included in SPF record
- DMARC still passes because DKIM alignment holds
- Add sender to SPF record to clean up the signal

### SPF fail + DKIM fail from your own IPs
- SPF misconfiguration — your sending infrastructure not authorized
- Verify `include:spf.protection.outlook.com` is in your SPF record
- Check for SPF lookup limit exceeded — max 10 DNS lookups

---

## Action Matrix

| DKIM | SPF | Source | Action |
|---|---|---|---|
| Pass | Pass | Known | No action — expected |
| Pass | Fail | Known | Add to SPF record |
| Fail | Pass | Known | Investigate DKIM config |
| Fail | Fail | Known | Full authentication investigation |
| Fail | Fail | Unknown | Spoofing attempt — verify p=reject is blocking |
| Pass | Pass | Unknown | Identify source — may be unauthorized sender using your domain |

---

## Policy Progression Triggers

Only advance DMARC policy when aggregate reports confirm:

**p=none -> p=quarantine**
- All legitimate senders passing DKIM and SPF
- No unknown sources generating volume
- RUA reports reviewed for minimum 2-4 weeks

**p=quarantine -> p=reject**
- Zero legitimate sources failing authentication
- Quarantine period showed no legitimate mail impacted
- All third-party senders aligned and validated

---

## Monitoring Cadence

| Frequency | Action |
|---|---|
| Weekly | Review DMARCian dashboard for new sources or failures |
| Monthly | Full report review — document findings in changelog |
| Quarterly | Verify reporting addresses still receiving reports |
| On change | Review immediately after any DNS or mail flow changes |

---

## Tools

| Tool | Purpose | URL |
|---|---|---|
| DMARCian | Aggregate report parsing and visualization | dmarcian.com |
| MXToolbox DMARC | Quick DMARC record validation | mxtoolbox.com/DMARC.aspx |
| Google Admin Toolbox | DMARC record syntax check | toolbox.googleapps.com |
| mail-tester.com | Full authentication test on sent message | mail-tester.com |

---

## Current Reporting Status

| Domain | RUA Address | Reports Receiving | Last Reviewed |
|---|---|---|---|
| `domain-1.io` | dmarc@domain-1.io | Yes | 2026-04-23 |
| `domain-2.dev` | dmarc@domain-1.io | Yes | 2026-04-23 |
| `domain-3.com` | dmarc@domain-1.io | Yes | 2026-04-23 |

---

*NextLayerSec*
*Last validated: April 2026*
