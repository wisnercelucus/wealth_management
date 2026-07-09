# Party / Tiers cluster Model Reference

The identity layer the rest of the platform points at (`PortfolioParty.partyId → Party`). One
unified identity for every person and organization ProFin references — clients, advisors,
issuers, banks, fund managers, valuation companies, employers, employees. Role-specific data on
extensions, never duplicated: _identity once, relationships separate._

## Models

| Model                                         | Purpose                                                                   |
| --------------------------------------------- | ------------------------------------------------------------------------- |
| [Party](Party.md)                             | The single identity (PERSON / ORGANIZATION)                               |
| [PartyRole](PartyRole.md)                     | The role catalog (CLIENT, EMETTEUR, BANQUE, …)                            |
| [PartyRoleMembership](PartyRoleMembership.md) | Party ↔ role (M:N)                                                        |
| [PartyProfile](PartyProfile.md)               | Party/client profile extension (1:1) — category, classifications, KYC ref |
| [PartyContact](PartyContact.md)               | Party emails / phones (1:many)                                            |
| [PartyAddress](PartyAddress.md)               | Party postal addresses, typed (1:many)                                    |
| [PartyGroup](PartyGroup.md)                   | Party groupings (M:N)                                                     |
| [BankAccount](BankAccount.md)                 | ProFin's own bank accounts                                                |
| [PartyDocument](PartyDocument.md)             | Party identity documents, typed (1:many) — PII                            |

## Relationships

```
Party         1───* PartyRoleMembership  *───1 PartyRole
Party         1──1  PartyProfile          (the party/client extension)
Party         1───* PartyContact
Party         1───* PartyDocument
Party         1───* PartyAddress
Party         1───* PartyGroupMembership  *───1 PartyGroup
Party(BANQUE) 1───* BankAccount
Party         1───* PortfolioParty          (→ portfolio cluster)
```

## Module extensions (designed with their module)

Some parties carry module-specific data in tables that FK `Party` and are defined in their own
module, not here. When an employer or employee is created in the pension / fund module, identity
goes to `Party` (+ profile / contacts / addresses) and the module-specific fields go to its
extension:

- `PensionEmployer` — employer business fields (sector, employee count, company type, registration
  numbers: numéro d'identification, matricule, patente, registration date).
- `PensionEmployee` — employment / enrollment fields (position, hire date, contract type, salary,
  plan entry date; national ID if pension reporting needs it).

## Reference FKs (→ référentiel)

`Country` and `Currency` (`citizenshipCountryId`, `residenceCountryId`, address `countryId`,
`reportingCurrencyId`) live in the **référentiel** layer, and `preferredLanguage` uses its shared
`Language` enum. `cloisonnementEntityId` → `Entity` lives in the **security** layer (not référentiel).

## Shared envelope — on every table

```
id          uuid / bigint   PK
isActive    bool            soft-delete — never hard-delete
externalId  string          external-system id (integration / reconciliation)
externalRef string?         external-system reference
createdAt   datetime
createdBy   FK→User
modifiedAt  datetime
modifiedBy  FK→User
```
