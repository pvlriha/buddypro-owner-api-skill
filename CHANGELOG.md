# Changelog — buddypro-owner-api skill

All notable changes documented per release. Format: human-readable, ordered newest first. Same content as the footer of `skill/SKILL.md`, but in a standalone file so update notifications can fetch only the relevant entry without parsing SKILL.md.

## v0.10.0 — 2026-05-08

**Sub-skill split + instance alias auto-trigger + fresh-agent TL;DR + force-update slash command. Major refactoring release.**

- **Sub-skill split.** `deep-research-architecture.md` reduced from 1465 → 779 lines and now serves as the entry point. Three new sub-skill files extracted:
  - `deep-research-topologies.md` (216 lines) — the 8 question topology patterns (KRUH, HLOUBKA, ŠÍŘKA, SPIRÁLA, STROM, ZIGZAG, INVERZE, META) + selection guidance per scenario + combining topologies within a single Type A branch.
  - `deep-research-scenarios.md` (258 lines) — 7 scenario adaptations (universal how-to, person+product, comparative, audience-tiered, content draft, knowledge audit, single-principle exhaustive) + Track A external research integration.
  - `deep-research-blueprints.md` (317 lines) — 8-phase pipeline, coordination & resource management, knowledge graph for synthesis, full Python implementation skeleton, branch-type decision tree, privacy & cost summary.
  - SKILL.md Quick reference table now routes to the right sub-skill based on user need.

- **Instance alias auto-trigger.** Onboarding now asks 2 explicit questions (instance NAME like „Online Strateg" + topic). The user's instance name is:
  - Saved to `state.env` as `BUDDYPRO_INSTANCE_NAME` (separate from `BUDDYPRO_INSTANCE_TOPIC`)
  - Persisted to `.instance-aliases` file (survives skill upgrades)
  - Injected into local `SKILL.md` frontmatter `description` so Claude Code auto-trigger fires when user mentions the custom name (not just „BuddyPro")
  - Wrapped in a per-instance slash command alias (e.g., `/online-strateg`)
  - Re-applied after every `auto-update` via `INSTALL.md` post-install hook (custom description survives skill upgrades)

- **Fresh-agent 30-second TL;DR.** New section at top of `SKILL.md` with 7 critical facts that prevent ~80% of mistakes a brand-new subagent makes (HTTP not Telegram, no history in messages, `user` field semantics, Step 0 auto-update, state.env onboarding, anti-clarification directive, MASTER PATTERN default).

- **`/buddypro-api-update` slash command.** Explicit force-update path for users who know there's a new release and want it now (bypasses 4h cache TTL). Honors `.pinned_version` if set.

- **Auto-promotion path** (existing-key discovery from v0.9.1) now also captures instance NAME via probe + name-extraction heuristic (regex on bot's self-introduction), or asks the user once if extraction fails.

- **All scenarios in `deep-research-scenarios.md`** now explicitly assume anti-clarification directive on every call + MASTER PATTERN as base structural shape (called out at top of file).

## v0.9.3 — 2026-05-08

**MASTER PATTERN promoted as default + anti-clarification propagated + standalone changelog.**

- `deep-research-architecture.md` Scenario 1 (Universal how-to) now leads with **MASTER PATTERN** (3 forks × 6-8 turns × different topology × different role) as the default pipeline. Old 7-phase Type B+A hybrid relegated to fallback for very narrow topics; 6-stage 40-65 call deluxe blueprint relegated to „exhaustive only when explicitly requested".
- Optimal Blueprint section now leads with MASTER PATTERN (18-24 calls / $0.90-1.20 / 3-5 min wall-clock — empirically validated on 2026-05-08 at 18 calls / 6289 words / 1.5-2.9% cross-fork sim) and presents the exhaustive 6-stage variant as the secondary option only.
- Anti-clarification directive code skeleton now visible in `use-cases.md` X3 (multi-step deep research) and A2 (bulk evaluation) — callers see the directive at the point of use, not just in the architecture file.
- `troubleshooting.md` adds gotcha „Bot is asking clarifying questions instead of answering" with full directive + curl example.
- This standalone `CHANGELOG.md` created for future update notifications.

## v0.9.2 — 2026-05-08

**Auto-update at every invocation (Step 0) + anti-clarification directive.**

- Auto-update check moved to TOP of SKILL.md as Step 0 (was buried at the bottom and easily skipped by auto-trigger invocations from description match).
- Changed from passive „mention once" to silent auto-update with 4-hour cache TTL via `.last_version_check` file (avoids GitHub ping every invocation, ~6 fetches/day max even in heavy use).
- Explicit pin support via `.pinned_version` marker for users who don't want auto-updates. To unpin: `rm $SKILL_DIR/.pinned_version`.
- Graceful degradation: if curl fails (network/GitHub down), continue with local version, no spam to user.
- `deep-research-architecture.md` now has MANDATORY anti-clarification directive in CZ + EN, applied via `x_buddy_systemPrompt` mode `add` on EVERY API call (without it, bot defaults to clarifying questions instead of framework answers).
- 3-5 call mini-probes deprecated (minimum 6 calls per stage; smaller stages should fold into larger ones).
- `add` mode preserves bot voice + persona while stripping clarifying behavior.

## v0.9.1 — 2026-05-08

**Onboarding state persistence (`state.env`) + auto-discovery.**

- Introduced `state.env` as the single source of truth (env vars don't survive between Claude Code sessions reliably; they depend on shell launcher, GUI vs terminal, subagent context).
- Self-check now exhaustively scans BOTH global home-level files (`~/.zshenv`, `~/.zshrc`, `~/.bash_profile`, `~/.bashrc`, `~/.profile`, `~/.env`) AND project-local `.env` files (`./.env`, `../.env`, `../../.env`, plus git-root `.env` if in a repo).
- Auto-promotion path: when key is found in any fallback location but `state.env` and `.onboarded` marker are missing, agent auto-creates both silently and skips full onboarding (privacy warning never re-shown — already acknowledged at original onboarding).
- INSTALL.md note clarifies that re-installs preserve `state.env` + `.onboarded` (they're outside the cp source list — verified end-to-end).
- Explicit reset procedure: `rm state.env + .onboarded` then re-run onboarding.

## v0.9.0 — 2026-05-08

**Onboarding overhaul: HTTP-not-Telegram clarity, mandatory `/test` profile, SaaS warning, Drive folder gotcha.**

- Description in frontmatter explicitly says *„NOT Telegram bot integration — this is an HTTP API for the bot's brain"*.
- Mandatory `/test` profile switch as Step 0 of onboarding (before generating the API key — prevents pollution of owner's real Telegram chat history + keeps management commands out of agent reach).
- Prominent SaaS / privacy warning citing official docs verbatim. The `user` field reframed as sub-profile within owner's account (NOT a tenant boundary).
- Drive folder identification gotcha: bot has no visibility into its own Drive folder; identify via service-email-share + SYSTEM PROMPT content match, NEVER ask the bot.
- Value-first onboarding ending with 3 owner-direct demo prompts (no more SaaS demos by default).
- 4-line mental model briefing including latency (15-25s cold start, 3-8s warm), rate limit (30/min), cost (~$0.05/call).
- Smart-default key storage (no A/B/C dialog menu).
- Auto-topic-detection from bot's first answer (no separate topic question).
- Categories B (end-user products) and C (community integrations) marked ⚠️ with privacy caveat. Categories A (owner-direct) and D (internal team) remain ✅ recommended.

## Earlier versions

Earlier versions (0.1.0 — 0.8.6) were pre-release iteration (architecture experiments, empirical validation suite, infrastructure setup). Production starts at v0.9.0.

---

*Maintained alongside `skill/SKILL.md`. When releasing a new version, update both this file and the SKILL.md footer.*
