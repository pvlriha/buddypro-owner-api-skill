# Changelog — buddypro-owner-api skill

## v1.0.0 — initial workshop release

First public release. The skill enables Claude Code to talk to a user's BuddyPro AI instance(s) via the Owner API (HTTPS REST endpoint at `https://api.buddypro.ai/v1/chat/completions`).

**What's included:**

- 5-step onboarding (privacy warning → test profile switch → API key generation → verify → capture instance NAME + TOPIC into `instances.json`)
- Multi-instance store: a user can have N BuddyPro instances; each with own `bapi_` key, name, topic. Managed conversationally (just say *„přidej další instanci"* / *„seznam mých instancí"* / *„přepni na X"* / *„odeber X"* / *„resetuj"*).
- Auto-update at every invocation: silently pulls new versions from GitHub via `https://docs.buddypro.ai/skill`, with 4-hour cache and pin support
- Auto-discovery of existing API keys in `~/.zshenv`, `~/.zshrc`, `~/.bash_profile`, `~/.bashrc`, `~/.profile`, `~/.env`, `./.env`, `../.env`, `../../.env`, git-root `.env`
- Persistent state in `~/.claude/skills/buddypro-owner-api/instances.json` survives skill upgrades
- Local SKILL.md description injection: when the user names their BuddyPro instance during onboarding, the skill auto-fires when they later mention that name in any conversation
- Two slash commands: `/buddypro-api` (primary entry) + `/buddypro-api-update` (force update bypassing cache)
- Reference library covers: API spec, use-case patterns (owner-direct + privacy-aware caveats for SaaS), code recipes (Python/Node/curl), `user` field semantics, management commands with risk-classified confirmation flow, instance/Drive folder identification, troubleshooting, official docs map
- Deep research sub-skill: 3-fork × 6-8 turn × topology+role MASTER PATTERN as default; 7 scenario adaptations; 8 question topology patterns; mandatory anti-clarification directive on every API call
