# Conditional Access Policies

Entra ID Conditional Access policy baseline for an M365 Business Premium
tenant. These policies complement the Exchange Online hardening runbook --
the runbook turns off legacy protocols; CA enforces what the modern protocols
can do.

> Always test policies in **Report-only** mode for at least 24 hours before
> turning them On. Mis-scoped CA policies are the most common cause of
> tenant-wide lockouts.

---

## Required Foundation

Before deploying any of these policies:

1. Confirm at least one **break-glass account** exists, is excluded from every
   CA policy below, and has phishing-resistant MFA (FIDO2) registered. See
   `exchange-online/hardening-runbook.md` Break Glass section.
2. Confirm **Security Defaults** are disabled in Entra ID (CA replaces them).
3. Inventory third-party / on-prem services that depend on legacy auth before
   blocking it -- they will break.

---

## Policy 1 -- Block Legacy Authentication

| Field | Value |
|---|---|
| Users | All users |
| Exclusions | Break-glass account |
| Cloud apps | All cloud apps |
| Conditions -> Client apps | Exchange ActiveSync clients, Other clients (legacy auth) |
| Grant | Block access |

This is the single most impactful CA policy. Pair with the Exchange Online
`Set-AuthenticationPolicy` baseline -- CA blocks the auth attempt, the
auth policy blocks the protocol negotiation.

---

## Policy 2 -- Require MFA for All Users

| Field | Value |
|---|---|
| Users | All users |
| Exclusions | Break-glass account |
| Cloud apps | All cloud apps |
| Grant | Require MFA |
| Session | Sign-in frequency: 30 days |

Use phishing-resistant methods where possible:
- FIDO2 / WebAuthn
- Windows Hello for Business
- Microsoft Authenticator (number matching enabled)

Disable SMS and voice as MFA methods in tenant-wide authentication methods
policy. They satisfy the MFA requirement but are SIM-swap vulnerable.

---

## Policy 3 -- Require Compliant Device for Mobile Mail

| Field | Value |
|---|---|
| Users | All users |
| Exclusions | Break-glass account |
| Cloud apps | Office 365 Exchange Online, Office 365 |
| Conditions -> Device platforms | iOS, Android |
| Grant | Require device to be marked as compliant **OR** Require approved client app |

Approved client apps: Outlook for iOS/Android, no other mail apps. Combined
with Intune compliance, this scopes ActiveSync access to managed devices
only. This is what the hardening runbook references as "ActiveSync scoped
via Conditional Access" -- it lives here.

---

## Policy 4 -- Block Sign-In from Unsupported Countries

| Field | Value |
|---|---|
| Users | All users |
| Exclusions | Break-glass account, travelers |
| Cloud apps | All cloud apps |
| Conditions -> Locations | Include: All locations. Exclude: Named locations you operate from |
| Grant | Block access |

Define **Named Locations** in Entra ID first. Start with an allowlist of
countries you operate from (US, Canada, UK, etc.) plus any traveling
employee exceptions in a separate group.

This blocks the bulk of credential-stuffing volume. Travel exceptions are
managed by adding users to a "Travel" group with a different CA scope.

---

## Policy 5 -- Risky Sign-In Response

Requires Entra ID P1 or P2 (P2 includes risky-user policies as well).

| Field | Value |
|---|---|
| Users | All users |
| Exclusions | Break-glass account |
| Cloud apps | All cloud apps |
| Conditions -> Sign-in risk | High, Medium |
| Grant | Require MFA + Require password change (for High) |

Medium risk triggers re-MFA. High risk forces password reset. Entra ID
classifies risk based on impossible travel, anonymous IP, leaked credentials,
unfamiliar sign-in properties.

---

## Policy 6 -- Block Unmanaged Browser Access to Mailbox Data

| Field | Value |
|---|---|
| Users | All users |
| Exclusions | Break-glass account |
| Cloud apps | Office 365 Exchange Online |
| Conditions -> Client apps | Browser |
| Grant | Require app protection policy **OR** Require compliant device |

Prevents downloading attachments to an unmanaged personal laptop while
allowing OWA preview-in-browser usage from compliant devices.

---

## Policy 7 -- Require Phishing-Resistant MFA for Admins

| Field | Value |
|---|---|
| Users | Directory roles: Global Admin, Privileged Auth Admin, Exchange Admin, SharePoint Admin, Security Admin, etc. |
| Exclusions | Break-glass account |
| Cloud apps | All cloud apps |
| Grant | Require authentication strength: Phishing-resistant MFA |

This forces FIDO2 / Windows Hello / cert-based auth for any privileged role
assignment. Push-MFA alone is not sufficient for admin actions.

---

## Break-Glass Exclusion Pattern

Every policy above must exclude the break-glass account. The cleanest
pattern:

1. Create an Entra ID group named `BreakGlass-Excluded` (no dynamic
   membership -- explicit only)
2. Add the break-glass account(s) to that group
3. Exclude the group from every CA policy
4. Monitor sign-ins to the group via Entra ID sign-in alerts -- any sign-in
   to a break-glass account outside of a documented emergency is an
   incident

---

## Deployment Order

1. Deploy in **Report-only** mode, all policies
2. Watch the "What If" tool and sign-in logs for 48-72 hours
3. Resolve any unexpected blocks (most commonly: a service account hitting
   legacy auth, a vendor app, a travel user)
4. Flip to **On**, one policy per day, starting with Block Legacy Auth
5. Document in changelog
6. Re-validate quarterly

---

## Validation

Entra ID > Conditional Access > **Insights and reporting** workbook shows:
- Sign-ins blocked by each policy
- Users impacted
- Apps impacted

PowerShell (requires Microsoft.Graph module):

```powershell
Connect-MgGraph -Scopes "Policy.Read.All"

# List all CA policies
Get-MgIdentityConditionalAccessPolicy | Format-Table DisplayName, State

# Detail on one policy
Get-MgIdentityConditionalAccessPolicy -ConditionalAccessPolicyId <id> | Format-List
```

---

## What CA Does Not Cover

CA evaluates at sign-in. It does not:
- Inspect message content (Defender for Office 365 does that)
- Catch token theft after a successful sign-in (use Token Protection policy + Continuous Access Evaluation)
- Replace authentication policies in Exchange Online -- both layers are needed
- Apply to service principals -- use App Registrations / Workload Identities

---

## References

- [Microsoft CA documentation](https://learn.microsoft.com/entra/identity/conditional-access/)
- [Authentication strengths](https://learn.microsoft.com/entra/identity/authentication/concept-authentication-strengths)
- [Continuous Access Evaluation](https://learn.microsoft.com/entra/identity/conditional-access/concept-continuous-access-evaluation)
