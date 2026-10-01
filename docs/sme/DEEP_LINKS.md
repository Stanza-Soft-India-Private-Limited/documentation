# Deep links — `/open/<key>[/<id>]`, hosts, `?exam=`, tracking

> New 2026-10-01. Source of truth for the routes: the app's
> `upsc_app/…/features/notifications/DeepLinkTarget.kt` (`DeepLinkMapper.ROUTES` +
> `legacyUrlTarget`), the web app's `prepmonkey-web/src/lib/appLinks.ts` and
> `src/app/open/[...slug]/page.tsx`, and the backend's `notification-exam.util.ts` and
> `links.service.ts`. Read alongside [SME_NOTIFICATIONS_API.md](./SME_NOTIFICATIONS_API.md) §4
> (the same keys as push `type` values) and [SME_BANNERS_API.md](./SME_BANNERS_API.md)
> (banner destinations).

One link format opens the same screen from a push, a share, an ad, an email or a dashboard
banner:

```
https://go.prepmonkey.com/open/<key>[/<id>][?param=value&…]
```

---

## 1. Hosts — which one to publish

| Host | What it does | Use it for |
|---|---|---|
| **`go.prepmonkey.com`** | The **hand-off** host. Records a click server-side (`link_events`, a `cid` is minted), shows a short interstitial, then hands off to `app.prepmonkey.com` with the original query string intact plus `nb=1` and `cid=<uuid>`. | **Everything published outside the app**: Instagram/ads, WhatsApp, email, SMS, QR codes. |
| `app.prepmonkey.com` | The canonical Universal Link / App Link host. App installed → the OS opens the app directly (no click is recorded). Not installed → the store on mobile, or the equivalent web screen on desktop. | What the app itself builds for in-app shares. Fine for a link tapped from Messages/Notes. |

Why two hosts: iOS will not open the app from a Universal Link while the page you are on is
served from that **same** domain, so a page on `app.` can never hand off to the app. Standing
on `go.` first makes the jump a real cross-domain Universal Link. The app accepts **both**
hosts (they must stay in sync with the Android manifest's `android:host` entries).

Do **not** add `nb` or `cid` yourself. `nb=1` is the hand-off's loop-breaker and `cid` is
minted by the click.

---

## 2. Query parameters every link may carry

| Param | Meaning | Who reads it |
|---|---|---|
| `exam` | The exam the link belongs to, as a slug: `upsc-cse`, `appsc-group-1` (the active catalogue: `GET /sme/exams`). See §3. | The app (switches exam before routing); the `go.` click and `/links/*` events (stored as `link_events.exam_id`). |
| `src` | Free-text source tag for the funnel, e.g. `instagram_bio`, `ad_appsc_oct`, `whatsapp_group`. | The `go.` click (`link_events.source`). Passed through to the app unchanged; the app does not act on it. |
| `cid` | Click id. **Appended by the `go.` interstitial**; never write it into a published link. | The app carries it into later funnel events (`APP_OPEN`, sign-in, purchase). |
| target params | Per-key arguments such as `filterType`, `filterValue`, `examType`, `subject`, `topic`, `formType`, `taskId`, `name`. See §4. | The app's route resolver. |

Values are percent-decoded; a blank value counts as absent. On a push these same arguments
travel inside `data.params` (one JSON-encoded string→string object), not a query string.

---

## 3. `?exam=` — which exam the link opens in

### 3.1 The rule (locked)

> **A link WITHOUT `exam` opens in UPSC. A link WITH `exam` opens in that exam.**

The user is switched to the link's exam **silently** before the target screen opens, and a
short toast says *"Switched to <exam name>"*. There is no confirmation dialog, because an ad
click has to convert and a dialog on the paid path is friction. The switch is parked until
login and the nav graph are ready, so a link tapped while logged out still lands in the
right exam after sign-in.

| Link | User currently in | Opens in |
|---|---|---|
| `…/open/premium?exam=appsc-group-1` | UPSC | **APPSC** (switched, toast) |
| `…/open/premium?exam=appsc-group-1` | APPSC | APPSC (no switch, no toast) |
| `…/open/premium?exam=upsc-cse` | APPSC | **UPSC** (switched, toast) |
| `…/open/premium` (no `exam`) | APPSC | **UPSC** — ⚠️ see §3.2 |
| `…/open/premium?exam=apsc` (typo) | either | ⚠️ see below: effectively UPSC, with a wrong toast |

So **every APPSC link must carry `exam=appsc-group-1`**. Adding `exam=upsc-cse` to UPSC
links is optional but makes them explicit.

⚠️ **Copy slugs from `GET /sme/exams`; never type them.** The app (2.0) does not check the
slug against the exam catalogue before switching. A typo "switches" to an exam that does not
exist: the toast shows the raw slug (*"Switched to apsc"*), the server serves UPSC content, and
the app resets itself to UPSC the next time it loads the exam list. The server-side click record
stores an unknown slug as `NULL` (§7).

### 3.2 ⚠️ What ships when, and old app versions

| App version | `exam` present | `exam` absent |
|---|---|---|
| **before 2.0** (no exam modes) | ignored: opens the target in UPSC, which is the only exam those builds have | UPSC |
| **2.0** (live since 2026-09-17) | **switches** to that exam | **no switch**: opens in whatever exam the user is currently in |
| **next app release** (mobile item **M5**, in progress) | switches to that exam | **switches to UPSC** (the rule in §3.1) |

**The "absent = UPSC" half of the rule ships with the next app release.** Until users update,
a link without `exam` opens in whatever exam they are currently in. Treat links without `exam` as
"current exam" for anyone on 2.0, and put `exam` on every link where the exam matters.

### 3.3 Where `exam` is NOT acted on

- **Dashboard banners.** A banner's `destinationUrl` is parsed as an *in-app* link: the route is
  honoured but `exam` is ignored. Banners are already served per exam, and a stray `?exam=` on
  artwork must never move a user between exams. Banner taps are also not recorded as
  smart-link clicks.
- **Desktop / web.** On a desktop browser the link opens the equivalent web screen, and the
  web app is UPSC-only today. An APPSC link on desktop lands on UPSC web content.
- **Store installs.** A user without the app goes to the store. Do not assume the target
  screen or the exam survives the install.

---

## 4. Every route the app supports

`id` = the path segment after the key (`/open/<key>/<id>`). ✅ = required (when it is missing the
link opens the **fallback** instead), "optional" = changes the destination when present, ➖ = not
used. **Unknown keys open Home**, so older builds never crash on a key added later.

| `/open/<key>` | `id` | params | opens | fallback |
|---|---|---|---|---|
| `home` | ➖ | — | Home / dashboard | — |
| `daily_task` | ➖ | — | Daily tasks (Home) | — |
| `chat` | ➖ | — | Chat | — |
| `chat_expert` | ➖ | — | Chat, Expert (SME) mode | — |
| `mains` | ➖ | — | PYQ tab → Mains landing | — |
| `mains_question` | ✅ question id | — | that Mains question | Mains landing |
| `reel` | optional reel id | — | that reel, playing | Reels feed |
| `reelblog` | ✅ reel id | — | that reel's blog article | Reels feed |
| `pyq` | ➖ | — | PYQ / Practice tab | — |
| `pyq_question` | ✅ question id | — | that prelims question (read-only shared view) | PYQ tab |
| `simulation` | ✅ simulation id | — | Library → Simulation, highlighted (no auto-start) | Library |
| `doc` | ✅ document id | — | that study document | Library |
| `library` | ➖ | — | Library / My Content | — |
| `flashcards` | ➖ | — | Chat with the flashcard **generator** | — |
| `mnemonics` | ➖ | — | Chat with the mnemonic **generator** | — |
| `saved` | ➖ | — | Saved questions | — |
| `premium` | ➖ | — | Upgrade / paywall (campaign-aware) | — |
| `offer` | ✅ campaign code | — | that campaign's paywall | standard paywall |
| `report` | ✅ report id | — | that feedback report | Home |
| `survey` | ✅ survey id | — | the survey runner | Home |
| `profile` | ➖ | — | Profile | — |
| `my_account` | ➖ | — | My Account | — |
| `my_activity` | ➖ | — | My Activity | — |
| `notification_feed` | ➖ | — | notification feed (bell) | — |
| `faq` | ➖ | — | FAQ | — |
| `terms` | ➖ | — | Terms & Conditions | — |
| `phone_verify` | ➖ | — | phone verification entry | — |
| `help_feedback` | ➖ | — | Help & Feedback | — |
| `feedback_form` | optional (`ISSUE`/`FEATURE`) | `formType` | the report form | `ISSUE` |
| `my_reports` | ➖ | — | My Reports | — |
| `survey_list` | ➖ | — | Surveys | — |
| `weak_topics` | ➖ | — | "Practice your mistakes" topics | — |
| `replay_session` | optional subject | `subject`, `topic` | mistake replay | all outstanding mistakes |
| `practice_list` | ➖ | `filterType` ✅ (`year`\|`subject`), `filterValue` ✅, `examType` (`prelims` default \| `mains`) | a filtered PYQ list: a **year folder** or a subject list | PYQ tab |
| `add_task` | optional task id | `taskId` | task editor | blank "add task" form |
| `doc_list` | optional subject | `subject` | that subject's documents | Library |
| `saved_flashcards` | ➖ | — | saved flashcards list | — |
| `saved_mnemonics` | ➖ | — | saved mnemonics list | — |
| `saved_reels` | ➖ | — | Updates / saved reels | — |
| `simulation_review` | ✅ **attempt** id | `name` (display only) | that attempt's review | Library |

**URL-only spellings** (older shapes still live in shares and emails; checked before the table
above, and **not** valid as push `type` values, where they open Home):

| Link | Opens |
|---|---|
| `/open/pyq/<id>`, `/open/practice/<id>` | that prelims question (same as `pyq_question`) |
| `/open/practice` | PYQ tab |
| `/open/mains/<id>` | that Mains question (same as `mains_question`) |
| `/open/reels`, `/open/updates` | Reels feed |
| `/open/chat/expert` | Chat, Expert mode |
| `/open/flashcard` | flashcards |
| `/open/savedquestions`, `/open/saved-questions`, `/open/saved_questions` | Saved questions |
| `/open/journey` | Home (no mobile journey screen) |

Note the asymmetry: **`/open/pyq/<id>` is a question link**, but a push with `type: "pyq"`
opens the tab. The same applies to `mains`.

---

## 5. Marketing examples

Always publish on `go.` with an `src`. Ids come from the SME portal (questions: `GET
/sme/content/pyq?exam=…`; campaigns: the campaign `code`).

**APPSC**

```
# Premium / paywall in APPSC
https://go.prepmonkey.com/open/premium?exam=appsc-group-1&src=ig_appsc_oct

# An APPSC campaign
https://go.prepmonkey.com/open/offer/APPSCDIWALI?exam=appsc-group-1&src=ad_appsc_diwali

# One APPSC prelims question (the id must be an APPSC row)
https://go.prepmonkey.com/open/pyq/<pyq-question-uuid>?exam=appsc-group-1&src=whatsapp_appsc

# The APPSC 2024 prelims year folder (both papers of that year; 1983-84 is year 1984)
https://go.prepmonkey.com/open/practice_list?filterType=year&filterValue=2024&exam=appsc-group-1&src=ig_appsc_2024

# APPSC prelims by subject
https://go.prepmonkey.com/open/practice_list?filterType=subject&filterValue=Polity&exam=appsc-group-1&src=email_appsc

# Reels, in APPSC
https://go.prepmonkey.com/open/reels?exam=appsc-group-1&src=ig_appsc_reels
https://go.prepmonkey.com/open/reel/<reel-id>?exam=appsc-group-1&src=ig_appsc_reels
```

**UPSC**

```
https://go.prepmonkey.com/open/premium?exam=upsc-cse&src=ig_upsc_oct
https://go.prepmonkey.com/open/pyq/<pyq-question-uuid>?exam=upsc-cse&src=telegram_upsc
https://go.prepmonkey.com/open/practice_list?filterType=year&filterValue=2025&exam=upsc-cse&src=email_upsc
https://go.prepmonkey.com/open/mains?exam=upsc-cse&src=ig_upsc_mains
```

`exam=upsc-cse` is written out deliberately. On 2.0 a link without `exam` opens in the user's
**current** exam (§3.2).

**Checklist before publishing:** `go.` host · `exam=` set · `src=` set · the id belongs to that
exam · the target key is in §4 (a typo opens Home, silently).

---

## 6. Push notifications — the `exam` they carry

A push uses the same keys (`data.type`, `data.id`, `data.params`; see
[SME_NOTIFICATIONS_API.md](./SME_NOTIFICATIONS_API.md) §4). The backend stamps
`params.exam` so a tapped notification opens in the exam it belongs to:

| Sent by | `params.exam` |
|---|---|
| Notification engine rule **scoped to an exam** | the **rule's exam** |
| Engine rule for **all exams**, event-triggered, where the event named an exam | that exam |
| Engine rule for **all exams**, otherwise | the **recipient's own active exam** (`user_profiles.active_exam_id`), looked up per send |
| `payment_success` ("Premium activated") | the **exam that was bought**, even if the user is in another one |
| SME `/broadcast` **with** `exam` (FCM topic `exam_<slug>`) | that exam |
| SME `/segment` **with** `exam` | that exam |
| SME `/broadcast` to **all users**, un-narrowed `/segment` | **none** |
| a failed recipient lookup | none (the push still sends; logged) |

- The stamped exam **wins** over an `exam` you typed into `params` by hand.
- `params` is capped at 10 keys / 1 KB. If adding `exam` would push a valid map over the cap,
  the push is sent **without** the exam (and a warning is logged) rather than losing the
  deep link.
- App side: a push **with** `exam` switches exam before routing; **without** one it does not
  switch. Unlike links, a push without `exam` does **not** mean UPSC. ⚠️ Acting on
  `params.exam` from a push is mobile work for the **next app release**. 2.0 and older ignore
  the key and route on `type`/`id` as before, which is forward-safe.

---

## 7. Analytics: the exam on a click

`/links/click`, `/links/event` and `/links/claim` take an optional `exam`, stored in
`link_events.exam_id`:

- validated against the **active** exam catalogue. An absent, unknown or inactive slug is
  stored as `NULL` (never a `400`, because refusing would lose the click);
- the `go.` click takes it from the link's `?exam=`;
- `claim` (at sign-in) fills `exam_id` only where it is still `NULL`, and never overwrites a
  recorded exam;
- rows before 2026-10-01 have `NULL` (exam unknown). Label any per-exam funnel *"since
  2026-10-01"*.
