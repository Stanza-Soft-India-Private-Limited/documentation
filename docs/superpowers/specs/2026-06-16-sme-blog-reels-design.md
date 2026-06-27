# SME Blog Management ↔ Reels — Design Spec (for the SME portal team)

**Date:** 2026-06-16
**Status:** Backend implemented (branch `feat/sme-management`)
**Audience:** SME portal engineers building the management UI

## Goal

Give the SME portal **blog management** for reels. Videos/reels are already
managed by the SME portal (Mux upload → `PUT /reels/bulk`). A **blog** is the
written companion to a reel (markdown → rendered article in the app's "Read more"
pane). This spec defines the endpoints AND the **required UX**: blog editing is
part of the **reel management screen — NOT a separate top-level screen**.

## The model (already in the backend)

- A blog is **1:1 with a reel** (`ReelBlog.reelId` is unique). A reel has zero or
  one blog.
- Blog content is authored as **raw markdown**. The backend parses it into
  `sections` (content + image blocks); **quiz blocks are stripped server-side**
  (reels have no quiz on any client).
- Deleting a reel cascades its blog. Deleting a blog leaves the reel intact.

## Required UX (build into the reel screen)

In the existing **reel management list/detail**:

1. Each reel row shows a **"Has blog"** indicator. The list endpoint
   (`GET /sme/blogs`) returns `hasBlog` + a blog summary per reel, and supports
   `?hasBlog=true|false` so the team can show a "reels missing a blog" filter.
2. From a reel's row/detail, an **"Add blog" / "Edit blog"** action opens an inline
   markdown editor (not a separate route):
   - No blog yet → **Add blog** → `POST /sme/blogs` (`reelId`, `title`, `rawMarkdown`).
   - Blog exists → **Edit blog** → `PATCH /sme/blogs/:reelId`. Send `rawMarkdown`
     to re-parse sections, or just `title` to rename without touching content.
   - **Remove blog** → `DELETE /sme/blogs/:reelId` (reel stays).
3. A **preview** of the parsed `sections` (returned by create/update/get) lets the
   author verify content/image blocks before publishing.

Do **not** build a standalone "Blogs" navigation entry. Blogs live where their
reel lives. This mirrors the app, where the blog is reached from the reel.

## Endpoints (see docs/sme/SME_PORTAL_API.md for full payloads)

| Method | Path | Purpose |
|---|---|---|
| GET | `/sme/blogs` | List reels + blog presence/summary (filter `hasBlog`, `search`, paginate) |
| GET | `/sme/blogs/:reelId` | Fetch a reel's blog (title, rawMarkdown, sections) |
| POST | `/sme/blogs` | Create a blog for a reel |
| PATCH | `/sme/blogs/:reelId` | Update title and/or rawMarkdown |
| DELETE | `/sme/blogs/:reelId` | Delete the reel's blog |

All are gated by `x-api-key`. The reel/video endpoints (`GET /reels`,
`POST /videos/upload-url`, `PUT /reels/bulk`, `DELETE /reels/:id`) are unchanged —
the SME portal continues to use them for video management.

## Markdown format

Same content/image/quiz markdown the content-doc handoff uses (see
`docs/sme/CONTENT_DOC_SME_API.md`). For blogs, quiz blocks are accepted but stripped — only
content + image blocks render.
