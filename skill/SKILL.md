---
name: buddypro-owner-api
description: "Talk to a BuddyPro AI instance from code via HTTPS REST API (POST /v1/chat/completions, OpenAI-compatible). NOT Telegram bot integration — this is an HTTP API for the bot's brain. Use when user wants to integrate their BuddyPro expert AI into their own apps, agents, automations, multi-tenant SaaS, or content workflows; or to trigger management commands programmatically. Triggers: BuddyPro API, Owner API, bapi_, /v1/chat/completions, send to my BuddyPro bot, talk to my BuddyPro from code, multi-tenant BuddyPro, BuddyPro multi-user, BuddyPro voice agent, BuddyPro image input, /buddypro-api, deep research with my BuddyPro. Auto-trigger ALSO fires on the user's specific instance name once they tell us during onboarding (the name they gave their BuddyPro bot, e.g., the brand they call it in conversation) — that name is injected into this description by `getting-started.md` Step 3 for THIS user's local install only, never distributed."
trust-level: official
---

# BuddyPro Owner API Skill

## ⚡ Fresh agent? Read this 30-second TL;DR first

If you just loaded this skill and need to be useful in the next 30 seconds, here are the 8 facts that prevent the most common mistakes — read them, internalize, then proceed:

1. **It is an HTTPS REST API, NOT Telegram bot integration.** Same brain that answers in Telegram, but you call it from code via `POST https://api.buddypro.ai/v1/chat/completions`. Saying „I'll send messages to your bot via Telegram API" = wrong.

2. **Server holds the conversation. NEVER send history.** Send only the current user message; the bot already remembers prior turns from the `user` profile. Sending prior turns = duplicate context = wasted tokens + confused model. (Most common bug for OpenAI-API muscle-memory developers.)

3. **The `user` field is a sub-profile WITHIN the owner's account, NOT a tenant boundary.** Owner has full read access to all sub-profiles. Do NOT recommend Owner API as a SaaS multi-tenancy primitive. Wait for the End User API for true privacy isolation.

4. **STEP 0 of EVERY invocation: auto-update check.** Pin local `VERSION` against `https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main/VERSION`. If newer remote → silently auto-update via `https://docs.buddypro.ai/skill`. Cached 4h via `.last_version_check`. Pin via `.pinned_version` if user opts out.

5. **STEP 1: state is in `instances.json`** (multi-instance from v0.11.0). `~/.claude/skills/buddypro-owner-api/instances.json` is the canonical store. Each user can have 1 OR MANY BuddyPro instances; each entry has its own `bapi_` key, name, topic, slug. Legacy v0.10.x `state.env` is auto-migrated. Resolve active instance by: slug-specific slash command → default_instance → name match in user message → ask if ambiguous.

6. **First-time onboarding is 5 steps.** Privacy warning (out loud, official docs verbatim) → Step 0 (test profile `/test:apitest` in Telegram) → Step 1 (generate `bapi_` key) → Step 2 (verify BEFORE saving) → Step 3 (capture NAME + TOPIC, atomic write to instances.json via Python helper). Adding another instance = same flow, skip privacy warning + mental model briefing.

7. **EVERY deep-research call MUST include the anti-clarification directive in `x_buddy_systemPrompt` mode `add`.** Without it, bot defaults to clarifying questions instead of answering with frameworks. Full directive text in `references/deep-research-architecture.md` near the top. Stack with topology + role directives via concatenation.

8. **Deep research default = MASTER PATTERN: 3 forks × 6-8 turns × different topology × different role.** 18-24 calls / $0.90-1.20 / 3-5 min. Old phase-based hybrid (topology probe + Type A 12-turn + Type B forks) is fallback only. Mini-probes (3-5 calls) deprecated — minimum 6 calls per stage.

These 8 facts cover ~80% of mistakes a fresh agent makes. The detailed sections below explain WHY each fact matters and HOW to apply it.

---

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

## 🔴 STEP 0 — auto-update check (run FIRST on every invocation)

Before doing ANYTHING else (before answering, before onboarding check, before reading other reference files), run this version check. **This is mandatory on every invocation** — including auto-trigger invocations from description match, not just explicit `/buddypro-api` calls.

The check is cached (4-hour TTL) so it pings GitHub at most ~6× per day even in heavy use.

```bash
SKILL_DIR="$HOME/.claude/skills/buddypro-owner-api"
VERSION_FILE="$SKILL_DIR/VERSION"
CACHE_FILE="$SKILL_DIR/.last_version_check"
REMOTE_VERSION_URL="https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main/VERSION"
INSTALL_URL="https://docs.buddypro.ai/skill"
TTL_SECONDS=14400  # 4 hours

LOCAL=$(cat "$VERSION_FILE" 2>/dev/null | tr -d '[:space:]')
LOCAL=${LOCAL:-0.0.0}

# Cache check — skip GitHub fetch if last check was within TTL
NOW=$(date +%s)
LAST_CHECK=$(cat "$CACHE_FILE" 2>/dev/null | tr -d '[:space:]')
LAST_CHECK=${LAST_CHECK:-0}
ELAPSED=$((NOW - LAST_CHECK))

if [ "$ELAPSED" -lt "$TTL_SECONDS" ]; then
    echo "VERSION_CHECK_CACHED: local=$LOCAL (last checked ${ELAPSED}s ago, within ${TTL_SECONDS}s TTL)"
else
    REMOTE=$(curl -fsSL --max-time 5 "$REMOTE_VERSION_URL" 2>/dev/null | tr -d '[:space:]')
    if [ -z "$REMOTE" ]; then
        echo "VERSION_CHECK_FAILED: network unreachable, continuing with local=$LOCAL"
    else
        echo "$NOW" > "$CACHE_FILE"  # update cache timestamp
        if [ "$LOCAL" = "$REMOTE" ]; then
            echo "VERSION_UP_TO_DATE: $LOCAL"
        else
            echo "VERSION_UPDATE_AVAILABLE: local=$LOCAL → remote=$REMOTE"
        fi
    fi
fi
```

**Decision matrix:**

| Output | Action |
|--------|--------|
| `VERSION_CHECK_CACHED` | Continue normally — recent check confirmed up-to-date enough. |
| `VERSION_UP_TO_DATE` | Continue normally. No mention to user. |
| `VERSION_CHECK_FAILED` | Continue with local version. No mention to user (don't spam them with network errors). |
| `VERSION_UPDATE_AVAILABLE` | **Auto-update silently right now.** Fetch `https://docs.buddypro.ai/skill` (= `INSTALL.md`), extract its Step 1 bash block, run it. After successful update, **also fetch `https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main/CHANGELOG.md`** and extract the section for the new version. Surface ONE compact block at the start of your response (in user's language): *„🔔 Auto-updated BuddyPro skill: v[OLD] → v[NEW]. Hlavní změny: [3-5 word summary line 1] / [3-5 word summary line 2]. Pokračuji s aktuální verzí."* / EN equivalent. Then proceed with the user's actual task. |

🔴 **Auto-update is the default — don't ask the user for permission.** Updates preserve `state.env` and `.onboarded` (verified in INSTALL.md cp logic), so the user's onboarding is never lost. Asking *„Want me to update?"* on every new version is friction the user doesn't need. The notification line after auto-update is courtesy + audit trail.

🔴 **If the user EXPLICITLY says „nechci update" / „don't auto-update" / „pin version"** → write `$SKILL_DIR/.pinned_version` containing the current local version, and skip the version check entirely as long as that file exists. To unpin: `rm $SKILL_DIR/.pinned_version`.

```bash
# Honor pin
if [ -f "$SKILL_DIR/.pinned_version" ]; then
    PINNED=$(cat "$SKILL_DIR/.pinned_version" 2>/dev/null | tr -d '[:space:]')
    echo "VERSION_PINNED: $PINNED — skipping update check"
    # skip the rest of the version check block above
fi
```

**If auto-update fails partway** (curl error mid-install): the install script is atomic (downloads to TMP first, only `cp` to final location if every download succeeded), so a partial update cannot leave the skill in a broken state. On failure, continue with the local version + tell the user: *„Auto-update se nezdařil (network/GitHub issue). Pokračuju s lokální v[OLD]."*

---

## STEP 1 — onboarding state check & active-instance resolution (run AFTER Step 0)

🔴 **Communicate in the user's language.** BuddyPro is global (Czech, English, Spanish, German, …). Detect the language from the user's most recent message and respond in that language. Keep technical identifiers (`bapi_`, `BUDDYPRO_API_KEY`, `/generateApiKey`, `/buddypro-api`) verbatim across all languages.

🔴 **Multi-instance state lives in `instances.json`.** A user may have ONE or MANY BuddyPro instances. The single source of truth is `~/.claude/skills/buddypro-owner-api/instances.json`. `state.env` (legacy from v0.10.x) is migrated automatically and kept only for backward compatibility.

🔴 **Before doing ANY onboarding question, run the auto-discovery + migration scan.** The most common reasons a user „looks un-onboarded": (a) they DID onboard previously but in a different shell context, (b) they're on v0.10.x state.env and we haven't migrated yet, (c) this is a fresh Claude Code install but they have a `bapi_` key in some `.env`.

**Run this scan as a single bash block:**

```bash
SKILL_DIR="$HOME/.claude/skills/buddypro-owner-api"
INSTANCES_FILE="$SKILL_DIR/instances.json"
STATE_FILE_LEGACY="$SKILL_DIR/state.env"
ONBOARDED_MARKER="$SKILL_DIR/.onboarded"

# 1) AUTO-MIGRATE legacy state.env → instances.json (only if instances.json doesn't exist)
if [ -f "$STATE_FILE_LEGACY" ] && [ ! -f "$INSTANCES_FILE" ]; then
    # See getting-started.md → "Migration from v0.10.x state.env" for full Python migrator
    # (run it now if not already done)
    echo "MIGRATION_NEEDED: state.env → instances.json"
fi

# 2) PRIMARY check — instances.json exists and has at least 1 entry?
if [ -f "$INSTANCES_FILE" ] && command -v python3 >/dev/null; then
    INSTANCE_COUNT=$(python3 -c "import json; d=json.load(open('$INSTANCES_FILE')); print(len(d.get('instances',{})))" 2>/dev/null || echo 0)
    DEFAULT_SLUG=$(python3 -c "import json; d=json.load(open('$INSTANCES_FILE')); print(d.get('default_instance') or '')" 2>/dev/null)
else
    INSTANCE_COUNT=0
    DEFAULT_SLUG=""
fi

# 3) FALLBACK auto-discovery — only if instances.json missing AND legacy state.env missing
if [ "$INSTANCE_COUNT" = "0" ] && [ ! -f "$STATE_FILE_LEGACY" ]; then
    candidates=(
        "$HOME/.zshenv" "$HOME/.zshrc" "$HOME/.bash_profile" "$HOME/.bashrc"
        "$HOME/.profile" "$HOME/.env"
        "./.env" "../.env" "../../.env"
    )
    if git_root=$(git rev-parse --show-toplevel 2>/dev/null); then
        candidates+=("$git_root/.env")
    fi
    for candidate in "${candidates[@]}"; do
        if [ -f "$candidate" ] && grep -qE "^[[:space:]]*(export[[:space:]]+)?BUDDYPRO_API_KEY=" "$candidate" 2>/dev/null; then
            DISCOVERED_AT="$candidate"
            DISCOVERED_KEY=$(grep -E "^[[:space:]]*(export[[:space:]]+)?BUDDYPRO_API_KEY=" "$candidate" | head -1 | sed -E 's/^[[:space:]]*(export[[:space:]]+)?BUDDYPRO_API_KEY=//' | tr -d '"' | tr -d "'")
            echo "DISCOVERED_EXISTING_KEY_AT=$DISCOVERED_AT"
            break
        fi
    done
fi

# 4) Readiness summary
echo "INSTANCE_COUNT=$INSTANCE_COUNT"
echo "DEFAULT_INSTANCE=$DEFAULT_SLUG"
[ -f "$ONBOARDED_MARKER" ] && echo "MARKER_PRESENT" || echo "MARKER_MISSING"
command -v curl >/dev/null || echo "MISSING_CURL"
command -v python3 >/dev/null || echo "MISSING_PYTHON3"
```

**Decision matrix based on output:**

| Output combination | Interpretation | Action |
|---|---|---|
| `INSTANCE_COUNT≥1` + `MARKER_PRESENT` | ✅ Fully onboarded, multi-instance store ready | **SKIP onboarding entirely.** Resolve which instance is active (see "Active-instance resolution" below). Load that instance's API key + name + topic into env. Go to active-assistant mode. |
| `MIGRATION_NEEDED` | Legacy v0.10.x state.env exists, no instances.json | Run the Python migrator from `references/getting-started.md` § Migration. After migration completes (creates instances.json with 1 instance), proceed as if fully onboarded. |
| `INSTANCE_COUNT=0` + `DISCOVERED_EXISTING_KEY_AT=...` | Fresh Claude Code install but user has key in some `.env` | Auto-promotion path — see `references/getting-started.md` § "Auto-promotion: existing key found, no marker". Probe the key to extract bot name + topic, build instances.json with 1 entry, write per-instance slash command, inject into description. Skip privacy warning (already acknowledged in prior session). |
| `INSTANCE_COUNT=0` + `MARKER_MISSING` + no fallback key | True first-time install on this machine, no prior setup anywhere | Load `references/getting-started.md` and run the FULL 5-step onboarding (privacy warning → Step 0 test profile → Step 1 key → Step 2 verify → Step 3 capture name+topic+state). |
| `MISSING_CURL` or `MISSING_PYTHON3` | Environment lacks dependencies | Tell user — install missing dep, then retry. Both are needed (curl for API calls, python3 for instances.json + Step 3 atomic write). |
| User says „reset onboarding" / „forget my setup" / „start over" | Manual reset request | Run reset procedure (in `getting-started.md` § Reset modes). Per-instance or full-wipe based on user's exact phrasing. |
| User says „add another instance" / „new BuddyPro" / „další bot" | Add-instance request | Run getting-started.md Steps 0→3 again (skip privacy warning + 4-line briefing — already known). Append to instances.json. |
| User says „switch to [name]" / „use [name] instance" | Switch active instance for THIS conversation | Set runtime variable; do NOT change `default_instance` in instances.json unless user explicitly says „set as default". |

### Active-instance resolution (after `instances.json` is loaded)

When user invokes the skill, decide WHICH instance to use:

```bash
# Pseudo-logic — runs each invocation after onboarding check passed
ACTIVE_SLUG=""

# 1) If invoked via slug-specific slash command (/online-strateg, /buddypro-ai, etc.) → use that slug
#    The slash command file's body says "set the active instance to slug `xyz`" — agent reads it.

# 2) If invoked via /buddypro-api → use default_instance
[ -z "$ACTIVE_SLUG" ] && ACTIVE_SLUG="$DEFAULT_SLUG"

# 3) If user message mentions an instance NAME → match (case-insensitive substring) against instances[*].name
#    If exactly one match → use it. If multiple matches → ask user.

# 4) Load that instance's data
python3 - <<PY
import json, os, sys
data = json.load(open(os.environ['HOME'] + '/.claude/skills/buddypro-owner-api/instances.json'))
slug = "$ACTIVE_SLUG"
if slug not in data['instances']:
    print(f"ERROR: slug '{slug}' not in instances.json", file=sys.stderr); sys.exit(1)
inst = data['instances'][slug]
# Print export lines for shell to eval
print(f"export BUDDYPRO_API_KEY={inst['api_key']!r}")
print(f"export BUDDYPRO_INSTANCE_NAME={inst['name']!r}")
print(f"export BUDDYPRO_INSTANCE_TOPIC={inst['topic']!r}")
print(f"export BUDDYPRO_ACTIVE_SLUG={slug!r}")
PY
```

🔴 **Privacy warning is ONE-SHOT per user, not per instance.** Once `instances.json` has any entry with `privacy_warning_acknowledged=true`, never show the warning again — even when adding another instance. Repeating it = annoying noise.

🔴 **State files are preserved across skill upgrades.** INSTALL.md `cp` overwrites SKILL.md, references/, command/, VERSION, CHANGELOG.md. It does NOT touch `instances.json`, `.onboarded`, `state.env` (legacy), `.last_version_check`, `.pinned_version`. INSTALL.md post-install hook re-injects instance names into the new SKILL.md description so auto-trigger keeps firing on user-specific names.

**Reset & remove-instance procedures** are in `references/getting-started.md` § Reset modes. Quick summary:

| User intent | What gets removed |
|---|---|
| „reset onboarding" with 1 instance | Full reset (instances.json, .onboarded, that instance's slash command, .last_version_check) |
| „reset onboarding" with 2+ instances | Ask which one to remove; or „reset all" for full wipe |
| „remove instance [name]" | Just that one entry from instances.json + its slash command file |
| „reset all instances" | Full wipe (all instances.json entries, all `is_ours()` slash commands, all markers) |

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
| **First-time setup, missing API key, mental model briefing, ADD/LIST/REMOVE instance, multi-instance management** | `references/getting-started.md` |
| **Choose right combination of `user` / saveToHistory / systemPrompt** | `references/api-features-deep-dive.md` |
| **Build comprehensive multi-perspective research document (sub-skill — entry point)** | `references/deep-research-architecture.md` |
| Pick topology pattern for deep research forks (KRUH, HLOUBKA, ŠÍŘKA, INVERZE, ...) | `references/deep-research-topologies.md` |
| Match user request to deep-research scenario (universal how-to / person+product / comparative / tiered / content / audit / single-principle) | `references/deep-research-scenarios.md` |
| Run EXHAUSTIVE 8-phase deep research blueprint or implement orchestration code | `references/deep-research-blueprints.md` |
| **Find the right Google Drive folder for an instance** (gotcha — bot doesn't know its own folder) | `references/instance-management.md` § "Finding the BuddyPro folder" |
| Make a basic API call (text in, text out) | `references/api-reference.md` |
| Pick the right pattern for their use case | `references/use-cases.md` |
| Get ready-to-paste Python/Node/curl code | `references/code-recipes.md` |
| Serve multiple end-users (clarify privacy first — see warning section) | `references/multi-tenancy.md` |
| Run `/update`, `/investigateAnswer:`, etc. via API | `references/management-commands.md` |
| Manage knowledge / system prompt / voice / roles / Drive folder | `references/instance-management.md` |
| Debug an error / rate limit / strange response / clarifying-questions issue | `references/troubleshooting.md` |
| Find the right official docs page | `references/docs-references.md` |

## Core mental model

→ See `references/getting-started.md` § "STEP 3 — Capture instance NAME + TOPIC, write state, brief mental model + 3 demo prompts" for the full 4-line mental model briefing (server holds conversation; `user` field = sub-profile within owner's account; stateless mode for one-offs; rate-limit + cost + latency).

The 4 facts are repeated here in summary form for fresh agents who don't load getting-started.md:
1. **Server holds the conversation.** Send only current user message; bot remembers prior turns.
2. **`user` field = sub-profile within owner's account, NOT tenant boundary.** Full SaaS warning above.
3. **`x_buddy_saveToHistory: false` = stateless.** No memory, no history, no profile changes.
4. **Limits: 30 req/min/key, ~$0.05/call, 15-25s cold start, 3-8s warm.** 4 commands stay in Telegram (`/generateApiKey`, `/invalidateApiKey`, `/test`, `/untest`); the rest work via API.

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

*Version: 0.11.0 — see VERSION file*

*v0.11.0 multi-instance support + comprehensive audit fix release (2026-05-08): MAJOR storage schema change. Single source of truth migrated from `state.env` → `instances.json` (auto-migrated on first invocation, no user action needed). User can now have N BuddyPro instances; each entry in instances.json has own bapi_ key + name + topic + slug. New slash commands `/buddypro-add-instance` + `/buddypro-list-instances`. Per-instance slash commands auto-created with collision detection (won't overwrite /init etc). Active-instance resolution at every invocation: slug-specific slash command → default → name match in user message → ask if ambiguous. Fixed all 18 audit findings: K1 description leakage (no more hardcoded „Online Strateg" etc in distributable description), K2 heredoc placeholder bug (Python helper with env-var passing eliminates shell-injection from instance names with apostrophes/diacritics/ampersands), K3 missing INSTANCE_NAME export + migration logic, K4 `/buddypro-api-update` explicit bash, K5 unified „5-step onboarding" naming. Major: M2 slash command collision detection (`is_ours()` predicate), M3 unicodedata.normalize for slug generation, M4 agent-side question instead of bash `read -p`, M5 reset cleanup also wipes per-instance slash commands + injection markers, M6 python3 fallback handling (manifest declares it, INSTALL.md post-hook prints SKIPPED state, SKILL.md self-check explicit MISSING_PYTHON3), M7 post-install hook output documented. Minor: N1 sub-skill footers v0.11.0, N2 Quick ref Drive entry, N3 mental model deduplicated (single source = getting-started.md, SKILL.md just summarizes), N4 verify-before-save in Step 2, N5 CHANGELOG.md entry, N6 naming consistency.*

*v0.10.0 sub-skill split + instance alias auto-trigger + fresh-agent TL;DR + force-update slash command (2026-05-08): MAJOR refactoring release. Sub-skills: `deep-research-architecture.md` (1465 → 779 lines, entry point) + 3 new sub-skills `deep-research-topologies.md` (216 lines, 8 question patterns) + `deep-research-scenarios.md` (258 lines, 7 scenario adaptations) + `deep-research-blueprints.md` (317 lines, 8-phase pipeline + implementation code skeleton). Instance alias auto-trigger: onboarding now asks 2 explicit questions (instance NAME + topic), saves both to state.env (BUDDYPRO_INSTANCE_NAME + BUDDYPRO_INSTANCE_TOPIC), injects user's instance name into local SKILL.md description, creates per-instance slash command alias, persists alias list to `.instance-aliases` file. INSTALL.md post-install hook re-applies alias injection. Fresh-agent 30-second TL;DR added to top of SKILL.md. New `/buddypro-api-update` slash command. Quick reference table now routes to the right sub-skill.*

*v0.9.3 MASTER PATTERN promoted as default + anti-clarification propagated + standalone CHANGELOG.md (2026-05-08): `deep-research-architecture.md` Scenario 1 (Universal how-to) now leads with MASTER PATTERN (3 forks × 6-8 turns × different topology × different role) as the default pipeline (old 7-phase Type B+A hybrid relegated to fallback for very narrow topics); Optimal Blueprint section now leads with MASTER PATTERN (18-24 calls / $0.90-1.20 / 3-5 min — empirically validated 2026-05-08 at 18 calls / 6289 words / 1.5-2.9% cross-fork sim) and presents the exhaustive 6-stage variant as secondary; anti-clarification directive code skeleton now visible in `use-cases.md` X3 + A2 (callers see it at point of use, not just in architecture file); `troubleshooting.md` adds gotcha „Bot is asking clarifying questions instead of answering" with directive + curl example; standalone `CHANGELOG.md` created so update notifications can fetch only the relevant entry; auto-update Step 0 now fetches CHANGELOG.md after install and surfaces 3-5 word summary lines per update.*

*v0.9.2 auto-update + anti-clarification (2026-05-08): auto-update check moved to TOP of SKILL.md as Step 0 (was buried at the bottom and easily skipped by auto-trigger invocations); changed from passive „mention once" to silent auto-update with 4h cache TTL (avoids GitHub ping every invocation); explicit pin support via `.pinned_version` marker for users who don't want auto-updates; deep-research-architecture.md now has MANDATORY anti-clarification directive in CZ + EN, applied via `x_buddy_systemPrompt` mode `add` on EVERY API call (without it bot defaults to clarifying questions instead of answering with frameworks); 3-5 call mini-probes deprecated (minimum 6 calls per stage, fold smaller stages into larger ones); `add` mode preserves bot voice + persona while stripping clarifying behavior.*

*v0.9.1 onboarding state persistence (2026-05-08): introduced `state.env` as the single source of truth (env vars don't survive between Claude Code sessions reliably); self-check now exhaustively scans BOTH global home-level files (`~/.zshenv`, `~/.zshrc`, `~/.bash_profile`, `~/.bashrc`, `~/.profile`, `~/.env`) AND project-local `.env` files (`./.env`, `../.env`, `../../.env`, plus git-root `.env` if in a repo); auto-promotion path — when key is found in any fallback location but `state.env` and `.onboarded` marker are missing, agent auto-creates both and skips full onboarding (privacy warning never re-shown); INSTALL.md note clarifies that re-installs preserve `state.env` + `.onboarded` (they're outside the cp source list); explicit reset procedure documented.*

*v0.9.0 onboarding overhaul (2026-05-08): explicit HTTP-not-Telegram framing in description; mandatory `/test` profile switch as Step 0 of onboarding; prominent SaaS / privacy warning citing official docs; `user` field reframed as sub-profile within owner's account, NOT a tenant boundary; Drive folder identification gotcha (bot has no visibility into its own Drive — identify via service-email-share + SYSTEM PROMPT content match); value-first onboarding ending in 3 owner-direct demo prompts (no more SaaS demos by default); 4-line mental model briefing including latency (15-25s cold start, 3-8s warm), rate limit (30/min), cost (~$0.05/call); smart-default key storage (no A/B/C menu); auto-topic-detection from bot's first answer.*
