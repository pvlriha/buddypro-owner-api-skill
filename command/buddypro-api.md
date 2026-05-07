# BuddyPro Owner API

Load the skill `buddypro-owner-api` from `~/.claude/skills/buddypro-owner-api/SKILL.md` and apply it to the user's request.

**Always start with the onboarding state check** described in SKILL.md. If `BUDDYPRO_API_KEY` is missing, or `BUDDYPRO_INSTANCE_TOPIC` is missing, or `.onboarded` marker is missing — load `references/getting-started.md` first.

After onboarding is complete, route based on the user's request:

- First-time, mental model briefing, missing config → `references/getting-started.md`
- Basic API call (text in/out, response parsing) → `references/api-reference.md`
- Choosing the right pattern → `references/use-cases.md`
- Ready-to-paste code (Python/Node/curl) → `references/code-recipes.md`
- Multi-tenant / multiple end-users → `references/multi-tenancy.md`
- `/update`, `/investigateAnswer:` etc. via API → `references/management-commands.md`
- Knowledge base, system prompt, voice clone, roles → `references/instance-management.md`
- Errors / rate limits / debugging → `references/troubleshooting.md`
- Official docs lookup → `references/docs-references.md`

🔴 **Always respond in the user's language.** BuddyPro owners are global. Detect language from how the user spoke to you. Keep technical identifiers (`bapi_`, `BUDDYPRO_API_KEY`, `/generateApiKey`, etc.) verbatim across languages.

$ARGUMENTS
