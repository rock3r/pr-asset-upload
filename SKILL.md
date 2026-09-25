---
name: pr-asset-upload
description: Upload a screenshot or short demo clip (PNG, JPEG, WebP, GIF, MP4, or WebM) to static.sebastiano.dev and get back a URL to hotlink in a GitHub PR description or comment. Use whenever a PR needs a before/after screenshot, UI diff image, or a short demo recording embedded or linked instead of committed to the repository.
license: MIT
compatibility: Requires outbound network access to https://static.sebastiano.dev and a valid upload token available as an environment variable.
metadata:
  owner: seb
  endpoint: https://static.sebastiano.dev/upload
---

# PR Asset Upload

Uploads a single image or video file to Cloudflare R2 via an authenticated HTTP endpoint and
returns a URL to embed or link in a GitHub PR. By default every asset key is an opaque UUID — no
owner/repo/PR/filename is ever exposed in the URL itself, for either visibility.

There is one exception: a **stable URL**. Here the caller picks a name, and a later upload with
the same name replaces the file at the same URL. Use it for files that a README or docs page links
permanently and that CI refreshes. See "Stable URLs" below.

## When to use this

- The user asks to attach a screenshot, before/after comparison, or short demo clip to a GitHub PR.
- A PR description or comment needs an externally-hosted image/video URL instead of committing
  the file to the reviewed repository.

## Before you upload

1. Confirm the file's mime type is one of: `image/png`, `image/jpeg`, `image/webp`, `image/gif`,
   `video/mp4`, `video/webm`. Anything else is rejected by the endpoint. Files over 50MB are
   rejected (`413`).
2. GitHub only renders images inline via markdown (`![alt](url)`) — an externally-hosted video is
   never played inline no matter the format, it always renders as a plain clickable link. That's a
   GitHub platform behavior, not something this pipeline can change. Animated WebP is the
   practical workaround: it's an image format (so it renders inline like a GIF) but can still
   carry a short demo recording — verified that GitHub's `camo` image proxy passes WebP through
   byte-for-byte, animation intact, not stripped or re-encoded. Prefer animated WebP for inline
   demo previews, and link an MP4/WebM alongside it as a fallback for viewers/tools that don't
   render WebP inline. Static screenshots should just be PNG/JPEG.
3. Decide visibility:
   - `public` — the containing repo is public, or the asset is fine being reachable by anyone
     with the link, no restriction. Requires owner/repo/PR number/name (see below) for
     operator-side bookkeeping — these are stored as metadata only, never in the URL. PR number
     must be numeric. `X-Asset-Allowed-Origins` is not accepted here — public assets are served
     directly with no access check possible.
   - `private` — the containing repo is private, or the asset shouldn't be casually reachable by
     anyone who finds the link. Requires `X-Asset-Allowed-Origins` (see below) — a private upload
     with no origin restriction at all is rejected, since that would just be a public asset with
     extra steps.
4. Get the upload token. It must already be available to you as an environment variable
   (e.g. `PR_ASSET_UPLOAD_TOKEN`) — never hardcode it, log it, or ask the user to paste it in chat.

## Request

```
POST https://static.sebastiano.dev/upload
Authorization: Bearer <token>
Content-Type: <mime type of the file>
X-Asset-Visibility: public | private
```

Additional headers, required only when `X-Asset-Visibility: public`:

| Header | Meaning |
|---|---|
| `X-Asset-Owner` | GitHub org/user, e.g. `acme` |
| `X-Asset-Repo` | Repo name, e.g. `widget` |
| `X-Asset-Pr` | PR number, e.g. `123` |
| `X-Asset-Name` | Semantic name, e.g. `settings-before` |

Required only when `X-Asset-Visibility: private`:

| Header | Meaning |
|---|---|
| `X-Asset-Allowed-Origins` | Comma-separated hostnames allowed to load this asset. Almost always just `github.com` for a PR embed — GitHub proxies every embedded image through its `camo` service, which strips `Origin`/`Referer` entirely, so this is matched against camo's distinctive request signature rather than a literal `Origin` header. Other hostnames are matched against `Origin`/`Referer` normally (useful for embedding on a site you control directly, not through GitHub). |

The request body is the raw file bytes — no multipart, no base64.

## Example: public screenshot

```bash
curl -sS -X POST "https://static.sebastiano.dev/upload" \
  -H "Authorization: Bearer $PR_ASSET_UPLOAD_TOKEN" \
  -H "Content-Type: image/png" \
  -H "X-Asset-Visibility: public" \
  -H "X-Asset-Owner: acme" \
  -H "X-Asset-Repo: widget" \
  -H "X-Asset-Pr: 123" \
  -H "X-Asset-Name: settings-before" \
  --data-binary @settings-before.png
```

## Example: private demo clip, embedded in a GitHub PR

```bash
curl -sS -X POST "https://static.sebastiano.dev/upload" \
  -H "Authorization: Bearer $PR_ASSET_UPLOAD_TOKEN" \
  -H "Content-Type: video/mp4" \
  -H "X-Asset-Visibility: private" \
  -H "X-Asset-Allowed-Origins: github.com" \
  --data-binary @demo.mp4
```

## Response

Success (`201`):

```json
{ "url": "https://static.sebastiano.dev/public/9a54b794-bcda-4cc4-a9e5-5c1d9e1b2ff3.png" }
```

Use the `url` value directly in the PR:

```md
![Before](https://static.sebastiano.dev/public/9a54b794-bcda-4cc4-a9e5-5c1d9e1b2ff3.png)
```

Errors return a JSON body with an `error` field and an appropriate status code (`400` bad, missing,
or invalid fields — e.g. `invalid_pr`, `allowed_origins_not_supported_for_public`,
`allowed_origins_required_for_private`; `401` bad token; `413` file too large; `415` unsupported
content type). Report the `error` field back to the user rather than retrying blindly.

## Stable URLs (overwrite in place)

Use a stable URL when the same link must always show the latest version of a file. A typical case
is a README preview that a CI job re-records on every release. Do not use it for PR screenshots:
a PR should keep showing what it showed at review time, so use the normal UUID flow there.

Add `X-Asset-Stable-Name` to a public upload:

```
POST https://static.sebastiano.dev/upload
Authorization: Bearer <token>
Content-Type: image/webp
X-Asset-Visibility: public
X-Asset-Owner: rock3r
X-Asset-Repo: bioparco
X-Asset-Stable-Name: grabby-stepper
```

The file is stored at `https://static.sebastiano.dev/stable/<token label>/<name>.<ext>`:

- The **token label** is the namespace. The server takes it from your token, and you cannot
  choose it. So a token can only create or overwrite stable files under its own label. This is
  also true for `admin` tokens.
- The **name** is `X-Asset-Stable-Name`. It may use lowercase letters, digits, `-` and `_`, and
  `/` to make sub-folders (for example `specimens/grabby-stepper`). It must not include the file
  extension. The maximum length is 128 characters.
- The **extension** comes from `Content-Type`. So `grabby-stepper` uploaded as `image/webp` and
  then as `video/mp4` gives two files, `grabby-stepper.webp` and `grabby-stepper.mp4`.

Rules for stable uploads:

- `X-Asset-Visibility` must be `public` (`400 stable_name_requires_public` otherwise).
- `X-Asset-Owner` and `X-Asset-Repo` are required. They are stored as metadata only.
  `X-Asset-Pr` is optional; if you send it, it must be numeric. `X-Asset-Name` is not used.
- If your token's label is not a safe path segment (lowercase letters, digits, `-`, `_`), the
  upload fails with `403 label_not_usable_as_namespace`. Ask the operator for a different token.

The response is `201` when the file is new and `200` when it replaced an existing file:

```json
{ "url": "https://static.sebastiano.dev/stable/bioparco/grabby-stepper.webp", "replaced": true }
```

Example for a CI job:

```bash
curl -sS --fail-with-body -X POST "https://static.sebastiano.dev/upload" \
  -H "Authorization: Bearer $STATIC_UPLOAD_TOKEN" \
  -H "Content-Type: image/webp" \
  -H "X-Asset-Visibility: public" \
  -H "X-Asset-Owner: rock3r" \
  -H "X-Asset-Repo: bioparco" \
  -H "X-Asset-Stable-Name: grabby-stepper" \
  --data-binary @grabby-stepper.webp
```

**How fast viewers see a new version.** Stable files are served with
`Cache-Control: public, max-age=60, must-revalidate`. Caches that follow this header (Cloudflare's
edge, browsers, and GitHub's image proxy) refresh within about one minute of an overwrite. The
old version can show for longer in a viewer whose browser or network cache ignores these rules.
The response `Content-Type` is the one you uploaded with (`image/webp`, `video/mp4`, ...).

## Deleting an asset

```
DELETE https://static.sebastiano.dev/upload
Authorization: Bearer <token>
X-Asset-Key: public/9a54b794-bcda-4cc4-a9e5-5c1d9e1b2ff3.png
```

Every token has a role, `admin` or `normal`. A `normal` token can only delete assets it uploaded
itself — deleting anyone else's key returns `404`, indistinguishable from that key never having
existed. An `admin` token can delete anything, any label. You don't get to choose which role your
token has; ask if you're not sure which one you were given.

Use `X-Asset-Key` (the `public/...`, `private/...` or `stable/...` path from a URL) to delete one asset — `404`
if it doesn't exist, or if it exists but you (a `normal` token) don't own it. Only delete assets
you or the user uploaded in this session — never delete on a guess.

`X-Asset-Delete-Uploader: <label>` instead of a key deletes **every asset uploaded under that
label**. A `normal` token may only pass its own label (anything else is `403`). There is no
per-PR bulk delete, since PR numbers aren't part of the key; if you need to clean up just what
you uploaded for one PR, track the individual URLs yourself and delete them one at a time by
exact key instead. Only use `X-Asset-Delete-Uploader` when a human has explicitly asked to wipe
everything a label uploaded — not as a routine cleanup step, even for your own label.

A deleted `public` URL can keep returning `200` for up to an hour afterward, and a deleted stable
URL for about a minute (Cloudflare's edge cache, not a failed delete) — don't retry a delete
because the URL still loads right after.

## Do not

- Do not upload files unrelated to the PR or demo in question.
- Do not retry more than once on a `401` — it means the token is wrong or missing, not transient.
- Do not guess owner/repo/PR values — ask the user if they're not obvious from context.
- Do not delete an asset unless the user or session context makes it unambiguous which one.
