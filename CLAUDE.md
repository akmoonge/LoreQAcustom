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
- Branch / copied chats get a new chat id, so `loreqa_inheritFromParent` (called once per scope from
  `scoutSnapshot`, setting `inheritBranch`) copies ledger + position + fixed facts from the parent when
  the new scope is empty. Parent = Risu's hidden `{{specialcomment::branchedfrom::<parentId>::…}}`
  message, else a chat whose messages start with this whole chat. Ledger is trimmed by reconcile;
  position gets `rejudge` (re-judged, backward move allowed) when the parent went past the fork.
  `scoutText` returns '' for the marker but keeps its index.
- `<scope>` = character id + chat id.

## Architecture (search these names)

- `LOREQA_DEFAULTS` — every setting and its default. New settings go here.
  Length / count caps are settings too (0 = no limit): read them with `loreqa_lim(key)`,
  `loreqa_capStr(text, key)`, `loreqa_capTail(list, key)` instead of hard-coding a number.
- `beforeRequest` → `loreqa_unifiedRequest` → `loreqa_prepareTurn` (ledger read,
  position, guard, guide) → per-mode Q&A `loreqa_runModeQA(mode, …)` (swaps
  `loreqa_cfg` with `modeCfg[mode]` and the mode's API) → `loreqa_injectUnified`.
- `afterRequest` → `loreqa_afterTurnBackground` (3 s later): `loreqa_ledgerCatchupRun`
  (reads unread messages to the end, 2 batches per serial task) and
  `loreqa_posModelFallback` (position judge every `posModelEvery` replies or on date change).
- Modes / header chips: 인물 (char), 세계관 (set), 분기 (branch = divergence ledger),
  전개 (flow = position + 시점 가드 + 서사 가이드). Tabs: 현황판 / 인물모드 / 세계관모드 /
  분기모드 / 전개모드 / 기본·프리셋 / 지침·자료 / API·MCP.
- 작품 인지도 one-button presets: `LOREQA_FAME` (major / semi / minor / niche) is the single table
  of on/off values (base keys + per-mode `char`/`set` cfg); `loreqa_applyFame` shows the diff and
  applies; `fameTier` + `loreqa_fameDirty` show "(수정됨)". Values are guesses until usage data.
- Settings UI layout in each mode tab: frequently used toggles first, then small topic sections
  (모델 · 검색/첨부, 캐릭터 & 보정, 시간 점프), and numbers / caps last in a collapsed
  `loreqa_foldSection('세부 설정 (숫자 · 상한)')`. New number settings go in that fold.
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
- Tiers: `core:true` (★). Main model gets `loreqa_mainDivergences(t.allDivergences)` (the full
  latest-state list, not the `helperDivMax`-capped `t.divergences`) per `branchMainTier`:
  0 off / 1 core only / 2 all / 3 (default) core first, then the rest newest-first, all within
  `divMainChars` (`loreqa_pickDivergences`; overflow drops the oldest). Helpers (Q&A, guard,
  guide) always get all latest states (`loreqa_latestStates`, `helperDivMax`).
- Hand-written records (`loreqa_manualEvent`): no evidence, `edited`+`manual`, so re-reads and
  tidy keep them. JSON: `loreqa_ledgerExport/Import` (`loreqacustom-ledger-v1`); imported
  records whose evidence hashes don't match this chat become manual records.
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
- JSON: `loreqa_posExport/Import` (`loreqacustom-position-v1`: `cur` + `byPos`); import sets
  `guideDivN` to this chat's count so the imported guide is not regenerated at once.

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
