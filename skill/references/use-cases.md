# Use Cases — 7 Production Patterns

When the user asks „what can I do with this?" — match their intent to one of these 7 patterns. Each pattern lists: when to use it, the API call shape, key parameters, gotchas, and code recipe location.

The active assistant should **suggest patterns proactively** based on the user's `$BUDDYPRO_INSTANCE_TOPIC`. A marketing coach instance has different natural fits than a fitness trainer.

## Pattern 1 — Ask My Bot From Code (single-call automation)

**When:** Owner wants their own automation/script to ask the bot a question and use the answer. Cron jobs, internal dashboards, scripted reports.

**Key choice:** Owner conversation pollution.

| Variant | When | Setup |
|---------|------|-------|
| Owner profile (default) | The owner is the actual user, just from code | No `user` field |
| Stateless | One-off questions, no memory needed | `x_buddy_saveToHistory: false` |
| Test profile | Repeated automated tests | `user: "automation-job-name"` |

**Minimal call:**
```bash
curl -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "Summarize the top 3 trends from my latest content"}]}'
```

**Gotcha:** Without a `user` field, every call writes to the owner's main profile. Fine for owner's own scripts, terrible for any non-owner traffic. See `multi-tenancy.md`.

**Code recipe:** `code-recipes.md` → "Pattern 1 — Single owner call"

## Pattern 2 — Multi-Tenant SaaS (every customer has their own memory)

**When:** Building a product where each end-customer interacts with the bot independently. Customer support tools, coaching apps, embedded widgets.

**Key parameter:** stable `user` field per customer.

```bash
curl -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "user": "customer-acme-42",
    "messages": [{"role": "user", "content": "What did we agree on last week?"}]
  }'
```

The bot remembers everything customer-acme-42 said. Customer-acme-43 gets a totally separate profile.

**Crucial:** Use a stable, unique value per end-customer (UUID, email hash, internal customer ID — NOT username, which can change). The same `user` value across calls = same profile = continued memory. Different value = fresh profile.

**Privacy note:** All profiles are visible to the API key owner. The Owner API is **not** designed for true end-user privacy (a future End User API will solve this). Disclose this in your product's privacy policy.

**Code recipe:** `code-recipes.md` → "Pattern 2 — Multi-tenant Express server"
**Deep dive:** `multi-tenancy.md`

## Pattern 3 — Stateless One-Shot Q&A (evaluation, batch processing)

**When:** Sending 100 questions to evaluate answer quality, generating drafts that shouldn't pollute any profile, doing market research, A/B testing prompts.

**Key parameter:** `x_buddy_saveToHistory: false`.

```bash
curl -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "x_buddy_saveToHistory": false,
    "messages": [{"role": "user", "content": "Score this email draft 1-10 and explain..."}]
  }'
```

Nothing persists — no chat history, no memory updates, no profile mutations. The AI still uses the instance's knowledge base + system prompt for its answer.

**Use case examples:**
- Quality audit: ask 50 questions covering all topics, evaluate consistency
- Bulk classification: feed 500 customer messages, classify each
- Research: ask the same question 100 times to see variance

**Code recipe:** `code-recipes.md` → "Pattern 3 — Batch quality audit"

## Pattern 4 — Custom Persona (one-shot system prompt override)

**When:** The instance has knowledge for one topic, but you need a totally different persona for a single use case. Or you want to constrain answers to a strict format.

**Key parameter:** `x_buddy_systemPrompt` + `x_buddy_systemPromptMode`.

```bash
curl -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "x_buddy_systemPrompt": "You are a JSON API. Respond ONLY with valid JSON in the format: {\"summary\": \"...\", \"keyPoints\": [...]}",
    "x_buddy_systemPromptMode": "replace",
    "messages": [{"role": "user", "content": "Explain your best framework for pricing"}]
  }'
```

| Mode | Effect |
|------|--------|
| `add` (default) | Appends to base instance prompt — extends behavior |
| `replace` | Override base prompt entirely — pure custom persona |

The bot still uses the instance's knowledge base, but the personality, voice, and response format are dictated per-request.

**Use case examples:**
- JSON API wrapper for structured output
- Translation persona ("respond in French")
- Strict format ("answer in exactly 3 bullet points")
- Different audience ("explain for a 10-year-old")

**Combine with `x_buddy_saveToHistory: false`** for clean evaluation experiments — change persona without polluting any profile.

**Code recipe:** `code-recipes.md` → "Pattern 4 — Structured output wrapper"

## Pattern 5 — Voice Assistant (audio in + TTS out)

**When:** Building a voice product. Phone IVR with the bot's brain. Voice memos to AI. Audio-first mobile app.

**Key parameters:** `input_audio` content + `modalities: ["text", "audio"]` + `audio.format`.

```bash
AUDIO_B64=$(base64 -i question.mp3 | tr -d '\n')

curl -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg b "$AUDIO_B64" '{
    modalities: ["text", "audio"],
    audio: {format: "mp3"},
    user: "voice-customer-42",
    messages: [{
      role: "user",
      content: [{type: "input_audio", input_audio: {data: $b, format: "mp3"}}]
    }]
  }')"
```

Response includes both `choices[0].message.content` (text transcript) AND `choices[0].message.audio` (base64 MP3 of the TTS reply).

**Quality boost:** Set up a voice clone of the expert via `/createVoiceClone` — TTS replies sound like the real expert, not a generic voice.

**Cost watch:** TTS adds ~2× cost per message. Voice clone with music gen: 2–6×. Set `/enableVoiceWithMusicGeneration:true:LIMIT` to cap.

**Code recipe:** `code-recipes.md` → "Pattern 5 — Voice loop"

## Pattern 6 — Image Analysis Pipeline

**When:** Bot needs to analyze user-uploaded images (handwritten notes, screenshots, photos of products, design feedback).

**Key parameter:** `image_url` content type.

```bash
curl -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "user": "designer-customer-7",
    "messages": [{
      "role": "user",
      "content": [
        {"type": "text", "text": "Critique this landing page mockup based on your conversion principles"},
        {"type": "image_url", "image_url": {"url": "https://example.com/mockup.png"}}
      ]
    }]
  }'
```

**Constraints:**
- Max 5 images per request
- Max 40 MB per image
- HTTPS only (SSRF protection blocks private IPs)
- Or base64 inline: `{"url": "data:image/png;base64,..."}`

**Use case examples:**
- Pre-launch design review (designer's own bot reviews their work in their voice)
- Visual customer support (user shows a screenshot, bot helps)
- OCR + analysis (handwritten notes → digital advice)

**Code recipe:** `code-recipes.md` → "Pattern 6 — Image critique"

## Pattern 7 — Programmatic Instance Management

**When:** Owner wants to script their own dashboard, admin tools, daily reports, or maintenance jobs.

**Key insight:** The owner profile (no `user` field) can run management commands like `/update`, `/checkSetup`, `/stats`, `/investigateAnswer:`, `/messageAllUsers`. See `management-commands.md` for the full list.

**Examples:**

```bash
# Daily morning report
curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "/stats"}]}' \
  | jq -r '.choices[0].message.content' \
  | mail -s "BuddyPro daily" you@example.com

# Auto-trigger update after content sync
rsync new-content/ /path/to/drive-mounted-folder/SOURCES/TEXTS/
curl -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "/update"}]}'

# Quality audit across 12 domains
for domain in pricing sales copywriting email mindset productivity ...; do
  curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
    -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg d "$domain" '{messages: [{role: "user", content: ("/investigateAnswer:Best framework for " + $d + "?")}]}')" \
    | jq -r '.choices[0].message.content' > "audit-$domain.txt"
  sleep 3
done
```

**Gotcha:** Management commands require **owner permissions** — the API key must have been generated WITHOUT being inside `/test:` mode. See `troubleshooting.md` if you get 403.

**Code recipe:** `code-recipes.md` → "Pattern 7 — Daily admin script"

## How to choose the right pattern

The active assistant should walk the user through this:

```
User wants to use the bot from code
│
├─ Just for myself (the owner)
│   ├─ One-off question → Pattern 1 (single call, no user field)
│   ├─ Repeating without polluting → Pattern 1 + stateless flag
│   └─ Daily reports / admin tasks → Pattern 7
│
├─ For other people / customers / users
│   ├─ Each has own memory → Pattern 2 (multi-tenant)
│   ├─ Voice product → Pattern 5
│   └─ Image-driven product → Pattern 6
│
└─ Special use cases
    ├─ Different persona for one task → Pattern 4 (custom system prompt)
    ├─ Bulk evaluation / batch processing → Pattern 3 (stateless)
    └─ Knowledge gap testing → Pattern 7 (programmatic /investigateAnswer:)
```

## Combining patterns

Patterns combine. Examples:

- **Pattern 2 + Pattern 5** → Voice SaaS where every customer has their own voice agent with their own memory
- **Pattern 3 + Pattern 4** → Bulk evaluation with custom persona override (e.g., test 100 questions in JSON output mode without polluting)
- **Pattern 2 + Pattern 6** → Multi-tenant image critique service
- **Pattern 1 + Pattern 7** → Internal admin dashboard that combines reports with on-demand questions

## Tailoring suggestions to instance topic

When `$BUDDYPRO_INSTANCE_TOPIC` is known, the active assistant prioritizes patterns:

| Instance topic example | Most-likely patterns |
|------------------------|----------------------|
| Marketing coach for entrepreneurs | 1 (own scripts), 2 (coach-bot SaaS), 4 (structured advice) |
| Personal fitness trainer | 5 (voice during workout), 2 (per-client plans) |
| Sales advisor for B2B | 1 (CRM enrichment), 4 (response generation), 6 (proposal review) |
| Mindfulness teacher | 5 (audio meditations), 2 (per-student journey) |
| Writing coach | 4 (style critique), 6 (manuscript review) |
| Product manager mentor | 6 (UI mockup feedback), 1 (decision support) |

The skill should suggest 2–3 patterns most relevant to the topic, not all 7.

*Last updated: 2026-05-07 (v0.2.x)*
