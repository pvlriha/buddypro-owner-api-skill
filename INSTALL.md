# BuddyPro Owner API — Auto-Install for Claude Code

Hello Claude Code! The user wants to install the BuddyPro Owner API skill. Follow these steps exactly.

> **Re-installing?** This script overwrites all skill files. If the user customized any of them locally, back up first: `cp -r ~/.claude/skills/buddypro-owner-api ~/buddypro-skill.backup`.

## Step 1: Download skill files

Run this single bash block. Do NOT modify it. Do NOT skip steps.

```bash
set -e
BASE="https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main"
DEST_SKILL="$HOME/.claude/skills/buddypro-owner-api"
DEST_CMD="$HOME/.claude/commands"

mkdir -p "$DEST_SKILL/references" "$DEST_CMD"

# Skill core
curl -fsSL --create-dirs "$BASE/skill/SKILL.md"                          -o "$DEST_SKILL/SKILL.md"
curl -fsSL --create-dirs "$BASE/VERSION"                                 -o "$DEST_SKILL/VERSION"

# Skill references
curl -fsSL --create-dirs "$BASE/skill/references/api-reference.md"       -o "$DEST_SKILL/references/api-reference.md"
curl -fsSL --create-dirs "$BASE/skill/references/use-cases.md"           -o "$DEST_SKILL/references/use-cases.md"
curl -fsSL --create-dirs "$BASE/skill/references/code-recipes.md"        -o "$DEST_SKILL/references/code-recipes.md"
curl -fsSL --create-dirs "$BASE/skill/references/multi-tenancy.md"       -o "$DEST_SKILL/references/multi-tenancy.md"
curl -fsSL --create-dirs "$BASE/skill/references/management-commands.md" -o "$DEST_SKILL/references/management-commands.md"
curl -fsSL --create-dirs "$BASE/skill/references/troubleshooting.md"     -o "$DEST_SKILL/references/troubleshooting.md"

# Slash command
curl -fsSL --create-dirs "$BASE/command/buddypro-api.md"                 -o "$DEST_CMD/buddypro-api.md"

# Sanity check — detect placeholder/stub references
STUB_COUNT=$(find "$DEST_SKILL/references" -name "*.md" -size -500c 2>/dev/null | wc -l | tr -d ' ')

VERSION=$(cat "$DEST_SKILL/VERSION")
echo "INSTALLED_VERSION=$VERSION"
echo "STUB_REFERENCE_FILES=$STUB_COUNT"
```

## Step 2: Verify installation

```bash
ls "$HOME/.claude/skills/buddypro-owner-api/SKILL.md" \
   "$HOME/.claude/skills/buddypro-owner-api/VERSION" \
   "$HOME/.claude/commands/buddypro-api.md"
```

If any file is missing, STOP and tell user the exact `curl` step that failed. Common causes: no internet, GitHub down, dependency `curl` missing.

## Step 3: Read the skill into context

Read this file: `$HOME/.claude/skills/buddypro-owner-api/SKILL.md` (use the literal `$HOME` expansion or the actual user's home path).

This loads the skill's instructions so you can apply it immediately — no Claude Code restart needed.

## Step 4: Pick the right confirmation message

Read the `INSTALLED_VERSION` and `STUB_REFERENCE_FILES` from Step 1's output.

### Case A — `STUB_REFERENCE_FILES > 0` (alpha release)

The reference files are placeholders. Be honest with the user. Send EXACTLY this message in Czech:

```
✅ BuddyPro Owner API skill nainstalován (v{INSTALLED_VERSION} alpha).

⚠️ Heads-up: tato verze obsahuje hlavní SKILL.md a slash command,
ale {STUB_REFERENCE_FILES} ze 6 reference souborů jsou ještě placeholdery
(plný obsah přijde v další verzi). Základní volání API funguje hned.

Další krok: vygeneruj si API klíč.

1. Otevři svého BuddyPro bota v Telegramu
2. Pošli mu příkaz: /generateApiKey:my-agent
3. Bot ti pošle klíč začínající `bapi_...` (jen jednou — zkopíruj hned!)
4. Ulož si ho jako proměnnou prostředí:
   echo 'export BUDDYPRO_API_KEY="bapi_xxxxxxxxxxxx"' >> ~/.zshrc
   source ~/.zshrc

Pak zkus: /buddypro-api pošli "ahoj" mojí instanci
```

Replace `{INSTALLED_VERSION}` with the actual version (e.g., `0.1.1`) and `{STUB_REFERENCE_FILES}` with the actual count.

### Case B — `STUB_REFERENCE_FILES == 0` (production release)

Reference files are complete. Send EXACTLY this message in Czech:

```
✅ BuddyPro Owner API skill nainstalován (v{INSTALLED_VERSION}).

Další krok: vygeneruj si API klíč.

1. Otevři svého BuddyPro bota v Telegramu
2. Pošli mu příkaz: /generateApiKey:my-agent
3. Bot ti pošle klíč začínající `bapi_...` (jen jednou — zkopíruj hned!)
4. Ulož si ho jako proměnnou prostředí:
   echo 'export BUDDYPRO_API_KEY="bapi_xxxxxxxxxxxx"' >> ~/.zshrc
   source ~/.zshrc

Pak zkus: /buddypro-api pošli "ahoj" mojí instanci
```

> **Note:** Confirmation message is hard-coded Czech because BuddyPro's primary audience is Czech-first. If the user is clearly using English or another language, translate proportionally — keep the structure (✅ icon, version, next steps).

## Uninstall (for reference)

If the user later asks to remove the skill:

```bash
rm -rf "$HOME/.claude/skills/buddypro-owner-api" \
       "$HOME/.claude/commands/buddypro-api.md"
```

## Notes for Claude Code

- **No Claude Code restart needed.** The skill is hot-loadable. After Step 3, you can use it immediately.
- **Idempotent.** If skill is already installed, this re-downloads and overwrites — safe to run multiple times.
- **Update flow.** When user asks to update the skill, fetch this same INSTALL.md again and re-run.
- **Dependency.** Only requires `curl` (default on macOS, Linux, Windows 10+).
- **Security.** All files come from `raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main` over HTTPS. Repo is public; user can audit at https://github.com/pvlriha/buddypro-owner-api-skill.
- **Reproducibility (advanced).** To pin to a specific commit instead of `main`, replace `main` in `BASE=` with a commit SHA or tag. Example: `BASE="https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/v0.1.1"`.
