# BuddyPro — List & Manage Instances

Show all configured BuddyPro instances + offer management commands (set default, switch active for this conversation, remove).

## Run this bash

```bash
SKILL_DIR="$HOME/.claude/skills/buddypro-owner-api"
INSTANCES_FILE="$SKILL_DIR/instances.json"

if [ ! -f "$INSTANCES_FILE" ]; then
    echo "Žádné instance nenakonfigurovány. Spusť /buddypro-api pro první onboarding."
    exit 0
fi

python3 - <<'PY'
import json, os
from pathlib import Path
data = json.load(open(Path(os.environ['HOME']) / '.claude/skills/buddypro-owner-api/instances.json'))
instances = data.get('instances', {})
default = data.get('default_instance')

if not instances:
    print("Žádné instance.")
else:
    print(f"\nMáš {len(instances)} BuddyPro instanci(í):\n")
    for slug, inst in instances.items():
        marker = " ⭐ [DEFAULT]" if slug == default else ""
        print(f"  {marker}")
        print(f"  /{slug}  →  {inst['name']}")
        print(f"     Topic: {inst.get('topic','(no topic)')}")
        print(f"     Onboarded: {inst.get('onboarded_at','(unknown)')}")
        print(f"     Test profile: /test:{inst.get('test_profile_used','apitest')}")
        print()

print("---")
print("Příkazy:")
print(f"  /[slug]                       — vyvolá konkrétní instanci (např. /{default or 'instance-slug'})")
print( "  /buddypro-add-instance        — přidá novou instanci")
print( "  „set [name] as default"       — změní default")
print( "  „switch to [name]"            — pro TUTO konverzaci, použij tu")
print( "  „remove instance [name]"      — smaže instanci (s potvrzením)")
print( "  „reset all instances"         — kompletní wipe (s potvrzením)")
PY
```

## After showing the list

Wait for user instruction. Common follow-ups:

| User says | Action |
|---|---|
| `/[slug]` (any specific slash) | Re-invoke skill with that instance active. |
| „set [name] as default" | Update `instances.json[default_instance] = matching slug`. Atomic write. Confirm. |
| „switch to [name]" | For THIS conversation only — load that instance's API key into env. Don't change `default_instance`. |
| „remove instance [name]" | Confirm: *„Opravdu smazat instanci „[name]"? Slash command `/[slug]` taky zmizí. (yes / no)"* If yes: delete from instances.json, delete its slash command file, re-inject remaining names into description. If it was the default, prompt for new default. |
| „reset all instances" | Confirm: *„Opravdu kompletně smazat VŠE? Tj. všech [N] instancí + všechny slash commands + onboarding marker. Budeš muset onboardovat od nuly. (yes / no)"* If yes: run full reset from `getting-started.md` § Reset modes. |

🔴 **Always confirm destructive actions.** „remove" / „reset" require explicit yes — never act on first mention.

$ARGUMENTS
