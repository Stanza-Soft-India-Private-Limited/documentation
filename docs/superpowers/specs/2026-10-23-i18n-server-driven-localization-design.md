# Server-driven UI localization — backend design (2026-10-23)

Approved plan: `~/.claude/plans/so-here-is-the-jiggly-snowglobe.md` (24 decisions D1–D24). This file is the
backend lane's design; the mobile lane's is `upsc_app/docs/superpowers/specs/2026-10-23-i18n-mobile-design.md`.
Anchored on HEAD c90c3fa. Two places where the code pushes back on the decisions are marked **CONTRADICTION**.

## 0. Findings that shape everything
1. `AllExceptionsFilter` (`src/common/filters/exception.filter.ts:114-167`) keeps only `message`/`error` on a 400
   (a `code` survives only for 402 / 403 FEATURE_DISABLED / 409 QUESTION_WITHDRAWN). A bulk-upsert rejection therefore
   carries its offender list in `message: string[]` — the shape ValidationPipe already emits.
2. Express 5 (`express@5.2.1`) turns a 2xx into 304 automatically when `If-None-Match` matches an `ETag` the handler
   set (`res.send` → `req.fresh`). The service still short-circuits BEFORE loading the bundle so the 304 path is one
   small Redis read, never a DB hit.
3. `CacheModule` and `ExamsModule` are `@Global()`; `src/generated` is git-ignored so `npx prisma generate` is part of
   every gate; no global ThrottlerGuard (public bundle route is unthrottled — keep its 404 path DB-free).

## 1. Schema
English lives ONLY in `i18n_labels.source_text` (no `en` translation rows): one truth for placeholder parity, one row
per key on sync, `source_edited_by_sme` is a label property, compile for `en` is the degenerate (no-overlay) case.
Consequence to accept: every language's `published_bundle` is SELF-CONTAINED (English fallbacks baked in at publish)
because D10 forbids a bundle's content changing without its version moving. So a source edit makes every language
`draftDirty` until republished — the languages list says so.

Migration `prisma/migrations/20261023000000_i18n/migration.sql` (hand-applied, idempotent, long header comment in
the repo style; VARCHAR statuses not Postgres enums — same reasoning as exam_banners 20260917000001):
```sql
CREATE TABLE IF NOT EXISTS "public"."i18n_languages" (
    "code" VARCHAR(8) NOT NULL, "name" TEXT NOT NULL, "native_name" TEXT NOT NULL,
    "enabled" BOOLEAN NOT NULL DEFAULT false, "sort_order" INTEGER NOT NULL DEFAULT 0,
    "published_version" INTEGER NOT NULL DEFAULT 0, "published_bundle" JSONB, "published_at" TIMESTAMP(3),
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP, "updated_at" TIMESTAMP(3) NOT NULL,
    CONSTRAINT "i18n_languages_pkey" PRIMARY KEY ("code"));
INSERT INTO "public"."i18n_languages" ("code","name","native_name","enabled","sort_order","updated_at")
VALUES ('en','English','English',true,0,CURRENT_TIMESTAMP) ON CONFLICT ("code") DO NOTHING;
CREATE TABLE IF NOT EXISTS "public"."i18n_labels" (
    "key" VARCHAR(160) NOT NULL, "source_text" TEXT NOT NULL, "context" TEXT,
    "status" VARCHAR(16) NOT NULL DEFAULT 'active', "source_edited_by_sme" BOOLEAN NOT NULL DEFAULT false,
    "first_seen_app_version" VARCHAR(20), "retired_at" TIMESTAMP(3),
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP, "updated_at" TIMESTAMP(3) NOT NULL,
    CONSTRAINT "i18n_labels_pkey" PRIMARY KEY ("key"));
CREATE INDEX IF NOT EXISTS "i18n_labels_status_key_idx" ON "public"."i18n_labels" ("status","key");
CREATE TABLE IF NOT EXISTS "public"."i18n_translations" (
    "key" VARCHAR(160) NOT NULL, "lang" VARCHAR(8) NOT NULL, "value" TEXT NOT NULL,
    "status" VARCHAR(16) NOT NULL DEFAULT 'machine', "updated_by" TEXT,
    "created_at" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP, "updated_at" TIMESTAMP(3) NOT NULL,
    CONSTRAINT "i18n_translations_pkey" PRIMARY KEY ("key","lang"));
CREATE INDEX IF NOT EXISTS "i18n_translations_lang_updated_at_idx" ON "public"."i18n_translations" ("lang","updated_at");
DO $$ BEGIN ALTER TABLE "public"."i18n_translations" ADD CONSTRAINT "i18n_translations_key_fkey"
  FOREIGN KEY ("key") REFERENCES "public"."i18n_labels"("key") ON DELETE CASCADE ON UPDATE CASCADE;
EXCEPTION WHEN duplicate_object THEN NULL; END $$;
DO $$ BEGIN ALTER TABLE "public"."i18n_translations" ADD CONSTRAINT "i18n_translations_lang_fkey"
  FOREIGN KEY ("lang") REFERENCES "public"."i18n_languages"("code") ON DELETE CASCADE ON UPDATE CASCADE;
EXCEPTION WHEN duplicate_object THEN NULL; END $$;
ALTER TABLE "public"."user_profiles" ADD COLUMN IF NOT EXISTS "ui_language" VARCHAR(8);
```
⚠️ JwtStrategy `include: { userProfile: true }` selects every scalar → `ui_language` MUST exist before the image that
knows it deploys (staging: orchestrator applies; prod: user by hand). Verification: `information_schema.tables` has the 3
names, `information_schema.columns` has `user_profiles.ui_language`, `SELECT code, enabled FROM i18n_languages` = `en`.

Prisma models appended after `AppConfig` (~line 1667): `I18nLanguage` (`@@map("i18n_languages")`, `publishedVersion`,
`publishedBundle Json?`, relations), `I18nLabel` (`@@map("i18n_labels")`, `@@index([status, key])`), `I18nTranslation`
(`@@id([key, lang])`, `@@index([lang, updatedAt])`, cascade relations), and `UserProfile.uiLanguage String? @map("ui_language")
@db.VarChar(8)` (validated at write time against i18n_languages, NOT a FK). Side effect: `prisma generate` adds `uiLanguage` to
the generated admin `UpdateUserProfileDto` — acceptable.

## 2. Public endpoint — `/api/v1/i18n/*` (under the global prefix; `/config` is bare only for legacy clients)
New module `src/modules/i18n/`: `i18n.module.ts` (exports `I18nService`), `i18n.controller.ts` (`@ApiTags('i18n')`,
`@Public()`), `i18n.service.ts` (compile, getPublicLanguages, getBundle, bustCaches), `i18n.constants.ts`
(`LANG_CODE_PATTERN = /^[a-z]{2,3}(?:-[a-z0-9]{2,4})?$/`, `LABEL_KEY_PATTERN = /^[a-z][a-zA-Z0-9]*(?:[._][a-zA-Z0-9]+)*$/` — camelCase segments as mobile
emits them, e.g. `auth.getStarted.title`, `premium.trialDaysLeft_one`; mobile's parity test enforces the stricter style,
the server only guarantees it never rejects a valid mobile key — corrected in B2, the original lowercase-only pattern
rejected every mobile key, limits),
`placeholders.util.ts` (pure `extractPlaceholders`, `placeholderDiff`, `formatPlaceholderMismatch`), `dto/i18n-bundle.dto.ts` (**the contract**).
Register in `app.module.ts`; `SmeModule` and `UserModule` import it.

`GET /i18n/languages` — always 200, enabled only, ordered sortOrder,code; Redis `I18n:languages` (TTL Long, busted on any
language write). `{ "languages": [ {code,name,nativeName,sortOrder} ] }`. `en` always present.

`GET /i18n/bundle/:code?v=N` (`@Res({passthrough:true})`): (1) lowercase; bad pattern → 404. (2) cached languages list
(internally carries enabled+publishedVersion); unknown or disabled → 404 `Language "xx" is not available` (en served even if
disabled). (3) `?v` int equals publishedVersion OR `If-None-Match` equals `"<code>-<version>"` → set `ETag`, status 304,
no body, blob never loaded. (4) `getOrSet('bundle:<code>', ns I18n, TTL Long)` → 200 `{code, version, languages[], labels{}}`
+ `ETag` + `Cache-Control: no-cache`. Servable = enabled AND (publishedVersion>0 OR code='en'); `en` with version 0 is a live
compile (**CONTRADICTION with D13, accepted: the app is not dead the day the migration lands; publish en in rollout anyway**).
**`en` at version 0 NEVER answers 304** — neither `?v=0` nor `If-None-Match: "en-0"` (step 3 is skipped for it). Its content
changes with every sync / English edit while its version cannot move, so "you hold v0" proves nothing; it is a pre-publish
state only (rollout publishes `en` right after the first sync), so the extra full fetch is acceptable. The `ETag: "en-0"`
header is still sent, and the controller strips the request's `If-None-Match` for this response so Express 5's own freshness
check (§0.2) cannot re-impose the 304. _(Amended 2026-10-23: the first build answered 304 to `?v=0`; proven stale on a rig.)_
`labels` holds EVERY active key (English for untranslated — D4/D9). No nulls in the body. Why 304 not `{unchanged:true}`:
Ktor with expectSuccess OFF reads `status == NotModified` in one branch with no body parse; a second 200 shape is a trap.

Cache namespace: add `CacheNamespace.I18n = 'I18n'` to `src/common/services/cache.service.ts` with a comment that
`clearNamespace(ApiCache)` (SmeFilterConfigService) must never reach bundles; keys `languages`, `bundle:<code>`.

Compile(code): active labels ordered by key; `labels[key] = translation.value` if a row exists AND `placeholderDiff(source,
value).ok`, else `sourceText`; returns `{labels, keyCount, translatedCount, droppedForPlaceholderMismatch[]}`.

**CONTRADICTION with D17 + 304**: a 304 hides a newly enabled language from an app holding the current version → the
mobile switcher calls `GET /i18n/languages` when opened (always 200). Resolved in both designs.

## 3. SME endpoints — `@ApiTags('SME') @SmeApiKey() @Controller('sme/i18n')`
Files `src/modules/sme/controllers/sme-i18n.controller.ts`, `services/sme-i18n.service.ts`, `dto/sme-i18n.dto.ts`; wire in
`sme.module.ts`. Audit union additions in `services/sme-audit.service.ts`: `I18N_LANGUAGE_CREATE|UPDATE|PUBLISH`,
`I18N_LABEL_UPDATE`, `I18N_TRANSLATION_UPSERT|DELETE`, `I18N_KEYS_SYNC` (targetUserId null). Conventions: tri-state PATCH
(undefined untouched / null clears only nullable columns, 400 on non-nullable / value set), every write audited, raw bodies.

### 3.1 Languages
- `GET /sme/i18n/languages` → `{data:[{code,name,nativeName,enabled,sortOrder,publishedVersion,publishedAt,
  coverage:{activeKeyCount,translatedCount,missingCount,reviewedCount,machineCount,placeholderMismatchCount},
  draftDirty,lastDraftChangeAt,createdAt,updatedAt}], total}`. draftDirty = publishedAt IS NULL OR max(translations.updated_at
  for lang) > publishedAt OR max(labels.updated_at) > publishedAt. Three GROUP BY queries, not per row. `en`: translated=active.
- `POST /sme/i18n/languages` `CreateI18nLanguageDto`: code (`@Matches(LANG_CODE_PATTERN) @MaxLength(8)`, lowercased),
  name/nativeName (`@IsNotEmpty @MaxLength(60)`), sortOrder? (`@IsInt @Min(0) @Max(1000)`), enabled? (default false). 201;
  duplicate → 409 ConflictException. Busts `I18n:languages`.
- `PATCH /sme/i18n/languages/:code` `UpdateI18nLanguageDto` (all optional, none nullable → null = 400 "omit it to leave
  unchanged"); `en` + `enabled:false` → 400 "English is the fallback for every other language and cannot be disabled".
  Busts `I18n:languages` and `I18n:bundle:<code>` when enabled flips.
- `POST /sme/i18n/languages/:code/publish` (`@HttpCode(200)`): `$transaction` with `SELECT published_version,
  published_bundle FROM i18n_languages WHERE code=$1 FOR UPDATE` (mirrors filter-config.service.ts:303-378), compile from
  tx reads, diff vs old bundle, write version+1/bundle/published_at, then bust `bundle:<code>` + `languages`, audit
  `{before:{version}, after:{version,keyCount,translatedCount,missingCount,diff}}`. Response `{code,version,publishedAt,
  keyCount,translatedCount,missingCount,droppedForPlaceholderMismatch[],diff:{added,changed,removed}}`. Publishing a disabled
  or zero-translation language is allowed (D14).
- `GET /sme/i18n/languages/:code/preview` → public bundle shape + `draft:true, wouldBeVersion, droppedForPlaceholderMismatch`.

### 3.2 Labels
- `GET /sme/i18n/labels` query: page (≥1, default 1), limit (1..500, default 50), search (ILIKE key OR sourceText, ≤120),
  prefix (`/^[a-z0-9_.-]+$/i`, `key LIKE 'prefix%'`), status (`active|retired|all`, default active), lang, missingIn (implies
  lang; only keys without a translation). Row: `{key,sourceText,context,status,sourceEditedBySme,firstSeenAppVersion,retiredAt,
  placeholders[],updatedAt, translation?:{lang,value,status,placeholderOk,updatedBy,updatedAt}|null}`; `{data,total,page,limit,hasMore}`.
- `PATCH /sme/i18n/labels/:key` `UpdateI18nLabelDto`: sourceText? (`@IsNotEmpty @MaxLength(2000)`, null → 400),
  context? (`@MaxLength(500)`, null clears). A CHANGED sourceText sets `sourceEditedBySme=true` (a no-op save does not).
  If the placeholder set changed, response carries `affectedTranslations:[{lang,placeholderOk:false}]`. 404 unknown key.

### 3.3 Translations
- `GET /sme/i18n/translations/:code` query page/limit/status/missing/search/prefix; active keys; `translation` always present
  (null when missing). 404 unknown language; `en` → 400 "English is edited through PATCH /sme/i18n/labels/:key".
- `PUT /sme/i18n/translations/:code` `BulkUpsertI18nTranslationsDto { updatedBy?: ≤80, items: @IsArray @ArrayMinSize(1)
  @ArrayMaxSize(500) @ValidateNested({each}) @Type(() => I18nTranslationItemDto) }`; item `{key (@Matches LABEL_KEY_PATTERN
  ≤160), value (@IsNotEmpty ≤2000), status? ('machine'|'reviewed', default machine)}`. **All-or-nothing**: 400 with every
  offender in `message[]` as `[idx] key: KIND detail` for UNKNOWN_KEY, RETIRED_KEY, DUPLICATE_KEY, PLACEHOLDER_MISMATCH
  (expected vs got), EMPTY_VALUE; `en` → 400. One `$transaction` of upserts; 200 `{code,received,created,updated}`. Audit
  `{after:{code,received,created,updated,status,updatedBy,sampleKeys(≤20)}}`. Busts nothing (drafts invisible).
- `DELETE /sme/i18n/translations/:code/:key` → `{deleted:true,key,lang}`; 404 if none; audit with before.

### 3.4 Keys sync — `POST /sme/i18n/keys/sync` (`@HttpCode(200)`)
`SyncI18nKeysDto { appVersion (@Matches(/^\d+(\.\d+){0,2}$/) ≤20), labels: Record<string,{text,context?}> (@IsObject, inner
shape validated by hand: key LABEL_KEY_PATTERN ≤160, text non-empty ≤2000, context ≤500; offenders → 400 message[]),
dryRun?: boolean }`. Max 5,000 keys. One transaction: absent → insert active + first_seen_app_version (**added**); present &
retired → active, retired_at null, text per SME rule (**reactivated**); present & active & equal → **unchanged** (context still
updated if provided and different); text differs & !sourceEditedBySme → update (**updated**); text differs & sourceEditedBySme
→ no write, `conflicts.push({key,repo,server})`; every active label not in payload → retired + retired_at (**retired**). Never
deletes. Response `{appVersion,dryRun,added,updated,unchanged,reactivated,retired,conflicts[],activeKeyCount,addedKeys(≤50),
retiredKeys(≤50)}`. Audit `I18N_KEYS_SYNC` (skipped on dryRun). Busts nothing (bundles change only on publish).

### 3.5 Export — `GET /sme/i18n/export/:code[?fill=true]`
Raw `{key:value}` of DRAFT translations for active keys (en → sourceText), sorted; `fill=true` adds English for missing keys.

### 3.6 `uiLanguage` on the user surface
`src/modules/user/dto/update-my-account.dto.ts`: `@IsOptional() @IsString() @Matches(LANG_CODE_PATTERN) @MaxLength(8)
uiLanguage?: string | null`. `UserService.updateMyAccount` (`user.service.ts:491`): `if (dto.uiLanguage !== undefined &&
dto.uiLanguage !== null)` → lowercase, `i18n.assertServableCode(code)` (cached list; en always valid; otherwise enabled AND
published — the public routes' servable rule, so a stored code is always fetchable; 400 `Unknown or unavailable language
"xx"`), set. _(Amended 2026-10-23: was `assertEnabledCode`/enabled-only, which accepted an enabled-but-unpublished language
whose bundle 404s.)_ The DTO also trims + lowercases before `@Matches` (`HI` → `hi`). **null = unchanged** exactly like `whatsappExamIds` (`:496`) — the Kotlin client has explicitNulls=true.
`findByUserId` (`:302`) returns `uiLanguage: profile.uiLanguage ?? null` explicitly. SME: `formatUserSummary`
(`sme-user.service.ts:696`) + the two `userProfile.select` blocks (`:133`, `:244`) add `uiLanguage`. `UserModule` imports `I18nModule`.

## 4. Scripts (`backend/scripts/`, `npx ts-node -T`, mirror `verify-exam-backcompat.ts` / `seed-staging-exam-plans.ts`)
Key: `process.env.SME_API_KEY` else `readVar(--env ?? '.env.staging', 'API_KEY_SECRET')`; `--base` default
`https://staging-api.prepmonkey.com`; host `app.stanzasoft.ai` requires `--yes`; never print the key; non-zero exit on failure.
- `i18n-sync.ts --file <en.json> --app-version 2.2 [--base] [--env] [--dry-run] [--yes]`: accepts flat `{key:text}` with
  `key@context` siblings (the mobile format) or `{key:{text,context}}`; POSTs keys/sync; prints counts, then conflicts as a
  unified-style diff (`- server:` / `+ repo:`), exit 2 on conflicts; ends with the publish reminder.
- `i18n-seed-language.ts --code hi --file hi.json --name Hindi --native-name हिन्दी [--status machine|reviewed] [--updated-by]
  [--publish] [--enable] [--base] [--env] [--yes]`: GET languages → create if absent (disabled unless --enable); drop keys not
  active (printed as skipped); PUT in batches of 200; on 400 print every message[] line and stop; `--publish` prints
  version/missingCount/dropped; ends with the coverage line.

## 5. Tests (positional-constructor harness like `sme-banner.service.spec.ts:48-70`)
`src/modules/i18n/placeholders.util.spec.ts` (table-driven: identical; reordered ok; `{count}` vs `{n}` names both; missing;
extra; duplicates ok; `{{name}}` = placeholder `name` plus literal braces — no escape syntax; `{ }`,`{1}`,`{a b}`,`{}` literal;
Devanagari around tokens; none/none). `i18n.service.spec.ts` (compile overlay, retired excluded, mismatch dropped+reported;
getBundle 304 w/o touching bundle cache; If-None-Match; unknown/disabled 404; en served when disabled; en v0 live; namespace
`I18n` asserted). `i18n.controller.spec.ts` (304 sets ETag + no body; 200 sets ETag + Cache-Control). `sme-i18n.service.spec.ts`
(publish row-locked, version N→N+1, both cache dels, audit diff; en disable 400 no write; create 409; sync never overwrites
sme-edited, retires absent via update never delete, reactivates, idempotent re-run all unchanged, dryRun writes/audits
nothing; bulk: one offender rejects all, `$transaction` not called, en 400, status default machine). `user.service.spec.ts`
extended (null untouched; unknown 400; lowercased). Gate per item: `npx prisma generate && npx tsc --noEmit && npx jest
<paths> && npx nest build`.

## 6. Docs
`docs/sme/SME_LOCALIZATION_API.md` (SME_FILTER_CONFIG_API.md format: banner, base URL/auth/Swagger, no-envelope +
frontend-design boxes; §0 mental model — keys born in the app, English = sourceText, draft vs publish, self-contained
snapshots and why a sync dirties every language, placeholder rule and stale-translation behaviour, machine/reviewed is a
work queue, en cannot be disabled, what the app calls; §1 languages, §2 labels, §3 translations with the exact `message[]`
rejection, §4 keys sync (portal never calls it), §5 export, §6 user uiLanguage, §7 public contract; Common errors; audit
actions). README banner + Contents row (`— n/a` exam status; languages are global). `rollout-and-verify.md` §9 (migration,
information_schema checks, smoke: bundle/en 200+ETag, ?v= 304, /bundle/xx 404, keyless /sme/i18n/languages 401).
Every example re-captured LIVE from staging in B9.

## 7. Items (serial, one commit each) — see the plan's Lane B list B1..B9.
Prod deploy order (user by hand): apply migration → deploy image → `i18n-sync.ts --base https://app.stanzasoft.ai --yes`
→ seed + publish + enable `hi` → mobile 2.2 release.

## 8. Traps for every brief
forbidNonWhitelisted global (new body fields need DTO props; column before image); explicitNulls on the Kotlin client (null =
unchanged for uiLanguage); `clearNamespace(ApiCache)` collateral (bundles live in `I18n:*`, never behind CacheInterceptor);
JwtStrategy scalar select (migration first); hand-applied idempotent migrations verified via information_schema never
`_prisma_migrations`; no envelope, 400 offenders in `message[]`; Swagger tags; CORS untouched; self-contained snapshots
documented; `@Type()` mandatory on nested arrays; retire never deletes (dryRun first; an empty en.json would retire everything
— reversible by re-sync); `src/generated` ignored → `prisma generate` in every gate; placeholder grammar `{identifier}` only,
no escape, identical on both lanes; unknown codes answered from the cached list, never a DB lookup.
