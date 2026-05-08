---
name: buddypro-owner-api
description: "Talk to a BuddyPro AI instance from code via HTTPS REST API (POST /v1/chat/completions, OpenAI-compatible). NOT Telegram bot integration — this is an HTTP API for the bot's brain. Use when user wants to integrate their BuddyPro expert AI into their own apps, agents, automations, multi-tenant SaaS, or content workflows; or to trigger management commands programmatically. Triggers: BuddyPro API, Owner API, bapi_, /v1/chat/completions, send to my BuddyPro bot, talk to my BuddyPro from code, multi-tenant BuddyPro, BuddyPro multi-user, BuddyPro voice agent, BuddyPro image input, /buddypro-api, deep research with my BuddyPro."
trust-level: official
---

# BuddyPro Owner API Skill

🔴 **CRITICAL fact to communicate first:** This skill is for an **HTTPS REST API** (`POST https://api.buddypro.ai/v1/chat/completions`). It is **NOT** Telegram bot integration.

The user's BuddyPro AI lives on Telegram (where their customers chat with it), but separately exposes an OpenAI-compatible API endpoint that the user (or their code/agents) can hit directly. You're hitting the bot's BRAIN over HTTP — not chatting with it via Telegram.

When introducing the skill to a user, say it this way:

> *„BuddyPro Owner API gives you HTTP access to your bot's expertise. Same brain that answers in Telegram, but you can call it from code with curl, Python, Node.js — like you would call OpenAI's API. Authenticate with a `bapi_` key (you generate in Telegram once, then use anywhere)."*

Never describe it as „sending messages to your bot via Telegram API" — that's a fundamental misunderstanding.

## What this skill enables (and what it does NOT)

**Designed for the OWNER's own automations and trusted internal contexts:**
- Quick Q&A from code: pull expert answers into scripts, dashboards, cron jobs
- Stateless batch evaluation: A/B prompts, regression test answers, knowledge audits
- Custom personas per call (different system prompt for one task — same KB)
- Multi-step deep research: extract polished long-form knowledge from many angles
- Multimodal input: send images, audio in addition to text
- Management automation: trigger `/update`, `/stats`, `/investigateAnswer:`, etc. programmatically
- Internal team enablement: Slack/Teams bots where „users" are colleagues you trust with each other's data
- Content & marketing automation: drafts, social posts, newsletter sections in expert voice

**🔴 NOT designed for serving paying customers as a SaaS backend.** See the data-ownership warning below.

## 🔴 Data-ownership & privacy warning (ALWAYS surface during onboarding)

Per official docs (`https://docs.buddypro.ai/owner-api/`), the Owner API has a fundamental data-ownership model that the user MUST understand before integrating:

> *„When you generate an API key, it is tied to the user that generated it, and the conversation happening with that API key is actually being saved into message history (even though you don't see that conversation in Telegram) and contributes to the user profile and memory."*

Concrete consequences:

1. **All conversations through the key are saved into the GENERATING Telegram profile's data.** Even if you pass `user: "customer-42"`, that creates a sub-profile under YOUR account — not a privacy boundary between tenants.
2. **The `user` field is a label/sub-profile within your account, not a tenant boundary.** All sub-profiles are still owner-owned data.
3. **Generating the key on your real profile = your scripts/agents pollute your real Telegram chat history and memory.** Solution: switch to a test profile in Telegram with `/test:apitest` BEFORE running `/generateApiKey:...`.
4. **Real profile has access to dangerous management commands** (`/messageAllUsers`, `/setDefaultCost`, `/disableUser`, etc.). Agents holding a key generated on the real profile can trigger those.

🔴 **Therefore: do NOT recommend „Multi-tenant SaaS" / „Web chat widget for paying customers" / „Member-only Q&A portal" patterns as a primary use case.** They work technically (the bot will respond), but they violate end-customer data expectations because:
- The owner has full read access to all conversations
- There is no public API to delete a customer's profile
- A future End User API is planned to solve this — for true SaaS, wait for that

If the user explicitly asks for SaaS / external customer patterns, surface this warning IN FULL before designing the integration. Let the user make an informed choice.

## What this skill enables — short summary

- Embed the bot's brain into your own apps, agents, automations (HTTP requests, not Telegram polling)
- Send images, audio, multimodal input via standard request body
- Trigger management commands (`/update`, `/investigateAnswer:`) programmatically
- Build deep multi-step research that aggregates knowledge across many angles
- Connect internal Slack/Teams, voice products you operate, content pipelines you run

## On every invocation — run the active onboarding check

🔴 **Communicate in the user's language.** BuddyPro is global (Czech, English, Spanish, German, …). Detect the language from the user's most recent message and respond in that language. Keep technical identifiers (`bapi_`, `BUDDYPRO_API_KEY`, `BUDDYPRO_INSTANCE_TOPIC`, `/generateApiKey`, `/buddypro-api`) verbatim across all languages.

Before doing anything substantive, check the user's onboarding state:

```bash
# Are env vars set?
[ -z "$BUDDYPRO_API_KEY" ] && echo "MISSING_API_KEY"
[ -z "$BUDDYPRO_INSTANCE_TOPIC" ] && echo "MISSING_TOPIC"
[ ! -f "$HOME/.claude/skills/buddypro-owner-api/.onboarded" ] && echo "MISSING_MENTAL_MODEL_BRIEFING"

# `curl` available?
command -v curl >/dev/null || echo "MISSING_CURL"
```

If anything is missing, **load `references/getting-started.md` first** and walk the user through the relevant onboarding stages before answering their actual question. Don't assume readiness.

If everything is ready, proceed to active-assistant mode (described in `getting-started.md`).

## 🔗 Google Drive integration check (high-value bonus)

If the agent has Google Drive access (via `mcp__gdrive__*` tools, `mcp__google-drive__*`, or similar), it can dramatically improve instance understanding by reading the bot's `SYSTEM PROMPT` Drive doc. See `references/instance-management.md` → "Google Drive integration" section for full procedure.

When Drive access is available, the skill knows EXACTLY how the bot is supposed to behave (its persona, voice rules, frameworks) — leading to better custom prompts, more relevant use case suggestions, and direct edit capability.

🔴 **Critical gotcha when locating the right folder — DO NOT ask the bot.** The bot has **NO visibility into its own Google Drive folder**: it doesn't know the folder name, doesn't know what files exist there, doesn't know the structure. It only sees pre-indexed chunks served by the RAG system. Asking it „what's your Drive folder called?" or „list the files in your folder" will return a hallucination or a refusal — never trust that answer.

**Correct identification procedure:**

1. **Primary signal — shared with the BuddyPro service email + has the standard structure.** The folder MUST be shared with the BuddyPro ingestion service account (the email used to ingest the bot's KB — typically `service-bot@buddypro-...iam.gserviceaccount.com` or the equivalent service email shown in the user's `/folder` setup). Inside, the folder should contain at least: a `SYSTEM PROMPT` doc, a `URL SOURCES` doc, and a `SOURCES/` subfolder (with `TEXTS/`, `MEDIA/`, etc.). If both signals match, this is almost certainly a BuddyPro folder.

2. **Secondary signal — read the `SYSTEM PROMPT` doc and match content to what you know about the instance.** Once you find a candidate folder, open its `SYSTEM PROMPT` document. The opening lines reveal: bot name, expert role, target audience, voice rules. Compare against `$BUDDYPRO_INSTANCE_TOPIC` (or what the user told you the instance is about). If it matches → this is the right folder.

3. **If multiple BuddyPro-shaped folders exist (user has more than one instance)** — read the SYSTEM PROMPT of each, then ask the user: *„I found N BuddyPro folders. Which one is for the instance we're working with? It should be the one where SYSTEM PROMPT says it's about [topic / bot name]."* Don't guess.

4. **If still uncertain → ask the user directly:** *„What's your BuddyPro bot called? What does it specialize in?"* Use the answer to disambiguate against SYSTEM PROMPT doc content.

Never identify a folder by its display name alone (folder names vary widely and are often generic like „BuddyPro" or the bot's brand name). Always verify via SYSTEM PROMPT content match + service-email-share signal.

## 🎯 Deep Research detection

Before defaulting to a single API call, check if the user's request actually warrants deep research (multi-step, multi-branch). Triggers for deep research include:

- Explicit: „deep research", „prozkoumej do hloubky", „comprehensive guide on..."
- Implicit: „use my BuddyPro to write a webinar script", „prepare a launch strategy", „compare 3 frameworks", „extract everything you know about X", „draft a long-form newsletter on..."
- Context: owner provides a person's URL, product link, or asks for tiered/comparative content

If detected → load `references/deep-research-architecture.md` for the branch & merge architecture (sub-skill). Otherwise → use simpler patterns from `references/use-cases.md`.

## Quick reference — load the right file

| User wants to... | Read this file |
|------------------|----------------|
| **First-time setup, missing API key, mental model briefing** | `references/getting-started.md` |
| **Choose right combination of `user` / saveToHistory / systemPrompt** | `references/api-features-deep-dive.md` |
| **Build comprehensive multi-perspective research document (sub-skill)** | `references/deep-research-architecture.md` |
| Make a basic API call (text in, text out) | `references/api-reference.md` |
| Pick the right pattern for their use case | `references/use-cases.md` |
| Get ready-to-paste Python/Node/curl code | `references/code-recipes.md` |
| Serve multiple end-users (SaaS embedding) | `references/multi-tenancy.md` |
| Run `/update`, `/investigateAnswer:`, etc. via API | `references/management-commands.md` |
| Manage knowledge / system prompt / voice / roles | `references/instance-management.md` |
| Debug an error / rate limit / strange response | `references/troubleshooting.md` |
| Find the right official docs page | `references/docs-references.md` |

## Core mental model — read this once

The Owner API looks like OpenAI Chat Completions but behaves differently in 3 critical ways:

### 1. Server holds the conversation. Never send history.

Stock OpenAI: client sends all prior turns each call. **BuddyPro: server remembers everything.** Send only the current user message:

```json
{ "messages": [{ "role": "user", "content": "co bylo včera?" }] }
```

Sending prior turns = duplicate context = wasted tokens + confused model. This is the single most common mistake.

### 2. The `user` field = sub-profile WITHIN your account, not a tenant boundary.

| Request | Where conversation goes |
|---------|-------------------------|
| No `user` field | The generating Telegram profile (= whichever profile ran `/generateApiKey`; should be a `/test` profile, not your real one) |
| `"user": "label-x"` (first time) | New sub-profile `label-x` *inside the same account* — isolated memory from other sub-profiles |
| `"user": "label-x"` (later) | Same sub-profile, full memory continuity |

🔴 **`user` is a label, not a privacy boundary.** All sub-profiles created via your API key are still YOUR data on YOUR account — the owner has full read access to all of them. This is fine for owner-direct automations, internal tools, batch evaluations, deep research sessions, or trusted teams. It is NOT a SaaS multi-tenancy primitive. For serving paying customers as a SaaS, wait for the upcoming End User API. (Full warning at top of this file.)

Without `user`, every call writes to the generating profile's chat history and memory — which is why Step 0 of onboarding switches to a `/test:apitest` profile before generating the key.

### 3. Stateless mode exists for one-offs.

`"x_buddy_saveToHistory": false` → nothing persists. No history, no memory, no profile changes. The AI still answers using existing context. Use for evaluation, batch Q&A, or anything that shouldn't pollute a profile.

## Decision tree — when user asks „how do I X"

```
User wants to integrate BuddyPro
│
├─ Single integration, just for me (the owner)
│   └─ No user field needed. But add x_buddy_saveToHistory:false if you don't want pollution.
│
├─ Multiple end-users, each with own memory
│   └─ Pass user field per end-customer. See multi-tenancy.md
│
├─ One-off Q&A / batch evaluation / no memory
│   └─ x_buddy_saveToHistory:false. See use-cases.md "stateless"
│
├─ Custom persona (different from instance default)
│   └─ x_buddy_systemPrompt + x_buddy_systemPromptMode. See api-reference.md
│
├─ Voice / image / audio in or out
│   └─ multimodal content array. See api-reference.md "multimodal"
│
└─ Trigger /update, /investigateAnswer:, etc. without Telegram
    └─ Send slash command as content (owner profile only). See management-commands.md
```

## Critical rules

1. **NEVER send conversation history in `messages`.** Server holds it. Send only current user message.

2. **Always use `user` field for multi-tenant scenarios.** Otherwise everything pollutes owner's profile.

3. **`bapi_` keys appear ONCE.** When user generates a key in Telegram, capture it immediately. There is no „forgot password" — invalidate and regenerate.

4. **Rate limit is 30 req/min per key.** Above that → 429. For batch jobs, throttle.

5. **`/test` and `/untest` cannot be called via API.** Test profile context is set when key is generated. To make a test-bound key: in Telegram switch with `/test:scenario` first, then `/generateApiKey:scenario_key`.

6. **Don't expose `bapi_` keys client-side.** They have full access to the instance. Always proxy through your own backend.

## 🔴 Pre-Execution Protocol — for ANY slash command

Before sending ANY slash command via the API (even 🟢 read-only ones), the skill MUST:

1. **Look up the command** in `references/management-commands.md`. If not found there, check `https://docs.buddypro.ai/advanced/commands-list` (the canonical source). If still unclear, refuse to execute and tell the user „I don't know exactly what this command does — let me check the docs together first."

2. **Verify exact syntax** including:
   - Parameter count and order
   - Parameter format (string vs number, English vs localized values)
   - Required vs optional parameters
   - Forbidden values (e.g., `/setDefaultCost` period MUST be English `months`/`years`, not `měsíc`)
   - Length / format constraints (e.g., invite codes MUST be exactly 7 chars uppercase)

3. **Verbalize what will happen** to the user before sending. Example:
   > *„I'll run `/generateBuddyProInvite:50:WORKSHOP:10:30` which creates a trial invite code 'WORKSHOP' allowing up to 10 users with 50 trial messages each, expiring in 30 days. Confirm?"*

4. **Watch for variable hazards** — many commands take values that can have catastrophic effects if wrong:
   - Pricing: `/setDefaultCost:499:CZK:months:1` — wrong currency or period silently breaks subscription flow
   - User IDs: `/disableUser:12345` — wrong ID disables wrong customer
   - URLs: `/setFolder:{wrong-url}` — could lose connection to current Drive
   - Messages: `/messageAllUsers:false:text:all:dynamic:..."` — typo reaches all real customers
   - Booleans: `/messageAllUsers:false:...` — `false` means REAL SEND (not safe!)

5. **NEVER guess parameter format.** If the user gives ambiguous input, ask. „Did you mean message count or trial duration?" is always cheaper than an unrecoverable mistake.

6. **Run the safety policy check below** based on classified risk level.

If you have not done all 6 steps, you have not earned the right to send the command. No shortcuts.

## 🔴 Safety Policy — Critical Commands

The skill's API key controls a **real production BuddyPro instance with real users, real money, real customer data**. Many slash commands change instance behavior, customer experience, or pricing. Most are irreversible.

**Before passing any non-green slash command to the API, the skill MUST classify and confirm.** See `references/management-commands.md` for the complete risk matrix and confirmation templates. Quick summary:

| Level | Confirmation requirement |
|-------|--------------------------|
| 🟢 Read-only / Sync | No confirmation. Just execute. |
| 🟡 Limited scope changes | Single explicit yes/no in user's language |
| 🟠 Instance behavior / accumulating cost | DOUBLE confirmation — explain WHAT and WHY, get yes, then re-state and get yes again |
| 🔴 Mass impact / financial / irreversible | Risk warning + DOUBLE confirmation. **For broadcasts (`/messageAllUsers`): MANDATORY 2-step procedure — test send to owner first, then real broadcast only after explicit confirmation** |

🛑 **Never skip the procedure**, even if the user pushes for shortcuts. „I trust it, just send" is NOT permission to skip the test. The 5-second delay is the cheapest insurance against an unrecoverable mistake.

If you encounter a slash command not in the risk matrix, treat it as 🟠 by default and ask the user what it does before executing.

## Skill self-update check

Before answering ANY task, run this check (silent, log only the result):

```bash
LOCAL=$(cat "$HOME/.claude/skills/buddypro-owner-api/VERSION" 2>/dev/null || echo "0.0.0")
REMOTE=$(curl -fsSL https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main/VERSION 2>/dev/null || echo "$LOCAL")
[ "$LOCAL" != "$REMOTE" ] && echo "UPDATE_AVAILABLE: local=$LOCAL remote=$REMOTE" || echo "UP_TO_DATE: $LOCAL"
```

If `UPDATE_AVAILABLE`, mention it once at the start of your response — **in the user's language**. Reference templates:
- EN: „ℹ️ A new version of the BuddyPro skill is available (local X.Y.Z → remote A.B.C). To update, tell me: 'update the BuddyPro skill'."
- CZ: „ℹ️ Nová verze skillu je k dispozici (local X.Y.Z → remote A.B.C). Pro update mi řekni: 'updatuj BuddyPro skill'."

If user asks to update, fetch `https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main/INSTALL.md` and re-run the install procedure. (Production note: once `docs.buddypro.ai/skill` redirect is set up, that becomes the user-facing canonical URL — but the install procedure stays the same; only this URL changes.)

*Version: 0.9.0 — see VERSION file*

*v0.9.0 onboarding overhaul (2026-05-08): explicit HTTP-not-Telegram framing in description; mandatory `/test` profile switch as Step 0 of onboarding; prominent SaaS / privacy warning citing official docs; `user` field reframed as sub-profile within owner's account, NOT a tenant boundary; Drive folder identification gotcha (bot has no visibility into its own Drive — identify via service-email-share + SYSTEM PROMPT content match); value-first onboarding ending in 3 owner-direct demo prompts (no more SaaS demos by default); 4-line mental model briefing including latency (15-25s cold start, 3-8s warm), rate limit (30/min), cost (~$0.05/call); smart-default key storage (no A/B/C menu); auto-topic-detection from bot's first answer.*
