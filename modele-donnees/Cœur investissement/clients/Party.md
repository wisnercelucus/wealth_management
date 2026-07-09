# Party (Tiers)

The single identity for every person and organization ProFin references. Lean identity only.
What a party _is_ → [PartyRole](PartyRole.md) via [PartyRoleMembership](PartyRoleMembership.md);
profile / client data → [PartyProfile](PartyProfile.md); emails / phones →
[PartyContact](PartyContact.md); addresses → [PartyAddress](PartyAddress.md); bank accounts →
[BankAccount](BankAccount.md).

## Essential fields

| Field       | Type        | Req | Description                                                                                |
| ----------- | ----------- | --- | ------------------------------------------------------------------------------------------ |
| `id`        | uuid/bigint | ●   | Surrogate PK                                                                               |
| `type`      | enum        | ●   | `PERSON \| ORGANIZATION`                                                                   |
| `code`      | string(50)  | ●   | Party code — unique business key (join on `id`, not this)                                  |
| `name`      | string      | ●   | Given / common name — a person's **first name**, or an organization's common name          |
| `legalName` | string      | ○   | Legal / family name — a person's **last name**, or an organization's registered legal name |
| `status`    | enum        | ●   | `ACTIVE \| ONBOARDING \| BLOCKED \| CLOSED`                                                |
| + envelope  |             | ●   | see README                                                                                 |

> Two name slots cover both types: a PERSON is `name` (first) + `legalName` (last); an
> ORGANIZATION is `name` (common) + `legalName` (registered). Display name is derived.

## Not included

No KYC, contact, address, or tax on Party — identity only. Those live on the extensions
(PartyProfile, PartyContact, PartyAddress) or in Zoho (KYC).

## Clean model

```
Party
  id          uuid    PK
  type        enum (PERSON | ORGANIZATION)
  code        string  unique
  name        string             -- first name (persons) / common name (orgs)
  legalName   string?            -- last name (persons) / legal name (orgs)
  status      enum (ACTIVE | ONBOARDING | BLOCKED | CLOSED)
  + envelope
```
