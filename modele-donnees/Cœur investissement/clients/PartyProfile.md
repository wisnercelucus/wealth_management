# PartyProfile

The profile extension 1:1 with a [Party](Party.md). Primarily a client's profile (category,
regulatory classifications, KYC / CRM references), but named generally since the same 1:1
extension holds any party-level profile data. Full KYC detail stays in Zoho; no tax (taxation layer).
Addresses live in [PartyAddress](PartyAddress.md); emails / phones in [PartyContact](PartyContact.md).

## Essential fields

| Field                   | Type        | Req | Description                                                                                          |
| ----------------------- | ----------- | --- | ---------------------------------------------------------------------------------------------------- |
| `id`                    | uuid/bigint | ●   | Surrogate PK                                                                                         |
| `partyId`               | FK→Party    | ●   | The party — unique (1:1)                                                                             |
| `category`              | enum        | ○   | Client type: `WEALTH \| PENSION \| NAOS` (clients only; HNW is portfolio-level, IPE)                 |
| `investorStatus`        | enum        | ○   | Statut d'investisseur: `AVERTI \| NON_AVERTI`                                                        |
| `riskProfile`           | enum        | ○   | Profil de risque: `FAIBLE \| MOYEN_FAIBLE \| MOYEN \| MOYEN_ELEVE \| TRES_ELEVE`                     |
| `birthDate`             | date        | ○   | Date of birth (persons) — pension / age                                                              |
| `sex`                   | enum        | ○   | `MALE \| FEMALE` (persons) — KYC / statement narrative                                               |
| `isMinor`               | bool        | ○   | Whether the person is a minor (persons)                                                              |
| `maritalStatus`         | enum        | ○   | État civil (persons): `CELIBATAIRE \| MARIE \| DIVORCE \| VEUF \| UNION_LIBRE`                       |
| `profession`            | string      | ○   | Profession / occupation (persons)                                                                    |
| `citizenshipCountryId`  | FK→Country  | ○   | Citizenship — AML / regulatory                                                                       |
| `residenceCountryId`    | FK→Country  | ○   | Country of residence — tax / AML                                                                     |
| `isPep`                 | bool        | ○   | Politically Exposed Person flag (PPE) — AML; detail stays in Zoho                                    |
| `reportingCurrencyId`   | FK→Currency | ○   | Primary statement / portal display currency _(optional)_                                             |
| `preferredLanguage`     | enum        | ○   | Correspondence language — shared `Language` enum (`FR \| EN`); statements / notifications                                      |
| `cloisonnementEntityId` | FK→Entity   | ○   | Chinese-wall segregation unit (security layer); the client **default** inherited by their portfolios |
| `kycReference`          | string      | ○   | Pointer to the KYC record in Zoho                                                                    |
| `zohoCrmLink`           | string      | ○   | Zoho CRM record link                                                                                 |
| `clientSince`           | date        | ○   | Client opening date                                                                                  |
| `closureDate`           | date        | ○   | Client dossier closure date                                                                          |
| `closureNote`           | string      | ○   | Free-text note on dossier closure                                                                    |
| + envelope              |             | ●   | see README                                                                                           |

## Notes & rules

- 1:1 with Party (`partyId` unique). Client-specific fields (`category`, the classifications) are
  set for client parties; `birthDate` applies to any person (e.g. pension employees).
- **KYC stays in Zoho** — `kycReference` / `zohoCrmLink` are the only KYC footprint here. The two
  classifications and citizenship / residence are kept locally because the platform _acts_ on them
  (eligibility, AML); `birthDate` because pension needs it.
- `profession` and `maritalStatus` (état civil) are kept locally on the profile; deeper KYC detail
  and ID numbers stay in Zoho / [PartyDocument](PartyDocument.md).
- `isPep` flags a Politically Exposed Person (PPE) for AML controls — detail / screening stays in
  Zoho / the AML module. `isMinor` is stored explicitly (not only derived from `birthDate`) for
  guardianship / eligibility. `closureDate` + `closureNote` record dossier closure; the lifecycle
  state itself is `Party.status = CLOSED`.
- `preferredLanguage` drives the language of statements / notifications (default FR if unset); `sex`
  applies to persons only.
- `cloisonnementEntityId` is the **client default**; a portfolio's own `cloisonnementEntityId` (on
  `Portfolio`) is the authoritative grain and may override it.

## Clean model

```
PartyProfile
  id                    uuid   PK
  partyId               FK Party  unique          -- 1:1
  category              enum?  (WEALTH | PENSION | NAOS)
  investorStatus        enum?  (AVERTI | NON_AVERTI)
  riskProfile           enum?  (FAIBLE | MOYEN_FAIBLE | MOYEN | MOYEN_ELEVE | TRES_ELEVE)
  birthDate             date?
  sex                   enum?  (MALE | FEMALE)
  isMinor               bool?
  maritalStatus         enum?  (CELIBATAIRE | MARIE | DIVORCE | VEUF | UNION_LIBRE)
  profession            string?
  citizenshipCountryId  FK Country?
  residenceCountryId    FK Country?
  isPep                 bool?
  reportingCurrencyId   FK Currency?
  preferredLanguage     enum?  Language (FR | EN)
  cloisonnementEntityId FK Entity?            -- client default; portfolios inherit/override
  kycReference          string?
  zohoCrmLink           string?
  clientSince           date?
  closureDate           date?
  closureNote           string?
  + envelope
```
