# User Phish Reporting

End-user reporting is a sensor, not a defense. Every reported message is
either training data for ZAP / Defender or a real campaign to investigate.
Make reporting frictionless and respond visibly when users do it.

---

## Reporting Channels

Three options, in increasing order of integration:

| Option | Where it goes | Pros | Cons |
|---|---|---|---|
| Outlook built-in "Report" button | Microsoft + admin mailbox | No add-in, default in new Outlook | Less customizable |
| Microsoft Report Message / Report Phishing add-in | Microsoft + admin mailbox | Same data, works in classic Outlook | Add-in deployment required |
| Custom add-in or mailto button | Admin mailbox only | Full control of UX | No Microsoft signal feedback |

Use the **Outlook built-in button** for new tenants. Deploy the **Report
Message add-in** if you have users on classic Outlook on Windows.

---

## Configuration

```powershell
Connect-ExchangeOnline

# Enable user reporting tenant-wide
Set-ReportSubmissionPolicy -Identity DefaultReportSubmissionPolicy `
  -EnableReportToMicrosoft $true `
  -ReportJunkToCustomizedAddress $true `
  -ReportNotJunkToCustomizedAddress $true `
  -ReportPhishToCustomizedAddress $true `
  -ReportJunkAddresses "phish-reports@<domain>" `
  -ReportNotJunkAddresses "phish-reports@<domain>" `
  -ReportPhishAddresses "phish-reports@<domain>" `
  -PostSubmitMessageEnabled $true `
  -PreSubmitMessageEnabled $true

# Verify
Get-ReportSubmissionPolicy | Format-List
```

Settings:
- **EnableReportToMicrosoft: true** -- Microsoft uses reports to train their classifier
- **ReportPhishToCustomizedAddress + ReportPhishAddresses** -- copy to your security mailbox for tracking
- **PostSubmitMessageEnabled / PreSubmitMessageEnabled** -- confirmation dialogs that make the report feel acknowledged

---

## Reporting Mailbox Setup

`phish-reports@<domain>` should be:
- A shared mailbox (no license needed)
- Members of the security team
- Inbox rule to auto-tag and route based on subject
- Mailbox audit fully enabled

```powershell
# Create shared mailbox
New-Mailbox -Shared -Name "Phish Reports" -DisplayName "Phish Reports" -Alias phish-reports

# Grant security team access
Add-MailboxPermission -Identity phish-reports@<domain> `
  -User secops@<domain> `
  -AccessRights FullAccess `
  -InheritanceType All

# Enable audit
Set-Mailbox -Identity phish-reports@<domain> -AuditEnabled $true -AuditLogAgeLimit 180
```

---

## Triage Workflow

For each reported message:

1. **Verify** -- open the message in Defender Threat Explorer (Email entity page) to see the full delivery context, headers, authentication results
2. **Classify** as: confirmed phish / spam / false report / legitimate
3. **Hunt** for the same campaign across other mailboxes -- search by sender, subject pattern, URL
4. **Purge** matches via Threat Explorer "Take action"
5. **Block** the IOCs via [Tenant Allow/Block List](tenant-allow-block.md)
6. **Notify** the reporting user that action was taken (this is critical for ongoing reporting)
7. **Submit** to Microsoft via the user-submitted phish queue if not auto-submitted

---

## Threat Explorer Pivot

```
Defender portal -> Email & collaboration -> Explorer
  -> Filter: All email
  -> Add filter: Sender = <reported sender>
  -> Time range: last 7 days
```

Per-message actions available:
- Soft delete from mailbox
- Hard delete
- Submit to Microsoft as user-reported phish
- Move to inbox / junk / quarantine

---

## Metrics Worth Tracking

| Metric | Target | Why |
|---|---|---|
| User reports per month | Higher = better engagement | Falling = users have stopped trusting the process |
| % of reports confirmed phish | 30-70% typical | <10% = users over-reporting (noise); >90% = real attacks are getting through |
| Median time from report to action | < 4 business hours | Users notice if reports go nowhere |
| % of reports notified back to user | 100% | Acknowledgment drives future reporting |

---

## User Communication

After action is taken:

```
Subject: Thanks for reporting -- action taken

Hi <user>,

Thanks for flagging the message from <sender>. We confirmed it was a
phishing attempt and have:
- Removed it from your inbox
- Blocked the sender across the organization
- Searched for and removed copies sent to other users

You did not need to do anything else.

If you clicked any link or entered credentials, please reply to this email
or call <secops-extension> immediately.

-- Security Team
```

Send this even for false positives -- explain why it was actually legitimate
so the user calibrates.

---

## Anti-Patterns

- **Silent triage**: users stop reporting if nothing visibly happens
- **Punishing false positives**: kills the sensor entirely
- **No metrics**: cannot tell if the channel is healthy
- **Single-person dependency**: shared mailbox + on-call rotation, not one person's inbox

---

## References

- [Microsoft user-reported settings](https://learn.microsoft.com/defender-office-365/submissions-user-reported-messages-files-custom-mailbox)
- [Report Message add-in](https://learn.microsoft.com/defender-office-365/submissions-users-report-message-add-in-configure)
