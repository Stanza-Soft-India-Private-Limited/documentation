# SME — Subject Filter Config API Guide

Reference for the **SME portal** to manage what the app's PYQ **Prelims** and **Mains**
subject filters look like: per-subject **display name**, **icon**, **visibility** and
**sort position** — plus flipping a mains subject between **Optional** and **GS**. Same
server-to-server model as the rest of the SME surface: we expose the endpoints, the SME
team builds the UI.

**Base URL:** `{{BASE_URL}}/api/v1` (prod `BASE_URL` = `https://app.stanzasoft.ai`)
**Authentication:** `x-api-key: <API_KEY_SECRET>` header on **every** request.
**Swagger:** `/api/docs` (group **SME**) documents these live with the `x-api-key` control.

---

## 0. The mental model (read this first)

Defaults are **auto-derived from the live question data** (every subject that has active
questions gets an entry, with an auto-matched icon and name). SME entries are **overrides**
layered on top — you only store what you change, and clearing an override falls back to
the default. The merged result is served to the apps on the public `GET /mains/filters`
and `GET /pyq/filters` endpoints (new `subjectsMeta` field), so the portal never has to
touch the apps.

**Icon keys:** the physical icon assets are **bundled inside the mobile app** — the portal
only assigns one of the canonical **keys** below (returned as `validIconKeys` for a picker;
any other key is rejected with 400). Adding a brand-new icon requires a mobile app release.

```
sme_art, sme_current_affairs, sme_economy, sme_environment, sme_ethics,
sme_geography, sme_governance, sme_history, sme_ir, sme_polity, sme_science_technology
```

**Propagation:** the public `/filters` endpoints are cached for up to 1 hour, but every
write below **clears that cache**, so changes are visible on the very next app request.
(Worst case — if the cache clear itself fails — within the hour.)

`examType` is `prelims` or `mains`. `:subject` path params are the **raw** subject value,
URL-encoded (e.g. `GS%20Paper%201`, `Sociology%20Optional`).

## 1. Get filter config (per exam type)
```
GET /sme/filter-config/:examType
```
Returns every subject present in the (active) question data — plus any stale override
entries whose questions are gone (`questionCount: 0`) so they stay clearable. Hidden
subjects ARE included here (the portal must see everything); the public endpoints drop them.

```json
{
  "examType": "mains",
  "validIconKeys": ["sme_art", "…", "sme_science_technology"],
  "entries": [
    {
      "subject": "GS Paper 1",
      "isOptional": false,
      "questionCount": 270,
      "effective": { "displayName": "GS Paper 1", "iconKey": "sme_history", "hidden": false, "sortOrder": 1 },
      "override": { "sortOrder": 1 }
    },
    {
      "subject": "Sociology Optional",
      "isOptional": true,
      "questionCount": 729,
      "effective": { "displayName": "Sociology", "iconKey": "sme_governance", "hidden": false, "sortOrder": null },
      "override": null
    }
  ]
}
```

- `effective` = what the app actually renders (defaults merged under the override).
- `override` = only what SME explicitly set (`null` = pure defaults).
- Default `displayName` strips a trailing `" Optional"` **only when the subject is
  actually optional**; default `iconKey` is auto-matched from the subject name.
- Entries are sorted like the app (explicit `sortOrder` first — lower = earlier —
  then alphabetically by display name).

Errors: **400** unknown `examType`.

## 2. Update one subject's override
```
PUT /sme/filter-config/:examType/:subject
```
| Body field | Type | Description |
|---|---|---|
| displayName | string \| null | Label shown in the app. `null` clears back to the default |
| iconKey | string \| null | One of `validIconKeys`. `null` clears back to the auto-derived icon |
| hidden | boolean \| null | `true` removes the subject from the public filters. `null` clears (=visible) |
| sortOrder | number \| null | Explicit position (lower first; subjects without one sort last, alphabetically). `null` clears |

Field semantics: **omitted = untouched, `null` = clear, value = set.** Clearing every
field removes the override entry entirely. Concurrent-safe (row-locked read-modify-write).

```json
// PUT /sme/filter-config/mains/Law%20Optional
{ "displayName": "Law", "iconKey": "sme_polity" }
```
Response — the updated entry:
```json
{
  "examType": "mains",
  "subject": "Law Optional",
  "isOptional": true,
  "effective": { "displayName": "Law", "iconKey": "sme_polity", "hidden": false, "sortOrder": null },
  "override": { "displayName": "Law", "iconKey": "sme_polity" }
}
```
Errors: **400** unknown `examType` or unknown `iconKey` (message lists the valid keys).
Audit: `FILTER_CONFIG_UPDATE` (before/after of just that subject's override).

## 3. Flip a mains subject between Optional and GS
```
POST /sme/filter-config/mains/:subject/optional
```
Bulk-updates `isOptional` on **all** mains questions of that subject (self-service
correction for content dumps that mislabeled optionality). Prelims has no optional concept.

Body: `{ "isOptional": true | false }`

```json
// POST /sme/filter-config/mains/History%20Optional/optional  { "isOptional": false }
{
  "subject": "History Optional",
  "isOptional": false,
  "updatedCount": 546,
  "effective": { "displayName": "History Optional", "iconKey": "sme_history", "hidden": false, "sortOrder": null }
}
```

Note the display-name behavior: a subject flipped to non-optional keeps its **raw** name
(no automatic `" Optional"` stripping) until SME sets a `displayName` override — this is
deliberate, so nothing gets silently mislabeled.

Errors: **404** no mains questions for that subject.
Audit: `MAINS_OPTIONAL_FLIP` (pre-flip row counts + updated count).

---

## Common errors

| Status | Cause | Fix |
|---|---|---|
| 401 | Missing/invalid `x-api-key` | Send the `x-api-key` header = `API_KEY_SECRET` |
| 400 | Unknown `examType` / unknown `iconKey` / bad body | Check the tables above; the 400 message lists valid values |
| 404 | Flip on a subject with no mains questions | Verify the raw subject value (URL-encoded) |

All writes are recorded in `sme_audit_log` (`FILTER_CONFIG_UPDATE`, `MAINS_OPTIONAL_FLIP`)
with before/after payloads.
