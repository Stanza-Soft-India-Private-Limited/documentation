# SME — UI Localization API Guide

> Changed 2026-10-23 — new surface.

Reference for the **SME portal** to manage the app's **UI languages**: which languages exist,
the translation of every on-screen label, and when a language goes live. Same
server-to-server model as the rest of the SME surface: we expose the endpoints, the SME team
builds the UI.

**Base URL:** `{{BASE_URL}}/api/v1` (prod `BASE_URL` = `https://app.stanzasoft.ai`)
**Authentication:** `x-api-key: <API_KEY_SECRET>` header on **every** `/sme/*` request.
**Swagger:** `/api/docs` (group **SME** for `/sme/i18n/*`, group **i18n** for the two public routes).

> **Examples below were captured live from staging (`https://staging-api.prepmonkey.com`) on
> 2026-10-23, backend commit `3952b34`.** Every JSON body in this doc is a real response (or a
> marked excerpt of one). Staging holds the app's real registry (1,261 active keys), `en`
> (published v3), Hindi `hi` (published v4, every key translated, all `machine`) and `zz`, a
> **disabled test language**. **`zz` is the only language the examples write to** — the
> translation writes and the language PATCH below were made against `zz` and undone afterwards;
> nothing here published, PATCHed or translated `hi` / `en`, and the keys sync is a `dryRun`.
> Examples that cannot be produced on staging without mutating `hi` / `en` (or without an app
> user) are kept from the earlier local rig (`60bdd89`, an 8-key mini registry with Hindi and a
> disabled Tamil `ta`; the §5 refusal at `045ab74`) and are **marked "Rig capture"** where they
> appear. Timestamps are UTC, from whichever host produced the capture.

> **There is no response envelope.** Success bodies are raw; errors are shaped by the
> exception filter (`success: false`, `message`, `error`, `statusCode`, …). Branch on the HTTP
> status code, never on a `success` field. List endpoints carry their own
> `{ data, total, page, limit, hasMore }` (or `{ data, total }`) wrapper, documented per endpoint.

> **Invoke the `frontend-design` skill before building any screen from this doc.** The
> localization screens are a work-queue tool (what is missing, what needs review, what is
> unpublished) — design them around those three questions, not around a raw table.

> **Languages are global, not per exam.** Nothing on this surface takes `?exam=`; one Hindi
> serves UPSC, APPSC and TPSC alike.

---

## 0. The mental model (read this first)

**Keys are born in the app, values are edited here.** Every piece of UI text has a dotted
key (`premium.trialDaysLeft_other`, `common.ok`) that a developer creates in the app repo's
`en.json`. Once per app release a developer runs a **keys sync** (§4) that pushes the
complete registry to the server. So:

- the portal edits **values only** — it never creates or deletes a key;
- a **new key** appears in the portal after the release that introduced it is synced;
- a key the app **stopped using** is **RETIRED, never deleted** (`status: "retired"`,
  `retiredAt` set). Its translations stay in the database; if a later release uses the key
  again, the sync reactivates it with its translations intact.

**English is a label's `sourceText`.** There is no "English translation" row: English lives
on the label itself and is edited with `PATCH /sme/i18n/labels/:key` (§2). Every translation
endpoint refuses `en` with `400 English is edited through PATCH /sme/i18n/labels/:key`. When an
SME changes the English, the label is marked **`sourceEditedBySme: true` — and the SME's
English wins over the repo forever**: later syncs never overwrite it, they report a
**conflict** instead (§4), for the developer to copy into `en.json`.

**Drafts vs Publish.** Every edit — a translation, an English correction, a sync — changes
**drafts**. **Nothing reaches a device until the language is published**
(`POST /sme/i18n/languages/:code/publish`). Publish compiles the drafts into a snapshot and
bumps that language's `publishedVersion` by **exactly 1**. Each language row tells you whether
it has unpublished changes: `draftDirty: true` + `lastDraftChangeAt` (the latest write that
affects it).

**Snapshots are SELF-CONTAINED.** A published bundle contains **every active key**: the
translation where there is a usable one, the **English baked in** everywhere else. That is
what lets a phone render a complete screen from one download, and why a version number can
be trusted: **a bundle's LABELS never change without its version moving.** (The `languages[]`
list inside a bundle is different: it can go stale behind a 304 — a language enabled after the
phone's last 200 is not in it. The switcher therefore calls `GET /i18n/languages`, which is
always a 200, and never trusts the list inside a bundle.) The consequence the
portal must show: **an English edit, or a sync that changes anything, makes EVERY language
`draftDirty`** — each language's snapshot holds its own copy of that English — so **republish
each language** (en first) for the change to reach everyone. A context-only change (the
translator note) also flags every language; republishing then ships an identical bundle under
a new version, which is harmless.

**Placeholders.** A label may carry `{identifier}` placeholders: `{count} days left`,
`Switched to {examName}`. The rule, enforced at write time:

- a translation must contain **exactly the same SET** of placeholder names as its English —
  reordering is fine (`{examName} पर स्विच किया`), repeating is fine, renaming / dropping /
  adding is not;
- the identifier is `[a-z][a-zA-Z0-9]*` and is never translated: `{count}` stays `{count}`;
- there is **no escape syntax**: `{ }`, `{1}`, `{a b}`, `{}`, `{Name}` are literal text.
  `{{name}}` is the placeholder `name` inside literal braces.

A bulk write with **one** bad item is rejected **whole** and writes nothing (§3). A translation
can still go **stale** later: if an SME edits the English and changes its placeholder set, the
existing translations no longer match. Nothing breaks on the phone — at publish a stale
translation is **silently replaced by the English** and listed in
`droppedForPlaceholderMismatch`; the language row counts it in
`coverage.placeholderMismatchCount`; and the `PATCH` that caused it returns
`affectedTranslations` so the portal can say so at the moment of the edit.

**"Translated" has TWO meanings — show them as two numbers.**

| Where | `translatedCount` means |
|---|---|
| `GET /sme/i18n/languages` → `coverage` | **rows present**: active keys that have a draft translation of any status, stale ones included |
| `POST …/publish`, `GET …/preview` | **translations actually used**: rows present **minus** the ones dropped for a placeholder mismatch |

On the local rig (the second publish in §1.4) Hindi showed `translatedCount: 4` in the list
and `translatedCount: 3` in the publish response at the same moment: four rows existed, one was
stale.

**`machine` vs `reviewed` is a work queue, not a gate.** Every translation carries a
`status`. `machine` (the default) is "produced by a person or a tool, not yet checked";
`reviewed` is "a reviewer signed it off". **Both are published.** The status exists so the
portal can show a review queue (`GET /sme/i18n/translations/hi?status=machine`) and a
progress bar (`coverage.reviewedCount` / `machineCount`).

**`en` cannot be disabled.** It is every other language's fallback. `PATCH …/languages/en
{"enabled": false}` is a 400.

**A language is visible to the apps only when it is `enabled` AND published**
(`publishedVersion ≥ 1`). Recommended order: **create (disabled) → seed translations →
publish → enable.** A language that is enabled but never published stays invisible (nothing
to serve); a published but disabled one is a staged release, waiting for the switch.
English is the exception: it is always served.

**What the app actually calls** (both public, no key — §6):

- `GET /api/v1/i18n/languages` when the language switcher opens — the list of languages it
  may offer;
- `GET /api/v1/i18n/bundle/:code?v=<version it holds>` on launch and on switch. If it already
  holds the published version it gets a **304 with an empty body** (one tiny request); a new
  version comes back as a **200** with the full bundle; a **404** means "this language is not
  available" (unknown, disabled, or never published) and the app **reverts to English**.

So the full loop for a portal edit is: **edit → (review) → publish → the next app launch
picks it up.** Disabling a language takes effect on the app's next call; devices that had it
selected fall back to English.

---

## 1. Languages

### 1.1 List — with coverage and draft state
```
GET /sme/i18n/languages
```
Every language, **disabled ones included**, ordered by `sortOrder` then `code`.

```json
{
  "data": [
    {
      "code": "en",
      "name": "English",
      "nativeName": "English",
      "enabled": true,
      "sortOrder": 0,
      "publishedVersion": 3,
      "publishedAt": "2026-10-23T00:56:50.176Z",
      "coverage": {
        "activeKeyCount": 1261,
        "translatedCount": 1261,
        "missingCount": 0,
        "reviewedCount": 0,
        "machineCount": 0,
        "placeholderMismatchCount": 0
      },
      "draftDirty": true,
      "lastDraftChangeAt": "2026-10-23T03:33:30.518Z",
      "createdAt": "2026-10-22T22:51:35.766Z",
      "updatedAt": "2026-10-23T00:56:50.179Z"
    },
    {
      "code": "hi",
      "name": "Hindi",
      "nativeName": "हिन्दी",
      "enabled": true,
      "sortOrder": 0,
      "publishedVersion": 4,
      "publishedAt": "2026-10-23T00:56:51.245Z",
      "coverage": {
        "activeKeyCount": 1261,
        "translatedCount": 1261,
        "missingCount": 0,
        "reviewedCount": 0,
        "machineCount": 1261,
        "placeholderMismatchCount": 0
      },
      "draftDirty": true,
      "lastDraftChangeAt": "2026-10-23T03:33:30.518Z",
      "createdAt": "2026-10-22T23:05:11.552Z",
      "updatedAt": "2026-10-23T00:56:51.254Z"
    },
    {
      "code": "zz",
      "name": "Zz test",
      "nativeName": "Zz",
      "enabled": false,
      "sortOrder": 0,
      "publishedVersion": 1,
      "publishedAt": "2026-10-22T23:08:47.155Z",
      "coverage": {
        "activeKeyCount": 1261,
        "translatedCount": 2,
        "missingCount": 1259,
        "reviewedCount": 1,
        "machineCount": 1,
        "placeholderMismatchCount": 0
      },
      "draftDirty": true,
      "lastDraftChangeAt": "2026-10-23T03:33:30.518Z",
      "createdAt": "2026-10-22T23:08:05.395Z",
      "updatedAt": "2026-10-22T23:08:58.669Z"
    }
  ],
  "total": 3
}
```

| Field | Meaning |
|---|---|
| `code` | Lowercase, permanent: it is the bundle URL and the value stored on user profiles |
| `name` / `nativeName` | English name / the language's own name. The app's switcher shows `nativeName` first |
| `enabled` | Offered to the apps (still needs a publish — §0) |
| `sortOrder` | Switcher order, ascending; ties broken by `code` |
| `publishedVersion` | `0` = never published. +1 per publish, never otherwise |
| `publishedAt` | When the live snapshot was published; `null` = never |
| `coverage.activeKeyCount` | Active keys — the size of every bundle |
| `coverage.translatedCount` | Active keys with a draft row (**rows present**, stale included). `en`: all of them |
| `coverage.missingCount` | `activeKeyCount − translatedCount` — keys that will show English |
| `coverage.reviewedCount` / `machineCount` | Split of `translatedCount` by status. `en`: 0 / 0 |
| `coverage.placeholderMismatchCount` | Of `translatedCount`: stale rows that publish will drop to English |
| `draftDirty` | Drafts differ from what the apps are served: never published, or a translation of this language **or any English label** changed after `publishedAt`. Show a "Publish" call to action |
| `lastDraftChangeAt` | The latest write that affects this language (its translations, or any label) |

Reading the capture above: `en` is live at v3 and Hindi at v4 — every one of the 1,261 keys
translated, all of them still `machine` (none reviewed yet). `zz` is disabled, was published
once (v1), and has 2 of 1,261 keys translated. **All three are `draftDirty`**, with the same
`lastDraftChangeAt`: a `zz` translation was deleted after the last publishes, and a deletion
touches the label, which flags **every** language (§3.3). The list response is wrapped as
`{ data, total }`.

### 1.2 Create
```
POST /sme/i18n/languages
```
| Body field | Type | Description |
|---|---|---|
| code | string | **required.** 2–3 letters, optionally `-` + 2–4 alphanumerics (`hi`, `ta`, `pt-br`). Case-insensitive on input, stored lowercase. **Permanent** |
| name | string | **required.** English name, ≤ 60 chars |
| nativeName | string | **required.** The language's own name, ≤ 60 chars |
| sortOrder | number | optional, 0–1000, default 0 |
| enabled | boolean | optional, default **false** — leave it off; enable after the first publish |

```json
// POST /sme/i18n/languages
{
  "code": "HI",
  "name": "Hindi",
  "nativeName": "हिन्दी",
  "sortOrder": 1
}
```
Response **201** — the new row, in the list shape (note `HI` was stored as `hi`):
> _Rig capture (local rig, `60bdd89`): staging examples never create a language._
```json
{
  "code": "hi",
  "name": "Hindi",
  "nativeName": "हिन्दी",
  "enabled": false,
  "sortOrder": 1,
  "publishedVersion": 0,
  "publishedAt": null,
  "coverage": {
    "activeKeyCount": 8,
    "translatedCount": 0,
    "missingCount": 8,
    "reviewedCount": 0,
    "machineCount": 0,
    "placeholderMismatchCount": 0
  },
  "draftDirty": true,
  "lastDraftChangeAt": "2026-10-22T22:12:37.363Z",
  "createdAt": "2026-10-22T22:12:37.444Z",
  "updatedAt": "2026-10-22T22:12:37.444Z"
}
```
Errors: **409** the code exists; **400** bad code / blank name. The 409, live:
```json
// POST /sme/i18n/languages   {"code": "zz", "name": "Zz test", "nativeName": "Zz"}
{
  "success": false,
  "message": "Language \"zz\" already exists",
  "error": "Conflict",
  "statusCode": 409,
  "timestamp": "2026-10-23T03:39:41.797Z",
  "path": "/api/v1/sme/i18n/languages",
  "method": "POST"
}
```
Audit: `I18N_LANGUAGE_CREATE`.

### 1.3 Update — MERGE, no field is nullable
```
PATCH /sme/i18n/languages/:code
```
Body: any of `name`, `nativeName`, `sortOrder`, `enabled`. Field semantics are **tri-state
with no null**: **omitted = untouched, value = set, `null` = 400**. Enabling or disabling
takes effect on the apps' next call; renames and reordering show on the next languages call.

```json
// PATCH /sme/i18n/languages/zz      (restored to sortOrder 0 straight after)
{
  "sortOrder": 90
}
```
Response **200** — the updated row:
```json
{
  "code": "zz",
  "name": "Zz test",
  "nativeName": "Zz",
  "enabled": false,
  "sortOrder": 90,
  "publishedVersion": 1,
  "publishedAt": "2026-10-22T23:08:47.155Z",
  "coverage": {
    "activeKeyCount": 1261,
    "translatedCount": 2,
    "missingCount": 1259,
    "reviewedCount": 1,
    "machineCount": 1,
    "placeholderMismatchCount": 0
  },
  "draftDirty": true,
  "lastDraftChangeAt": "2026-10-23T03:33:30.518Z",
  "createdAt": "2026-10-22T23:08:05.395Z",
  "updatedAt": "2026-10-23T03:39:42.185Z"
}
```

The two 400s, verbatim:
```json
// PATCH /sme/i18n/languages/zz   {"enabled": null}
{
  "success": false,
  "message": "`enabled` is not nullable; omit it to leave it unchanged",
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-10-23T03:39:41.969Z",
  "path": "/api/v1/sme/i18n/languages/zz",
  "method": "PATCH"
}
```
> _Rig capture (local rig, `60bdd89`): staging examples never PATCH `en`._
```json
// PATCH /sme/i18n/languages/en   {"enabled": false}
{
  "success": false,
  "message": "English is the fallback for every other language and cannot be disabled",
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-10-22T22:12:37.547Z",
  "path": "/api/v1/sme/i18n/languages/en",
  "method": "PATCH"
}
```
**A PATCH that changes nothing is a no-op.** Only fields whose value actually differs from the
stored one (names compared after trimming) are written. Re-sending the current values — or an
empty body — writes nothing, clears no cache, records **no** audit row, and returns the row
unchanged with a 200 (the same rule as `PATCH /sme/i18n/labels/:key`). So the portal may send
the whole form on Save. (On staging, re-sending `zz`'s current `name`, `nativeName`,
`sortOrder` and `enabled` returned 200 with `updatedAt` unchanged, and no audit row was written.)

Errors: **404** unknown code. Audit: `I18N_LANGUAGE_UPDATE` (before/after) — only when
something changed.

### 1.4 Publish — ship the drafts
```
POST /sme/i18n/languages/:code/publish
```
No body. Compiles every **active** key — this language's translation where it exists **and**
its placeholders match, the English otherwise — stores the snapshot, bumps the version by
exactly 1, and makes it live on the apps' next check. Allowed for a disabled language (staging
it before the switch) and for one with zero translations. Concurrent publishes of the same
language serialize (row lock), so two clicks give v2 and v3, never two v2s.

> _Rig capture (local rig, `60bdd89`): both publish examples — staging examples never publish._

First publish of Hindi:
```json
{
  "code": "hi",
  "version": 1,
  "publishedAt": "2026-10-22T22:12:37.852Z",
  "keyCount": 8,
  "translatedCount": 5,
  "missingCount": 3,
  "droppedForPlaceholderMismatch": [],
  "diff": {
    "added": 8,
    "changed": 0,
    "removed": 0,
    "addedKeys": [
      "common.cancel",
      "common.language.title",
      "common.ok",
      "common.retry",
      "nav.examSwitched",
      "premium.trialDaysLeft_one",
      "premium.trialDaysLeft_other",
      "profile.title"
    ],
    "changedKeys": [],
    "removedKeys": []
  }
}
```

Second publish, after an SME edited two English labels (one of them changing a placeholder set)
and deleted one Hindi translation:
```json
{
  "code": "hi",
  "version": 2,
  "publishedAt": "2026-10-22T22:12:38.258Z",
  "keyCount": 8,
  "translatedCount": 3,
  "missingCount": 5,
  "droppedForPlaceholderMismatch": [
    "nav.examSwitched"
  ],
  "diff": {
    "added": 0,
    "changed": 3,
    "removed": 0,
    "addedKeys": [],
    "changedKeys": [
      "common.cancel",
      "nav.examSwitched",
      "profile.title"
    ],
    "removedKeys": []
  }
}
```

| Field | Meaning |
|---|---|
| `version` / `publishedAt` | The new live version and time |
| `keyCount` | Keys in the bundle (= active keys) |
| `translatedCount` | Translations **actually used** (stale ones excluded — §0) |
| `missingCount` | `keyCount − translatedCount`: keys served in English |
| `droppedForPlaceholderMismatch` | Keys whose translation exists but was replaced by English because its placeholders no longer match. Fix them, then publish again |
| `diff` | Key-level comparison with the **previous snapshot**: `added` / `changed` / `removed` are exact counts; `addedKeys` / `changedKeys` / `removedKeys` list up to 50 keys each, sorted. On a first publish every key is `added` |

Read the second diff: `common.cancel` changed because its translation was deleted (now
English); `nav.examSwitched` because its translation went stale (now the new English);
`profile.title` because the SME reworded the **English** — Hindi had no translation for it,
so the English baked into the Hindi snapshot moved. That third line is the self-contained rule
in action.

Errors: **404** unknown code. Audit: `I18N_LANGUAGE_PUBLISH` (old/new version, counts, up to
50 dropped keys, the diff).

### 1.5 Preview — what publish would ship
```
GET /sme/i18n/languages/:code/preview
```
Writes nothing. The public bundle shape (§6) compiled from the **current drafts**, plus
`draft: true`, `wouldBeVersion`, and the same counts as publish. Use it for a "preview in
Hindi" panel and to show `droppedForPlaceholderMismatch` before the SME presses Publish. Works
for disabled languages. `version` is the currently published one.

```json
// GET /sme/i18n/languages/hi/preview   — excerpt: 3 of the 1,261 labels
{
  "code": "hi",
  "version": 4,
  "languages": [
    {
      "code": "en",
      "name": "English",
      "nativeName": "English",
      "sortOrder": 0
    },
    {
      "code": "hi",
      "name": "Hindi",
      "nativeName": "हिन्दी",
      "sortOrder": 0
    }
  ],
  "labels": {
    "auth.error.appleSignInFailed": "Apple से साइन इन नहीं हो सका: {detail}",
    "auth.error.appleTokenExchangeFailed": "Apple टोकन एक्सचेंज नहीं हो सका: {detail}",
    "auth.error.backendExchangeFailed": "सर्वर पर टोकन एक्सचेंज नहीं हो सका: {detail}"
  },
  "draft": true,
  "wouldBeVersion": 5,
  "keyCount": 1261,
  "translatedCount": 1261,
  "missingCount": 0,
  "droppedForPlaceholderMismatch": []
}
```
Errors: **404** unknown code.

---

## 2. Labels — the key registry and the English

### 2.1 List
```
GET /sme/i18n/labels
```
| Query | Description |
|---|---|
| page / limit | default 1 / 50; `limit` ≤ 500. Key ascending |
| search | case-insensitive substring of the key **or** the English text |
| prefix | keys starting with this (case-sensitive), e.g. `premium.` |
| status | `active` (default) · `retired` · `all` |
| lang | attach that language's draft as `translation` on every row (`null` when missing). Not `en` |
| missingIn | only keys with **no** translation in that language — the translator's queue. Implies `lang`; if both are sent they must match |

Every row carries `placeholders` — the `{identifier}`s every translation must reproduce — so
the editor can show them as chips and validate as the SME types.

```json
// GET /sme/i18n/labels?prefix=premium.expiry.&lang=hi   — excerpt: the first 2 of the 5 rows
{
  "data": [
    {
      "key": "premium.expiry.active",
      "sourceText": "Active",
      "context": "Subscription status when the expiry date is missing",
      "status": "active",
      "sourceEditedBySme": false,
      "firstSeenAppVersion": "2.2",
      "retiredAt": null,
      "placeholders": [],
      "updatedAt": "2026-10-22T23:04:49.148Z",
      "translation": {
        "lang": "hi",
        "value": "एक्टिव",
        "status": "machine",
        "placeholderOk": true,
        "updatedBy": "claude-hi-v1",
        "updatedAt": "2026-10-22T23:05:12.041Z"
      }
    },
    {
      "key": "premium.expiry.activeUntil",
      "sourceText": "Active until {date}",
      "context": "Subscription status line; {date} is e.g. 12 Jul 2026",
      "status": "active",
      "sourceEditedBySme": false,
      "firstSeenAppVersion": "2.2",
      "retiredAt": null,
      "placeholders": [
        "date"
      ],
      "updatedAt": "2026-10-22T23:04:49.148Z",
      "translation": {
        "lang": "hi",
        "value": "{date} तक एक्टिव",
        "status": "machine",
        "placeholderOk": true,
        "updatedBy": "claude-hi-v1",
        "updatedAt": "2026-10-22T23:05:12.041Z"
      }
    }
  ],
  "total": 5,
  "page": 1,
  "limit": 50,
  "hasMore": false
}
```
`translation.placeholderOk: false` flags a stale row (§0).

```json
// GET /sme/i18n/labels?missingIn=zz&limit=2
{
  "data": [
    {
      "key": "auth.error.appleSignInFailed",
      "sourceText": "Apple sign in failed: {detail}",
      "context": "error shown in a banner/toast; {detail} is the technical reason (server or exception text), keep it last",
      "status": "active",
      "sourceEditedBySme": false,
      "firstSeenAppVersion": "2.2",
      "retiredAt": null,
      "placeholders": [
        "detail"
      ],
      "updatedAt": "2026-10-22T23:33:50.975Z",
      "translation": null
    },
    {
      "key": "auth.error.appleTokenExchangeFailed",
      "sourceText": "Apple token exchange failed: {detail}",
      "context": "error shown in a banner/toast; {detail} is the technical reason (server or exception text), keep it last",
      "status": "active",
      "sourceEditedBySme": false,
      "firstSeenAppVersion": "2.2",
      "retiredAt": null,
      "placeholders": [
        "detail"
      ],
      "updatedAt": "2026-10-22T23:33:50.975Z",
      "translation": null
    }
  ],
  "total": 1259,
  "page": 1,
  "limit": 2,
  "hasMore": true
}
```
> _Rig capture (local rig, `60bdd89`): staging has no retired label today (live: `total: 0`), and retiring one takes a real keys sync._
```json
// GET /sme/i18n/labels?status=retired      (rows carry no `translation` without `lang`)
{
  "data": [
    {
      "key": "dashboard.oldBanner",
      "sourceText": "New: daily quiz",
      "context": "Removed in 2.2",
      "status": "retired",
      "sourceEditedBySme": false,
      "firstSeenAppVersion": "2.1",
      "retiredAt": "2026-10-22T22:12:37.363Z",
      "placeholders": [],
      "updatedAt": "2026-10-22T22:12:37.363Z"
    }
  ],
  "total": 1,
  "page": 1,
  "limit": 50,
  "hasMore": false
}
```
Errors: **400** `lang=en` (verbatim below), `lang` ≠ `missingIn`; **404** unknown language.
```json
// GET /sme/i18n/labels?lang=en
{
  "success": false,
  "message": "English is edited through PATCH /sme/i18n/labels/:key",
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-10-23T03:39:39.781Z",
  "path": "/api/v1/sme/i18n/labels?lang=en",
  "method": "GET"
}
```

### 2.2 Edit the English (or the translator note)
```
PATCH /sme/i18n/labels/:key
```
| Body field | Type | Description |
|---|---|---|
| sourceText | string | The English, ≤ 2,000 characters (Unicode code points). **Not nullable** (omit to leave it). A **changed** value sets `sourceEditedBySme: true` — the repo can no longer overwrite it (§4). Saving the same text is a no-op |
| context | string \| null | Translator note, ≤ 500 characters (code points). `null` or `""` clears it. ⚠️ **Developer-owned when the repo has one** — see below |

Drafts only: the change reaches devices when each language is republished (every language
becomes `draftDirty`). Retired labels are editable.

**The translator note is NOT protected like the English.** Only `sourceText` has an SME-wins
rule. If the app repo carries a note for the key (`"<key>@context"` in `en.json`), the **next
release sync overwrites the SME's note with the repo's** — silently, no conflict reported (it
counts under `contextUpdated`). The note is developer-owned: an SME note survives only on keys
whose repo entry has no `@context`. To change a repo-owned note for good, ask the developer to
change it in `en.json`.

If the edit changes the **placeholder set**, `affectedTranslations` lists the languages whose
translation no longer matches — those keys will fall back to English at their next publish
until someone fixes them:
> _Rig capture (local rig, `60bdd89`): all three label examples — editing a label's English changes it for every language, `hi` / `en` included._
```json
// PATCH /sme/i18n/labels/nav.examSwitched   {"sourceText": "Now preparing for {examName} ({examCode})"}
{
  "key": "nav.examSwitched",
  "sourceText": "Now preparing for {examName} ({examCode})",
  "context": "Snackbar after the exam switcher changes exam",
  "status": "active",
  "sourceEditedBySme": true,
  "firstSeenAppVersion": "2.1",
  "retiredAt": null,
  "placeholders": [
    "examName",
    "examCode"
  ],
  "updatedAt": "2026-10-22T22:12:38.149Z",
  "affectedTranslations": [
    {
      "lang": "hi",
      "placeholderOk": false
    }
  ]
}
```
Same placeholders, different words — nothing affected:
```json
// PATCH /sme/i18n/labels/profile.title
{
  "sourceText": "Your profile",
  "context": "Profile screen title (SME wording)"
}
```
```json
{
  "key": "profile.title",
  "sourceText": "Your profile",
  "context": "Profile screen title (SME wording)",
  "status": "active",
  "sourceEditedBySme": true,
  "firstSeenAppVersion": "2.1",
  "retiredAt": null,
  "placeholders": [],
  "updatedAt": "2026-10-22T22:12:38.077Z",
  "affectedTranslations": []
}
```
Errors: **400** `sourceText` null (below) or blank; **404** unknown key. Audit:
`I18N_LABEL_UPDATE` (before/after, plus the affected languages).
```json
{
  "success": false,
  "message": "`sourceText` is not nullable; omit it to leave it unchanged",
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-10-22T22:12:38.102Z",
  "path": "/api/v1/sme/i18n/labels/profile.title",
  "method": "PATCH"
}
```

---

## 3. Translations

### 3.1 Get one language's drafts
```
GET /sme/i18n/translations/:code
```
Every **active** key is a row (the label fields of §2.1), and `translation` is always present —
`{ lang, value, status, placeholderOk, updatedBy, updatedAt }`, or `null` when missing.

| Query | Description |
|---|---|
| page / limit | default 1 / 50; `limit` ≤ 500 |
| missing | `true` → only untranslated keys |
| status | `machine` \| `reviewed` → only keys whose translation has that status (the review queue) |
| search / prefix | as on labels |

`missing=true` and `status` together are a 400 (`status` filters existing translations).

```json
// GET /sme/i18n/translations/hi?status=machine&limit=2
{
  "data": [
    {
      "key": "auth.error.appleSignInFailed",
      "sourceText": "Apple sign in failed: {detail}",
      "context": "error shown in a banner/toast; {detail} is the technical reason (server or exception text), keep it last",
      "status": "active",
      "sourceEditedBySme": false,
      "firstSeenAppVersion": "2.2",
      "retiredAt": null,
      "placeholders": [
        "detail"
      ],
      "updatedAt": "2026-10-22T23:33:50.975Z",
      "translation": {
        "lang": "hi",
        "value": "Apple से साइन इन नहीं हो सका: {detail}",
        "status": "machine",
        "placeholderOk": true,
        "updatedBy": "claude-hi-v1",
        "updatedAt": "2026-10-22T23:36:30.532Z"
      }
    },
    {
      "key": "auth.error.appleTokenExchangeFailed",
      "sourceText": "Apple token exchange failed: {detail}",
      "context": "error shown in a banner/toast; {detail} is the technical reason (server or exception text), keep it last",
      "status": "active",
      "sourceEditedBySme": false,
      "firstSeenAppVersion": "2.2",
      "retiredAt": null,
      "placeholders": [
        "detail"
      ],
      "updatedAt": "2026-10-22T23:33:50.975Z",
      "translation": {
        "lang": "hi",
        "value": "Apple टोकन एक्सचेंज नहीं हो सका: {detail}",
        "status": "machine",
        "placeholderOk": true,
        "updatedBy": "claude-hi-v1",
        "updatedAt": "2026-10-22T23:36:30.532Z"
      }
    }
  ],
  "total": 1261,
  "page": 1,
  "limit": 2,
  "hasMore": true
}
```
```json
// GET /sme/i18n/translations/hi?missing=true&status=machine
{
  "success": false,
  "message": "`status` filters existing translations; it cannot be combined with missing=true",
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-10-23T03:39:40.068Z",
  "path": "/api/v1/sme/i18n/translations/hi?missing=true&status=machine",
  "method": "GET"
}
```
Errors: **400** `en`; **404** unknown language.

### 3.2 Bulk upsert — ALL-OR-NOTHING
```
PUT /sme/i18n/translations/:code
```
| Body field | Type | Description |
|---|---|---|
| updatedBy | string | optional, ≤ 80 chars. Free-text attribution stored on every row the call **writes** (`sme:anita`). Omitted = stored as `null` on the rows it writes |
| items | array | **1–500** of `{ key, value, status? }`. `status` = `machine` (default) \| `reviewed`. `value` ≤ 2,000 characters, counted in **Unicode code points** (an emoji is 1) — the same measure on every i18n route |

**Every item is checked before anything is written. One bad item rejects the whole batch,
nothing is written, and `message[]` lists EVERY offender** as `[index] <key>: KIND detail`
(the index is the item's position in `items`):

```json
// PUT /sme/i18n/translations/zz   (staging has no retired key, so four of the five kinds below)
{
  "updatedBy": "sme:b9b",
  "items": [
    {
      "key": "common.ok",
      "value": "ZZ Ok"
    },
    {
      "key": "exams.switcher.daysToGo_other",
      "value": "{n} ZZ days to go"
    },
    {
      "key": "common.nope",
      "value": "x"
    },
    {
      "key": "common.cancel",
      "value": "   "
    },
    {
      "key": "common.ok",
      "value": "ZZ Okay"
    }
  ]
}
```
```json
{
  "success": false,
  "message": [
    "[1] exams.switcher.daysToGo_other: PLACEHOLDER_MISMATCH expected {count} got {n}",
    "[2] common.nope: UNKNOWN_KEY not in the label registry",
    "[3] common.cancel: EMPTY_VALUE value is blank",
    "[4] common.ok: DUPLICATE_KEY also at [0]"
  ],
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-10-23T03:39:42.809Z",
  "path": "/api/v1/sme/i18n/translations/zz",
  "method": "PUT"
}
```

| KIND | Cause | Fix |
|---|---|---|
| `PLACEHOLDER_MISMATCH expected {a},{b} got {c}` | the value's placeholder set ≠ the English's (names listed both sides; `got (none)` when the value has none) | use exactly the English's `{…}` names |
| `UNKNOWN_KEY` | no such label | the key is not in the app (yet): check spelling, or wait for the release sync |
| `RETIRED_KEY` | the app no longer uses the key | drop the item |
| `EMPTY_VALUE` | blank / whitespace-only value | drop the item to leave the key untranslated |
| `DUPLICATE_KEY also at [i]` | the same key twice in one batch | send each key once |

A malformed item (bad key shape, `status` not `machine`/`reviewed`, `items` empty or > 500) is a
400 from validation, also with `message[]`. Display every line; the SME fixes them all and
resubmits.

A valid batch:
```json
{
  "updatedBy": "sme:b9b",
  "items": [
    {
      "key": "common.ok",
      "value": "ZZ Ok",
      "status": "reviewed"
    },
    {
      "key": "common.cancel",
      "value": "ZZ Cancel"
    },
    {
      "key": "exams.switcher.daysToGo_one",
      "value": "{count} ZZ day to go"
    },
    {
      "key": "exams.switcher.daysToGo_other",
      "value": "{count} ZZ days to go"
    },
    {
      "key": "nav.examSwitched",
      "value": "ZZ now on {examName}"
    }
  ]
}
```
```json
{
  "code": "zz",
  "received": 5,
  "created": 5,
  "updated": 0,
  "unchanged": 0
}
```
`created` / `updated` are exact, and **a row whose value and status are unchanged is not
rewritten** — it counts as `unchanged` and does not mark the language dirty. Re-sending two of
the rows above, one now `reviewed`:
```json
{
  "code": "zz",
  "received": 2,
  "created": 0,
  "updated": 1,
  "unchanged": 1
}
```
Drafts only: publish the language to ship them. Errors: **400** offenders / `en`
(`English is edited through PATCH /sme/i18n/labels/:key`); **404** unknown language. Audit:
`I18N_TRANSLATION_UPSERT` (counts, status split, `updatedBy`, up to 20 sample keys).

### 3.3 Delete one translation
```
DELETE /sme/i18n/translations/:code/:key
```
The key falls back to English at the next publish.
```json
// DELETE /sme/i18n/translations/zz/common.cancel
{
  "deleted": true,
  "key": "common.cancel",
  "lang": "zz"
}
```
Its effect on `draftDirty`: a deletion leaves no newer row behind, so the server touches the
**label** to record it — which marks **every** language dirty, not just this one (an accepted
over-signal: republishing an unaffected language ships an identical bundle). Errors: **400**
`en` (any case) — `English is edited through PATCH /sme/i18n/labels/:key`, as on GET / PUT;
**404** no such translation (`No "zz" translation for "common.cancel"` — the same DELETE sent twice). Audit:
`I18N_TRANSLATION_DELETE` (the deleted value).

### 3.4 Export — a flat file for a translator or a script
```
GET /sme/i18n/export/:code[?fill=true]
```
Raw `{ "<key>": "<text>" }` of the **drafts** (not what is published), active keys only, sorted
by key. Stale translations are exported as stored.
```json
// GET /sme/i18n/export/zz        — translated keys only (all 7 of them)
{
  "common.cancel": "ZZ Cancel",
  "common.ok": "ZZ Ok",
  "exams.switcher.daysToGo_one": "{count} ZZ day to go",
  "exams.switcher.daysToGo_other": "{count} ZZ days to go",
  "nav.examSwitched": "ZZ now on {examName}",
  "quota.resetLine": "{time} ZZ reset",
  "streak.monthYear": "{year} / {month}"
}
```
```json
// GET /sme/i18n/export/zz?fill=true   — every key; English where zz has none (excerpt: 5 of 1,261)
{
  "common.cancel": "ZZ Cancel",
  "common.close": "Close",
  "common.confirm": "Confirm",
  "common.ok": "ZZ Ok",
  "quota.resetLine": "{time} ZZ reset"
}
```
```json
// GET /sme/i18n/export/en        — every key's current English, SME edits included (excerpt: 4 of 1,261)
{
  "auth.error.appleSignInFailed": "Apple sign in failed: {detail}",
  "auth.error.appleTokenExchangeFailed": "Apple token exchange failed: {detail}",
  "auth.error.backendExchangeFailed": "Backend exchange failed: {detail}",
  "auth.error.backendSignInFailed": "Backend sign-in failed"
}
```
Errors: **404** unknown language.

---

## 4. Keys sync — release tooling, NOT the portal

```
POST /sme/i18n/keys/sync
```
**The portal never calls this.** A developer runs it once per app release with the app's
complete `en.json`, through `scripts/i18n-sync.ts` (which always dry-runs first and refuses a
suspiciously small file — see [rollout-and-verify.md](./rollout-and-verify.md) §9). It is
documented here so the SME team knows where new keys come from and what a **conflict** is.

What a sync does, per key:

| On the server | In the file | Result |
|---|---|---|
| absent | present | **added** (active, `firstSeenAppVersion` = the release) |
| active, same English | present | **unchanged** (the note is updated if it changed → `contextUpdated`) |
| active, different English, **not** SME-edited | present | **updated** to the repo's English |
| active, different English, **SME-edited** | present | **no write** — reported in `conflicts`; the SME's English stays |
| retired | present | **reactivated** (with its translations) |
| active | absent | **retired** — never deleted |

A **dry run** of the next release's file — staging's live registry (every key's text and note)
with `dashboard.explore.journey` removed and a new `dashboard.quizStreak` added, sent with
`"dryRun": true`, so nothing was written:
```json
{
  "appVersion": "2.2",
  "dryRun": true,
  "added": 1,
  "updated": 0,
  "unchanged": 1260,
  "reactivated": 0,
  "retired": 1,
  "contextUpdated": 0,
  "conflicts": [],
  "activeKeyCount": 1261,
  "addedKeys": [
    "dashboard.quizStreak"
  ],
  "updatedKeys": [],
  "reactivatedKeys": [],
  "retiredKeys": [
    "dashboard.explore.journey"
  ],
  "languagesDraftDirty": true,
  "note": "DRY RUN — nothing written. Bundles change only on publish: every language is now draftDirty — publish each language (en first) to ship these keys."
}
```

**The conflict report.** After an SME reworded two English labels (§2.2), the next sync of the
unchanged repo file keeps the SME text and says so:
> _Rig capture (local rig, `60bdd89`): staging has no SME-edited label, and making one means editing English for every language._
```json
{
  "appVersion": "2.2",
  "dryRun": true,
  "added": 0,
  "updated": 0,
  "unchanged": 6,
  "reactivated": 0,
  "retired": 0,
  "contextUpdated": 0,
  "conflicts": [
    {
      "key": "nav.examSwitched",
      "repo": "Switched to {examName}",
      "server": "Now preparing for {examName}"
    },
    {
      "key": "profile.title",
      "repo": "Profile",
      "server": "Your profile"
    }
  ],
  "activeKeyCount": 8,
  "addedKeys": [],
  "updatedKeys": [],
  "reactivatedKeys": [],
  "retiredKeys": [],
  "languagesDraftDirty": false,
  "note": "Nothing changed; no publish needed for this sync."
}
```
The SME's English wins over the repo **forever** for that key; the developer copies it into
`en.json`. In the portal, `sourceEditedBySme: true` on a label is the marker that it is
SME-owned. `languagesDraftDirty: true` in a sync response means every language needs a
republish (in a dry run: **would** need one — the `note` then starts `DRY RUN — nothing
written.`).

Payload validation reports every offender at once (this one was sent with `"dryRun": true`;
validation runs first either way):
```json
{
  "success": false,
  "message": [
    "Bad-Key: INVALID_KEY must be dotted camelCase segments like auth.getStarted.title",
    "common.empty: EMPTY_TEXT text must be a non-empty string",
    "common.extra: UNKNOWN_FIELD note"
  ],
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-10-23T03:39:41.657Z",
  "path": "/api/v1/sme/i18n/keys/sync",
  "method": "POST"
}
```
**Limits.** At most **5,000 keys** per sync (`labels: TOO_MANY_KEYS 5001 > 5000`); each `text`
≤ 2,000 and each `context` ≤ 500 characters, counted in **Unicode code points** exactly as on
PUT translations and PATCH labels (`zz.emoji: TEXT_TOO_LONG 2001 > 2000`). The whole request
body must also fit the API's **5 MB** JSON limit — at 5,000 keys that is **≈1,000 bytes per key
on average** (key + English + note + JSON punctuation). A larger body is refused before any
validation with **413** `Request body too large (limit 5 MB)` and nothing is written; split a
larger registry (or shorten notes) rather than retrying. The app's registry is far below both
limits today.

**Every key in the file is honoured, whatever its name.** Keys spelled like JavaScript object
internals (`constructor`, `toString`, `valueOf`, `hasOwnProperty`, `isPrototypeOf`,
`propertyIsEnumerable`, `toLocaleString`, `prototype`) are ordinary keys and are added,
updated and retired like any other. `__proto__` does not match the key pattern and is a 400
`__proto__: INVALID_KEY …`.

Audit: `I18N_KEYS_SYNC` (counts, conflict keys, up to 50 added/updated/reactivated/retired keys).
A `dryRun: true` call writes nothing and records no audit row.

---

## 5. The user's `uiLanguage`

Every `/sme/users` row, and the user detail, carries **`uiLanguage`** — the UI language the
user picked in the app. It is synced from the app: when a user switches language, the app sends
it to `PATCH /user/profile/me` (`uiLanguage: "hi"`). **`null` = never chosen** (the device
decides; most users), deliberately not coerced to `en`.

**A user's language can only ever be one the app can fetch.** The write accepts `en`, or a
language that is **enabled AND published** — exactly the languages `GET /i18n/languages`
lists (§6). An enabled language that has never been published, a disabled one, or an unknown
code is refused:
> _Rig capture (local rig, `045ab74`): it needs an app user and a user token; staging examples use no user accounts._
```json
// PATCH /user/profile/me   {"uiLanguage": "mr"}   — mr exists and is enabled, but was never published
{
  "success": false,
  "message": "Unknown or unavailable language \"mr\"",
  "error": "Bad Request",
  "statusCode": 400,
  "timestamp": "2026-10-22T22:18:01.956Z",
  "path": "/api/v1/user/profile/me",
  "method": "PATCH"
}
```
(A `null` in that PATCH means "unchanged", never "reset" — the app sends explicit nulls.)

**Send `uiLanguage` in its own PATCH.** `PATCH /user/profile/me` is all-or-nothing: an
unavailable `uiLanguage` fails the **whole** request with the 400 above, so any other profile
fields in the same body are not saved either. This is deliberate (an explicit refusal beats a
silently dropped field), and it is why the app sends the language change as a PATCH carrying
only `uiLanguage`. Any other client should do the same.

> _Rig capture (local rig, `60bdd89`): same reason as above._
```json
// GET /sme/users?search=asha   — one row (excerpt: identity + the language fields)
{
  "data": [
    {
      "id": "b9-user",
      "email": "asha@example.com",
      "name": "Asha",
      "status": "ACTIVE",
      "activeExamId": null,
      "uiLanguage": "hi"
    }
  ],
  "total": 1
}
```
```json
// GET /sme/users/b9-user   — excerpt: the top-level field and the same value inside `profile`
{
  "id": "b9-user",
  "uiLanguage": "hi",
  "profile": {
    "uiLanguage": "hi"
  }
}
```
Read-only here: the portal does not set a user's language. A stored language can still stop
being servable later — an SME disables it — and then that user simply sees English: nothing
cascades, the stored value stays, and it applies again if the language is re-enabled.

---

## 6. The public contract — what the apps (and the portal's preview) call

Both routes are **public** (no key, no login — the app needs labels on the login screen),
exam-agnostic, and under the global prefix.

### 6.1 `GET /api/v1/i18n/languages`
Languages the app may switch to: **enabled AND published**, plus `en` always, ordered by
`sortOrder` then `code`. Always 200.
```json
{
  "languages": [
    {
      "code": "en",
      "name": "English",
      "nativeName": "English",
      "sortOrder": 0
    },
    {
      "code": "hi",
      "name": "Hindi",
      "nativeName": "हिन्दी",
      "sortOrder": 0
    }
  ]
}
```

### 6.2 `GET /api/v1/i18n/bundle/:code[?v=<held version>]`
| Request | Response |
|---|---|
| no `v`, or `v` ≠ the published version | **200**, the full bundle, `ETag: "<code>-<version>"`, `Cache-Control: no-cache` |
| `?v=<published version>`, or `If-None-Match: "<code>-<version>"` | **304**, empty body, same `ETag` and `Cache-Control: no-cache` (RFC 9110 §15.4.5) — except `en` while it is still at version 0, which always gets the 200 (below) |
| unknown, disabled, or never-published language | **404** `Language "xx" is not available` → the app reverts to English |

```
GET /api/v1/i18n/bundle/hi
HTTP/2 200
ETag: "hi-4"
Cache-Control: no-cache
```
```json
// excerpt: 5 of the 1,261 labels
{
  "code": "hi",
  "version": 4,
  "languages": [
    {
      "code": "en",
      "name": "English",
      "nativeName": "English",
      "sortOrder": 0
    },
    {
      "code": "hi",
      "name": "Hindi",
      "nativeName": "हिन्दी",
      "sortOrder": 0
    }
  ],
  "labels": {
    "nav.ai": "AI",
    "nav.home": "होम",
    "common.ok": "ठीक है",
    "faq.title": "अक्सर पूछे जाने वाले सवाल",
    "pyq.mains": "Mains"
  }
}
```
(`labels` is a JSON object; its key order is not meaningful.)

```
GET /api/v1/i18n/bundle/hi?v=4                       → HTTP/2 304, ETag: "hi-4", Cache-Control: no-cache, empty body
GET /api/v1/i18n/bundle/hi  If-None-Match: "hi-4"    → HTTP/2 304, ETag: "hi-4", Cache-Control: no-cache, empty body
GET /api/v1/i18n/bundle/en?v=3                       → HTTP/2 304, ETag: "en-3", Cache-Control: no-cache, empty body
```
```json
// GET /api/v1/i18n/bundle/zz   (exists and was published once, but disabled)
{
  "success": false,
  "message": "Language \"zz\" is not available",
  "error": "Not Found",
  "statusCode": 404,
  "timestamp": "2026-10-23T03:39:38.638Z",
  "path": "/api/v1/i18n/bundle/zz",
  "method": "GET"
}
```

**`en` before its first publish.** `en` is always served. Until it is published at least once
it is served at `version: 0` as a **live compile** of the current English (`ETag: "en-0"`), and
**it never answers 304 until it is published** — neither to `?v=0` nor to
`If-None-Match: "en-0"`: its content changes with every sync and English edit while its version
cannot move, so holding "v0" proves nothing and every call gets the full 200. This is a
pre-publish state only: **publish `en` immediately after the first sync** (rollout §9); from v1
on, `en` behaves like every other language (304 when current, English changes reach devices
only through publishes). Staging's `en` is already past this state (v3 — the `?v=3` 304 above),
so the version-0 behaviour is verified on the local rig, not shown from staging.

For the portal: the **preview** route (§1.5) returns this same shape from the drafts, so one
renderer serves both "what users see now" (`/i18n/bundle/:code`) and "what Publish would ship".

---

## Common errors

| Status | Cause | Fix |
|---|---|---|
| 401 | Missing/invalid `x-api-key` (`Invalid or missing API key`) | Send the `x-api-key` header = `API_KEY_SECRET` |
| 400 | `null` on a non-nullable field — `` `enabled` is not nullable; omit it to leave it unchanged `` (languages), `` `sourceText` is not nullable… `` (labels) | Omit the field to leave it unchanged |
| 400 | Bulk translation batch rejected — `message[]` = every offender as `[idx] <key>: KIND detail` (§3.2) | Fix every line, resend the whole batch (nothing was written) |
| 400 | `en` on a translation route — GET, PUT **and DELETE** (`English is edited through PATCH /sme/i18n/labels/:key`); disabling `en`; `missing` + `status` together; `lang` ≠ `missingIn` | See the section for that route |
| 400 | A malformed language code in a **query** parameter — `GET /sme/i18n/labels?lang=hi_IN` → `` lang must be a language code like `hi` ``, `?missingIn=…` → `` missingIn must be a language code like `hi` ``. (A malformed code in the **path**, e.g. `/sme/i18n/translations/hi_IN`, is a 404 `Language "hi_in" not found`.) | Send a code like `hi`, `ta`, `pt-br` |
| 400 | A text over its limit — `… must be at most 2000 characters (Unicode code points)` (PUT / PATCH), `TEXT_TOO_LONG` / `CONTEXT_TOO_LONG` (sync). Lengths are code points everywhere | Shorten it |
| 404 | Unknown language / label / translation (`Language "xx" not found`, `Label "…" not found`, `No "hi" translation for "…"`) | Check the code or key; keys come from the app (§0) |
| 409 | `POST /sme/i18n/languages` with an existing code (`Language "hi" already exists`) | `PATCH` it instead |
| 413 | The JSON body is over the API's 5 MB limit (`Request body too large (limit 5 MB)`); nothing was written | Split the request (§4 limits) |

```json
// any /sme/i18n/* call without the key
{
  "success": false,
  "message": "Invalid or missing API key",
  "error": "Unauthorized",
  "statusCode": 401,
  "timestamp": "2026-10-23T03:39:40.626Z",
  "path": "/api/v1/sme/i18n/languages",
  "method": "GET"
}
```

All writes are recorded in `sme_audit_log` with before/after payloads:

| Action | Fires when |
|---|---|
| `I18N_LANGUAGE_CREATE` | a language is created |
| `I18N_LANGUAGE_UPDATE` | a language's name / nativeName / sortOrder / enabled actually changes (an empty PATCH, or one re-sending the current values, records nothing) |
| `I18N_LANGUAGE_PUBLISH` | a language is published (old/new version, counts, dropped keys, diff) |
| `I18N_LABEL_UPDATE` | a label's English or note actually changes (+ the languages it made stale) |
| `I18N_TRANSLATION_UPSERT` | a bulk translation write succeeds (a rejected batch records nothing) |
| `I18N_TRANSLATION_DELETE` | a translation is deleted (its last value) |
| `I18N_KEYS_SYNC` | a real (non-dry-run) keys sync completes |
