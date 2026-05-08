# BuddyPro Owner API — Force Update

Force an immediate update check + atomic re-install of the BuddyPro Owner API skill, bypassing the 4-hour cache TTL.

Use this when you know there's been a new release and want it NOW (instead of waiting for the next auto-trigger), or when something seems off and you want to ensure you're on the latest version.

## Run this exact bash sequence

The agent runs this AS-IS (no abstraction):

```bash
SKILL_DIR="$HOME/.claude/skills/buddypro-owner-api"

# 1) Honor pin if present — refuse to update, tell user how to unpin
if [ -f "$SKILL_DIR/.pinned_version" ]; then
    echo "VERSION_PINNED: $(cat $SKILL_DIR/.pinned_version) — skipping forced update"
    echo "To unpin: rm $SKILL_DIR/.pinned_version"
    exit 0
fi

# 2) Bust the cache so SKILL.md Step 0 logic re-fetches GitHub VERSION
rm -f "$SKILL_DIR/.last_version_check"

# 3) Run SKILL.md Step 0 logic verbatim (cache-miss path → real fetch)
LOCAL=$(cat "$SKILL_DIR/VERSION" 2>/dev/null | tr -d '[:space:]')
LOCAL=${LOCAL:-0.0.0}
REMOTE=$(curl -fsSL --max-time 5 "https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main/VERSION" 2>/dev/null | tr -d '[:space:]')

if [ -z "$REMOTE" ]; then
    echo "VERSION_CHECK_FAILED: GitHub unreachable. Continuing with local=$LOCAL."
    exit 1
fi

if [ "$LOCAL" = "$REMOTE" ]; then
    echo "✓ Already on latest version: $LOCAL. No update needed."
    date +%s > "$SKILL_DIR/.last_version_check"
    exit 0
fi

echo "Updating: $LOCAL → $REMOTE"

# 4) Run INSTALL.md Step 1 atomic install logic
set -euo pipefail
BASE="https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main"
DEST_CMD="$HOME/.claude/commands"
TMP=$(mktemp -d -t bpoasinstall.XXXXXX)
trap 'rm -rf "$TMP"' EXIT
mkdir -p "$TMP/skill/references" "$TMP/command"

files=(
  "VERSION:VERSION" "CHANGELOG.md:CHANGELOG.md"
  "skill/SKILL.md:skill/SKILL.md"
  "skill/references/getting-started.md:skill/references/getting-started.md"
  "skill/references/api-reference.md:skill/references/api-reference.md"
  "skill/references/api-features-deep-dive.md:skill/references/api-features-deep-dive.md"
  "skill/references/deep-research-architecture.md:skill/references/deep-research-architecture.md"
  "skill/references/deep-research-topologies.md:skill/references/deep-research-topologies.md"
  "skill/references/deep-research-scenarios.md:skill/references/deep-research-scenarios.md"
  "skill/references/deep-research-blueprints.md:skill/references/deep-research-blueprints.md"
  "skill/references/use-cases.md:skill/references/use-cases.md"
  "skill/references/code-recipes.md:skill/references/code-recipes.md"
  "skill/references/multi-tenancy.md:skill/references/multi-tenancy.md"
  "skill/references/management-commands.md:skill/references/management-commands.md"
  "skill/references/instance-management.md:skill/references/instance-management.md"
  "skill/references/troubleshooting.md:skill/references/troubleshooting.md"
  "skill/references/docs-references.md:skill/references/docs-references.md"
  "command/buddypro-api.md:command/buddypro-api.md"
  "command/buddypro-api-update.md:command/buddypro-api-update.md"
)

for entry in "${files[@]}"; do
  curl -fsSL --create-dirs --retry 2 --connect-timeout 10 \
    "$BASE/${entry%%:*}" -o "$TMP/${entry##*:}"
done

mkdir -p "$SKILL_DIR/references" "$DEST_CMD"
cp "$TMP/VERSION" "$SKILL_DIR/VERSION"
cp "$TMP/CHANGELOG.md" "$SKILL_DIR/CHANGELOG.md"
cp "$TMP/skill/SKILL.md" "$SKILL_DIR/SKILL.md"
cp "$TMP/skill/references/"*.md "$SKILL_DIR/references/"
cp "$TMP/command/"*.md "$DEST_CMD/"

# 5) Re-inject instance names (post-install hook from INSTALL.md)
if [ -f "$SKILL_DIR/instances.json" ] && command -v python3 >/dev/null; then
    python3 - <<'PY'
import os, json, re
from pathlib import Path
sd = Path(os.environ['HOME']) / '.claude/skills/buddypro-owner-api'
inst = json.loads((sd / 'instances.json').read_text())
names = sorted({i['name'] for i in inst.get('instances', {}).values()})
sm = sd / 'SKILL.md'
if sm.exists() and names:
    c = sm.read_text(encoding='utf-8')
    c = re.sub(r' AUTO-INJECTED-INSTANCE-NAMES:.*? :END-AUTO-INJECTED\.', '', c)
    m = re.search(r'^description:\s*"([^"]+)"', c, re.MULTILINE)
    if m:
        old = m.group(1)
        injected = f", {', '.join(names)}" if names else ''
        new = old.rstrip(' .') + f". AUTO-INJECTED-INSTANCE-NAMES:{injected} :END-AUTO-INJECTED."
        sm.write_text(c.replace(f'description: "{old}"', f'description: "{new}"', 1), encoding='utf-8')
        print(f"INSTANCE_ALIASES_REAPPLIED={','.join(names)}")
PY
fi

# 6) Refresh cache + report success
date +%s > "$SKILL_DIR/.last_version_check"
echo ""
echo "✅ Auto-updated: $LOCAL → $(cat $SKILL_DIR/VERSION)"

# 7) Surface CHANGELOG.md entry for the new version (best-effort)
if [ -f "$SKILL_DIR/CHANGELOG.md" ]; then
    awk -v target="^## v$(cat $SKILL_DIR/VERSION)" '
        $0 ~ target {found=1}
        found && /^## v/ && $0 !~ target {exit}
        found {print}
    ' "$SKILL_DIR/CHANGELOG.md" | head -20
fi
```

## After running

Tell the user (in their language):

> *„🔔 Aktualizováno: v[OLD] → v[NEW]. Hlavní změny: [3-5 word summary z CHANGELOG output]. Pokračuju s aktuální verzí. Tvoje instances.json a onboarding state zůstávají beze změny."*

Honors `.pinned_version` (refuses to update if pinned). Preserves `instances.json`, `.onboarded`, `state.env` (legacy), `.pinned_version`. Re-injects all instance names into the new SKILL.md description so auto-trigger continues firing on user-specific names.

$ARGUMENTS
