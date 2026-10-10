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
  분기모드 / 전개모드 / 원클릭 세팅 / 기본·프리셋 / 지침·자료 / API·MCP.
- 원클릭 세팅 tab: 작품 인지도 (`loreqa_buildQuickFame`, table `LOREQA_FAME`: mode chips and
  branch/flow sub-feature on/off only; `loreqa_applyFame` shows the diff; `fameTier` +
  `loreqa_fameDirty` show "(수정됨)") and 내 사용 환경 (`loreqa_buildQuickEnv`: context /
  fixed-prompt / response / input tokens + language → `loreqa_envCalc` with ratios in
  `LOREQA_ENV_RULE`, user overrides in `envRule` via the "계산 기준 (고급)" fold; no prompt size =
  40% of context assumed; 계산 shows
  current → computed, 적용 writes them). Separate token budgets per use (인물 4k / 세계관 6k /
  위치 10k / 분기 묶음 6k, max 3 turns: bigger batches missed and blurred records); ledger read interval = min(batch, remembered turns / 3, helper window
  + maxGap 2), chosen from a 10-environment simulation. All ratios and tiers are guesses until
  usage data.
- 진행 상황 창: `loreqa_callLLM` is now a thin wrapper that records each helper call (step name
  from `requestOptions.step`, else inferred) and calls `loreqa_callLLMRaw`. Cards are drawn into
  Risu's main page via `risuai.getRootDocument()` (`.loreqa-float`, fixed, pointer-events none),
  like provider-manager's floating window. No streaming: running time, then output / thinking
  tokens and t/s. Settings `floatOn`, `floatPos` (default bottom-right: Yumi Provider Manager uses
  top-right). Cards take the pointer and can be dragged (listeners on
  the persistent `.loreqa-float`, body pointermove only while dragging) → `floatPos` "custom" + `floatXY`.
  Pass `step:` when adding a new helper call.
- Usage stats (`loreqacustom_stats`, no chat text / work title / keys / model names):
  `loreqa_stat(key, n)`, `loreqa_statTime(key, ms)`, `loreqa_statUsage(prefix, usage)`; hooks in
  position judge (reason + changed/same), time-jump, ledger extract/audit/batch/added/rewind, tidy,
  Q&A per mode, inheritance, turns. Exported from 원클릭 세팅 as `loreqacustom-stats-v1` with a
  settings summary (`loreqa_statsSettings`). Add a stat when adding a feature whose default is a guess.
- Every "턴" setting counts turns (user input + reply = 1 turn). `maxLogs` and `posReadMsgs` used
  to count messages; `turnUnit` marks converted configs/presets (`loreqa_toTurnUnit` halves old
  values once), and `loreqa_turnStart(list, n)` finds where the last n turns start.
- Settings window: default `min(1120px, 96vw)` × 86vh; bottom-right grip resizes it (the plugin
  iframe is spread full-screen while resizing, then re-fitted; size in `windowSize`). Text size
  `uiScale` (%, default 115) is CSS `zoom` on the tabs and body only, so the window rect and the
  iframe passthrough stay in pixels. The saved size is not clamped when the hidden plugin frame reports a
  0-wide window at build time (that used to reset it to the minimum).
- Settings UI layout in each mode tab: frequently used toggles first, then small topic sections
  (모델 · 검색/첨부, 캐릭터 & 보정, 시간 점프), and numbers / caps last in a collapsed
  `loreqa_foldSection('세부 설정 (숫자 · 상한)')`. New number settings go in that fold; settings
  shared by several modes go in the 기본·프리셋 tab instead.
- Editable prompts: `LOREQA_PROMPTS` + `loreqa_prompt(key, vars)`. A user-saved prompt
  overrides the default, so default changes only reach users who press "기본값으로".

### Divergence ledger (분기모드)

- Extraction: `scoutLedgerSyncWork` → `scoutLedgerExtractBatch` (extract, + coverage
  audit only when `ledgerAudit` is on; default off, it doubles the requests) → `scoutLedgerExtractRequest` → `scoutLedgerValidate(…, {lenient:true})`.
  Evidence quotes must be substrings of the message (normalized for quotes/whitespace);
  bad events are dropped, not the whole batch; one corrective retry.
- Prompt = `LOREQA_PROMPTS.ledger` (gate A/B/C) + locked JSON part from
  `SCOUT_LEDGER_EXTRACT` (patched in `scoutLedgerExtractRule`: adds `review` lines and
  the `core` flag).
- Gate in plain words: record only what a writer who knows the original but not this
  chat's history would get wrong, that is still a current state, and that involves at
  least one canon character / group / place / event. No OC–OC-only records, no canon
  events with the same outcome, no trips/meals/"was present", no in-progress states.
- `SCOUT_LEDGER_KEEP_TRUE` (in the locked part, so customised gate prompts get it too): a new
  message that ends / reverses a state already in the ledger must update that same entity +
  dimension, overriding the gate (joining a group passed as "side or group" but leaving was
  filtered as "travel", leaving a false record). `SCOUT_TIDY_LOCKED` has the same rule for tidy.
  The locked part also has NOTHING BEYOND THE EVIDENCE: details (which parent a sibling shares,
  titles, dates, places, reasons) must be in the quotes or player_persona; "write in detail" made the
  model invent 異父兄 from "そなたの兄". Each extraction log entry has `attached` (persona /
  author-note chars, cut by attachChars?). `attachChars` (default 0 = no cap; old saved 4000 → 0 once via
  `attachV`) is shared by every mode, so it lives in the 기본·프리셋 tab with a "페르소나 확인" row. Persona comes from `getDatabase(['personas','selectedPersona'])`
  (needs the user's DB permission; Risu's own `personaPrompt` key is not exposed to plugins, chat-bound
  personas may differ); `loreqa_personaLast` keeps why it failed, "페르소나 확인" shows it.
  OC–OC-only facts (e.g. an OC's death with no canon link) stay out of the ledger (gate C);
  that is long-term memory's job, not the divergence ledger's. The locked part also defines gate C: the
  canon element must be what changed or what is affected; a canon character who only tells / warns /
  witnesses / is present does not count (a record about an invented bandit fort passed because Kagome
  told the OC about it). Tidy's locked part drops such records too.
  RELATIONSHIPS (locked): how a canon character feels about / treats the OC is a canon change and its own
  "relationship with <name>" record, never mixed with knowledge records; time together is not recorded but the
  resulting change in feelings is; the extractor compares the ledger's relationship record with new messages and
  updates it when the relationship has moved on (2-turn batches never saw slow arcs: ledger-04 had 21 records,
  all first meetings / joins / the sword, none about feelings, and Inuyasha was still "confronting" her).
- Player character slots (`LOREQA_PC_SLOTS`, locked PLAYER CHARACTER STATE rule): records on `player_character`
  with `slot` = party / home / standing (one record each) or items / condition / secrets / promises (one per thing).
  Only what the story changed vs the persona; exempt from gate C, nothing else about her is. `loreqa_stateKey` dedupes
  by entity + slot (+ dimension for multi slots), so differently worded "同行" records still replace each other.
  The slot survives validation, projection, manual add / edit (select), import / export and tidy merges. Main
  injection: slot records always go in with the branch block (also on tier 1 "★만"), first, under "## <name>의 지금
  상태". Relationship records hold both directions; where she is / what she carries never goes in canon records.
  Knowledge records hold no feelings; in-progress states are updated when finished.
  `loreqa_slotOf` trusts a dimension that names a slot over the slot field (tidy once tagged a "party" record as
  standing, so a stale "alone" and a stale "with the group" both reached the main model); tidy keeps the source slot.
  The review line must report joins / leaves, items gained / used up and promises (the farewell at 188 and the
  finished sword at 197 were missed); used-up items are rewritten, a forged item gets its own record.
  INVALIDATES (locked + default prompt): for OC records write the canon assumption the record overturns ("In the
  original [canon mother] has one child, [canon son]"); only "she doesn't exist / they never met" is banned. The old
  wording ("if no specific fact is broken, use ''") left every relationship record empty. Then every record repeated the same
  premise ("Izayoi's only child is Inuyasha") into the main block: now a premise is stated once, other records give
  only what they overturn beyond it (tidy dedupes too). "after" (locked) = the current state in one or two sentences, no scene
  details (blushing, smiles, smells); an update carries over what is still true from the record it replaces. Same
  entity + dimension duplicates never reach the main model (`loreqa_latestStates` keeps the newest). Tidy (locked)
  must merge them, following the newest where they differ (a merge once kept "still travelling together" after she
  left), and single-record-rewrites scene-heavy "after" / repeated premises; any same-key group the model leaves is
  cut to the newest in code (`scoutLedgerTidyWork`, reason '같은 항목의 더 최근 기록이 대신함'). A merge output also replaces
  leftover originals with its key. Don't rely on tidy for quality: it only touches records it mentions and its
  rewrites can be wrong. Record quality comes from extraction rules; deterministic cleanup belongs in code.
- NAMES (locked, extract + tidy): one spelling/script per person, copied from the ledger or as the story writes it,
  never romanized; dimensions in the record language (the English "relationship with <name>" template produced
  "knowledge of Sayo" next to 「小夜」).
- Prompt examples use [bracketed placeholders], never a real work's names or titles (they were Harry Potter): when the
  user runs that work, the model copies the example into the ledger / labels without evidence.
- Tiers: `core:true` (★). Main model gets `loreqa_mainDivergences(t.allDivergences)` (the full
  latest-state list, not the `helperDivMax`-capped `t.divergences`) per `branchMainTier`:
  0 off / 1 core only / 2 all / 3 (default) core first, then the rest newest-first, all within
  `divMainChars` (`loreqa_pickDivergences`; overflow drops the oldest). Helpers (Q&A, guard,
  guide) always get all latest states (`loreqa_latestStates`, `helperDivMax`).
- Hand-written records (`loreqa_manualEvent`): no evidence, `edited`+`manual`, so re-reads and
  tidy keep them. JSON: `loreqa_ledgerExport/Import` (`loreqacustom-ledger-v1`); imported
  records whose evidence hashes don't match this chat become manual records.
- Tidy: `scoutLedgerTidyWork` merges/drops/re-flags core every `ledgerTidyEvery` (default 20; the
  calculator sets main-block records × `tidyFill` 0.5, 8–60) new
  records or via "지금 정리"; unmentioned and protected records are kept; undo backup. Protection is
  its own flag (`scoutLocked`: `locked`, or a manual record unless `locked:false`), toggled per record
  ("정리에서 보호"); editing a record no longer protects it. "처음부터 다시 읽기" keeps only protected records
  (manual ones are protected by default) plus records whose evidence is all before `startAt`; with a
  start position it is labelled "N번부터 다시 읽기".
- Batching: `ledgerBatchTurns` (turns), `ledgerBatchChars` (0 = no cap), `ledgerEvery`
  (auto-read interval). Panel: "이어서 읽기", "처음부터 다시 읽기", "기록 두고 다시 훑기"
  (resets read position; `rescanPrev` lets "읽기 중지" roll it back), "시작 위치" input.
  "여기부터 읽기" sets `ledger.startAt` and reads from that number: `scoutLedgerSyncWork` never reads before it (redo, rescan
  and rewinds included); "시작 위치 해제" removes it.
- `scoutLedgerReconcile` rewinds when an earlier message changed; it logs the index.
  `scoutMessageHash` hashes `scoutHashNorm(text)` (no whitespace / tags / markdown / width
  differences): GigaTrans rewrites the last reply after we read it (original moves into
  `<GigaTrans>`, trimmed), which used to rewind and re-read one message every turn.
  `scoutHashMatch` still accepts the older un-normalised hashes.
  `ledgerDeferLatest` (default on): `scoutCompleted` stops before the newest reply, so it is read
  next turn, after GigaTrans / status-panel / illustration plugins have rewritten it. Reconcile
  only logs a rewind when a message really changed (a shorter readable range is trimmed quietly).
- Branch PDF toggle exists, but PDF renders text as images, so quote matching suffers.

### Position (전개모드)

- `loreqa_posModelFallback`: web search, 4-line answer (label / `Index:` / Confidence /
  Source). Keys `n:<nums>`, `pre:` (before the original starts), `post:`.
- Backward moves are held once (`st.backCand`) and accepted only if the next judgement
  agrees; "지금 판정" and manual set bypass this.
- 시점 가드 + 서사 가이드 = one 세계 상태표 per position (`loreqa_generateWorld`, prompt `world`,
  web search, reads no chat turns, only position + ledger): `[PUBLIC]` = what ORIGINAL characters /
  factions are doing now (never the story's own characters' whereabouts: those change scene by
  scene), `[HIDDEN]` = one-line secrets, `[BEATS]` = next original events as
  "(Ch.N) event / at: place / present: people / changed: how this story's confirmed changes alter it (or \"as canon\") /
  then: canon outcome" (`changed` restores the old guide's [changed]/[as canon] notes users liked; it stays in the
  injection for every stance, only `then:` is stance-gated) (+ "(broken: …)"; old "/ needs:" still parsed).
  `[NOW]` comes first (last event done | next: first event not done) and the order is NOW, BEATS,
  PUBLIC, HIDDEN, so HIDDEN is written after BEATS and can avoid them. READY needs the beat's
  "present" people in the current scene (off-screen events never READY); "then:" is injected only
  for follow / canon stances. Stored as `byPos[key].world` {raw,pub,hidden,beats,v,n,nb,
  divN}; regenerated on position / ledger-count / `worldCount` / `beatCount` change.
  PUBLIC skips what NOW / BEATS say (only PUBLIC: a global "each fact once" made HIDDEN drop the real secret
  "the party doesn't know Naraku sent Tsubaki" and fill up with weak lines); HIDDEN may be empty, keeps plans set
  in motion that their targets don't know, drops facts that become true after the cut; first beat = NOW's next. HIDDEN drops news ("hasn't heard X won") and
  "known to" lists only on-page knowers, no guessing. The locked format starts with SOURCE (only `mediumName`, from
  `canonMedium`; auto = the medium the position is counted in, so Vol/Ch = manga, never anime-original arcs: Tsubaki's
  anime-only 爆流破 death leaked in) and HOW TO SEARCH (chapter list + per-chapter summaries after current_point first;
  no character / anime episode pages for beats). Each beat starts with its chapter "(Ch.195)" and must come from that
  chapter's summary. No "drop unverified details" rule: it would strip beats to bare bones. The output format lives in the locked `LOREQA_WORLD_FORMAT` (appended even to a customised `world`
  prompt: an old saved prompt produced tables without [BEATS]); the parser accepts [TAG], 【TAG】, TAG:.
- 장면 판단 (`loreqa_judgeScene`, prompt `sceneJudge`, 전개모드 API with its profile reasoning, no search,
  last `sceneTurns` turns) runs when `st.sceneGen` bumps (time-jump detector answers time + SAME/NEW),
  the table changes, or every `sceneTurns` replies if jump detection is off. One call returns PLACE,
  PRESENT, SECRETS (≤ `sceneSecretMax` → 시점 가드), READY / NEAR beats and DONE beats. Stored as
  `byPos[key].scene`; a locked extra line (`LOREQA_SCENE_ONSCREEN`, appended even to a customised prompt)
  asks ONSCREEN = numbered world_state lines whose character / group is in the current scene; those lines
  are left out of the injected "World state (off-screen)" (the story wins). With world on and any PUBLIC
  line, the judge runs on scene change even when there are no beats / few secrets. DONE beats go to `byPos[key].played` (canon-as-written events never reach the
  ledger) and are excluded from later judgments and regeneration. Skipped when nothing to choose.
- 원작 흐름 (`canonStance`: follow / canon / balance / change / free; off = `compGuide` 0) picks the
  stance prompt in the `[Canon Guide]` block; only READY beats are injected (follow also NEAR and broken
  beats). Beats whose conditions don't hold are never given to the main model: it would pull them in.
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
