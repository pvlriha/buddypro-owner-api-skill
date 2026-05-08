# BuddyPro Owner API

Load the skill `buddypro-owner-api` from `~/.claude/skills/buddypro-owner-api/SKILL.md` and apply it to the user's request.

## Always start with STEP 0 + STEP 1 from SKILL.md

1. **STEP 0** — auto-update check (see SKILL.md). Bust cache if user says „force update" or invokes `/buddypro-api-update`.
2. **STEP 1** — onboarding state + active-instance resolution (see SKILL.md):
   - If `instances.json` doesn't exist → run full 5-step onboarding via `references/getting-started.md`
   - If `instances.json` has 1 entry → use it as the active instance
   - If `instances.json` has 2+ entries → resolve which is active:
     - Did user mention a specific instance name? → use that one
     - Default to `instances.json["default_instance"]`
     - Tell user briefly which instance is active so they know

After STEP 0 + STEP 1, route based on the user's request:

- First-time onboarding, multi-instance management (add/list/switch/remove — all conversational), mental model briefing → `references/getting-started.md`
- Basic API call (text in/out, response parsing) → `references/api-reference.md`
- Choosing the right pattern → `references/use-cases.md`
- Ready-to-paste code (Python/Node/curl) → `references/code-recipes.md`
- Deep research (multi-step, multi-perspective) → `references/deep-research-architecture.md` (entry; sub-skills `deep-research-topologies.md`, `deep-research-scenarios.md`, `deep-research-blueprints.md`)
- Privacy story / `user` field semantics → `references/multi-tenancy.md`
- `/update`, `/investigateAnswer:` etc. via API → `references/management-commands.md`
- Knowledge base, system prompt, voice clone, roles, Drive folder identification → `references/instance-management.md`
- Errors / rate limits / clarifying-question gotcha → `references/troubleshooting.md`
- Official docs lookup → `references/docs-references.md`

## The only slash commands

- `/buddypro-api` (this) — primary entry point
- `/buddypro-api-update` — force update bypassing 4h cache TTL

**No `/buddypro-add-instance`, no `/buddypro-list-instances`, no per-instance slash commands.** Multi-instance management is conversational — user says *„přidej další instanci"*, *„seznam mých instancí"*, *„přepni na [name]"*, *„odeber [name]"* in plain language and the skill recognizes intent + runs the matching flow.

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
