# ProFin Modèle de données

Inventaire des entités du modèle de données, chacune décrite par son **rôle**. Les champs, types et relations sont laissés à l'implémentation.

**Conventions**

- _prebuilt_ : fourni par le framework ou un package (non modélisé à la main).
- _« instrument technique »_ : terme de travail pour les instruments internes non investissables.

---

## Vue d'ensemble

```
modele-donnees/
│
├── references                    (socle partagé — shared kernel)
│
├── Cœur investissement
│   ├── clients
│   ├── tiers-et-comptes
│   ├── portefeuilles
│   ├── instruments
│   ├── transactions
│   ├── market-data
│   ├── frais
│   ├── taxation
│   ├── comptabilite
│   ├── actions-corporatives
│   ├── fonds
│   └── pension
│
├── Services et plateforme       (hors cœur métier)
│   ├── securite-acces
│   ├── maker-checker
│   ├── am-services-ia
│   ├── notifications
│   ├── taches-batch
│   ├── documents
│   ├── audit
│   └── integration
│
├── Portail client
│   └── profin-online
│
└── Reporting
    └── reporting
```

---

## Référentiel

### `references`

Le socle de données de référence , liste de modeles utilitaires

- **Currency** — les devises opérées (HTG / USD / EUR).
- **Country** — la liste ISO des pays (adresses, domicile, nationalité).
- **Calendar** — les calendriers de jours ouvrés nommés (Haïti, US).
- **CalendarHoliday** — les dates fériées par calendrier.
- **CurrencyPair** — la définition d'une paire FX (base / quote), pas le taux.
- _Enums partagés_ — `Language`, `BusinessDayConvention`, `DayCountConvention`, `PeriodFrequency` (voir `enums.md`).

---

## Cœur investissement

### `clients`

- **Client** — l'entité servie par ProFin ; un même client peut tenir plusieurs rôles sur des portefeuilles (bénéficiaire, mandataire, etc.).
- **ConseillerEnPlacement (CP)** — l'advisor qui porte la relation et le KYC, rattaché à ses clients.
- **ProfilKYC** — la référence légère du profil ; le KYC complet vit dans le CRM.
- **GroupeClient** — une catégorisation logique des clients (sert aux règles de frais et de taxation).

### `tiers-et-comptes`

- **Émetteur** — l'entité qui émet un instrument (sociétés, BRH, Raymond James), avec sa société de valorisation externe.
- **CompteBancaire** — les comptes en banque (omnibus, clients) et leur catégorie.

### `portefeuilles`

- **Portefeuille** — le conteneur des avoirs d'un client, dans une devise et une entité, avec sa politique de frais, son benchmark et son statut.
- **RolePortefeuille** — associe un **client** à un portefeuille avec un rôle (bénéficiaire, PoA, UBO, signataire).
- **GroupePortefeuille** — un regroupement logique (ex. retail 1, retail 2…).

### `instruments`

- **Instrument** — la définition commune à tous les titres et flux ; chaque famille en précise les spécificités.
- **Action ordinaire** — titres de capital de sociétés privées, négociés de gré à gré, valorisés en externe.
- **Action privilégiée** — dividende fixe (type A) ou sur décision (type B), mécanisme HTG/USD éventuel.
- **Obligation ordinaire** — taux fixe, coupon, retenue à la source.
- **Obligation BRH** — protection HTG/USD et rachat anticipé.
- **Fonds NAOS** — fonds mutuels internes, à NAV calculée.
- **Fonds étranger (RJ)** — prix NAV externe.
- **Cash / compte** — liquidités HTG / USD.
- **Instrument technique** — regroupe les flux internes non investissables : income, expense, blocage / réservation de disponibilité (ordre en attente), provisions, et autres au besoin.
- **Benchmark / indice** — un instrument de type indice (dont l'indice HTG/USD), dont les valeurs sont des prix.
- **GroupeInstrument** — un regroupement logique d'instruments.

### `transactions`

- **Transaction** — l'événement qui modifie un avoir ou un solde ; porte son type, ses dates, ses montants, son statut de workflow et son maker/checker.
- Types : achat, vente, souscription, rachat, dividende, coupon/intérêt, remboursement, transfert / mouvement interne, saisie de transaction, FX deal (client) et FX deal interne, frais, dépôt, reversement, accrual.

### `market-data`

- **Prix** — le prix courant de chaque instrument (y compris indices et benchmarks).
- **PrixHistorique** — l'historique des prix.
- **TauxFX** — les taux entre devises (HTG / USD).

### `frais`

Frais configurables, distincts des frais intrinsèques d'un instrument (commission émetteur, retenue, taxe).

- **RègleFrais** — définit quel frais s'applique selon le type de transaction et le périmètre (groupe client, groupe portefeuille, groupe instrument, émetteur) ; barèmes flat / % / paliers, frais périodiques (gestion, advisory, custody) et de performance (high watermark, hurdle).

### `taxation`

Même logique de règles configurables, étroitement liée aux frais.

- **RègleTaxation** — détermine une retenue ou la variation d'un prélèvement selon le groupe (client, portefeuille, instrument, émetteur) et le type de flux.

### `comptabilite`

- **PlanComptable** — un plan par entité (ProFin, NAOS, par fonds).
- **CompteComptable** — les comptes du plan.
- **ModèleÉcriture** — les gabarits de journaux par type de transaction et d'instrument.
- **ÉcritureComptable** — les écritures générées, agrégées pour l'export vers Zoho Books.

### `actions-corporatives`

Volontairement simple.

- **ÉvénementCorporatif** — pour le MVP, les dividendes (actions) et les distributions (fonds), avec éligibilité et paiement.
- **InstructionClient** — le choix du client pour un événement électif.

### `fonds`

Administration des fonds, _à approfondir._

- **ClasseFonds** — les classes d'un fonds.
- **PlanSystématique (SIP)** — les souscriptions récurrentes sur les fonds NAOS.
- etc.

### `pension`

_À approfondir._

- **Employeur**, **Employé**, **Contribution**, **PlanPension** (type, fréquence, vesting, allocation vers fonds NAOS).
- etc.

---

## Services et plateforme (hors cœur métier)

### `securite-acces`

Les utilisateurs (interne LDAP / portail), les rôles et permissions (RBAC, maker/checker), les menus et écrans visibles, et l'**entité** qui ségrège les données qu'un utilisateur voit.

### `maker-checker`

Le modèle d'approbation générique, activable par module et applicable à n'importe quelle entité : une action (création, modification, suppression) saisie par un **maker** reste en attente jusqu'à validation par un **checker**.

- **RègleApprobation** — définit les entités et modules soumis au double contrôle, le nombre de validations requises et les rôles habilités.
- **DemandeApprobation** — l'action en attente, avec son contenu, son demandeur (maker) et son statut (en attente, approuvée, rejetée).
- **DécisionApprobation** — la validation ou le rejet d'un checker, avec horodatage, motif et traçabilité.

### `am-services-ia`

Le service de surveillance et d'IA (détection d'anomalies, contrôles de cohérence, surveillance) ; il lit le cœur, sans entités métier propres.

### `notifications`

La production et la diffusion des messages (in-app / email, criticité, templates FR/EN) ; la couche WebSocket est _prebuilt_ (Channels + Redis).

### `taches-batch`

L'ordonnancement et les résultats des traitements, _prebuilt_ par les packages Celery ; les tâches (accruals, NAV, export GL, PDF) sont du code.

### `documents`

Les métadonnées des fichiers ; les binaires vivent dans MinIO.

### `audit`

Des tables d'audit qui conservent les traces de modifications (qui, quoi, quand) ; rétention BRH 10 ans.

### `integration`

Les échanges avec Zoho (CRM / Books) et le suivi des imports de migration depuis Axia.

---

## Portail client (app autonome — base séparée)

### `profin-online`

Le compte d'accès du client, l'authentification 2FA par email, les préférences (langue, affichage) ; les sessions sont _prebuilt_. Aucune donnée financière n'y est stockée.

---

## Reporting

### `reporting`

L'ensemble des rapports de la plateforme à identifier : positions et soldes (dérivés des transactions, pas des entités stockées), AUM, allocations, P&L, cash ledger, états de compte client, rapports BRH, etc.
