# WhatsApp media folder — upload API

> Added 2026-11-12. A public folder of pictures and videos for WhatsApp (Wylto) message
> templates, with a token-guarded upload surface so the marketing side never needs
> AWS access.

## What it is

- **Folder:** the `wa-media/` prefix of the public media bucket (the same bucket that
  serves question figures, banners and offer artwork).
- **Links:** `https://<bucket>.s3.ap-south-1.amazonaws.com/wa-media/<name>.<ext>` —
  plain HTTPS, HTTP 200, no login, no redirect, no bot protection. Meta's servers
  fetch template media from exactly this kind of URL.
- **Permanent:** a file stays until someone deletes it through this API.
- **Accepted:** JPG and PNG up to 5 MB, MP4 up to 16 MB (WhatsApp's own template
  limits). The type is decided from the file bytes, never from the name or the
  declared `Content-Type`, so a renamed `.exe` cannot be published under a `.png` link.

## Auth

Every data route needs the header `x-wa-media-token: <token>` (or
`Authorization: Bearer <token>`). The token is `WA_MEDIA_TOKEN` in the backend env — a
**separate** secret from the SME `x-api-key`, and it opens nothing else. Unset = every
route answers 401.

## Routes (under `/api/v1`)

| Route | What |
|---|---|
| `GET /wa-media/ui` | Browser page: paste the token once, upload, list, copy links, delete. No auth on the page itself. |
| `POST /wa-media/upload` | `multipart/form-data`: `file` (required), `name` (optional link name; letters, digits, dashes), `overwrite` (`true` to replace an existing name). Returns `201 { name, url, size, contentType }`. |
| `GET /wa-media` | `[{ name, url, size, lastModified }]`, newest first. |
| `DELETE /wa-media/:name` | Removes the file; its link stops working. `404` if absent. |

Errors: `400` not a JPG/PNG/MP4 or over the cap, `401` bad token, `409` name exists and
`overwrite` not set, `503` storage not configured or the server's S3 role lacks the
action (the message names it: `AccessDenied`).

```bash
curl -X POST https://app.stanzasoft.ai/api/v1/wa-media/upload \
  -H "x-wa-media-token: $WA_MEDIA_TOKEN" -F "file=@diwali.png" -F "name=diwali-offer"
# → {"name":"diwali-offer.png","url":"https://…/wa-media/diwali-offer.png","size":48213,"contentType":"image/png"}
```

## Operating notes

- Production's ECS task role already has `s3:PutObject` on the bucket (that is how
  question figures are uploaded). `ListBucket` / `DeleteObject` are reported as a
  `503 … AccessDenied` if the role lacks them — nothing is hidden.
- Staging's task role cannot write to the bucket (known), so on staging only the
  page, the auth and the validation paths can be proven; uploads are proven on prod.
- The "name already exists" check tries a signed HeadObject and, if the role may not HEAD (403), falls back to an anonymous HEAD on the public link — no permission needed, because the bucket serves every key publicly.
- Bytes go through the API (multer, memory storage, 16 MB cap per request), unlike the
  SME presigned two-step — chosen because the uploader is a person with a browser, not
  a portal.
- Code: `modules/wa-media/*`, guard `common/guards/wa-media-token.guard.ts`, config
  `storage.waMediaToken` / `storage.waMediaPrefix`.
