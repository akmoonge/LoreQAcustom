# LoreQAcustom — 원작견 개조판 (RisuAI plugin)

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
  `//@display-name` is "원작견 개조판 vX.Y.Z": bump the number there too (it is what Risu's plugin list shows).
  Bump `const LOREQA_VER` (line ~8) to the same number: it is the only way to see the installed version (console
  "로드 완료" line, tooltip on the settings window title); Risu's plugin list shows no version.
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
- 원클릭 세팅 tab, first section 질문으로 설정 만들기 (3.9.0, `loreqa_buildQuickSurvey`): work name + 7 five-option questions
  (`LOREQA_SURVEY`: fame, stance, player character, time frame, context, reply length, helper cost) + story language ->
  `loreqa_surveyPlan` (fame tier base from `LOREQA_FAME`, stance, OC rule / persona, AU turns flow mode off, after-ending
  turns the guide off, PDF on for Gemini/Vertex at cost <= 3, position-judge interval by cost) and `loreqa_surveyValues`
  (the calculator with prompt = 40%, input = reply / 4, then cost multipliers on batch / tidy) -> diff list -> saved as a
  preset and applied, with the "대략적인 추천 설정값" note. Mostly guesses; the measured parts are noted in the code.
- 원클릭 세팅 tab: 작품 인지도 (`loreqa_buildQuickFame`, table `LOREQA_FAME`: mode chips and
  branch/flow sub-feature on/off only; `loreqa_applyFame` shows the diff; `fameTier` +
  `loreqa_fameDirty` show "(수정됨)") and 내 사용 환경 (`loreqa_buildQuickEnv`: context /
  fixed-prompt / response / input tokens + language → `loreqa_envCalc` with ratios in
  `LOREQA_ENV_RULE`, user overrides in `envRule` via the "계산 기준 (고급)" fold; no prompt size =
  40% of context assumed; 계산 shows
  current → computed, 적용 writes them). Separate token budgets per use (인물 4k / 세계관 6k /
  위치 10k / 분기 묶음 6k, max 3 turns: bigger batches missed and blurred records); ledger read interval = min(batch, remembered turns / 3, helper window
  + maxGap 2), chosen from a 10-environment simulation. All ratios and tiers are guesses until
  usage data, except two measured in 3.7.6 (harness, two works x 2 runs): ledger batch 1/2/3/4 turns -> recall about
  equal for 1-3 (1 turn costs 2x), 4 adds junk, so the 6000-token / 1-3 turn rule stays; position judge 2/3/5/8 turns ->
  no accuracy gain past 3 (5-8 gave some wild jumps), so posTokens 10000 -> 6000 and the range 3-8 -> 3-5.
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
- Prompt (rewritten in 3.3.0 from the Inuyasha test log): ALL judgement rules live in the editable default
  `LOREQA_PROMPTS.ledger`, in sections: 1 what is a record (A wrong by default / B true now and needed later /
  C about the original; invented people/places/plans never, generic words = invented), 2 kinds (canon change,
  knowledge without feelings, relationship both directions one per canon character, changed event), 3 player
  character slots, 4 keep the ledger true (same entity+dimension+slot, carry over what is still true), 5 how to
  write (names never romanized, lasting-state dimension, after = 1-2 sentences without scene details, invalidates
  "" when the only point is her existence / meeting / a relationship the original lacks, nothing beyond the
  evidence, core list, story's language). `scoutLedgerExtractRule` appends only quote rules and the locked JSON tail of
  1.3's `SCOUT_LEDGER_EXTRACT` (review line, core, slot). The old patch pile (`SCOUT_LEDGER_KEEP_TRUE`) and the OC
  rule that said "never record the OC's facts or who knows them" are gone. Tidy: same structure, `SCOUT_TIDY_LOCKED`
  is ids + slot + JSON only. Prompt examples use [bracketed placeholders], never a real work's or the owner's names.
- Player character slots (`LOREQA_PC_SLOTS`): records on `player_character` with `slot` = party / home / standing
  (one record each) or items / condition / secrets / promises (one per thing). `loreqa_stateKey` dedupes by entity +
  slot (+ dimension for multi slots); `loreqa_slotOf` trusts a dimension that names a slot. The slot survives
  validation, projection, manual add / edit, import / export and tidy merges. Main injection: slot records always go
  in with the branch block (also on tier 1), first, under "## <name>의 지금 상태".
  Slot `events` (3.4.4, one record): the original's events she took part in, one short line each; the model writes only
  new lines and `loreqa_appendPcEvents` appends them to the previous record (dedupe, `pcEventsChars` 4000 drops oldest;
  validator allows 6000 chars for this slot). It covers what she learned at canon events (secrets heard there go here).
  Party = journeys of days, not a one-day escort; home never names companions; a later name updates the same item.
  Inner thoughts in narration / status panels are not evidence of knowledge (prompt rule).
- Tidy (3.4.6–3.4.7): merges an item recorded before and after its name, and an earlier stage a newer record settled
  (suspicion → confirmed): never dropped, the earlier stage goes into `change` (the main model sees change in
  parentheses via `loreqa_divLine`). Drop is only for non-records (e.g. knowledge of her money / fears) and records that
  no longer hold at all; the locked format says so, because a "by" field made the model drop superseded records.
- 기록 점검 (3.6.0, `scoutLedgerCheck`, button "기록 점검", editable prompt `ledgerCheck` + locked `SCOUT_CHECK_LOCKED`): one
  branch-API call gives a verdict for EVERY record (ok / outdated / conflict / unsupported / minor, with a "note" first);
  flags only, never edits. Stored in `ledger.checks.flags[id]` with a sig of entity+dimension+after, so an edited record
  hides its flag. A "list only the problems" format missed the stale record 6/6; per-record verdicts caught it 3/3 (HP: the
  same 3 superseded records every run). "note" stays in the records' language and only "reason" uses the output
  language (3.6.1): writing notes in Korean dropped the catch rate to 2/7; with notes in the record language 5/5. Code-only hint `loreqa_staleTurns`: condition / promises slots with no new
  evidence for `ledgerStaleTurns` (40) turns get ⏳ (party / home were noise: long-stable is normal).
  Fix (3.8.0): flagged rows get a checkbox; "체크한 기록 고치기" runs `scoutLedgerFixChecked` = `scoutLedgerTidyWork` in fix
  mode (`opts.only` = checked ids, `opts.ctx` = their related records via `withId`, records carry `check_flag` + quotes,
  locked `SCOUT_CHECK_FIX`); undo via 정리 되돌리기; `sinceTidy` untouched. Test: INU promise rewritten to "only the 灰刃坊
  duty remains", HP superseded relationship records merged into the newest (history in change), 21 -> 14-17 records.
- Same entity + dimension duplicates never reach the main model (`loreqa_latestStates` keeps the newest); tidy's code
  fallback cuts any same-key group to the newest, and a merge output replaces leftovers with its key. Don't rely on
  tidy for quality: record quality comes from the extraction prompt; deterministic cleanup belongs in code.
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
  `scoutText` = `scoutNoise(scoutTextKeep(msg))`: strips other plugins' `<reasoning>`/`<thinking>` notes and illustration
  markup (`class="am-…"`, `<img>`) before any helper reads it; `scoutCompleted` keeps `keep` (pre-strip text) so evidence
  hashes saved before 3.4.5 still match.
  `ledgerDeferLatest` (default on): `scoutCompleted` stops before the newest reply, so it is read
  next turn, after GigaTrans / status-panel / illustration plugins have rewritten it. Reconcile
  only logs a rewind when a message really changed (a shorter readable range is trimmed quietly).
- Output limit (3.5.3): ledger calls use max(budget, profile maxTokens). An empty body that ended on the output limit
  (reasoning used it all) is returned at once with `diagnostic.emptyByLimit` instead of retried; extraction then doubles
  the budget up to 32768 before splitting the batch.
- PDF (`scoutPdfEncode`, Yumi Provider Manager's method): a text-layer PDF, not images. Type0/CIDFontType2 font with
  no glyph file, one CID per distinct character + ToUnicode CMap, content streams deflated. Gemini reads the text
  layer exactly and bills per page, so `pdfFontPt` (default 1pt, ~1190×841 chars per page) keeps it to a page or two
  (28k chars → 1 page, 2.4 KB). `pdfMediaRes` (low default / medium / high / default) is sent per document part on
  Gemini 3+ and as generationConfig.mediaResolution on older Gemini. The old rasterized JPEG PDF garbled quotes
  (結界→結果) and hit the 15 MB cap; `scoutQuoteFuzzy` (≥ 85% edit-distance match, stores the original span) stays.

### Position (전개모드)

- `loreqa_posModelFallback`: web search, 4-line answer (label / `Index:` / Confidence /
  Source). Keys `n:<nums>`, `pre:` (before the original starts), `post:`.
- Backward moves are held once (`st.backCand`) and accepted only if the next judgement
  agrees; "지금 판정" and manual set bypass this.
- `posPrevAnchor` (3.4.1, default on): the judge gets `previous_point` and moves only on progress markers (a later
  chapter's event done, season / term / time skip, characters who appear later), never on a similar-looking scene.
  Sequential harness test: INU stopped jumping to Vol.10 / Vol.15 lookalikes; HP unchanged (dates).
- 전개 방식 `flowStyle` (3.4.0): 'beats' = 비트 모드 (세계 상태표, below) / 'list' = 목록 모드, the 3.2.16 behaviour
  restored verbatim: per position `loreqa_generateGuard` (prompt `guard`, bucket.secrets) + `loreqa_generateGuide` (prompt
  `guide`, `guideCount` events marked [as canon]/[changed]/[impossible], bucket.guide, regenerated when the ledger count
  changes, skipped on a time-jump turn), injected whole with lead `guideSteer` (follow/canon) or `guideRef`. Only change:
  written in the story language and fed `loreqa_divFor('world')` (3.4.12: all records except her party / home / condition slots; it used to drop every slot, so the table never saw her finished sword and wrote 刀々斎 still forging it). `flowSceneGate` (beats mode, default 0 since 3.3.15):
  0 = all beats with at/present conditions + all secrets + PUBLIC minus the judge's ONSCREEN lines (3.4.11: before, gate 0 ignored ONSCREEN and an on-screen group went in as off-screen), main model decides; 1 = scene judge picks
  (READY / SECRETS / ONSCREEN). A harness A/B (testdata/harness/gen.js) showed no pull-in either way.
- 실험 미리보기 (`loreqa_renderAltPreview`, 3.4.9): the position card builds the other style (beats ↔ list) for the same
  position + ledger and shows it next to the current one; memory only (`loreqa_altPreview`), never saved or injected.
- 보기용 번역 (3.4.10): "한국어로 번역" buttons (`loreqa_trBox`) under the world table, guard / guide lists and the
  preview; the ledger window translates all records (`loreqa_ledgerTranslate`, 40 per call, `[n]` lines) and shows one
  line under each. Editable prompt `translate`, 전개모드 API, memory only (`loreqa_trCache`, `loreqa_ledgerTr`), never injected.
- World table prompt (3.3.1): all content rules (source/search, NOW, BEATS, PUBLIC, HIDDEN) are in the editable
  `world` prompt in numbered sections; the locked `LOREQA_WORLD_FORMAT` is only the four-tag line format.
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
- Stance groups (3.5.0, `loreqa_stanceGroup`): canon (follow / canon) appends editable `worldCanon` to the world prompt
  (beats in order + "/ route:" how the present people come together); change (change / free) appends `worldChange`
  (events adapted to the changes + "(ripple)" lines from ledger records); balance = no add-on. The table stores `sg` and is
  regenerated when the group changes. Canon group injects the first live beat as "다음 원작 이정표" (`t.beatNext`).
  Harness (testdata/harness/stance.js, 3 → 7 turns): no measurable steering difference between versions or stances (at a
  quiet INU point every run reached the next arc by turn 4-7); change group showed slightly more invented / ripple. Test
  stance effects with 7+ turns: 3 turns only finish the current scene.
  3.5.2: change group leaves "as canon" beats out of the injection; ripples first. Quiet-point tests (judge scores how
  closely the original's next events are replayed, 0-3, 4 points x 2 runs x 7 turns): follow 1.5, balance 0.6, free 0.8,
  free + "do not replay the original" 0.9 (not added), flow off 1.0. Only follow differs; balance / change / off are
  within noise (balance does not pull canon in). Owner keeps all stances.
- 화면 밖 진행 (3.7.x, `loreqa_offscreenHours` / `loreqa_offscreenRule`): the scene judge also answers TIME (story hours over
  `turns_since_last_check` turns) and OFFSCREEN_DONE (beats whose people are away from the story's characters). Code adds the
  hours to `st.offClock`; when it reaches the stance threshold (`LOREQA_OFFSCREEN_HOURS`: canon 336, balance 48, change 24,
  free 12; follow = never) the first such beat passes off-screen: it joins `played`, goes to `st.offscreen` (main model gets
  the last 3 for `offscreenTurns` (5) replies as "그사이 화면 밖에서 일어난 일"), and the position moves to its chapter
  ("Ch.N (화면 밖 진행)"). Letting the model apply "only when days passed" itself skipped 12 chapters in 8 turns on minutes
  of story time (3.7.0 test); with the code clock: follow 0, canon 0 (under 2 weeks), balance / free 0-6 by story time.
- 원작 브리핑 language: `briefLang` dropdown (default 한국어, or "output" = 출력 언어), because 출력 언어 is the Q&A injection
  language and users often set it to the story language (the briefing came out in Japanese).
- 원작 브리핑: reads flow mode's persona / author-note toggles (`flowPersona`, `flowAuthorNote`; used to follow the Q&A
  toggle and was usually off). Its last section is [TIME JUMP OPTIONS] (3-4 canon events with "to get involved", 1-2 of the
  player character's own threads), shown only: split off before the audit pass and the main-model reference.
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
