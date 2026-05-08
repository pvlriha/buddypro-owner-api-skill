# BuddyPro — Add Another Instance

User wants to add a NEW BuddyPro instance to their existing setup. They already onboarded at least one instance previously (privacy warning + mental model briefing already acknowledged).

## What this does

1. Load `references/getting-started.md` from the skill
2. Run STEPS 0 → 1 → 2 → 3 of onboarding (skip the privacy warning section + skip the 4-line mental model briefing in Step 3 — those are one-shot per user, already done)
3. Step 3's Python helper APPENDS to existing `~/.claude/skills/buddypro-owner-api/instances.json` (does NOT overwrite, does NOT change `default_instance` unless user explicitly asks)
4. Creates a per-instance slash command `/[slug-of-new-instance]` (with collision detection — uses `-2`, `-3` suffix if needed)
5. Re-injects ALL instance names (old + new) into local `SKILL.md` description so auto-trigger fires on each

## Before starting, sanity-check that instances.json exists

```bash
SKILL_DIR="$HOME/.claude/skills/buddypro-owner-api"
INSTANCES_FILE="$SKILL_DIR/instances.json"

if [ ! -f "$INSTANCES_FILE" ]; then
    echo "ERROR: instances.json doesn't exist yet. Use /buddypro-api for first-time onboarding instead."
    exit 1
fi

CURRENT_COUNT=$(python3 -c "import json; d=json.load(open('$INSTANCES_FILE')); print(len(d.get('instances',{})))" 2>/dev/null || echo 0)
echo "Currently $CURRENT_COUNT instance(s). Adding one more..."
```

If `instances.json` doesn't exist yet, redirect user to `/buddypro-api` (full onboarding from scratch).

## Tell the user

> *„Přidáme novou instanci. Privacy warning a základy už znáš — přeskočíme. Ujistím se ale, že:*
>
> *1. Tvoje nová instance je v test profilu (různé `/test:scenario` pro různé instance je dobrá hygiena, jinak by sis chtěl-bys do test-apitest klíče smíchávaly konverzace z různých botů). Jaký test profile použijeme pro tuhle? (default: `apitest-[guess-from-name]` — ale klidně vyber svůj)*
>
> *2. Přepneš se v Telegramu na ten profil (`/test:[profile-name]`)*
>
> *3. Vygeneruješ klíč (`/generateApiKey:[label]`)*
>
> *4. Pošleš mi klíč zpět*
>
> *Až budeš mít, řekni mi a vrátíme se na test → key → verify → name+topic."*

## Reuse Step 3 logic

When user provides the new key + name + topic, run the SAME Python helper from `getting-started.md` § STEP 3. The helper auto-detects existing instances.json and appends. No special case needed.

After successful add, tell user:

> *„✅ Přidána instance „[NAME]" (slug `[slug]`). Aktuálně máš [N] instancí.*
>
> *Default zůstává „[default-name]". Pokud chceš tuhle novou jako default: řekni „set [NAME] as default".*
>
> *Auto-trigger nyní reaguje i na jméno „[NAME]" v jakékoli další zprávě. Také můžeš použít `/[slug]` přímo."*

$ARGUMENTS
