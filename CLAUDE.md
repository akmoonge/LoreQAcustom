# LoreQAcustom — 원작견 통합판 (RisuAI plugin)

One-file RisuAI plugin (`LoreQAcustom.js`, Plugin API 3.0). It merges the original
원작견 (LoreQA v2.13) with a modified CanonScout (원작 사전정보) build, for AU / OC fan
roleplay on top of an existing work. Users RP in English through the GigaTrans
translation plugin (`<GigaTrans>` holds the source text) and read Korean. The UI and
comments are Korean.

## Release rules (auto-update)

- Header lines must stay within the first ~500 bytes:
  `//@name LoreQAcustom` · `//@display-name` · `//@version X.Y.Z` · `//@api 3.0` ·
  `//@update-url https://raw.githubusercontent.com/akmoonge/LoreQAcustom/main/LoreQAcustom.js`
- **Never change `//@name`.** Risu refuses updates whose name differs.
- `//@version` must be numbers and dots only (Risu splits on `.` and casts to Number;
  any letters make that part 0). Bump it on every push to main, or the update never shows.
- raw.githubusercontent caches for ~5 minutes.
- Check syntax before every commit: `node --check LoreQAcustom.js`.
- Never put API keys, proxy URLs or user chat text in the file.

## Storage

`risuai.pluginStorage` is ONE store shared by every plugin, not per plugin.
- Settings / presets / saved lore: `loreqacustom_config`, `loreqacustom_presets`,
  `loreqacustom_saved_lores` via `loreqa_storeGet/Set` (copies old `loreqa_*` once).
  The original 원작견 still uses `loreqa_*`; do not write there.
- Divergence ledger: `canon_scout_major_v1:<scope>` (+ `_backup:` for tidy undo).
- Position store: `canonpos_v1:<scope>` (`cur`, `byPos[key]` with guard/guide/qa, `backCand`).
- `<scope>` = character id + chat id.

## Architecture (search these names)

- `LOREQA_DEFAULTS` — every setting and its default. New settings go here.
- `beforeRequest` → `loreqa_unifiedRequest` → `loreqa_prepareTurn` (ledger read,
  position, guard, guide) → per-mode Q&A `loreqa_runModeQA(mode, …)` (swaps
  `loreqa_cfg` with `modeCfg[mode]` and the mode's API) → `loreqa_injectUnified`.
- `afterRequest` → `loreqa_afterTurnBackground` (3 s later): `loreqa_ledgerCatchupRun`
  (reads unread messages to the end, 2 batches per serial task) and
  `loreqa_posModelFallback` (position judge every `posModelEvery` replies or on date change).
- Modes / header chips: 인물 (char), 세계관 (set), 분기 (branch = divergence ledger),
  전개 (flow = position + 시점 가드 + 서사 가이드). Tabs: 현황판 / 인물모드 / 세계관모드 /
  분기모드 / 전개모드 / 기본·프리셋 / 지침·자료 / API·MCP.
- Editable prompts: `LOREQA_PROMPTS` + `loreqa_prompt(key, vars)`. A user-saved prompt
  overrides the default, so default changes only reach users who press "기본값으로".

### Divergence ledger (분기모드)

- Extraction: `scoutLedgerSyncWork` → `scoutLedgerExtractBatch` (extract + coverage
  audit) → `scoutLedgerExtractRequest` → `scoutLedgerValidate(…, {lenient:true})`.
  Evidence quotes must be substrings of the message (normalized for quotes/whitespace);
  bad events are dropped, not the whole batch; one corrective retry.
- Prompt = `LOREQA_PROMPTS.ledger` (gate A/B/C) + locked JSON part from
  `SCOUT_LEDGER_EXTRACT` (patched in `scoutLedgerExtractRule`: adds `review` lines and
  the `core` flag).
- Gate in plain words: record only what a writer who knows the original but not this
  chat's history would get wrong, that is still a current state, and that involves at
  least one canon character / group / place / event. No OC–OC-only records, no canon
  events with the same outcome, no trips/meals/"was present", no in-progress states.
- Tiers: `core:true` (★) goes to the main model (setting `branchMainTier`: 0 off /
  1 core / 2 all); helpers (Q&A, guard, guide) always get all latest states
  (`loreqa_latestStates`, max 60).
- Tidy: `scoutLedgerTidyWork` merges/drops/re-flags core every `ledgerTidyEvery` new
  records or via "지금 정리"; unmentioned and user-edited records are kept; undo backup.
- Batching: `ledgerBatchTurns` (turns), `ledgerBatchChars` (0 = no cap), `ledgerEvery`
  (auto-read interval). Panel: "이어서 읽기", "처음부터 다시 읽기", "기록 두고 다시 훑기"
  (resets read position; `rescanPrev` lets "읽기 중지" roll it back), "시작 위치" input.
- `scoutLedgerReconcile` rewinds when an earlier message changed; it logs the index.
- Branch PDF toggle exists, but PDF renders text as images, so quote matching suffers.

### Position (전개모드)

- `loreqa_posModelFallback`: web search, 4-line answer (label / `Index:` / Confidence /
  Source). Keys `n:<nums>`, `pre:` (before the original starts), `post:`.
- Backward moves are held once (`st.backCand`) and accepted only if the next judgement
  agrees; "지금 판정" and manual set bypass this.
- Guard regenerates only on position change; guide on position change or ledger count change.

## Testing

No build step. Extract functions with node and stub `risuai` (see past tests), or run
`node --check`. Behaviour depends on live LLM output, so ask the owner for the
"진단 로그" JSON when extraction misbehaves.

## Owner's working preferences

- Give a clear conclusion; no "half right, half wrong" hedging.
- When asked for an idea or an opinion, answer first; build only when asked.
- Check the actual code/source before agreeing or explaining; don't just go along.
- Don't lengthen the 3 s background delay (GigaTrans is slow, users chat fast).
- Wording injected into the main model says "story", not "RP/chat".
- Every setting should be exposed in the UI; never hard-disable a feature silently.
