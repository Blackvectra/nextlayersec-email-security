# ARC (Authenticated Received Chain)

ARC (RFC 8617) preserves authentication results across intermediaries that
modify messages -- mailing lists, forwarders, gateways. Without ARC, those
intermediaries break SPF and often DKIM, causing DMARC to fail on otherwise
legitimate mail.

> Microsoft 365 implements ARC automatically as both signer and validator.
> There is nothing to configure for inbound mail. This doc explains the
> mechanism and how to verify it is working.

---

## The Problem ARC Solves

Without ARC:

```
sender -> SPF pass, DKIM pass -> mailing list -> reformats message ->
your tenant -> SPF fail (list's IP not in sender's SPF),
DKIM fail (body modified) -> DMARC fail -> message rejected
```

The mailing list did nothing wrong -- it just modified the message in a
normal way. But DMARC has no concept of "trusted intermediary."

---

## How ARC Works

Each intermediary that touches the message adds three headers:
- `ARC-Authentication-Results` -- the auth results it observed
- `ARC-Message-Signature` -- a DKIM-like signature over the headers and body
- `ARC-Seal` -- a signature over the previous two plus prior ARC sets

The receiving server can then chain through the ARC sets to verify what each
intermediary saw, and choose to trust the chain if all signatures validate
and the chain is unbroken.

DMARC does not natively consider ARC results -- the receiving MTA's policy
decides whether to honor them. M365 honors trusted ARC senders configured
in `Set-ArcConfig`.

---

## Adding Trusted ARC Senders in M365

If a known mailing list or forwarder repeatedly causes DMARC failures on
mail you want delivered, add it as a trusted ARC sealer:

```powershell
Connect-ExchangeOnline

# Add a trusted ARC sealer
Set-ArcConfig -Identity default -ArcTrustedSealers "list.example.org"

# Verify
Get-ArcConfig | Format-List ArcTrustedSealers
```

Use sparingly -- each trusted sealer is a delegation of authentication trust.
Only add intermediaries you have a documented relationship with.

---

## Verifying ARC on Inbound Mail

Inspect message headers (Outlook: File -> Properties -> Internet headers):

```
ARC-Authentication-Results: i=1; mx.list.example.org;
  dkim=pass header.d=sender.example
  spf=pass smtp.mailfrom=sender.example
  dmarc=pass header.from=sender.example

ARC-Message-Signature: i=1; a=rsa-sha256; c=relaxed/relaxed; d=list.example.org;
  s=arc20240101; ...

ARC-Seal: i=1; a=rsa-sha256; cv=none; d=list.example.org; s=arc20240101; ...

Authentication-Results: spf=fail (your tenant validated) dmarc=fail ARC=pass
```

`ARC=pass` confirms the chain validated. The presence of `cv=` field shows
the chain validity at each hop -- `cv=none` for the first hop, `cv=pass`
for subsequent.

---

## ARC and Outbound

When your tenant forwards mail to another organization, M365 adds your own
ARC seal. The downstream receiver can verify your seal to recover the
original auth results.

No configuration is needed. Verify by looking at the headers of any mail
you forward externally -- you should see an `ARC-Seal` header with
`d=<onmicrosoft-tenant>.onmicrosoft.com`.

---

## When ARC Does Not Help

- Forwarders that strip headers entirely (some legacy systems)
- Intermediaries that re-sign the message as themselves without ARC sealing
- Receivers that do not honor ARC chains (most do now, but not all)

If a critical workflow breaks despite ARC, the alternative is a
mail-flow rule that bypasses spam/DMARC for messages from a specific
sender + connector -- but this widens trust and should be carefully scoped.

---

## References

- RFC 8617 -- Authenticated Received Chain (ARC) Protocol
- [Microsoft ARC in Exchange Online](https://learn.microsoft.com/defender-office-365/email-authentication-arc-configure)
- [ARC at the IETF DMARC WG](https://datatracker.ietf.org/wg/dmarc/about/)
