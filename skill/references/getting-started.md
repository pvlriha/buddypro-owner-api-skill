# Getting Started — Fast onboarding to BuddyPro Owner API

> 🔴 **First, get one fact straight:** This skill talks to your BuddyPro instance via **HTTPS REST API** (`POST /v1/chat/completions`). It is **NOT** Telegram bot integration — your bot continues to live on Telegram, but you (and your code) talk to its brain over a normal authenticated HTTP endpoint. Think „OpenAI-compatible API" but for YOUR expert AI.

🔴 **Communicate in the user's language.** All templates here are reference English. Translate naturally to whatever the user is speaking. Keep technical strings (`bapi_`, `BUDDYPRO_API_KEY`, `/generateApiKey`, `/test`, `/buddypro-api`) verbatim.

---

## 🔴 BEFORE ANYTHING ELSE — read this safety warning out loud to the user

This is the official safety warning from `https://docs.buddypro.ai/owner-api/`. You MUST surface it before generating any API key. Translate to the user's language but preserve all four points:

> *„Heads up before we generate an API key:*
>
> *1. The key is tied to whatever Telegram profile generates it. Conversations made through that key DO get saved into message history and contribute to that profile's memory — even though you won't see them in your Telegram chat.*
>
> *2. If you generate the key on your **real** Telegram profile, anything your scripts/agents send through the API will pollute your real chat history and memory.*
>
> *3. Your real profile also has access to management commands (`/messageAllUsers`, `/setDefaultCost`, etc.) — you don't want agents triggering those by accident.*
>
> *4. So before we generate the key: switch to a test profile in Telegram by sending `/test:apitest` to your bot. Then we'll generate the key under that test profile."*

If the user has ALREADY generated a key on their real profile, gently flag that this is suboptimal and offer to: invalidate the old key, switch to test profile, generate a fresh key. Don't just continue silently.

---

## The 4-step quick onboarding (≈90 seconds)

Goal: from zero to first real bot answer in under 90 seconds — using a test profile, not your real one.

### STEP 0 — Switch to test profile in Telegram (mandatory, ~10 seconds)

Tell the user:

> *„In Telegram, send your bot: `/test:apitest`*
> *Bot replies confirming you're now in test mode (test scenario name `apitest`). Anything we do via API now writes into a sandboxed test profile, not your real chat. Tell me when done."*

If user says „I'm already on a test profile" or „I already use /test daily" — fine, skip ahead. Otherwise wait for confirmation.

### STEP 1 — Generate the API key (Telegram, ~20 seconds)

Once on test profile, tell the user:

> *„Now still in Telegram (still in your test profile), send: `/generateApiKey:my-agent`*
> *Bot replies with a key starting `bapi_...`. Copy it now — shown only once. Paste it here."*

Validate format: `^bapi_[A-Za-z0-9_-]+$`, length > 20. Confirm with last 4 chars only (e.g., *„got key ending in `...cd30`"*) — never repeat the full key.

### STEP 2 — Smart-default save + verify (one call, ~20 seconds)

Decide WHERE to save the key by reading the environment, NOT by asking the user:

```bash
# Decision tree (run silently):
if [ -d ".claude" ] && [ -f ".gitignore" ]; then
    SAVE_TO=".env"           # in a project context → use local .env
elif [ -f "$HOME/.zshrc" ]; then
    SAVE_TO="$HOME/.zshrc"   # user's shell profile (default)
else
    SAVE_TO="$HOME/.bashrc"  # fallback
fi
```

**Don't present an A/B/C menu** — pick a sane default. If the user later wants to move it, they will say so.

Save:
```bash
# For ~/.zshrc or ~/.bashrc:
echo 'export BUDDYPRO_API_KEY="bapi_..."' >> "$SAVE_TO"

# For .env (in project):
echo 'BUDDYPRO_API_KEY=bapi_...' >> .env
chmod 600 .env
grep -q '^\.env$' .gitignore 2>/dev/null || echo '.env' >> .gitignore
```

Then test the key with a stateless ping:

```bash
curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"x_buddy_saveToHistory": false, "messages": [{"role": "user", "content": "What is your specialty in one sentence?"}]}'
```

`x_buddy_saveToHistory: false` keeps even the test profile clean during the verification call.

**Expected behavior:**
- ✅ Status 200 with `choices[0].message.content` containing a real one-sentence answer about the bot's expertise
- ⏱️ **First call typically takes 15-25 seconds** (cold start) — this is normal, mention it to the user so they don't think it's stuck. Subsequent calls run in 3-8 seconds.
- ❌ Status 401 → wrong key, restart Step 1
- ❌ Status 429 → rate-limit hit (30 req/min/key), wait 60s

The bot's reply already TELLS YOU what the instance does — you just got the topic for free. **Don't ask the user to describe their instance separately.**

### STEP 3 — Confirm topic + brief mental model + 3 demo prompts (~30 seconds)

Use the bot's own answer to infer `BUDDYPRO_INSTANCE_TOPIC`. Confirm with one question:

> *„Your bot says: „[exact bot answer]". I'll remember that as: „**[short topic phrase you extract from it]**". Sound right? (yes / clarify)"*

If user says yes — save:
```bash
echo 'export BUDDYPRO_INSTANCE_TOPIC="<extracted topic>"' >> "$SAVE_TO"
mkdir -p "$HOME/.claude/skills/buddypro-owner-api"
touch "$HOME/.claude/skills/buddypro-owner-api/.onboarded"
```

If user clarifies — save their version.

Then deliver this 4-line mental model briefing (translate to user's language):

> *„Quick orientation — 4 things to know about this API:*
>
> *1. **Server holds the conversation.** Send only the current user message; the bot already knows the history (just like in Telegram). This is the most common mistake when coming from OpenAI-style APIs.*
>
> *2. **`user` field = sub-profile within YOUR account.** Without `user` → writes to your test profile (good — you switched in Step 0). With `user: "label-x"` → separate sub-profile, isolated memory. ⚠️ This is **not** privacy isolation between paying customers — all sub-profiles are still your data on your account. (See `references/multi-tenancy.md` for the full privacy story.)*
>
> *3. **Stateless mode**: `x_buddy_saveToHistory: false` → nothing persists. Good for testing, batch evals, A/B prompt comparisons.*
>
> *4. **Limits**: 30 requests/minute per key. Cost ≈ $0.02-0.10 per call (typically $0.05). First call after idle = 15-25s; warm calls = 3-8s. Management commands (`/update`, `/stats`, `/investigateAnswer:`) work via API, except 4 (`/generateApiKey`, `/invalidateApiKey`, `/test`, `/untest`) which stay in Telegram for security."*

Then **immediately** show 3 ready-to-paste demo prompts they can run RIGHT NOW. Tailor topic mentions to `$BUDDYPRO_INSTANCE_TOPIC`. EN reference template:

> *„✅ You're set up. Try one of these to see what's possible — pick a number:*
>
> *1. **Quick Q&A from your bot** — `/buddypro-api ask "what is the most underrated principle in [topic-area] that most people miss?"`*
>
> *2. **Get a structured answer** — `/buddypro-api ask "give me 3 concrete frameworks for [topic-area], with one example each"`*
>
> *3. **Deep research document** — `/buddypro-api research [specific subtopic] and produce a 5-section markdown brief synthesizing your knowledge` (this kicks in the deep-research sub-skill — multi-step, ~$2-3, 8-15 min, polished long-form output)*
>
> *Or just describe what you want — I'll route it to your bot for you."*

For Czech audience, mirror with Czech demo prompts.

That's the END of onboarding. Total: ~90 seconds. The user already saw their bot answer once, knows the safety model (test profile + privacy boundary), has 4-line mental model, and has 3 concrete next moves.

---

## What the active assistant does in steady state (post-onboarding)

After all three env vars are set and `.onboarded` marker exists:

When the user invokes `/buddypro-api` without specifics, **don't show a generic menu**. Pick the 3 most relevant scenarios for `$BUDDYPRO_INSTANCE_TOPIC` and ask:

> *„What can I help with? For your [topic] expert, the most common things owners do are:*
> *1. [pattern 1 specific to this topic]*
> *2. [pattern 2 specific to this topic]*
> *3. [pattern 3 specific to this topic]*
> *4. Something else — describe it."*

(See `use-cases.md` „Tailoring suggestions to instance topic" table for which patterns map to which instance themes.)

🔴 **Do NOT proactively suggest „Multi-tenant SaaS chat" / „web chat widget for paying customers" / „member-only Q&A portal" patterns** — those have unresolved privacy issues (see warning at top of this file and in `multi-tenancy.md`). If user explicitly asks for them, surface the warning before designing the integration.

---

## Re-running onboarding

If user says „reset onboarding", or env vars / markers are missing, restart from Step 0. No friction — just go through it again.

```bash
# Reset:
rm -f "$HOME/.claude/skills/buddypro-owner-api/.onboarded"
unset BUDDYPRO_API_KEY BUDDYPRO_INSTANCE_TOPIC
# Then walk Steps 0-3 again
```

## Anti-patterns to avoid

- ❌ **Don't skip Step 0** (test profile switch). Generating the key on the user's real profile contaminates their real Telegram chat history and exposes management commands to agents.
- ❌ **Don't say „Stage 1 of onboarding"** — internal jargon. Say „first, let's switch to a test profile."
- ❌ **Don't ask „where to save the key" with a menu.** Pick a default; user can move it later.
- ❌ **Don't promise a „5-bullet mental model" and then stop.** If you say you'll explain something, explain it. The 4-line briefing in Step 3 IS the explanation — deliver it, don't defer.
- ❌ **Don't ask for instance topic separately when the bot's first answer already reveals it.** That's redundant work for the user.
- ❌ **Don't end onboarding with another question.** End with a concrete suggestion they can act on right now.
- ❌ **Don't describe the skill as „sending messages to your bot via Telegram API."** It is HTTP REST API. Misnaming this confuses the user about what they're getting.
- ❌ **Don't lead with „Multi-tenant SaaS" demo prompts** — that pattern has privacy issues (see warning at top). Lead with owner-direct prompts (Q&A, structured answer, deep research) which are unambiguously safe.

*Last updated: 2026-05-08 (v0.9.0 — onboarding overhaul: HTTP-not-Telegram clarity, mandatory `/test` profile switching, value-first flow, smart defaults, auto-topic-detect, 4-line mental model with latency+cost+rate-limit, no SaaS demo prompts)*
