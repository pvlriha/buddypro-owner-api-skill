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

## Prerequisites — verify before any call

🔴 **Communicate in the user's language.** BuddyPro is global (Czech, English, Spanish, German, …). Detect the language from the user's most recent message and respond in that language. The templates below are reference English; translate to whatever the user is speaking.

1. **API key.** User must have a `bapi_` key. If `BUDDYPRO_API_KEY` env var is unset, tell user (in their language, EN reference template):
   > „I need your BuddyPro API key. Open your bot in Telegram, send `/generateApiKey:my-agent`, the bot will reply with a key starting with `bapi_...`. Then save it: `echo 'export BUDDYPRO_API_KEY=\"bapi_...\"' >> ~/.zshrc && source ~/.zshrc`."

   Wait for confirmation before proceeding. Keep the technical strings (`bapi_`, `BUDDYPRO_API_KEY`, `/generateApiKey`) verbatim across all languages.

2. **`curl` available.** Default on macOS/Linux/Win10+. If missing, instruct user to install it (in their language).

3. **Instance is set up.** The bot must have knowledge uploaded and `/update` run at least once. If user asks „why does it answer nothing?" → likely empty knowledge base.

## Quick reference — load the right file

| User wants to... | Read this file |
|------------------|----------------|
| Make a basic API call (text in, text out) | `references/api-reference.md` |
| Pick the right pattern for their use case | `references/use-cases.md` |
| Get ready-to-paste Python/Node/curl code | `references/code-recipes.md` |
| Serve multiple end-users (SaaS embedding) | `references/multi-tenancy.md` |
| Run `/update`, `/investigateAnswer:`, etc. via API | `references/management-commands.md` |
| Debug an error / rate limit / strange response | `references/troubleshooting.md` |

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

*Version: 0.1.2 — see VERSION file*
