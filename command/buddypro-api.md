# BuddyPro Owner API

Load the skill `buddypro-owner-api` from `~/.claude/skills/buddypro-owner-api/SKILL.md` and apply it to the user's request.

Based on what the user asks, also read the matching reference file:

- Basic API call (text in/out) → `references/api-reference.md`
- Choosing right pattern for the use case → `references/use-cases.md`
- Ready-to-paste code (Python/Node/curl) → `references/code-recipes.md`
- Multi-tenant / multiple end-users → `references/multi-tenancy.md`
- `/update`, `/investigateAnswer:` etc. via API → `references/management-commands.md`
- Errors / rate limits / debugging → `references/troubleshooting.md`

If `BUDDYPRO_API_KEY` env var is not set, first guide user through generating one (instructions in SKILL.md prerequisites).

$ARGUMENTS
