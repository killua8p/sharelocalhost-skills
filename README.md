# ShareLocalHost skills

The agent skill for [ShareLocalHost](https://sharelocal.host): one-curl HTML
publishing for AI agents. An agent POSTs a self-contained HTML file and gets
back a public URL to hand to a human.

This repo is a Claude Code plugin marketplace and an `npx skills`-compatible
skills repo. It contains only what an agent needs to install:

- `skills/sharelocalhost/SKILL.md` — when and how to publish, update, protect
  and expire pages; defers to the live contract at `curl https://sharelocal.host/`.
- `bin/slh-publish` — the allowlistable publish wrapper (`Bash(slh-publish:*)`).
- `.claude-plugin/` — marketplace and plugin manifests.

## Install

Paste this into your agent:

```
Install the ShareLocalHost skill. If you're in Claude Code, run `claude plugin marketplace add killua8p/sharelocalhost-skills`, then `claude plugin install sharelocalhost@sharelocalhost-skills`. If you're in another agent, run `npx skills add killua8p/sharelocalhost-skills --skill sharelocalhost` and select your agent. Use one installation method. You can read the skill directly at https://github.com/killua8p/sharelocalhost-skills/blob/main/skills/sharelocalhost/SKILL.md (raw: https://raw.githubusercontent.com/killua8p/sharelocalhost-skills/main/skills/sharelocalhost/SKILL.md). Then use the ShareLocalHost skill whenever you need to hand me a shareable link to an HTML page in this project.
```

Or by hand:

```bash
claude plugin marketplace add killua8p/sharelocalhost-skills
claude plugin install sharelocalhost@sharelocalhost-skills
```

```bash
npx skills add killua8p/sharelocalhost-skills --skill sharelocalhost
```

## You still need a key

Keys are issued by hand; there is no sign-up. Ask support@sharelocal.host.
Store it where your agent's non-interactive shell can see it:

```sh
mkdir -p ~/.config/sharelocalhost && chmod 700 ~/.config/sharelocalhost
printf 'export SLH_KEY=%s\n' 'slh_live_...' > ~/.config/sharelocalhost/env
chmod 600 ~/.config/sharelocalhost/env
echo '[ -f ~/.config/sharelocalhost/env ] && source ~/.config/sharelocalhost/env' >> ~/.zshenv
```

## Wrapper

`bin/slh-publish` reads `$SLH_KEY`, prints the URL, `page_id` and `page_token`,
and surfaces the server's error `hint`. Put it on your PATH, or use the copy a
Claude Code plugin install ships at `${CLAUDE_PLUGIN_ROOT}/bin/slh-publish`.

```sh
slh-publish report.html --password 0303 --slug q3-report --ttl 30d
slh-publish report.html --update <page_id>      # re-publish to the same URL
```

To let Claude Code publish without a per-call exfiltration prompt, allow
exactly this command in the project's `.claude/settings.json`:

```json
{ "permissions": { "allow": ["Bash(slh-publish:*)"] } }
```

That is a real trust decision: it lets the agent publish anything it can read
to a public URL.
