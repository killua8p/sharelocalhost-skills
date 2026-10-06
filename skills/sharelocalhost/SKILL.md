---
name: sharelocalhost
description: >
  Publish a self-contained HTML file to a public URL a human can open, using
  ShareLocalHost (https://sharelocal.host). Use whenever you have produced an
  HTML report, dashboard, visual review, prototype or demo and need to hand
  the user a link instead of a file path; also to re-publish updated content
  to the same URL, password-protect a page, or set an expiry. Covers first-run
  setup: if no key exists yet, it gets one from a claim code the user pastes
  into the chat, so the user never has to open a terminal.
---

# Publish HTML with ShareLocalHost

ShareLocalHost turns one HTML file into a public page at
`https://<id>.local-8000.site/`. You hand the user the URL. You keep the
`page_id` and a per-page `page_token` so you can update the page later or
delegate it to a subagent. Everything goes through one command, `slh-publish`,
which covers the whole API: `slh-publish --help` lists every subcommand and
option, so you never need to read the API document.

## 1. Find the wrapper

Try in this order and remember the path as `SLH`:

1. `command -v slh-publish` (already on PATH).
2. `${CLAUDE_PLUGIN_ROOT}/bin/slh-publish` (shipped with the Claude Code plugin).
3. Otherwise fetch it once from the skill's own public repo:
   ```bash
   mkdir -p ~/.config/sharelocalhost/bin && curl -fsSL \
     https://raw.githubusercontent.com/killua8p/sharelocalhost-skills/main/bin/slh-publish \
     -o ~/.config/sharelocalhost/bin/slh-publish && chmod +x ~/.config/sharelocalhost/bin/slh-publish
   ```
   Then `SLH=~/.config/sharelocalhost/bin/slh-publish`. Tell the user you did this.

Do not fall back to raw `curl` with the key on the command line; the wrapper
keeps the key out of your transcript.

## 2. Make sure there is a key (first run)

Run `$SLH doctor`.

- **`auth: ok`**: carry on.
- **`key: NOT FOUND`**: the user needs a one-time claim code. Say:
  "ShareLocalHost needs a one-time claim code to set up. Ask
  support@sharelocal.host for one and paste it here." When they paste
  something like `slh_claim_...`, run `$SLH login <code>`. It exchanges the
  code for a key, stores it at `~/.config/sharelocalhost/key` with mode 600,
  and prints nothing secret. Then re-run `doctor`.
- **`auth: FAILED`**: read the printed hint. A 401 means the key is bad or
  revoked; ask for a new claim code and `login` again.

Never ask the user for, print, or store the `slh_live_` key itself in the
chat. The claim code is the only credential that should ever appear there,
and it is single-use. If the user pastes an `slh_live_` key anyway, write it
to `~/.config/sharelocalhost/key` (mode 600) without echoing it, and suggest
they rotate it since it is now in a transcript.

## 3. Publish

Content rules first: one self-contained HTML file, max 2 MiB. Inline all CSS
and JS; embed images as `data:` URIs. By default the page can make **no
network requests**. Only pass `--assets remote` when the page genuinely needs
CDN scripts, remote images or `fetch`, and tell the user. Pages are public to
anyone with the link and never indexed. Never publish credentials, tokens or
private data; for sensitive-ish reports add a password (defense in depth, not
a vault).

```bash
$SLH report.html
$SLH report.html --slug q3-report --ttl 30d --password 0303
```

Options: `--slug my-report` gives a readable URL prefix (3-30 chars of
`[a-z0-9-]`); `--ttl 7d` auto-expires (`30m`, `12h`, `7d`, up to `365d`);
`--password <pw>` enables HTTP Basic protection (visitors enter the password
with any username).

The wrapper prints `Published: <url>`, `page_id` and `token`. Give the user
the URL (and the password if you set one). Keep `page_id` and the token for
yourself: the token grants update, read and delete rights to that one page,
cannot be revoked on its own, and is what you pass to a subagent instead of
the account key.

## 4. Update instead of re-creating

When the user asks for changes to something you already shared, re-publish to
the same URL:

```bash
$SLH report.html --update <page_id>
```

The output includes `Fresh: <url>?v=...`, a cache-busting variant that shows
the new content immediately; share that if the user will look right away.

## 5. Everything else the API offers

All through the same wrapper; add `--json` to any of these for the raw API
response.

```bash
$SLH list                         # every page on this key (100 per call; prints a --cursor for more)
$SLH get <page_id>                # metadata: version, size, sha256, expiry, protected, token
$SLH content <page_id> -o f.html  # read back the stored HTML (or omit -o to print it)
$SLH delete <page_id>             # the URL then serves 410 forever; ids are never reused
$SLH publish f.html --idempotency-key task-42     # a retried create returns the same page
$SLH update <id> f.html --if-match <sha256>       # 412 if someone changed it since you read it
$SLH update <id> f.html --no-password             # remove protection; --password sets/replaces it
$SLH update <id> f.html --ttl 7d --assets remote  # change expiry or the network policy
$SLH get <id> --token pt_...      # any per-page command can use a page token instead of the key
```

Use `--idempotency-key` whenever a publish might be retried (flaky network,
re-run task). Use `get` before `update --if-match` when another agent may be
editing the same page. When a subagent should manage one page, give it the
page's token and have it pass `--token`; never give it the account key.

## 6. Errors

Every failure prints `code: message` and a `hint`. Read the hint, fix the
request, retry once. Common: `too_large` (compress or drop inlined assets),
`not_html` (body must look like HTML), `rate_limited` (wait the printed
seconds), `page_limit_reached` (delete old pages or publish with `--ttl`),
`bad_key` (run `doctor`; probably needs a new claim code).

## Permissions in Claude Code

Publishing sends a local file to an external host, so Claude Code prompts on
the first `slh-publish` call. The user can choose "don't ask again" there, or
add `"Bash(slh-publish:*)"` to `.claude/settings.json` `permissions.allow`.
Do not add that rule yourself; it is the user's trust decision. If it is
absent, expect and accept the prompt.

## Live contract

`curl -sS https://sharelocal.host/` returns the full API reference (endpoints,
limits, error table). Read it when a response surprises you.
