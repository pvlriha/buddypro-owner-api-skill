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

## 🔴 Pre-execution protocol — for EVERY slash command

Before sending ANY slash command via the API:
1. **Verify exact syntax** in `references/management-commands.md` (parameter count, order, format, forbidden values)
2. **Verbalize what will happen** to the user in their language before sending
3. **Never guess parameters** — if ambiguous, ASK
4. **Watch for variable hazards** — pricing, customer IDs, URLs, language-sensitive keywords (`/setDefaultCost` period must be English)

## 🔴 Safety policy — risk-based confirmation

The API key controls the user's real production BuddyPro instance with real customers and real money. Many slash commands change instance behavior — most are irreversible.

**After pre-execution checks pass, classify and confirm:**

| Risk | Confirmation |
|------|--------------|
| 🟢 Read-only / sync | None |
| 🟡 Limited scope | Single yes/no |
| 🟠 Behavior/cost change | DOUBLE confirmation |
| 🔴 Mass impact / financial / irreversible | Risk warning + DOUBLE confirmation. Broadcasts (`/messageAllUsers`) require MANDATORY 2-step procedure (test send to owner first, then real broadcast) |

Full risk matrix in `references/management-commands.md`. **Never skip the procedure even if user pushes for shortcuts.**

🔴 **Always respond in the user's language.** BuddyPro owners are global. Detect language from how the user spoke to you. Keep technical identifiers (`bapi_`, `BUDDYPRO_API_KEY`, `/generateApiKey`, etc.) verbatim across languages.

$ARGUMENTS
