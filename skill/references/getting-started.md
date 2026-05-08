# Getting Started — Onboarding flow (multi-instance aware)

> 🔴 **First, get one fact straight:** This skill talks to BuddyPro instance(s) via **HTTPS REST API** (`POST /v1/chat/completions`). It is **NOT** Telegram bot integration — your bot continues to live on Telegram, but you (and your code) talk to its brain over a normal authenticated HTTP endpoint. Think „OpenAI-compatible API" but for YOUR expert AI.

🔴 **Communicate in the user's language.** All templates here are reference English. Translate naturally to whatever the user is speaking. Keep technical strings (`bapi_`, `BUDDYPRO_API_KEY`, `/generateApiKey`, `/test`, `/buddypro-api`) verbatim.

🔴 **Multi-instance store, conversational management.** A user may have ONE BuddyPro instance or MANY (different brands/topics, each with its own `bapi_` key). Internally the skill stores all instances in `~/.claude/skills/buddypro-owner-api/instances.json`. The user manages this store **in plain language** — they say things like *„přidej další instanci"* / *„add another instance"* / *„přepni na X"* / *„seznam mých instancí"* / *„odeber instanci X"* / *„resetuj"*. The skill recognizes those intents and runs the appropriate flow. **There are NO `/buddypro-add-instance` or `/buddypro-list-instances` slash commands** — keep it simple, conversational.

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
> *(or pick a different test scenario name. If you're adding a SECOND/THIRD instance, use a unique scenario per instance to keep their test conversations separate, e.g. `/test:apitest-2`.)*
> *Bot replies confirming you're now in test mode. Anything we do via API now writes into a sandboxed test profile, not your real chat. Tell me when done."*

Capture the test scenario name (default `apitest`) — we'll save it in instances.json so the user knows which test profile this key is bound to.

If user says „I'm already on a test profile" or „I already use /test daily" — fine, just confirm which scenario name and skip ahead.

### STEP 1 — Generate the API key (Telegram, ~20 seconds)

Once on test profile, tell the user:

> *„Now still in Telegram (still in your test profile), send: `/generateApiKey:my-agent`*
> *(or pick a more descriptive label — e.g., `/generateApiKey:[your-bot-nickname]-api`.)*
> *Bot replies with a key starting `bapi_...`. Copy it now — shown only once. Paste it here."*

Validate format: `^bapi_[A-Za-z0-9_-]+$`, length > 20. Confirm with last 4 chars only (e.g., *„got key ending in `...cd30`"*) — never repeat the full key.

### STEP 2 — Verify the key BEFORE saving (one stateless call, ~20 seconds)

🔴 **Verify first, save second.** Don't write a possibly-invalid key to `instances.json`. Test it with a stateless ping:

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
| 429 | ⏱️ Rate-limit hit. Wait 60s, retry. |
| 5xx | 🔧 Transient server error. Retry 3x with backoff. If still failing, tell user to try in 5 min. |

⏱️ **First call typically takes 15-25 seconds** (cold start) — this is normal, mention it to the user so they don't think it's stuck.

### STEP 3 — Capture instance NAME + TOPIC + add to instances.json (~30 seconds)

🔴 **Ask the user TWO explicit questions** (out-of-band — agent asks user, user replies normally):

> *„Dvě rychlé otázky, ať vím, jak na tu instanci odkazovat:*
>
> *1. **Jak se jmenuje tvoje instance?** (jak jí říkáš v konverzaci — to bude i tvůj auto-trigger; když to jméno potom zmíníš v jakékoli zprávě, skill se sám načte)*
>
> *2. **O čem je?** (jednou větou — koho učí, v čem pomáhá)"*

Pre-fill suggestions from `PROBE_BODY` (Step 2 returned 1-sentence specialty). Most BuddyPro bots open with „I'm [name]…" or „Jsem [name]…" — extract the name as a suggestion. The user just confirms or tweaks; if extraction fails, ask without a pre-fill.

🔴 **Why both fields:** Once captured into `instances.json`, the skill:
1. Stores the entry atomically with all metadata
2. **Re-injects all instance names from instances.json into the LOCAL `SKILL.md` description** (between special markers, so auto-trigger fires when user mentions any of their instance names — purely local; never distributed)
3. Sets this as `default_instance` if it's the first one (otherwise leaves default unchanged)

🔴 **NO per-instance slash command is created.** The user manages instances conversationally — see "Conversational management" section below.

**Atomic write to instances.json + description injection:**

```bash
SKILL_DIR="$HOME/.claude/skills/buddypro-owner-api"
mkdir -p "$SKILL_DIR"

# Pass user-supplied values via env vars (Python reads them safely — no shell-injection risk)
export BP_INSTANCE_NAME='<user's instance name verbatim — agent-substituted, no bash expansion>'
export BP_INSTANCE_TOPIC='<user's topic verbatim>'
export BP_API_KEY='<the bapi_... key>'
export BP_TEST_PROFILE='<test scenario from Step 0, e.g. apitest>'

python3 - <<'PY'
import os, json, re, sys, datetime, unicodedata
from pathlib import Path

SKILL_DIR = Path(os.environ['HOME']) / '.claude/skills/buddypro-owner-api'
INSTANCES_FILE = SKILL_DIR / 'instances.json'
SKILL_MD = SKILL_DIR / 'SKILL.md'

name = os.environ['BP_INSTANCE_NAME'].strip()
topic = os.environ['BP_INSTANCE_TOPIC'].strip()
api_key = os.environ['BP_API_KEY'].strip()
test_profile = os.environ.get('BP_TEST_PROFILE', 'apitest').strip() or 'apitest'

if not name or not api_key:
    print("ERROR: BP_INSTANCE_NAME and BP_API_KEY required", file=sys.stderr)
    sys.exit(1)

# Slug — internal identifier in instances.json (NOT a slash command, just a dict key)
ascii_name = unicodedata.normalize('NFKD', name).encode('ascii', 'ignore').decode('ascii')
slug = re.sub(r'[^a-z0-9-]', '', ascii_name.lower().replace(' ', '-')).strip('-')[:40] or 'buddypro-instance'

# Load existing instances.json or initialize
if INSTANCES_FILE.exists():
    with open(INSTANCES_FILE) as f:
        store = json.load(f)
else:
    store = {"schema_version": "0.11.1", "default_instance": None, "instances": {}}

# If slug already exists, suffix with -2, -3, ...
base_slug = slug
n = 2
while slug in store['instances']:
    slug = f"{base_slug}-{n}"
    n += 1

# Add this instance
now = datetime.datetime.now(datetime.UTC).strftime('%Y-%m-%dT%H:%M:%SZ')
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

# Re-inject ALL instance names into local SKILL.md description (auto-trigger)
if SKILL_MD.exists():
    content = SKILL_MD.read_text(encoding='utf-8')
    m = re.search(r'^description:\s*"([^"]+)"', content, re.MULTILINE)
    if m:
        old_desc = m.group(1)
        # Strip any prior auto-injection block, then re-add
        cleaned = re.sub(r' AUTO-INJECTED-INSTANCE-NAMES:.*? :END-AUTO-INJECTED\.', '', old_desc).rstrip(' .')
        names = sorted({inst['name'] for inst in store['instances'].values()})
        injected = f", {', '.join(names)}" if names else ''
        new_desc = cleaned + f". AUTO-INJECTED-INSTANCE-NAMES:{injected} :END-AUTO-INJECTED."
        content = content.replace(f'description: "{old_desc}"', f'description: "{new_desc}"', 1)
        SKILL_MD.write_text(content, encoding='utf-8')

# Onboarded marker
(SKILL_DIR / '.onboarded').touch()

print(json.dumps({
    "ok": True,
    "slug": slug,
    "default_instance": store['default_instance'],
    "total_instances": len(store['instances']),
}, indent=2))
PY
```

**After successful run**, tell the user (in their language):

> *„✅ Instance „[NAME]" uložena. Aktuálně máš [N] instance(í). Auto-trigger reaguje na jméno „[NAME]" v jakékoli další zprávě — když ho zmíníš, skill se sám načte.*
>
> *Pokud chceš přidat další instanci, řekni „přidej další instanci". Pokud chceš přehled, řekni „seznam mých instancí". Vše konverzačně, žádné speciální příkazy."*

Then deliver the 4-line mental model briefing (only on FIRST instance — skip if `total_instances > 1`):

> *„Quick orientation — 4 things to know about this API:*
>
> *1. **Server holds the conversation.** Send only the current user message; the bot already knows the history (just like in Telegram).*
>
> *2. **`user` field = sub-profile within YOUR account.** Without `user` → writes to your test profile. With `user: "label-x"` → separate sub-profile, isolated memory. ⚠️ **Not** privacy isolation between paying customers — all sub-profiles are still your data.*
>
> *3. **Stateless mode**: `x_buddy_saveToHistory: false` → nothing persists. Good for testing, batch evals.*
>
> *4. **Limits**: 30 requests/minute per key. Cost ≈ $0.05/call. Cold start 15-25s; warm 3-8s. 4 commands stay in Telegram (`/generateApiKey`, `/invalidateApiKey`, `/test`, `/untest`); the rest work via API."*

Then **immediately** show 3 ready-to-paste demo prompts (only on FIRST instance, tailored to topic):

> *„✅ Hotovo. Vyzkoušej:*
>
> *1. „Zeptej se [name]: jaký je nejvíc podceňovaný princip v [topic-area]?"*
>
> *2. „Použij [name] a dej mi 3 konkrétní rámce na [topic-area], každý s příkladem"*
>
> *3. „Spusť deep research s [name] na [konkrétní subtopic] a vytvoř 5-sekční playbook" (kicks in deep-research sub-skill — multi-step, ~$2-3, 8-15 min)*
>
> *Nebo prostě napiš, co potřebuješ — routnu to k tvému botovi."*

That's the END of onboarding for this instance.

---

## Conversational management (no slash commands needed)

The user manages multi-instance state in plain language. The skill recognizes these intents:

| User says (CZ / EN — patterns) | What the skill does |
|---|---|
| „přidej další instanci" / „add another instance" / „mám ještě jeden BuddyPro" / „chci připojit nový bot" | Run STEPS 0→3 again (skip privacy warning + 4-line briefing — already known). Append to `instances.json`. Tell user how many they have now. |
| „seznam mých instancí" / „list my instances" / „kolik mám botů" / „co mám připojeno" | Read `instances.json` → pretty-print: name, topic, slug (internal), test profile, onboarded date, mark default with ⭐. |
| „přepni na [name]" / „switch to [name]" / „použij [name]" | For THIS conversation, set the active instance to the matching one. Don't change `default_instance` in instances.json unless user explicitly says „set as default". |
| „nastav [name] jako default" / „set [name] as default" | Update `instances.json["default_instance"] = matching slug`. Atomic write. Confirm. |
| „odeber instanci [name]" / „remove instance [name]" / „smaž [name]" | Confirm: *„Opravdu smazat instanci „[name]"? (yes / no)"* If yes: delete from instances.json, re-inject remaining names into description. If it was default, prompt for new default. |
| „resetuj BuddyPro" / „reset onboarding" / „začni od nuly" | If 1 instance → full reset (delete instances.json, .onboarded, .last_version_check, strip injection block from description). If 2+ → ask „kterou? nebo všechny?" |

🔴 **Always confirm destructive actions.** „remove" / „reset" require explicit yes — never act on first mention.

🔴 **Active-instance resolution at every invocation:**
1. If user message mentions an instance name (case-insensitive substring match) → use that one
2. Otherwise use `default_instance` from `instances.json`
3. If multiple matches → ask: *„Mám [N] instancí, které jméno vyhovuje: [list]. Kterou použít?"*
4. Tell user briefly which instance is active (e.g., *„Použiju tvou „[name]" instanci."*) so they know

---

## Migration from v0.10.x state.env → instances.json

If the user installed an older version with `state.env` but no `instances.json`, the skill auto-migrates on first invocation:

```bash
SKILL_DIR="$HOME/.claude/skills/buddypro-owner-api"
STATE_FILE="$SKILL_DIR/state.env"
INSTANCES_FILE="$SKILL_DIR/instances.json"

if [ -f "$STATE_FILE" ] && [ ! -f "$INSTANCES_FILE" ]; then
    python3 - <<'PY'
import os, json, re, datetime, unicodedata
from pathlib import Path

SKILL_DIR = Path(os.environ['HOME']) / '.claude/skills/buddypro-owner-api'
state_file = SKILL_DIR / 'state.env'
target = SKILL_DIR / 'instances.json'

# Parse state.env (shell-format KEY="value")
data = {}
for line in state_file.read_text().splitlines():
    line = line.strip()
    if not line or line.startswith('#') or '=' not in line:
        continue
    k, v = line.split('=', 1)
    data[k.strip()] = v.strip().strip('"').strip("'")

api_key = data.get('BUDDYPRO_API_KEY', '').strip()
name = data.get('BUDDYPRO_INSTANCE_NAME', '').strip() or 'My BuddyPro'
topic = data.get('BUDDYPRO_INSTANCE_TOPIC', '').strip() or 'BuddyPro AI assistant'

if not api_key:
    print("MIGRATION_SKIPPED: no API key")
else:
    ascii_name = unicodedata.normalize('NFKD', name).encode('ascii', 'ignore').decode('ascii')
    slug = re.sub(r'[^a-z0-9-]', '', ascii_name.lower().replace(' ', '-')).strip('-')[:40] or 'buddypro-instance'
    store = {
        "schema_version": "0.11.1",
        "default_instance": slug,
        "instances": {
            slug: {
                "slug": slug, "name": name, "topic": topic, "api_key": api_key,
                "test_profile_used": data.get('TEST_PROFILE_USED', 'apitest'),
                "onboarded_at": data.get('ONBOARDED_AT', datetime.datetime.now(datetime.UTC).strftime('%Y-%m-%dT%H:%M:%SZ')),
                "skill_version_at_onboarding": data.get('SKILL_VERSION_AT_ONBOARDING', 'pre-0.11.0'),
                "privacy_warning_acknowledged": data.get('PRIVACY_WARNING_ACKNOWLEDGED', '0') in ('1','true','True','yes'),
                "onboarded_via": "migrated-from-v0.10.x-state.env",
            }
        }
    }
    tmp = target.with_suffix('.json.tmp')
    with open(tmp, 'w') as f:
        json.dump(store, f, indent=2, ensure_ascii=False)
    os.chmod(tmp, 0o600)
    os.replace(tmp, target)
    print(f"MIGRATION_OK: '{slug}' -> instances.json")
PY
fi
```

After migration, `state.env` is left in place for backward compat with shell scripts that source it, but `instances.json` becomes the canonical source.

---

## Reset (full wipe — only when explicitly requested)

```bash
SKILL_DIR="$HOME/.claude/skills/buddypro-owner-api"
rm -f "$SKILL_DIR/instances.json" "$SKILL_DIR/.onboarded" "$SKILL_DIR/.last_version_check"
rm -f "$SKILL_DIR/state.env" "$SKILL_DIR/.instance-aliases"  # legacy from v0.10.x

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

After reset, re-run full onboarding from privacy warning.

---

## Anti-patterns to avoid

- ❌ **Don't skip Step 0** (test profile switch). Generating the key on the user's real profile contaminates their real Telegram chat history and exposes management commands to agents.
- ❌ **Don't say „Stage 1 of onboarding"** — internal jargon. Say „first, let's switch to a test profile."
- ❌ **Don't save the key BEFORE verifying it.** Step 2 verifies first.
- ❌ **Don't assume single instance.** Always read `instances.json` and resolve which instance is active.
- ❌ **Don't create per-instance slash commands.** Multi-instance management is conversational. The user says „přidej instanci" / „seznam" / „přepni" — no `/[slug]` slash commands.
- ❌ **Don't end onboarding with another question.** End with a concrete suggestion they can act on right now.
- ❌ **Don't describe the skill as „sending messages to your bot via Telegram API."** It is HTTP REST API.
- ❌ **Don't lead with „Multi-tenant SaaS" demo prompts.** Privacy issues. Lead with owner-direct prompts.
- ❌ **Don't use bash `read -p`** — no interactive stdin in Claude Code. Agent asks user, user replies normally.
- ❌ **Don't use bash heredoc with un-escaped user content.** Use Python (env-var passing) for any heredoc that contains user input.

*Last updated: 2026-05-08 (v0.11.1 — drasticky zjednodušeno: smazány slash commands `/buddypro-add-instance` + `/buddypro-list-instances`, smazány per-instance `/[slug]` slash commands. Multi-instance management = pure konverzační.)*
