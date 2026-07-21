# Mains "Optional" toggle + SME-managed filter config — design spec (condensed)

> 2026-07-22. Backend portion of the cross-repo plan (mobile toggle ships separately).

## Context

Two asks: (1) a big optional-subject mains dump encoded optionality in the subject *name*
(`"Economics Optional"`, 6,403 of 7,466 rows) — make it a real `isOptional` flag with a
Mains-screen toggle; (2) move subject→icon/filter config off the mobile app's static map
into a backend-served, SME-manageable config (assets stay bundled; backend assigns keys).

## Decisions (interview, 2026-07-21/22)

- `isOptional Boolean @default(false)` **column on `mains_papers`** (no new table); backfill
  `subject LIKE '% Optional'` — verified exact against prod. Prelims has no optional concept.
- Config storage: **`filter_config` JSONB on the `app_config` singleton** —
  `{prelims:{[rawSubject]:{displayName?,iconKey?,hidden?,sortOrder?}}, mains:{…}}`.
  Defaults auto-derived from live data; SME entries are overrides; merge is SERVER-side.
- Delivery: `GET /mains/filters` + `GET /pyq/filters` gain
  `subjectsMeta: [{subject, displayName, iconKey, isOptional}]` (hidden dropped, sorted by
  `sortOrder ?? MAX` then displayName). Legacy `subjects` (old builds/web) = non-optional AND
  non-hidden raw names — the optionals dump disappears for old builds.
- Ingest: create paths derive `isOptional ?? subject.endsWith(' Optional')`
  (MainsService.create + CmsMainsService.create/bulkCreate — the real dump path).
- SME can also bulk-flip a mains subject's `isOptional` (audited).

## Key validated flaws baked in

- **F1** no bool aggregate in Prisma → `groupBy(['subject','isOptional'])` + OR-reduce in TS
  (subject optional if ANY row is).
- **F2** `/filters` routes are Redis-cached 1h (`ApiCache` ns) → `filter_config` is read
  **fresh from DB** in getFilters (no in-process cache; runs only on Redis miss) and every
  SME write calls `cacheService.clearNamespace(ApiCache)`.
- **F3** `optional` query param mirrors `bookmarked`'s `@Transform`; services check
  `!== undefined` (NEVER truthiness — `optional=false` is the toggle's default state).
  Absent param → no clause. (Verified: class-transformer 0.5 does not run Transform on
  absent keys.)
- **F4** `getMetrics` gets the same optional scoping (incl. bookmark intersection); metrics
  matches subject EXACT vs list `contains` → clients always send RAW `subject`;
  `displayName` is render-only.
- **F6** nestjsDto drops `@default` fields → `/// @DtoCreateOptional` +
  `/// @DtoUpdateOptional` on the schema field + regenerate.
- **F8** default displayName strips `" Optional"` ONLY when isOptional=true — an SME flip
  to false shows the raw name (no silent mislabel).
- **F9** SME JSON read-modify-write inside `$transaction` with
  `SELECT "filter_config" … FOR UPDATE`; audit before/after = ONLY the affected subject's
  override entry.
- **F10** canonical icon keys (11, mirror the app's bundled `sme_*` assets): sme_art,
  sme_current_affairs, sme_economy, sme_environment, sme_ethics, sme_geography,
  sme_governance, sme_history, sme_ir, sme_polity, sme_science_technology.

## API shapes

Public (enriched, back-compat):

```
GET /mains/filters → { subjects: string[]   // legacy: non-optional + non-hidden raw names
                     , years, difficulties
                     , subjectsMeta: [{subject, displayName, iconKey, isOptional}] }
GET /pyq/filters   → same; isOptional always false
GET /mains?optional=true|false&…      // absent = legacy behavior
GET /mains/metrics?optional=true|false&…
```

SME (x-api-key, `@SmeApiKey`, audited, cache-busting):

```
GET  /sme/filter-config/:examType            → {examType, validIconKeys, entries:[{subject,
       isOptional, questionCount, effective:{displayName,iconKey,hidden,sortOrder}, override|null}]}
PUT  /sme/filter-config/:examType/:subject   body {displayName?,iconKey?,hidden?,sortOrder?}
       // omitted=untouched, null=clear, value=set; 400 unknown iconKey/examType
       // audit FILTER_CONFIG_UPDATE
POST /sme/filter-config/mains/:subject/optional  body {isOptional}
       // updateMany all rows of subject; 404 if 0; audit MAINS_OPTIONAL_FLIP w/ counts
```

## Backend pieces

- Migration `20260722000000_add_mains_is_optional_and_filter_config` (idempotent,
  hand-applied on prod; backfill expects 6,403 optional rows).
- `src/modules/app-config/filter-config.service.ts` — VALID_ICON_KEYS, default icon
  derivation (exact map + contains chain ported from mobile `getSmeIconForSubject`, default
  `sme_history`), F8 name rule, `buildSubjectFilters`, row-locked `updateOverride`.
- `src/modules/sme/{controllers,services,dto}/sme-filter-config.*` — portal surface.
- Docs: `docs/sme/SME_PORTAL_API.md` §6 (handed to portal team).
- Tests: `mains.service.filters.spec.ts`, `sme-filter-config.service.spec.ts`.

## Rollout order (load-bearing)

1. Hand-apply pending `20260720000000_trial_extension` on prod, 2. hand-apply this
migration (verify 6,403 backfilled), 3. `mau deploy` backend + smoke `/mains/filters`,
4. mobile ships in next store build (old builds + web keep working on legacy `subjects`).
