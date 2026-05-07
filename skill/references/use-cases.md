# Use Cases — Production Patterns by Category

When the user asks „what can I do with this?" — match their intent to one of the patterns below. Patterns are organized into 4 categories. Each lists: when to use it, the API call shape, key parameters, gotchas, and code recipe location.

The active assistant should **suggest patterns proactively** based on `$BUDDYPRO_INSTANCE_TOPIC`. A marketing coach instance has different natural fits than a fitness trainer.

## The 4 categories

| Category | Examples | Common pattern |
|----------|----------|----------------|
| **A. Owner-direct automation** | Own scripts, daily reports, batch eval | API key alone, owner profile or stateless |
| **B. End-user products** | SaaS apps, web chat, member portals | `user` field per end-customer with stable memory |
| **C. Community integrations** | Telegram/WhatsApp/Facebook/Skool group bots | `user` per group member + context injection |
| **D. Team enablement** | Slack, MS Teams, internal tools | Team Slack bot, expert-brain backend for agents |
| **Cross-cutting** | Content generation, marketing automation | Mix of A+C+D depending on workflow |

## Category A — Owner-Direct Automation

The owner uses their bot from code for their own purposes. No end-customers involved.

### A1 — Single owner question (default, simplest)

**When:** Owner wants a one-off answer from their bot from a script. Cron job report, dashboard widget, internal alert response.

**Setup:** No `user` field. Default `saveToHistory: true` is fine for owner's own conversation log, OR `false` to keep owner's profile clean.

```bash
curl -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"messages": [{"role": "user", "content": "Summarize the top 3 trends in my latest content"}]}'
```

**Code recipe:** `code-recipes.md` → Pattern 1.

### A2 — Bulk evaluation / batch testing

**When:** 50–500 questions to evaluate answer quality, prompt variants A/B test, regression test before content update.

**Setup:** `x_buddy_saveToHistory: false` (no pollution). Optionally `user` per question for true isolation.

```bash
for q in test_questions/*.txt; do
  curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
    -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
    -H "Content-Type: application/json" \
    -d "$(jq -n --arg q "$(cat $q)" '{
      x_buddy_saveToHistory: false,
      messages: [{role: "user", content: $q}]
    }')"
  sleep 3
done
```

**Code recipe:** `code-recipes.md` → Pattern 3 (batch quality audit).

### A3 — Daily admin / monitoring scripts

**When:** Cron-driven reports, weekly summaries, automated KB freshness checks, after-hours diagnostics.

**Setup:** Owner profile (no `user`). Use management commands like `/stats`, `/checkSetup` (see `management-commands.md`). Read-only commands are 🟢 safe.

**Code recipe:** `code-recipes.md` → Pattern 7 (daily admin script).

### A4 — Programmatic content audit

**When:** Owner wants to systematically test their bot across all expertise domains and find knowledge gaps.

**Setup:** `/investigateAnswer:` via API + parsing of returned chunks. Optionally save to dashboard.

```bash
for domain in pricing sales copywriting email mindset productivity; do
  curl -s -X POST https://api.buddypro.ai/v1/chat/completions \
    -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
    -d "$(jq -n --arg d "$domain" '{
      messages: [{role: "user", content: ("/investigateAnswer:Best framework for " + $d + "?")}]
    }')" \
    | jq -r '.choices[0].message.content' > "audit-$domain.txt"
  sleep 3
done
```

## Category B — End-User Products

The owner builds a product where end-customers interact with the bot. Each customer has their own memory.

### B1 — Multi-tenant SaaS chat

**When:** Customer support tool, coaching app, courseware companion, embedded chat widget.

**Setup:** Stable `user` field per customer (UUID, email hash, internal customer ID). `saveToHistory: true` (default).

```bash
curl -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $BUDDYPRO_API_KEY" \
  -d '{
    "user": "customer-acme-42",
    "messages": [{"role": "user", "content": "What did we agree on last week?"}]
  }'
```

Customer #42 has full memory continuity. Customer #43 has totally separate profile.

**Code recipe:** `code-recipes.md` → Pattern 2 (Multi-tenant Express server).

### B2 — Public website chat widget (anonymous visitors)

**When:** Marketing site chatbot, lead-gen widget, „ask the AI version of [expert]" demo.

**Setup:** Generate session ID in browser (UUID stored in localStorage). Use as `user` field. Memory persists for that visitor across page reloads.

```javascript
// Browser
let userId = localStorage.getItem("buddypro_session");
if (!userId) {
  userId = `web-visitor-${crypto.randomUUID()}`;
  localStorage.setItem("buddypro_session", userId);
}

// Backend POST /chat → call BuddyPro API with user: userId
```

**Privacy:** Disclose in privacy policy that conversations are visible to the website owner.

### B3 — Member-only Q&A portal

**When:** Course platform, paid community, customer-only support area where members ask the AI version of the expert.

**Setup:** `user` = member ID from your auth system. Bot remembers each member's questions over time, builds profile.

```python
def member_chat(member_id, message):
    return call_buddypro({
        "user": f"member-{member_id}",
        "messages": [{"role": "user", "content": message}],
    })
```

### B4 — Different conversation contexts per same person

**When:** Owner runs multiple programs (sales calls, coaching, mood-tracking) with the same customers, but doesn't want them blending.

**Setup:** Suffixed `user` IDs per context.

```python
# Sales context — separate memory
call_buddypro({"user": "client-42-sales", "messages": [...]})

# Coaching context — different memory, no pricing baggage
call_buddypro({"user": "client-42-coaching", "messages": [...]})

# Mood-tracking — emotional space, isolated
call_buddypro({"user": "client-42-mood", "messages": [...]})
```

### B5 — Voice product (audio in + TTS out)

**When:** Phone IVR with bot's brain, voice memo to AI, audio-first mobile app.

**Setup:** `input_audio` content + `modalities: ["text", "audio"]` + per-customer `user` for memory.

```python
def voice_chat(audio_path, customer_id):
    audio_b64 = base64.b64encode(open(audio_path, 'rb').read()).decode()
    return call_buddypro({
        "user": f"voice-{customer_id}",
        "modalities": ["text", "audio"],
        "audio": {"format": "mp3"},
        "messages": [{
            "role": "user",
            "content": [{"type": "input_audio", "input_audio": {"data": audio_b64, "format": "mp3"}}]
        }],
    })
```

**Tip:** Set up voice clone of the expert (`/createVoiceClone`) — TTS replies sound like the real person.

**Code recipe:** `code-recipes.md` → Pattern 5 (voice loop).

### B6 — Image-driven product

**When:** Designer's portfolio review tool, visual support, OCR + analysis pipeline, product photo critique.

**Setup:** `image_url` content + per-customer `user`.

```python
def image_critique(image_url, instruction, customer_id):
    return call_buddypro({
        "user": f"customer-{customer_id}",
        "messages": [{
            "role": "user",
            "content": [
                {"type": "text", "text": instruction},
                {"type": "image_url", "image_url": {"url": image_url}}
            ]
        }],
    })
```

**Code recipe:** `code-recipes.md` → Pattern 6 (image critique).

### B7 — Tier-based persona (same KB, different feel per customer tier)

**When:** Beginners vs Pros vs VIPs all use the same knowledge but should get different treatment.

**Setup:** `user` per customer + `x_buddy_systemPrompt` per call (remember: per-call only, must send every time).

```python
TIER_PROMPTS = {
    "beginner": "## YOUR ROLE\nYou are a patient teacher for beginners...",
    "pro": "## YOUR ROLE\nYou are a no-fluff strategist for experienced practitioners...",
    "vip": "## YOUR ROLE\nYou are a peer-level expert. Treat user as your equal...",
}

def chat(customer_id, tier, message):
    return call_buddypro({
        "user": customer_id,
        "x_buddy_systemPrompt": TIER_PROMPTS[tier],
        "x_buddy_systemPromptMode": "add",
        "messages": [{"role": "user", "content": message}],
    })
```

### B8 — DM communication & sales (Instagram, Facebook, LinkedIn, X/Twitter)

**When:** Owner gets DMs (direct messages) on social platforms — mostly inquiries, sales conversations, lead qualification, customer support. Automate replies in the expert's voice while preserving memory of each individual conversation.

**Why this is huge:** Sales via DMs is one of the most time-consuming founder activities. With BuddyPro as the brain, the owner can have an AI version handling 80% of inbound DMs in their authentic voice — qualifying leads, answering questions, scheduling calls — while owner steps in only for complex/closing moments.

**Setup:**
- `user` = stable platform-user ID per DM contact (e.g., `ig-{instagram_user_id}`, `fb-{fb_user_id}`, `x-{twitter_handle}`, `linkedin-{member_id}`)
- Each contact has full memory of past DM exchanges
- For sales workflows: combine with `x_buddy_systemPrompt` (per-call) to load current campaign context, current promotions, qualifying questions

```python
# Generic DM handler (works across platforms via webhook)
def on_dm(platform, contact_id, message_text, sender_name=None):
    # Stable user ID per contact across calls
    user_id = f"{platform}-{contact_id}"
    
    # Optional: inject current sales context per call
    sales_context = f"""## CURRENT CONTEXT
- Campaign: Mid-year cohort enrollment ends Friday
- Active offer: 20% off annual plan with code SUMMER20
- Available calendar: book.example.com/15min

## YOUR APPROACH
- Match the contact's energy and language
- For pricing questions: send book.example.com link, suggest a 15-min call
- For tire-kickers: provide value, no hard pitch
- For warm leads: invite to call, don't oversell
- Reply in 1-3 short paragraphs, conversational tone
- Match their language (Czech/English/Spanish/etc.)"""
    
    response = call_buddypro({
        "user": user_id,
        "x_buddy_systemPrompt": sales_context,
        "x_buddy_systemPromptMode": "add",
        "messages": [{
            "role": "user",
            "content": f"{sender_name or 'Contact'} just DMed me: \"{message_text}\""
        }],
    })
    return response['choices'][0]['message']['content']
```

**3 sub-patterns within B8:**

#### B8a — Auto-reply (full automation)
Bot responds automatically. Owner reviews replies via dashboard or notification, intervenes if needed. Best for: low-stakes inquiries, FAQs, lead qualification.

```python
@webhook("/instagram/dm")
def on_instagram_dm(event):
    reply = on_dm("ig", event['from_id'], event['text'], event['from_name'])
    instagram_api.send_dm(event['from_id'], reply)
```

#### B8b — Suggested-reply (human-in-the-loop)
Bot drafts reply, owner reviews and approves before sending. Best for: high-stakes sales conversations, brand-sensitive contacts, closing moments.

```python
@webhook("/dm-incoming")
def draft_dm_reply(event):
    suggested = on_dm(event['platform'], event['from_id'], event['text'])
    # Push to owner's review UI (Slack notification, dashboard widget, etc.)
    notify_owner(suggested, original_msg=event['text'], approve_url=f"/approve/{event['id']}")
```

#### B8c — Sales sequence orchestration
Bot handles multi-message DM sales sequences — initial reply, follow-up if no response, qualifying questions, soft pitch, calendar booking link.

```python
def progress_sales_conversation(contact_id, last_msg_from_contact):
    # Get conversation state from your CRM
    state = get_lead_state(contact_id)
    
    stage_prompt = {
        "first_contact": "First reply — be warm, ask one qualifying question.",
        "qualifying": "They engaged. Ask one more discovery question to understand fit.",
        "qualified": "They're qualified. Offer the 15-min call link soft, no pressure.",
        "objection": "They have an objection. Acknowledge, reframe, redirect to a call.",
    }
    
    response = call_buddypro({
        "user": f"ig-{contact_id}",
        "x_buddy_systemPrompt": f"## SALES STAGE\n{stage_prompt[state['stage']]}",
        "x_buddy_systemPromptMode": "add",
        "messages": [{"role": "user", "content": last_msg_from_contact}],
    })
    
    advance_stage_if_appropriate(contact_id, response['choices'][0]['message']['content'])
    return response['choices'][0]['message']['content']
```

**Critical for B8 — always test in human-review mode first.** Never auto-send DMs to real prospects without weeks of supervised testing. One off-tone reply to a high-value lead costs more than the time saved.

**Privacy / brand caveats:**
- Disclose AI usage if platform/region requires (EU AI Act, FTC guidelines)
- Keep human handoff trigger phrases („let me get the founder on this")
- Audit responses weekly — drift detection
- For DMs containing sensitive info (medical, legal, financial), default to human-review or add explicit refuse policy in system prompt

**Code recipe:** see `code-recipes.md` Pattern 2 (multi-tenant Express server) — same structure, just different sources of `user_id`.

## Category C — Community Integrations

Bot embedded in group communication platforms. Each group member is a separate user. Group context (recent posts, ongoing thread) gets injected per call.

### C1 — Telegram group bot

**When:** Bot is added to a Telegram group, ideally as a participant. Each group member can ask, bot remembers per member.

**Setup:** `user` = `f"tg-{telegram_user_id}"`. Each member has their own memory.

```python
@bot_handler
def on_group_message(msg):
    member_user = f"tg-{msg.from_user.id}"
    return call_buddypro({
        "user": member_user,
        "messages": [{"role": "user", "content": msg.text}],
    })
```

### C2 — WhatsApp group bot (via relay)

**When:** WhatsApp Business API or Twilio bot relays messages from group to BuddyPro.

**Setup:** `user` = WhatsApp phone number hash. Same pattern as C1, different platform.

```python
def on_whatsapp_msg(phone_number, message):
    user = f"wa-{hashlib.sha256(phone_number.encode()).hexdigest()[:12]}"
    return call_buddypro({
        "user": user,
        "messages": [{"role": "user", "content": message}],
    })
```

**Privacy gotcha:** Hash phone numbers before using as user IDs — don't expose real numbers in BuddyPro storage.

### C3 — Facebook Group bot (Pages/Groups API)

**When:** Bot replies to comments or DMs in a Facebook Group as the page admin.

**Setup:** `user` = FB user ID, treated like Telegram member.

### C4 — Skool community bot

**When:** Bot answers questions in a Skool community as the founder's expert AI version.

**Setup:** `user` per Skool member ID, similar to C1-C3.

### C5 — Discord server bot

**When:** AI assistant for Discord communities, channel-specific or DM-based.

**Setup:** `user` = Discord user ID. Optionally suffix with channel ID for context separation: `discord-{user_id}-{channel_id}`.

### Critical for ALL community use cases — context injection

In a group chat, the bot's reply often needs context that's not in the bot's memory or knowledge base:

- **What did the user just say** (the trigger)
- **What was said in the group recently** (last 5–20 messages, possibly from multiple speakers)
- **The thread/topic of the ongoing conversation**
- **External context** (current campaign, ongoing event, this week's theme)

There are 3 ways to inject this context. Choose based on what you want to achieve:

#### Method 1 — Inject in user message (simplest, recommended)

Pass the context as part of the user message itself:

```python
context = "\n".join(f"{m.from_name}: {m.text}" for m in last_20_messages)

call_buddypro({
    "user": f"tg-{trigger.from_user.id}",
    "messages": [{
        "role": "user",
        "content": f"""Recent group conversation:
---
{context}
---

The user {trigger.from_user.name} just asked: "{trigger.text}"

Reply briefly to the user's question, taking the recent group conversation into account."""
    }],
})
```

**Why this works:** The bot sees the entire context within the user's „message" and can reason about who said what. Memory stays clean (only the user's actual question goes into long-term memory if you write a clean follow-up turn).

**Tip:** End the prepended context with a clear marker (`---`, `### END OF CONTEXT`) so the bot knows where the actual question begins.

#### Method 2 — Inject via `x_buddy_systemPrompt` (for stable group context)

If the context is „about this group/community" rather than per-message:

```python
GROUP_CONTEXT = """## GROUP YOU ARE IN

You are participating in 'Marketing Mastermind' — a private Telegram group for online entrepreneurs running 6–7 figure businesses. Topics: pricing strategy, paid ads, scaling.

## TODAY'S THEME
This week the group is discussing email marketing optimization.

## RECENT POSTS BY OTHERS (not the user asking)
- Jan: 'I lost 30% open rate after switching to ConvertKit'
- Maria: 'Has anyone tried Klaviyo flows?'
- Tom: 'Subject line A/B testing — recommendations?'

When responding, you can reference these recent posts if relevant."""

call_buddypro({
    "user": f"tg-{trigger.from_user.id}",
    "x_buddy_systemPrompt": GROUP_CONTEXT,
    "x_buddy_systemPromptMode": "add",
    "messages": [{"role": "user", "content": trigger.text}],
})
```

**Why this works:** Group context is treated as instance-level info per call, not as user-typed message. Cleaner separation between „who said what" (in user message) and „where this conversation lives" (in system prompt).

#### Method 3 — Hybrid: system prompt for stable, user message for fresh

Best for active group chats where context changes minute-to-minute:

```python
# Stable group identity in system prompt
STABLE_CONTEXT = "## GROUP\nThis is 'Marketing Mastermind'. Tone: peer-to-peer, no fluff."

# Fresh recent activity in user message
recent_msgs = format_last_n_messages(group_history, n=10)

call_buddypro({
    "user": f"tg-{trigger.from_user.id}",
    "x_buddy_systemPrompt": STABLE_CONTEXT,
    "x_buddy_systemPromptMode": "add",
    "messages": [{
        "role": "user",
        "content": f"""LAST 10 GROUP MESSAGES (most recent last):
{recent_msgs}

NOW {trigger.from_user.name} just asked:
{trigger.text}

Reply concisely."""
    }],
})
```

#### Multi-speaker context — formatting tips

When context contains messages from multiple people, format clearly so the bot can attribute correctly:

```
Maria: I'm losing customers after the trial.
Jan: How long is your trial?
Maria: 14 days.
Tom: 14 is too short for SaaS — try 30.
```

Don't just concatenate raw text. Always include speaker name + colon + message. Optionally add timestamps.

#### Don't write group context to long-term memory

Important: if you save group context into the user's profile every call, the bot's memory of the user gets polluted with other people's posts. Either:
- Use `x_buddy_saveToHistory: false` for context-heavy calls (memory of the OTHER members' content is NOT saved)
- Or strip context before saving (not directly possible via API — would require external memory management)
- Or accept that the user's profile contains group context (sometimes desirable for power users)

## Category D — Team Enablement

The most powerful pattern: connect the BuddyPro instance as a knowledge backend for the owner's TEAM, not just end-customers. Team members get on-demand access to the expert's brain.

### D1 — Slack bot for team

**When:** Owner has a team (employees, contractors, partners). Team members ask the AI version of the expert questions while working — pricing decisions, copy reviews, strategic guidance — without bothering the founder.

**Setup:** Slack bot relays @-mentions or slash commands to BuddyPro. `user` = Slack user ID per team member, so each team member's memory is isolated. Optionally team-shared user for collaborative context.

```python
# Slack event handler
@app.event("app_mention")
def on_mention(event, say):
    slack_user_id = event['user']
    question = event['text'].replace(f"<@{BOT_ID}>", "").strip()
    
    response = call_buddypro({
        "user": f"slack-{slack_user_id}",
        "messages": [{"role": "user", "content": question}],
    })
    say(response['choices'][0]['message']['content'])
```

**Why this is powerful:** instead of every team member needing to internalize the expert's knowledge, they have it on tap in their daily work environment. The bot remembers each team member's role, their projects, their patterns of asking. Over time it becomes more useful per person.

### D2 — MS Teams / other team platform bot

Same pattern as D1, different platform integration. `user` = Teams user ID.

### D3 — Internal Q&A in workspace tools (Notion, Linear, etc.)

**When:** Workflow automation that fetches expert opinion when filling out templates, reviewing PRs, or drafting comms.

**Setup:** Stateless calls (`saveToHistory: false`) — these are tool-driven, not conversation. No `user` field needed (owner profile is fine).

```python
def get_expert_opinion(context, question):
    return call_buddypro({
        "x_buddy_saveToHistory": False,
        "messages": [{
            "role": "user",
            "content": f"Context: {context}\n\nExpert question: {question}\n\nGive a focused answer in <100 words."
        }],
    })

# Use in Notion automation, GitHub Action, Linear webhook, etc.
```

### D4 — Agent backend (autonomous agent uses BuddyPro as expert brain)

**When:** Building an autonomous agent (Claude Code, custom LLM agent, n8n workflow) where the agent calls BuddyPro for domain expertise during reasoning.

**Setup:** Stateless or fresh `user` per agent task. Custom system prompt to frame the request properly.

```python
def agent_consults_expert(task_context, specific_question, agent_id):
    return call_buddypro({
        "user": f"agent-{agent_id}-{int(time.time())}",  # fresh per task
        "x_buddy_saveToHistory": False,
        "x_buddy_systemPrompt": "You are being consulted by an automated agent. Be precise, cite frameworks by name, give actionable next steps.",
        "x_buddy_systemPromptMode": "add",
        "messages": [{
            "role": "user",
            "content": f"AGENT TASK CONTEXT:\n{task_context}\n\nSPECIFIC QUESTION: {specific_question}",
        }],
    })
```

**Why this matters:** Owner's AI expert brain becomes a reusable component for many agents. Build agents that draft emails, write copy, analyze data — and have them consult the expert at decision points.

## Cross-cutting use cases (combinations of A+C+D)

### X1 — Content & marketing automation

**Examples:**
- **Reply to ad comments** — fetch comment, ask bot for on-brand response, post.
- **Social media post drafts** — bot generates 5 variants in expert's voice.
- **Newsletter writing assistant** — bot drafts newsletter sections based on theme.
- **Blog article research + draft** — bot does research call (deep query) then drafts in voice.
- **YouTube video script** — bot drafts hook, structure, key talking points.

**Setup:** Usually stateless (`saveToHistory: false`) per task. Custom system prompt for output format. No `user` field (owner-only workflow).

```python
def draft_social_post(topic, platform="twitter"):
    return call_buddypro({
        "x_buddy_saveToHistory": False,
        "x_buddy_systemPrompt": f"Output format: a {platform} post in the expert's voice. Single post, max 280 chars for Twitter, max 2200 for LinkedIn. End with one rhetorical question.",
        "x_buddy_systemPromptMode": "add",
        "messages": [{"role": "user", "content": f"Draft a {platform} post on: {topic}"}],
    })
```

### X2 — Deep research / market intelligence

**When:** Owner wants the bot to synthesize known knowledge with a new question (competitive analysis, trend report, opportunity scan).

**Setup:** Stateless (or fresh user) — single deep query. Often combined with `x_buddy_systemPrompt` for output format (executive summary, structured report).

```python
def deep_research(question):
    return call_buddypro({
        "x_buddy_saveToHistory": False,
        "x_buddy_systemPrompt": "## OUTPUT FORMAT\n## Executive Summary (3 sentences)\n## Key Insights (3-5 bullet points)\n## Recommendations (numbered actions)\n## Sources/Frameworks Used",
        "x_buddy_systemPromptMode": "add",
        "messages": [{"role": "user", "content": question}],
    })
```

## Decision flowchart — pick the right pattern

```
Who is the user of the bot?
│
├─ The owner themselves (scripts, automation, content creation)
│   └─ Category A
│       ├─ Single one-off question? → A1
│       ├─ Bulk testing / eval? → A2
│       ├─ Daily monitoring? → A3
│       └─ Knowledge audit? → A4
│
├─ End-customers of a product (each gets own memory)
│   └─ Category B
│       ├─ Multi-tenant chat? → B1
│       ├─ Web visitor? → B2
│       ├─ Paid member? → B3
│       ├─ Multiple programs per person? → B4
│       ├─ Voice product? → B5
│       ├─ Image product? → B6
│       ├─ Tier-based personas? → B7
│       └─ DM communication / sales (Instagram, FB, LinkedIn, X)? → B8
│
├─ Members of a community/group (Telegram, WhatsApp, FB, Skool, Discord)
│   └─ Category C
│       ├─ Pick platform (C1-C5)
│       └─ Always inject context (Method 1, 2, or 3)
│
├─ Owner's TEAM (Slack, Teams, internal tools)
│   └─ Category D
│       ├─ Slack/Teams bot? → D1/D2
│       ├─ Embedded in workspace? → D3
│       └─ Backend for autonomous agent? → D4
│
└─ Cross-cutting — content/research/automation
    └─ X1 (content gen) or X2 (deep research)
```

## Combining patterns

The real power: combine. Examples:

- **B1 + B5** = Voice SaaS where each customer has their own voice agent with own memory
- **B1 + X1** = Multi-tenant SaaS that auto-drafts replies in customer's voice
- **C1 + D4** = Telegram group bot that delegates complex queries to background research agent
- **B7 + D1** = Member portal (tier-based) plus Slack team support that knows the expert's voice
- **A2 + A4** = Daily KB audit script that surfaces gaps, posts to dashboard

## Tailoring suggestions to instance topic

When `$BUDDYPRO_INSTANCE_TOPIC` is known, the active assistant prioritizes patterns:

| Instance topic example | Most-likely patterns |
|------------------------|----------------------|
| Marketing coach for entrepreneurs | A4, B1, B7, B8 (DM sales), C-platforms, D1, X1, X2 |
| Personal fitness trainer | B1, B5 (voice during workout), B8 (Instagram DM coaching), C1 (community group), B6 (form check images) |
| Sales advisor B2B | A1, B1, B8 (LinkedIn DMs!), D1, D4 (agent backend for CRM), X1 (proposal drafts) |
| Mindfulness teacher | B5 (audio meditations), B3 (member sessions), B8 (DM check-ins), C2 (WhatsApp coaching group) |
| Writing coach | B7 (tier feedback), B8 (DM critique requests), D3 (in workspace), X1 (content drafts) |
| Community founder (Skool, Discord, Telegram) | C-platforms PRIMARY, B8 (DMs from members), D1 (mod team), B3 (member portal) |
| Internal team trainer / corporate L&D | D1, D2, D3, D4 — all team patterns |
| Solo content creator / influencer | B8 PRIMARY (Instagram/X DMs), X1 (content drafts), A4 (audience research) |

The skill should suggest the **2–3 patterns most relevant to the topic**, not all 15.

*Last updated: 2026-05-07 (v0.5.0 — expanded to 4 categories + context injection patterns + 8 new patterns)*
