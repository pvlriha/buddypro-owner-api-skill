# Getting Started — Onboarding flow (multi-instance aware, v0.11.0)

> 🔴 **First, get one fact straight:** This skill talks to BuddyPro instance(s) via **HTTPS REST API** (`POST /v1/chat/completions`). It is **NOT** Telegram bot integration — your bot continues to live on Telegram, but you (and your code) talk to its brain over a normal authenticated HTTP endpoint. Think „OpenAI-compatible API" but for YOUR expert AI.

🔴 **Communicate in the user's language.** All templates here are reference English. Translate naturally to whatever the user is speaking. Keep technical strings (`bapi_`, `BUDDYPRO_API_KEY`, `/generateApiKey`, `/test`, `/buddypro-api`) verbatim.

🔴 **Multi-instance from the start.** A user may have ONE BuddyPro instance or MANY (different brands/topics, each with its own `bapi_` key). The skill stores all instances in `~/.claude/skills/buddypro-owner-api/instances.json` and supports adding, listing, switching, removing per-instance state. When the user says „my Online Strateg" or invokes `/online-strateg` or just says „my BuddyPro" — the skill resolves which instance is active from this canonical store.

---

## 🔴 BEFORE FIRST API KEY EVER — read this safety warning out loud to the user

(Skip this section if `instances.json` already exists with at least 1 entry — privacy warning is one-shot per user, not per instance.)

This is the official safety warning from `https://docs.buddypro.ai/owner-api/`. You MUST surface it before generating the user's FIRST API key. Translate to the user's language but preserve all four points:

> *„Heads up before we generate any API key:*
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

## The 5-step onboarding (≈100 seconds for first instance, ≈60 seconds for additional)

This flow runs once per instance the user wants to add. Privacy warning fires only on the very first instance (since `instances.json` doesn't exist yet).

### STEP 0 — Switch to test profile in Telegram (mandatory, ~10 seconds)

Tell the user:

> *„In Telegram, send your bot: `/test:apitest`*
> *(or pick a different test scenario name if you'd prefer — e.g., `/test:scenario-online-strateg-api`. Just remember which one you used.)*
> *Bot replies confirming you're now in test mode. Anything we do via API now writes into a sandboxed test profile, not your real chat. Tell me when done."*

Capture the test scenario name (default `apitest`) — we'll save it in instances.json so the user knows which test profile this key is bound to.

If user says „I'm already on a test profile" or „I already use /test daily" — fine, just confirm which scenario name and skip ahead.

### STEP 1 — Generate the API key (Telegram, ~20 seconds)

Once on test profile, tell the user:

> *„Now still in Telegram (still in your test profile), send: `/generateApiKey:my-agent`*
> *(or pick a more descriptive label — e.g., `/generateApiKey:online-strateg-api` so you can track which agent uses which key.)*
> *Bot replies with a key starting `bapi_...`. Copy it now — shown only once. Paste it here."*

Validate format: `^bapi_[A-Za-z0-9_-]+$`, length > 20. Confirm with last 4 chars only (e.g., *„got key ending in `...cd30`"*) — never repeat the full key.

### STEP 2 — Verify the key BEFORE saving (one stateless call, ~20 seconds)

🔴 **Verify first, save second.** Don't write a possibly-invalid key to the user's shell profile. Test it with a stateless ping:

```bash
PROBE_RESPONSE=$(curl -s -w "\n%{http_code}" -X POST https://api.buddypro.ai/v1/chat/completions \
  -H "Authorization: Bearer $USER_PROVIDED_KEY" \
  -H "Content-Type: application/json" \
  -d '{"x_buddy_saveToHistory": false, "messages": [{"role": "user", "content": "What is your name and what is your specialty? One sentence each."}]}')
PROBE_HTTP=$(echo "$PROBE_RESPONSE" | tail -n1)
PROBE_BODY=$(echo "$PROBE_RESPONSE" | sed '$d')
```

`x_buddy_saveToHistory: false` keeps even the test profile clean during verification.

**Decision matrix:**

| HTTP | Action |
|------|--------|
| 200 | ✅ Key valid. Continue to Step 3 with `PROBE_BODY` (extract bot's self-intro). |
| 401 | ❌ Wrong key. Don't save. Go back to Step 1 (regenerate). |
| 429 | ⏱️ Rate-limit hit (30 req/min/key on a key being used elsewhere). Wait 60s, retry. |
| 5xx | 🔧 Transient server error. Retry 3x with backoff (3s, 9s, 27s). If still failing, tell user to try in 5 min. |

⏱️ **First call typically takes 15-25 seconds** (cold start) — this is normal, mention it to the user so they don't think it's stuck. Subsequent calls run in 3-8 seconds.

The bot's reply already TELLS YOU what the instance is about — you just got the topic for free.

### STEP 3 — Capture instance NAME + TOPIC + add to instances.json (~30 seconds)

🔴 **Ask the user TWO explicit questions** (not via bash `read` — out-of-band, agent asks user, user replies normally):

> *„Dvě rychlé otázky, ať vím, jak na tu instanci odkazovat:*
>
> *1. **Jak se jmenuje tvoje instance?** (jak jí říkáš v konverzaci, např. „Online Strateg", „BuddyPro AI", „Pavel AI", atd.)*
>
> *2. **O čem je?** (jednou větou — koho učí, v čem pomáhá)"*

Pre-fill suggestions from `PROBE_BODY` (Step 2 returned 1-sentence specialty, often with self-intro). Most BuddyPro bots open with „I'm [name]…" — extract the name as a suggestion. The user just confirms or tweaks; if extraction fails, ask without a pre-fill.

🔴 **Why both fields:** Once captured into `instances.json`, the skill:
1. Generates a slug from the name (e.g., „Online Strateg" → `online-strateg`)
2. Creates a per-instance slash command at `~/.claude/commands/[slug].md` (with collision check — never overwrite an existing command)
3. Re-injects all instance names into the LOCAL `SKILL.md` description (auto-trigger now fires on those names)
4. Stores everything atomically in `instances.json`

🔴 **Topic provides context** for tailoring suggestions (use-cases.md table); name provides the auto-trigger handle.

**Atomic write to instances.json + slash command + description injection:**

The skill should call this Python helper (the skill ships a snippet — agent just runs it). All shell-escaping is handled by Python (NO bash heredocs with user content):

```bash
SKILL_DIR="$HOME/.claude/skills/buddypro-owner-api"
mkdir -p "$SKILL_DIR" "$HOME/.claude/commands"

# Pass user-supplied values via env vars (Python reads them safely)
export BP_INSTANCE_NAME='<user's instance name verbatim — agent-substituted, no bash expansion>'
export BP_INSTANCE_TOPIC='<user's topic verbatim>'
export BP_API_KEY='<the bapi_... key>'
export BP_TEST_PROFILE='<test scenario from Step 0, e.g. apitest>'

python3 - <<'PY'
import os, json, re, sys, datetime
from pathlib import Path

SKILL_DIR = Path(os.environ['HOME']) / '.claude/skills/buddypro-owner-api'
COMMANDS_DIR = Path(os.environ['HOME']) / '.claude/commands'
INSTANCES_FILE = SKILL_DIR / 'instances.json'
SKILL_MD = SKILL_DIR / 'SKILL.md'

name = os.environ['BP_INSTANCE_NAME'].strip()
topic = os.environ['BP_INSTANCE_TOPIC'].strip()
api_key = os.environ['BP_API_KEY'].strip()
test_profile = os.environ.get('BP_TEST_PROFILE', 'apitest').strip() or 'apitest'

if not name or not api_key:
    print("ERROR: BP_INSTANCE_NAME and BP_API_KEY required", file=sys.stderr)
    sys.exit(1)

# Slug — lowercase, ASCII-only, dashes, max 40 chars
import unicodedata
ascii_name = unicodedata.normalize('NFKD', name).encode('ascii', 'ignore').decode('ascii')
slug = re.sub(r'[^a-z0-9-]', '', ascii_name.lower().replace(' ', '-')).strip('-')[:40] or 'buddypro-instance'

# Load existing instances.json or initialize
if INSTANCES_FILE.exists():
    with open(INSTANCES_FILE) as f:
        store = json.load(f)
else:
    store = {"schema_version": "0.11.0", "default_instance": None, "instances": {}}

# If slug already exists in store, suffix with -2, -3, ...
base_slug = slug
n = 2
while slug in store['instances']:
    slug = f"{base_slug}-{n}"
    n += 1

# If slash command file already exists at that slug AND it's not ours, suffix further
target_slash = COMMANDS_DIR / f"{slug}.md"
def is_ours(p: Path) -> bool:
    return p.exists() and p.read_text().startswith("# Alias for /buddypro-api")
n = 2
while target_slash.exists() and not is_ours(target_slash):
    slug = f"{base_slug}-{n}"
    target_slash = COMMANDS_DIR / f"{slug}.md"
    n += 1

# Add this instance
now = datetime.datetime.utcnow().strftime('%Y-%m-%dT%H:%M:%SZ')
store['instances'][slug] = {
    "slug": slug,
    "name": name,
    "topic": topic,
    "api_key": api_key,
    "test_profile_used": test_profile,
    "onboarded_at": now,
    "skill_version_at_onboarding": (SKILL_DIR / 'VERSION').read_text().strip() if (SKILL_DIR / 'VERSION').exists() else 'unknown',
    "privacy_warning_acknowledged": True,
    "onboarded_via": "manual",
}

# Set as default if first instance
if store['default_instance'] is None:
    store['default_instance'] = slug

# Atomic write
tmp = INSTANCES_FILE.with_suffix('.json.tmp')
with open(tmp, 'w') as f:
    json.dump(store, f, indent=2, ensure_ascii=False)
os.chmod(tmp, 0o600)
os.replace(tmp, INSTANCES_FILE)

# Write per-instance slash command
target_slash.write_text(
    f"""# Alias for /buddypro-api — instance: {name}

User invoked the skill via the instance-specific alias for `{slug}`. Topic: {topic}.

Load the skill `buddypro-owner-api` from `$HOME/.claude/skills/buddypro-owner-api/SKILL.md`. Run STEP 0 (auto-update) and STEP 1 (state hydration) per the skill's standard flow. **For STEP 1, set the active instance to slug `{slug}` from `instances.json`** (the user invoked this slug-specific command, so use this instance's API key/topic/name, not the default).

$ARGUMENTS
"""
)

# Re-inject ALL instance names into local SKILL.md description (replace any prior injection)
if SKILL_MD.exists():
    content = SKILL_MD.read_text(encoding='utf-8')
    m = re.search(r'^description:\s*"([^"]+)"', content, re.MULTILINE)
    if m:
        old_desc = m.group(1)
        # Strip prior injection block (between known markers) if present, then re-add
        marker_start = " AUTO-INJECTED-INSTANCE-NAMES:"
        marker_end = " :END-AUTO-INJECTED."
        if marker_start in old_desc:
            cleaned = re.sub(re.escape(marker_start) + r'.*?' + re.escape(marker_end), '', old_desc).rstrip(' .')
        else:
            cleaned = old_desc.rstrip(' .')
        names = sorted({inst['name'] for inst in store['instances'].values()})
        injected = f", {', '.join(names)}" if names else ''
        new_desc = cleaned + f".{marker_start}{injected} {marker_end}"
        content = content.replace(f'description: "{old_desc}"', f'description: "{new_desc}"', 1)
        SKILL_MD.write_text(content, encoding='utf-8')

# Write .onboarded marker (any instance counts)
(SKILL_DIR / '.onboarded').touch()

print(json.dumps({
    "ok": True,
    "slug": slug,
    "default_instance": store['default_instance'],
    "total_instances": len(store['instances']),
    "slash_command_path": str(target_slash),
    "instances_json_path": str(INSTANCES_FILE),
}, indent=2))
PY
```

**After successful run**, tell the user (in their language):

> *„✅ Instance „[NAME]" uložena. Slug: `[slug]`. Slash command `/[slug]` aktivní. Auto-trigger reaguje i na samotné jméno „[NAME]" v jakékoli další zprávě.*
>
> *Aktuálně máš [N] instance(í). Přidání další: `/buddypro-add-instance`. Seznam/přepnutí: `/buddypro-list-instances`."*

Then deliver the 4-line mental model briefing (only on FIRST instance — skip if `len(store['instances']) > 1`):

> *„Quick orientation — 4 things to know about this API:*
>
> *1. **Server holds the conversation.** Send only the current user message; the bot already knows the history (just like in Telegram). Most common mistake when coming from OpenAI-style APIs.*
>
> *2. **`user` field = sub-profile within YOUR account.** Without `user` → writes to your test profile (you switched in Step 0). With `user: "label-x"` → separate sub-profile, isolated memory. ⚠️ This is **not** privacy isolation between paying customers — all sub-profiles are still your data on your account. (See `references/multi-tenancy.md` for the full privacy story.)*
>
> *3. **Stateless mode**: `x_buddy_saveToHistory: false` → nothing persists. Good for testing, batch evals, A/B prompt comparisons.*
>
> *4. **Limits**: 30 requests/minute per key. Cost ≈ $0.02-0.10 per call (typically $0.05). First call after idle = 15-25s; warm calls = 3-8s. Management commands (`/update`, `/stats`, `/investigateAnswer:`) work via API, except 4 (`/generateApiKey`, `/invalidateApiKey`, `/test`, `/untest`) which stay in Telegram for security."*

Then **immediately** show 3 ready-to-paste demo prompts (only on FIRST instance, tailored to topic):

> *„✅ You're set up. Try one of these to see what's possible — pick a number:*
>
> *1. **Quick Q&A from your bot** — `/[slug] ask "what is the most underrated principle in [topic-area] that most people miss?"`*
>
> *2. **Get a structured answer** — `/[slug] ask "give me 3 concrete frameworks for [topic-area], with one example each"`*
>
> *3. **Deep research document** — `/[slug] research [specific subtopic] and produce a 5-section markdown brief synthesizing your knowledge` (this kicks in the deep-research sub-skill — multi-step, ~$2-3, 8-15 min, polished long-form output)*
>
> *Or just describe what you want — I'll route it to your bot for you."*

For Czech audience, mirror with Czech demo prompts.

That's the END of onboarding for this instance. Total: ~100 seconds for first instance, ~60 seconds for additional.

---

## STEP 0/1 in steady state (post-onboarding)

After `instances.json` exists with ≥1 entry:

When the user invokes the skill (via `/buddypro-api`, slash alias, or auto-trigger from instance name mention):

1. **Resolve which instance is active:**
   - If invoked via slug-specific slash command → use that slug
   - If invoked via `/buddypro-api` → use `instances.json["default_instance"]`
   - If user message mentions an instance name → match against `instances[*].name` (case-insensitive substring match) and use that one
   - If multiple matches or unclear → ask: *„Máš [N] instancí: [list]. Kterou použít pro tento task?"*

2. **Load that instance's API key into env:**
   ```bash
   export BUDDYPRO_API_KEY=$(python3 -c "import json; print(json.load(open('$HOME/.claude/skills/buddypro-owner-api/instances.json'))['instances']['$ACTIVE_SLUG']['api_key'])")
   export BUDDYPRO_INSTANCE_NAME=$(...same for name)
   export BUDDYPRO_INSTANCE_TOPIC=$(...same for topic)
   ```

3. **Skip onboarding entirely.** No privacy warning, no 4-line briefing, no demo prompts. Go straight to active-assistant mode and answer the user's actual task.

When the user invokes `/buddypro-api` without specifics, **don't show a generic menu**. Pick the 3 most relevant scenarios for the active instance's topic and ask:

> *„What can I help with? For your [active instance name] expert, the most common things owners do are:*
> *1. [pattern 1 specific to this topic]*
> *2. [pattern 2 specific to this topic]*
> *3. [pattern 3 specific to this topic]*
> *4. Something else — describe it.*
>
> *Or — chceš pracovat s jinou instancí? `/buddypro-list-instances` zobrazí všechny."*

(See `use-cases.md` „Tailoring suggestions to instance topic" table for which patterns map to which instance themes.)

🔴 **Do NOT proactively suggest „Multi-tenant SaaS chat" / „web chat widget for paying customers" / „member-only Q&A portal" patterns** — those have unresolved privacy issues (see warning at top of this file and in `multi-tenancy.md`). If user explicitly asks for them, surface the warning before designing the integration.

---

## Migration from v0.10.x state.env → v0.11.0 instances.json

If the user installed an older version with `state.env` but no `instances.json`, the skill auto-migrates on first invocation:

```bash
SKILL_DIR="$HOME/.claude/skills/buddypro-owner-api"
STATE_FILE="$SKILL_DIR/state.env"
INSTANCES_FILE="$SKILL_DIR/instances.json"

if [ -f "$STATE_FILE" ] && [ ! -f "$INSTANCES_FILE" ]; then
    python3 - <<'PY'
import os, json, re, datetime
from pathlib import Path

SKILL_DIR = Path(os.environ['HOME']) / '.claude/skills/buddypro-owner-api'
state_file = SKILL_DIR / 'state.env'
target = SKILL_DIR / 'instances.json'

# Parse state.env (shell-format, simple KEY="value")
data = {}
for line in state_file.read_text().splitlines():
    line = line.strip()
    if not line or line.startswith('#'):
        continue
    if '=' in line:
        k, v = line.split('=', 1)
        v = v.strip().strip('"').strip("'")
        data[k.strip()] = v

api_key = data.get('BUDDYPRO_API_KEY', '').strip()
name = data.get('BUDDYPRO_INSTANCE_NAME', '').strip() or 'My BuddyPro'
topic = data.get('BUDDYPRO_INSTANCE_TOPIC', '').strip() or 'BuddyPro AI assistant'

if not api_key:
    print("MIGRATION_SKIPPED: no API key in state.env")
else:
    import unicodedata
    ascii_name = unicodedata.normalize('NFKD', name).encode('ascii', 'ignore').decode('ascii')
    slug = re.sub(r'[^a-z0-9-]', '', ascii_name.lower().replace(' ', '-')).strip('-')[:40] or 'buddypro-instance'

    store = {
        "schema_version": "0.11.0",
        "default_instance": slug,
        "instances": {
            slug: {
                "slug": slug,
                "name": name,
                "topic": topic,
                "api_key": api_key,
                "test_profile_used": data.get('TEST_PROFILE_USED', 'apitest'),
                "onboarded_at": data.get('ONBOARDED_AT', datetime.datetime.utcnow().strftime('%Y-%m-%dT%H:%M:%SZ')),
                "skill_version_at_onboarding": data.get('SKILL_VERSION_AT_ONBOARDING', 'pre-0.11.0'),
                "privacy_warning_acknowledged": data.get('PRIVACY_WARNING_ACKNOWLEDGED', '0') in ('1','true','True','yes'),
                "onboarded_via": "migrated-from-v0.10.x-state.env",
            }
        }
    }

    import os as _os
    tmp = target.with_suffix('.json.tmp')
    with open(tmp, 'w') as f:
        json.dump(store, f, indent=2, ensure_ascii=False)
    _os.chmod(tmp, 0o600)
    _os.replace(tmp, target)
    print(f"MIGRATION_OK: {slug} → instances.json (default_instance={slug})")
PY
fi
```

After migration, `state.env` is left in place (backward compat for users with shell scripts that source it), but `instances.json` becomes the canonical source.

---

## Adding another instance — `/buddypro-add-instance`

When user has ≥1 instance already and wants to add another:

1. Skip privacy warning (already acknowledged in `instances.json`)
2. Skip 4-line mental model briefing (already known)
3. Run Steps 0 → 1 → 2 → 3 again
4. Step 3's Python helper appends to `instances.json` (does NOT change `default_instance` — first one stays default unless user explicitly says „set new one as default")

After successful add, tell user:

> *„✅ Přidána instance „[NEW NAME]" (slug `[slug]`). Aktuálně máš [N] instancí.*
>
> *Default zůstává „[default-name]". Pokud chceš tento nový jako default: řekni „set [NEW NAME] as default"."*

---

## Listing / switching / removing instances — `/buddypro-list-instances`

When user runs this command, show:

```
You have N BuddyPro instances:

1. ⭐ [default] online-strateg — Online Strateg
   Topic: marketingový kouč pro online podnikatele
   Onboarded: 2026-05-08

2. buddypro-ai — BuddyPro AI
   Topic: assistant for BuddyPro platform
   Onboarded: 2026-05-09

Commands:
- /[slug]                          — invoke that specific instance
- /buddypro-add-instance           — add a new one
- "set [name] as default"          — change default
- "remove instance [name]"         — delete one (asks confirmation)
- "switch to [name]"               — for THIS conversation, use that one
```

Backed by reading `instances.json` and pretty-printing.

---

## Reset modes

🔴 **Reset is per-instance, not all-or-nothing.** When user says:

| User says | Action |
|---|---|
| „reset onboarding" / „forget my BuddyPro" | If 1 instance → full reset (delete instances.json, .onboarded, all per-instance slash commands, .last_version_check). If 2+ instances → ask which one. |
| „remove instance [name]" | Delete that instance from instances.json. Delete its slash command file. If it was default, prompt for new default. Re-inject names into SKILL.md description (now without that name). |
| „reset all instances" / „start completely over" | Full reset. Wipe instances.json, .onboarded, .instance-aliases, all `~/.claude/commands/*.md` files that match `is_ours()` predicate (start with „# Alias for /buddypro-api"). |

```bash
# Full reset (use only when explicitly requested)
SKILL_DIR="$HOME/.claude/skills/buddypro-owner-api"

# Read instance slugs to know which slash commands to remove
if [ -f "$SKILL_DIR/instances.json" ]; then
    python3 - <<'PY'
import json, os
from pathlib import Path
inst_file = Path(os.environ['HOME']) / '.claude/skills/buddypro-owner-api/instances.json'
cmd_dir = Path(os.environ['HOME']) / '.claude/commands'
data = json.loads(inst_file.read_text())
for slug in data.get('instances', {}):
    target = cmd_dir / f"{slug}.md"
    if target.exists() and target.read_text().startswith("# Alias for /buddypro-api"):
        target.unlink()
        print(f"removed: {target}")
PY
fi

rm -f "$SKILL_DIR/instances.json"
rm -f "$SKILL_DIR/.onboarded"
rm -f "$SKILL_DIR/.instance-aliases"  # legacy from v0.10.x — also remove if present
rm -f "$SKILL_DIR/state.env"           # legacy from v0.10.x — also remove if present

# Strip injection markers from local SKILL.md description
python3 - <<'PY'
import os, re
from pathlib import Path
sm = Path(os.environ['HOME']) / '.claude/skills/buddypro-owner-api/SKILL.md'
if sm.exists():
    c = sm.read_text(encoding='utf-8')
    c = re.sub(r' AUTO-INJECTED-INSTANCE-NAMES:.*? :END-AUTO-INJECTED\.', '', c)
    sm.write_text(c, encoding='utf-8')
PY
```

Then re-run full onboarding from privacy warning + STEP 0.

---

## Anti-patterns to avoid

- ❌ **Don't skip Step 0** (test profile switch). Generating the key on the user's real profile contaminates their real Telegram chat history and exposes management commands to agents.
- ❌ **Don't say „Stage 1 of onboarding"** — internal jargon. Say „first, let's switch to a test profile."
- ❌ **Don't save the key BEFORE verifying it.** Step 2 verifies first. If 401, never write the bad key to instances.json.
- ❌ **Don't assume single instance.** Always read `instances.json` and resolve which instance is active for the current task. Never hardcode a slug.
- ❌ **Don't ask „where to save the key" with a menu.** Pick canonical (`instances.json`); user can later opt to also export to `~/.zshrc` if they need shell-script compatibility.
- ❌ **Don't promise a „5-bullet mental model" and then stop.** If you say you'll explain something, explain it. The 4-line briefing in Step 3 IS the explanation — deliver it on first instance, skip it on subsequent.
- ❌ **Don't ask for instance topic separately when the bot's first answer already reveals it.** That's redundant work for the user.
- ❌ **Don't end onboarding with another question.** End with a concrete suggestion they can act on right now.
- ❌ **Don't describe the skill as „sending messages to your bot via Telegram API."** It is HTTP REST API. Misnaming this confuses the user about what they're getting.
- ❌ **Don't lead with „Multi-tenant SaaS" demo prompts** — that pattern has privacy issues. Lead with owner-direct prompts.
- ❌ **Don't use bash `read -p` to ask questions in Claude Code.** No interactive stdin in agent subprocess. Agent should ASK the user (out-of-band, plain message) and use the user's reply.
- ❌ **Don't use bash heredoc with un-escaped user content.** Apostrophes, ampersands, diacritics break shell. Use Python (env-var passing) for any heredoc that contains user input.
- ❌ **Don't overwrite an existing slash command at `~/.claude/commands/[slug].md`** unless our `is_ours()` predicate confirms it's already a BuddyPro alias. Otherwise add `-2`, `-3` suffix.

*Last updated: 2026-05-08 (v0.11.0 — multi-instance support: instances.json schema, per-instance slash commands with collision detection, atomic Python helper for Step 3 (no shell-injection from instance NAME), verify-before-save, migration from v0.10.x state.env, /buddypro-add-instance + /buddypro-list-instances commands, reset modes per-instance)*
