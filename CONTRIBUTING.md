# Contributing

Thanks for considering a contribution. This repo is small on purpose — each skill should pull its weight.

## What makes a good skill

A skill belongs here if it:

1. **Encodes a real workflow** you've run on a client site (not a hypothetical).
2. **Maps to BrightSite MCP tools** that exist today. If the workflow needs a tool that doesn't exist, open an issue instead of a skill — we'll consider adding the tool.
3. **Prevents a class of mistakes** the model would otherwise make. The most valuable section in any skill is usually "Anti-patterns to avoid."

If you're not sure whether your idea fits, open an issue first and we'll talk about it before you write the skill.

## Adding a skill

1. Fork the repo.
2. Create `skills/<your-skill-name>/SKILL.md` (kebab-case, one workflow per skill).
3. Use the frontmatter format the existing skills use — `name` and a `description` that includes the natural-language phrasings that should trigger it.
4. Reference MCP tools by their canonical `mcp__brightsite__<tool>` names. Don't invent tools.
5. Test the skill against a real (or test) BrightSite account before submitting. If you don't have one, say so in the PR and we'll run it.
6. Add a row to the table in `README.md`.

## Style

- **Be specific.** "Validate the input" is filler. "Coerce `redirect_type` from string to integer before passing to the MCP" is useful.
- **Call out the dangerous step.** Every skill has one — name it explicitly.
- **No emoji in skill outputs.** The skills tell the model not to use emoji unless the user asked; keep that consistent.
- **No marketing language.** Skills are read by a model deciding whether to invoke them. "Powerful," "seamless," "robust" are noise.

## Reporting bugs in a skill

Open an issue with:

- Which skill.
- The MCP tool call that misbehaved (paste the request and response if you have them).
- What you expected vs. what happened.

If a skill is misaligned with the current MCP schema (a field was renamed, a tool was added, etc.), that's a high-priority fix — please flag it.

## License

By contributing, you agree your contribution is licensed under the MIT license in this repo.
