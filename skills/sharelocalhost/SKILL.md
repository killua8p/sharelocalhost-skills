---
name: sharelocalhost
description: >
  Publish a self-contained HTML file to a public URL a human can open, using
  ShareLocalHost (one-curl API at https://sharelocal.host). Use whenever you
  have produced an HTML report, dashboard, visual review, prototype or demo
  and need to hand the user a link rather than a local file path; also for
  re-publishing updated content to the same URL, password-protecting a page,
  or setting an expiry. Read the live contract at https://sharelocal.host/llms.txt
  for the current API, limits and error codes.
---

# Publish HTML with ShareLocalHost

ShareLocalHost turns one HTML file into a public page at
`https://<id>.local-8000.site/`. You POST the file, you get back the URL, an
`update_command` to replace the content at the same URL, and a per-page
`page_token` you can delegate to a subagent.

## Read the live docs

**The served contract is the source of truth.** This skill gives direction;
`curl -sS https://sharelocal.host/llms.txt` carries the current endpoints,
query params, limits and the full error table. Read it before your first
publish in a session and whenever a response surprises you.

## Before you publish

1. **Credential.** The API key is `$SLH_KEY`. Check `[ -n "$SLH_KEY" ]`. If it
   is unset, stop and tell the user: keys are issued manually by the operator,
   and the key belongs in `~/.config/sharelocalhost/env` sourced from
   `~/.zshenv` (not `~/.zshrc`; agent shells are non-interactive). Never ask
   the user to paste the key into chat.
2. **Tool.** Prefer the `slh-publish` wrapper when it is on PATH
   (`command -v slh-publish`). It reads `$SLH_KEY`, prints the URL, `page_id`
   and `page_token`, and surfaces the server's `hint` on error. In a Claude
   Code plugin install it also lives at `${CLAUDE_PLUGIN_ROOT}/bin/slh-publish`.
   Without it, fall back to the raw `curl` below.
3. **Content.** One self-contained HTML file, max 2 MiB, `Content-Type:
   text/html`. Inline all CSS and JS; embed images as `data:` URIs. By default
   the page can make **no network requests** (strict CSP). Only pass
   `--assets remote` / `?assets=remote` when the page genuinely needs CDN
   scripts, remote images or `fetch`, and say so to the user.
4. **Secrets.** Pages are public to anyone with the link and are never
   indexed. Never publish credentials, tokens or private data. For
   sensitive-ish reports add a password (defense in depth, not a vault).

## Publish

With the wrapper:

```bash
slh-publish report.html
slh-publish report.html --slug q3-report --ttl 30d --password 0303
```

Without it:

```bash
curl -sS https://sharelocal.host/v1/pages \
  -H "Authorization: Bearer $SLH_KEY" \
  -H "Content-Type: text/html" \
  -H "Idempotency-Key: <unique-per-task>" \
  --data-binary @report.html
```

Always use `--data-binary @file` (never `-d`, never `-F`). Always send an
`Idempotency-Key` on creates so a retry does not mint a duplicate page.

Options: `?slug=my-report` gives a readable URL prefix (3-30 chars of
`[a-z0-9-]`); `?ttl=7d` auto-expires (`30m`, `12h`, `7d`, up to `365d`);
header `X-Page-Password: <pw>` enables HTTP Basic protection (visitors enter
the password with any username).

## Hand the result to the user

Give the user the `url` (and the password, if you set one). Keep `page_id`
and `page_token` for yourself: the token grants update, read and delete rights
to that one page, cannot be revoked on its own, and should be passed to a
subagent instead of `$SLH_KEY` when delegating.

## Update an existing page

Re-publish to the same URL instead of creating a new page when the user asks
for changes to something you already shared:

```bash
slh-publish report.html --update <page_id>
# or: PUT https://sharelocal.host/v1/pages/<page_id> with the same headers/body
```

The PUT response includes `fresh_url`, a cache-busting variant that reflects
the new content immediately; share that if the user will look right away.
`If-Match: "<sha256>"` makes the update conditional (412 on mismatch).

## Errors

Every non-2xx response is JSON `{"error": {"code", "status", "message",
"hint"}}`. Read `hint`, fix the request, retry once. Common cases: `bad_key`
(check `$SLH_KEY`), `too_large` (compress or drop inlined assets), `not_html`
(body must look like HTML), `rate_limited` (wait `retry_after_seconds`),
`page_limit_reached` (delete old pages or publish with a `ttl`).

## Permissions in Claude Code

Publishing sends a local file to an external host, which Claude Code gates as
exfiltration. The project can pre-authorize exactly this command with
`"Bash(slh-publish:*)"` in `.claude/settings.json` `permissions.allow`. Do not
add that rule yourself; it is the user's trust decision. If it is absent,
expect and accept the per-call prompt.
