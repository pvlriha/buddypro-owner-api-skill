# Workshop Guide — for Pavel

Jak demonstrovat BuddyPro Owner API skill majitelům instancí.

## Před workshopem (1× setup)

### 1. Ověř, že distribuční URL funguje

```bash
curl -s https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main/INSTALL.md | head -5
# Should return markdown starting with: "# BuddyPro Owner API — Install Guide"

curl -s https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main/VERSION
# Should return latest version
```

### 2. (Volitelně, ale doporučeno) Setup `docs.buddypro.ai/skill` redirect

Hezčí URL pro klienty než `raw.githubusercontent.com/...`. Setup:
- Docusaurus site má `static/skill.html` nebo redirect rule
- Nebo Cloudflare redirect: `docs.buddypro.ai/skill` → `https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main/INSTALL.md`

Obě URL fungují stejně pro Claude Code (WebFetch markdown).

### 3. Připrav demo BuddyPro instanci

Použij Online Stratéga (nebo jiného Pavla testovacího bota) s **dedikovaným test API klíčem**, který nepoužiješ jinde:

```
V Telegramu na svém botovi:
/untest    (ujisti se, že jsi v owner profilu)
/generateApiKey:workshop-demo
→ Bot pošle bapi_... klíč. ULOŽ HO.
```

Pro workshop chceš mít vlastní klíč, který můžeš ukázat na obrazovce (kromě posledních 4 znaků pro maskování) a klidně po workshopu invalidovat.

## Workshop demo flow (45-60 min)

### Část 1 — Úvod (5 min)

**Ukaž problém:** *„Vaše BuddyPro instance má skvělé know-how, ale je uzamčená v Telegramu. Co když ji potřebujete propojit s vlastním webem? Se Slackem? S agentem, který dělá research za vás?"*

Naznač řešení: **BuddyPro Owner API + Claude Code skill**.

### Část 2 — Live install (5 min)

Otevři Claude Code. Napiš (na obrazovce, viditelné):

```
Install the skill from https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main/INSTALL.md
```

(nebo `https://docs.buddypro.ai/skill` pokud máš redirect setup)

Sleduj, jak Claude Code:
1. Načte URL přes WebFetch
2. Spustí curl bash skript
3. Stáhne 12 souborů
4. Načte SKILL.md do active session
5. Pošle confirmation message v jazyce, kterým jsi mluvil (CZ nebo EN)

**Klíčový moment k zdůraznění:** *„Tohle byl jeden řádek. Teď tenhle Claude Code rozumí mé instanci."*

### Část 3 — Active onboarding (5 min)

Přepni do Claude Code session, kde ještě nemáš nastavený `BUDDYPRO_API_KEY`. Napiš:

```
/buddypro-api
```

Skill detekuje chybějící klíč → spustí Stage 1 onboardingu → vede tě krok po kroku.

Ukaž 4-stage onboarding:
- Stage 1: API key generation
- Stage 2: Validate (test call)
- Stage 3: Capture instance topic
- Stage 4: Mental model briefing

**Co ukazuje:** Skill aktivně vede klienta, není to pasivní reference.

### Část 4 — Use case demos (25-30 min)

**Vyber 4-5 use cases relevantních pro workshop audience.** Doporučeno:

#### Demo A — Single owner question (Pattern A1) — 3 min

```
/buddypro-api zeptej se mojí Buddy, jaké jsou top 3 principy pricingu pro coaching
```

Ukaž rychlost, kvalitu odpovědi. Vysvětli, že tahle 1 otázka je default.

#### Demo B — Multi-tenant SaaS (Pattern B1) — 5 min

Otevři connect Slack / web preview / atd. Vysvětli `user` field koncept:

```
Customer ACME-42 píše: "What did we agree on last week?"
→ Bot pamatuje pro každého zákazníka samostatně
```

Ukaž 2 různé customer profily, demonstruj memory isolation.

#### Demo C — DM communication & sales (Pattern B8) — 5 min

Toto je **killer use case** pro většinu majitelů.

```
"Pošli mi ten DM, který mi přišel na Instagramu, a navrhni odpověď v mém hlase"
```

Ukaž:
- Skill přijme DM jako kontext
- Pošle do BuddyPro
- Dostane on-brand reply
- Owner schválí → odešle

#### Demo D — Team Slack bot (Pattern D1) — 5 min

```
"Uvažuji, že napojím svůj Slack tým na moji instanci, aby mohli strávit 
expertní mozek během práce. Jak to nastavit?"
```

Skill vede přes architekturu: Slack bot → API → user field per team member → každý má svou paměť.

#### Demo E — DEEP RESEARCH (Pattern X3 / sub-skill) — 8-10 min

**TENTO DEMO JE ZÁVĚREČNÝ** — nejimpozantnější.

```
"Použij mou BuddyPro instanci a udělej mi komplexní průvodce, jak dělat 
úspěšné webináře. Markdown report, 5 sekcí, s frameworky, příklady, 
3 doporučeními na konci."
```

Skill detekuje deep research scenario → spustí pipeline:
- Phase 2 broad probe (~8 angles)
- Phase 4 deep dive (4 principů × 5-6 turns)
- Phase 5 perspective probe (3 profile)
- Phase 7 synthesis

**Reálný čas:** ~3-5 minut, ~35 calls, ~$2.

Během běhu vysvětli, co se děje. Po dokončení ukaž **finální dokument** — ne raw conversation.

**Klíčový bod:** *„Toto by mě ručně stálo hodiny v Telegramu. Tady to máte za 4 minuty, polished, ready to publish."*

### Část 5 — Q&A + closing (5-10 min)

Časté otázky které se mohou objevit:

**Q: „Funguje to jen pro Pavlovu instanci?"**
A: Ne. Funguje pro libovolnou BuddyPro instanci. Klíč je `bapi_` token, který si vygenerují u svého bota.

**Q: „Kolik to stojí?"**
A: Jednoduché otázky $0.02-0.05 každá. Deep research $1-5 za session. Žádný subscription poplatek za skill samotný.

**Q: „Jak bezpečné je to s API klíčem?"**
A: Klíč zůstává na klientově stroji v env var. Skill nikdy klíč neukládá ani neloguje. Klient může klíč kdykoliv invalidovat příkazem `/invalidateApiKey:`.

**Q: „Co když chci jiný output formát než markdown?"**
A: Skill umí markdown / executive brief / blog post / structured JSON / Q&A reference. Pro custom formáty stačí říct.

**Q: „Funguje to s mým neangličtinovým botem?"**
A: Ano. Skill je language-adaptive — odpovídá v jazyce, kterým s ním mluvíš. Bot odpovídá v jazyce klienta. Plně CZ/EN/Slovak/atd.

**Q: „Jak nainstalovat na můj počítač?"**
A: Stejně jako jsi to viděl: jeden řádek do Claude Code: *„Install skill from [URL]"*. Plus `curl` musí být dostupný (Mac/Linux/Win10+ default).

**Q: „Co když mi skill udělá něco špatně? Pošle e-mail všem mým zákazníkům?"**
A: Nepošle. Skill má 4-úrovňovou risk klasifikaci. Mass actions jako `/messageAllUsers` vyžadují **mandatory 2-step procedure** — nejdřív test pošle jen tobě, pak až po explicit confirmation reálný broadcast. Nemůžeš to omylem udělat.

## Po workshopu

### Sběr feedback

Pošli klientům 3 otázky:
1. *Která ukázka tě nejvíc zaujala?*
2. *Jaký use case bys chtěl použít první?*
3. *Co ti chybělo / co bys změnil?*

### Update skillu

Pokud klienti zmíní něco, co skill neumí nebo umí špatně:
1. Otevři issue na `pvlriha/buddypro-owner-api-skill` repo
2. Po fixu bumpni VERSION (X.Y+1.0 pro features, X.Y.Z+1 pro fixy)
3. Klienti dostanou auto-notifikaci při příštím použití (skill kontroluje VERSION při každém invoku)
4. Klient řekne „updatuj BuddyPro skill" → re-run install URL

### Šíření

Skill je open-source MIT — sdílej URL volně:
- `https://raw.githubusercontent.com/pvlriha/buddypro-owner-api-skill/main/INSTALL.md`
- (nebo) `https://docs.buddypro.ai/skill` po setup redirect
- (nebo) přímo `https://github.com/pvlriha/buddypro-owner-api-skill` pro audity

## Failure modes during demo — co dělat

**🚨 WebFetch refused / no content**
- Pravděpodobně cached starší verze. Zkus refresh URL: `?v=$(date +%s)` na konci.
- Backup: `git clone` přímo do `~/.claude/skills/`.

**🚨 API call returns 401**
- Test API klíč invalidován. Ukaž rychlou regeneraci v Telegramu (`/generateApiKey:workshop-demo-2`).
- Prevence: vygeneruj 2 záložní klíče před workshopem.

**🚨 Bot odpovídá pomalu / 429 rate limit**
- 30 req/min limit. Pokud demo včera dělalo 30+ calls, počkej minutu.
- Prevence: před workshopem nedělej heavy testy 1 hodinu předem.

**🚨 Deep research trvá dlouho na demo**
- Skip Demo E nebo zkrať na light tier (15 calls / ~$0.75 / ~90s).
- Nebo: pre-spuštěn 30 min před → ukaž rychlý replay s předpřipraveným outputem.

**🚨 Klient nemá API klíč**
- Pošli ho do Telegramu na `/generateApiKey:` — 30 sekund.
- Pokud nemá BuddyPro instanci → demo s tvým klíčem (s vědomím, že jejich akce poletí na Online Stratég).

## Checklist před workshopem

- [ ] GitHub repo public, INSTALL.md fetchable
- [ ] (Volitelně) `docs.buddypro.ai/skill` redirect funguje
- [ ] Test API klíč (`workshop-demo`) vygenerován a otestován
- [ ] Záložní 2 klíče připraveny
- [ ] Stabilní internet
- [ ] Claude Code aktualizovaný na nejnovější verzi
- [ ] Backup: `git clone` URL pro případ, že WebFetch selže
- [ ] Live demo na svém Online Stratégovi otestován v plném rozsahu (Demo A-E) den předem
- [ ] Pre-workshop: 1 hodinu před žádné heavy API testy (rate limit reset)
- [ ] Browser tabs: GitHub repo (pro audit), tvá Telegram konverzace s botem, Slack workspace pro Demo D, atd.

## Dlouhodobá údržba skillu

- **Týdně:** projdi GitHub issues, fix bugs, bump verzi
- **Měsíčně:** review která čísla / metriky se v skillu mění (cost, rate limits, model tiers) a aktualizuj
- **Quartal:** anti-AI-slop pass přes všechny soubory, anti-stale check pro odkazy na docs.buddypro.ai
- **Roční:** major version bump (v1.0.0 produkce, v2.0.0 první breaking change)

## Související soubory v repo

- `INSTALL.md` — distribuční vstupní bod
- `README.md` — public-facing repo description
- `VERSION` — aktuální verze
- `MANIFEST.sha256` — integrity verification
- `manifest.json` — file mappings
- `skill/SKILL.md` — entry point pro Claude Code
- `skill/references/*.md` — 11 reference files (~180 KB)
- `command/buddypro-api.md` — slash command

## Stats

- 16+ verzí (v0.1.0 → v0.7.x), git tags
- ~180 KB content, ~5000 řádků
- 17 use case patterns ve 4 kategoriích
- Deep research jako standalone sub-skill s 7 scénáři
- Source-verified Layer architecture (BuddyPro core + Layer 2)
- Empirically validated (~30 live tests on Pavel Říha AI)

*Last updated: 2026-05-07 (during v0.7.x development)*
