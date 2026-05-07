# Getting Started — Active Onboarding

> 🔴 **Communicate in the user's language.** All templates below are reference English. Translate naturally to the user's language. Keep technical strings (`bapi_`, `BUDDYPRO_API_KEY`, `BUDDYPRO_INSTANCE_TOPIC`, `/generateApiKey`, `/buddypro-api`) verbatim.

The first time a user invokes this skill (via `/buddypro-api` or by mentioning BuddyPro), check what state they're in and walk them through whatever's missing. Never assume readiness.

## Onboarding Decision Tree

```
Is BUDDYPRO_API_KEY set in the environment?
├─ No  → Run STAGE 1 (Generate API key)
└─ Yes → Run STAGE 2 (Verify it works)

Is BUDDYPRO_INSTANCE_TOPIC set?
├─ No  → Run STAGE 3 (Capture instance topic)
└─ Yes → Skip to "active assistant mode"

Has the user used the skill before?
├─ No  → Run STAGE 4 (Mental model briefing)
└─ Yes → Skip to "active assistant mode"
```

State for stages 1-3 is captured in the user's shell profile (`~/.zshrc`, `~/.bashrc`, etc.) so it persists across sessions:
- `BUDDYPRO_API_KEY` = the `bapi_` token
- `BUDDYPRO_INSTANCE_TOPIC` = a 1-line description of what the bot does, used to tailor advice

For stage 4 completion, write a marker file: `$HOME/.claude/skills/buddypro-owner-api/.onboarded` (zero bytes, presence = done).

## Stage 1 — Generate the API key

Tell the user:

> *„Before we can talk to your BuddyPro instance from code, we need an API key. Here's how to generate one — takes 30 seconds:*
> 
> *1. Open your BuddyPro bot in Telegram (the bot that's already set up with your knowledge).*
> *2. Send it: `/generateApiKey:my-agent`*
> *3. The bot replies with a key starting with `bapi_...`. **Copy it immediately** — it's shown only once.*
> *4. Paste it here when you have it."*

When the user pastes the key, validate format (`^bapi_[A-Za-z0-9_-]+$`, length > 20). If valid:

```bash
# Detect shell profile
if [ -n "${ZSH_VERSION:-}" ] || [ "$(basename "$SHELL")" = "zsh" ]; then
  PROFILE="$HOME/.zshrc"
else
  PROFILE="$HOME/.bashrc"
fi

# Append (don't overwrite previous values)
echo "" >> "$PROFILE"
echo "# BuddyPro Owner API" >> "$PROFILE"
echo 'export BUDDYPRO_API_KEY="bapi_THE_KEY_USER_PASTED"' >> "$PROFILE"

echo "✅ Saved to $PROFILE"
echo "Run: source $PROFILE  (or open a new terminal)"
```

🔴 **Never print the user's key back at them.** Confirm with the last 4 characters only: *„Saved key ending in `...{last4}`."*

After save, ask the user to run `source ~/.zshrc` (or open a new terminal), then proceed to Stage 2.

## Stage 2 — Verify the key works

Send a single test request to confirm the key is valid:

```bash
curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "x_buddy_saveToHistory": false,
    "messages": [{"role": "user", "content": "Reply with exactly: PONG"}]
  }'
```

Note: `x_buddy_saveToHistory: false` so this verification doesn't pollute the user's profile.

Parse the response:
- ✅ HTTP 200 + `choices[0].message.content` containing useful text → key works
- ❌ HTTP 401 + `error.code: invalid_api_key` → wrong key, restart Stage 1
- ❌ HTTP 429 → rate limit, wait 1 minute and retry
- ❌ Other → show user the raw error and the troubleshooting reference

If success, tell user (in their language):
> *„Your key works. Bot responded in {N} ms. Now I have one quick question to give you better recommendations..."*

## Stage 3 — Capture instance topic

Ask:

> *„In one sentence — what is your BuddyPro instance about? Who is the AI expert and what do they help with?"*
> 
> *Examples:*
> *- „Marketing coach for Czech online entrepreneurs"*
> *- „Personal fitness trainer specialized in calisthenics"*  
> *- „Sales advisor for B2B SaaS founders"*
> *- „Mindfulness teacher in the tradition of Jamie Smart"*

When user replies, save:

```bash
echo "export BUDDYPRO_INSTANCE_TOPIC=\"<their answer, escaped>\"" >> "$PROFILE"
```

This `$BUDDYPRO_INSTANCE_TOPIC` env var is read by the skill on every invocation so suggestions are tailored. Example: a marketing coach instance gets different use case suggestions than a fitness trainer.

## Stage 4 — Mental model briefing

Before the user does anything substantive, give them a 5-line mental model. **In their language.** EN reference:

> *„Quick orientation, 5 things you need to know:*
> 
> *1. **The server remembers everything.** When you call your bot through the API, you only ever send the current user message — never include past messages. The bot already remembers them, just like in Telegram.*
> 
> *2. **Two profile modes:** Without a `user` field, every API call writes to your own personal Telegram chat history with the bot. With `user: \"customer-123\"`, you create an isolated profile per end-customer (perfect for SaaS apps where each customer should have their own memory).*
> 
> *3. **Stateless mode** — set `x_buddy_saveToHistory: false` and nothing persists. Useful for batch testing or one-off questions.*
> 
> *4. **You can run all your management commands through the API too.** `/update`, `/investigateAnswer:`, `/stats`, etc. — all work via API, except 4 (`/generateApiKey`, `/invalidateApiKey`, `/test`, `/untest` are Telegram-only for security).*
> 
> *5. **Rate limit:** 30 requests per minute per key. Plenty for normal use; for batch jobs we'll throttle.*
> 
> *Want me to show you the most common patterns now, or do you have a specific task in mind?"*

After this, write the `.onboarded` marker:
```bash
mkdir -p "$HOME/.claude/skills/buddypro-owner-api"
touch "$HOME/.claude/skills/buddypro-owner-api/.onboarded"
```

## Active Assistant Mode (post-onboarding)

After all stages are complete, when user invokes `/buddypro-api` without specifics, **don't wait passively** — offer a menu of common scenarios tailored to their `$BUDDYPRO_INSTANCE_TOPIC`:

> *„What would you like to do? Common things owners of {topic} bots do via API:*
> 
> *1. **Send a one-off question** to your bot from code (great for testing)*
> *2. **Build an embedded chat** — multi-tenant version where every customer has their own memory*
> *3. **Run management commands** — refresh knowledge (`/update`), inspect answer quality (`/investigateAnswer:`), check stats*
> *4. **Voice/image processing** — send audio/images to your bot and get TTS replies*
> *5. **Custom persona for a single use case** — keep your knowledge base but override the system prompt for one request*
> *6. **Something else** — describe what you want."*

If user picks 1-5, jump to the matching reference file. If 6, ask for details, then map to the closest pattern in `use-cases.md`.

## Re-running onboarding

If user says „reset onboarding" or the markers/env vars are missing, restart from Stage 1. Don't be shy about asking again — it's better than acting on stale state.

*Last updated: 2026-05-07 (v0.2.0)*
