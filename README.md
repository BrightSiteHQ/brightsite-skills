# BrightSite Skills

Claude skills for agencies running client sites on [BrightSite](https://onbrightsite.com).

Each skill is a markdown file Claude reads and executes against the hosted [BrightSite MCP server](https://onbrightsite.com/mcp). Drop them into Claude Code, Claude Desktop, or Cursor and your assistant gains workflows like "migrate a WordPress blog," "generate location pages from a CSV," or "audit a site's SEO before handoff."

> **Status: early.** These skills target the BrightSite MCP as of this commit. They've been drafted from the tool definitions and reviewed for correctness, but each workflow benefits from being run against a real client account before you trust it with production data. Always start in `draft` mode and review in the editor.
>
> **Scope is bounded by the MCP.** Some QA checks are intentionally weaker than what you could do in the BrightSite dashboard, because the MCP doesn't expose every field over the wire (e.g. form delivery/recipient config). Skills will tell you when a check is limited and ask you to verify in the UI.

## Install

You need a [BrightSite](https://onbrightsite.com) account — the MCP is a hosted service, not something you run locally. Sign up, then follow the install instructions at [onbrightsite.com/mcp](https://onbrightsite.com/mcp) to connect the MCP server to Claude Code, Claude Desktop, or Cursor. The install snippet registers the server under the name `brightsite`, which is what these skills reference (`mcp__brightsite__<tool>`).

Once your MCP is connected, install the skills. Pick whichever path fits your setup:

### Option 1: `npx skills` (recommended, works with any agent)

```bash
# project-local (./.claude/skills)
npx skills add BrightSiteHQ/brightsite-skills

# or globally (~/.claude/skills)
npx skills add -g BrightSiteHQ/brightsite-skills
```

Powered by [vercel-labs/skills](https://github.com/vercel-labs/skills). Run `npx skills list` to see what's installed, `npx skills update` to pull latest.

### Option 2: Claude Code plugin marketplace

```text
/plugin marketplace add BrightSiteHQ/brightsite-skills
/plugin install brightsite-skills@brightsite
```

### Option 3: manual git clone

```bash
TMP=$(mktemp -d) && \
  git clone --depth 1 https://github.com/BrightSiteHQ/brightsite-skills.git "$TMP" && \
  mkdir -p ~/.claude/skills && \
  cp -R "$TMP"/skills/* ~/.claude/skills/ && \
  rm -rf "$TMP"
```

The final layout should be:

```text
~/.claude/skills/migrate-wordpress-blog/SKILL.md
~/.claude/skills/generate-location-pages/SKILL.md
~/.claude/skills/bulk-seo-audit/SKILL.md
~/.claude/skills/redirect-map-from-csv/SKILL.md
~/.claude/skills/client-handoff-checklist/SKILL.md
~/.claude/skills/blog-post-from-outline/SKILL.md
~/.claude/skills/visual-editor-authoring/SKILL.md
~/.claude/skills/visual-editor-authoring/editability-contract.md
~/.claude/skills/visual-editor-audit/SKILL.md
```

For project-local manual installs, copy individual skill directories (not just the `SKILL.md` files) into your project's `.claude/skills/` directory.

## Skills

| Skill | What it does |
|---|---|
| [migrate-wordpress-blog](skills/migrate-wordpress-blog/SKILL.md) | Import posts from a WordPress XML export into BrightSite. Preserves slugs and generates 301 redirects for any that change. |
| [generate-location-pages](skills/generate-location-pages/SKILL.md) | Programmatic SEO. Given a CSV of locations and a template, create one page per row. |
| [bulk-seo-audit](skills/bulk-seo-audit/SKILL.md) | Walk every page and post on a site, check titles, meta descriptions, headings, and canonicals, and output a prioritized fix list. |
| [redirect-map-from-csv](skills/redirect-map-from-csv/SKILL.md) | Turn a spreadsheet of old → new URLs into BrightSite redirects in one pass. |
| [client-handoff-checklist](skills/client-handoff-checklist/SKILL.md) | Pre-launch QA. Verify site identity, error pages, blog settings, forms, and analytics are configured before going live. |
| [blog-post-from-outline](skills/blog-post-from-outline/SKILL.md) | Turn an outline into a draft post with proper structure, meta tags, and excerpt. Always creates as draft. |
| [visual-editor-authoring](skills/visual-editor-authoring/SKILL.md) | Author pages, components, and layouts that are editable in the visual editor — so non-technical clients can change text, images, links, and props without a second pass. Load before writing HEEx. |
| [visual-editor-audit](skills/visual-editor-audit/SKILL.md) | Walk an existing site and flag what a client can't edit in the visual editor (no `data-bs-edit` markers, empty `props_schema`, bare loops, hardcoded links), then optionally fix it. |

## How to use a skill

Once installed, just ask your AI assistant in plain language. The skill descriptions are tuned to trigger on common phrasing — for example:

- "I have a WordPress export I need to import into account `8fd2k3jq91pn`."
- "Generate location pages for these 40 cities."
- "Run an SEO audit on this site before we hand it off."

The skill loads automatically and walks the assistant through inputs, confirmations, and the underlying MCP tool calls.

> **Account IDs.** BrightSite account IDs are unprefixed NanoIDs (e.g. `8fd2k3jq91pn`). You can find yours in the dashboard URL or under **Settings → API**. Wherever a skill asks for `YOUR_ACCOUNT_ID`, substitute your actual ID.

## Contributing

If you run sites on BrightSite and have a workflow worth sharing — a migration from another CMS, a particular pSEO pattern, an internal QA process — open a PR. See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

Quick version:

- One workflow per skill.
- Reference real MCP tools by their actual names (see [BrightSite MCP docs](https://onbrightsite.com/mcp)).
- No filler in the `SKILL.md` body — every section should help the model do the job better.
- Include an "Anti-patterns to avoid" section. The most useful skills are the ones that prevent mistakes, not just enable actions.

## License

MIT.
