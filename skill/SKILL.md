---
name: buddypro-owner-api
description: "Connect agents to a BuddyPro AI instance via the OpenAI-compatible Owner API (POST /v1/chat/completions). Use when user wants to integrate their BuddyPro bot with code, automate it, embed it in apps, run it for multiple end-users, send it text/images/audio, or trigger management commands programmatically. Triggers: BuddyPro API, bapi_, /v1/chat/completions, send to my BuddyPro bot, multi-tenant BuddyPro, BuddyPro multi-user, BuddyPro voice agent, BuddyPro image input, /buddypro-api."
trust-level: official
---

# BuddyPro Owner API Skill

Use this skill whenever the user wants to call their BuddyPro instance from code or agents — instead of chatting with it manually in Telegram.

## What is this skill for

The user owns a BuddyPro AI instance (a Telegram bot built from their expert knowledge). Now they want to:
- Embed the bot's brain into their own apps, agents, or automations
- Serve multiple end-users from one instance (each with isolated memory)
- Send images or audio to the bot
- Trigger management commands (like `/update`, `/investigateAnswer:`) without opening Telegram

This is done via the **Owner API**, an OpenAI-compatible REST endpoint at `https://api.buddypro.ai/v1/chat/completions`.

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

## Quick reference — load the right file

| User wants to... | Read this file |
|------------------|----------------|
| **First-time setup, missing API key, mental model briefing** | `references/getting-started.md` |
| **Choose right combination of `user` / saveToHistory / systemPrompt** | `references/api-features-deep-dive.md` |
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

### 2. The `user` field routes conversation to a profile.

| Request | Where conversation goes |
|---------|-------------------------|
| No `user` field | Owner's main profile (= the Telegram account that generated the key) |
| `"user": "joe-from-acme"` | Isolated profile named `joe-from-acme` (created on first use) |
| Same `user` value next time | Same profile, full memory continuity |

Use `user` for multi-tenancy (each end-customer = unique stable ID). **Without `user`, you pollute the owner's main profile fast** — every API call writes to the owner's chat history and memory.

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

*Version: 0.4.1 — see VERSION file*
