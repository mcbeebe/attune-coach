# Product Plan: Training Plan Generator — Defect Resolution & Performance Improvement

**Date:** 2026-08-17
**Trigger:** Review of the generated plan `broken-arrow-mike-plan-20260816.pdf` (Oakland Hills half + HYROX Anaheim dual-race season) against the independent v2 rebuild (`Mike-Training-Plan-v2.xlsx`), the findings review (`Mike-Plan-Review-Findings.docx`), and the race-day playbook.
**Scope:** `mcbeebe/Broken-Arrow-Training` — plan-generation engines, onboarding inputs, season splicing, Hyrox generator, QA/test infrastructure.
**Status of code references:** all file:line references verified against `Broken-Arrow-Training@main` (commit `5aa4706` deploy line) on 2026-08-17.

---

## 1. Executive summary

The generated plan failed in three compounding ways, and all three are systemic, not one-off bugs:

1. **It trained the athlete for the wrong race.** The real event is a 13.3 mi trail half with 2,900 ft of gain (~218 ft/mi — "Mountainous" on the app's own iRunFar tiers). The app has a full terrain/descent/vert engine stack (`engines/terrain`, `engines/descent`, `vertPrescription.ts`) — and every bit of it was silently skipped because **the product never collects elevation gain for a user-entered race**. `isClimby` evaluated false on `vertFtPerMi = 0`, so vert targets, downhill/eccentric sessions, and power-hiking all short-circuited. The longest run came out at 8.4 mi / ~88 min for a race that will take 2:15–3:00.

2. **The output contradicts itself.** Every session shows two conflicting durations because the header time and the workout steps are computed by two code paths that never reconcile. Weekly mileage is a top-down target that only easy/long runs consume, so quality sessions are invisible in the totals ("24.2 mi peak week" was really ~38 mi of running). Two different zone systems (%maxHR vs %LTHR) coexist in one plan, headers get renormalized but descriptions don't, and a 7 bpm band (155–162) belongs to no zone at all.

3. **Repetition instead of progression.** The layered Hyrox Monday session is a static string repeated verbatim for 8 weeks. The Hyrox generator keys all content on a 4-value `phase` enum, so weeks within a phase are byte-identical (weeks 14/15), and on short runways the `peak` phase is mathematically unreachable — which is why the race simulation and full-distance station sessions never appeared. No station is ever prescribed at race spec because the station list is hardcoded at half distance.

**There is no post-generation QA of any kind** — no validator checks internal consistency, day-before-race load, duplicate weeks, or race-spec coverage. Several tests actively lock in the defective behavior. The v2 workbook's "QA Checks" tab (18 automated validations) is effectively the spec for the QA gate this product lacks.

The plan below is sequenced in five phases: **P0 correctness hotfixes → P1 plan QA gate → P2 race intelligence → P3 Hyrox engine rebuild → P4 athlete calibration & safety**, plus a cross-cutting season-coherence track. P0+P1 stop us shipping self-contradictory plans; P2 fixes the wrong-race failure class; P3+P4 close the gap to the v2 standard.

---

## 2. Defect → root cause map

Every finding from the review, traced to its mechanism in code.

### F1. Wrong race profile (road plan for a mountain trail race)

| Mechanism | Location |
|---|---|
| No elevation-gain input exists in onboarding; only race name, date, a 9-value distance enum, and a free-text description | `src/components/Onboarding.tsx:752-820`; `OnboardingConfig.elevationGainFt` declared (`src/hooks/useOnboarding.ts:137`) but never set by any UI |
| Vert falls back to regex-scraping the description for a literal "N ft/feet" | `src/utils/raceVert.ts:26-37, 80-84` |
| `vertFtPerMi = 0` → `isClimby = false` → the entire vert prescription short-circuits | `src/engines/planGenerator/generatePlan.ts:659-666`; `vertPrescription.ts:73-74` |
| Downhill/eccentric sessions, long-run vert targets, power-hiking all gated on `isClimby` | `vertPrescription.ts:40-56`; `trailSessions.ts:58-68` |
| 13.3 mi snaps to the `half_marathon` enum = 13.1 mi; terrain is unrepresented in the distance model | `generatePlan.ts:51-61, 409` |
| Method selection infers terrain from the raceType radio only; vert is not an input | `methodSelection.ts:85-92` |
| Course resolution is curated-only (3 Broken Arrow courses); generic synthesis explicitly parked | `src/utils/resolveCourse.ts:5-20, 78-117` |
| Long run sized as 0.35 × weekly volume (peak floored at 25 mi/wk), never against expected race duration; `racePacing` computes ETA bands but only for curated courses and only for display | `weekPlan.ts:313-330, 235-237`; `racePacing/index.ts:108-141`; `App.tsx:614-622` |
| No external race lookup or verification of any kind | (absent — only Open-Meteo/Nominatim/Strava calls exist) |

The engines to fix this **already exist and are tested** — Minetti locomotion costs, GAP, eccentric scoring, repeated-bout protection, race pacing. They are analysis-side only today; the single missing link is a structured vert/terrain input and the wiring from it.

### F2. Internal contradictions and arithmetic errors

| Finding | Mechanism | Location |
|---|---|---|
| 2a. Two durations per session | Workout segments are copied verbatim from method JSON (`"150 min - Long steady run below AeT"` is a static template value); the mileage-scaled duration is written only into the `day.time` string. Nothing rewrites `segment.duration`. The same unscaled segments are pushed to Garmin watches. `DaySchedule.volumeModifier` ("short"/"long") is declared but read nowhere — dead data | `workouts.ts:161-177, 183-208`; `generatePlan.ts:389-403`; `roche_swap.json:105, 169-171`; `pdfExport.ts:352-368, 481`; `garminWorkout.ts:106-108`; `types/training-method.ts:244` |
| 2b. Mileage excludes quality | `week.miles` is a top-down target from `buildWeeklyMileage`, never a sum of days. Only easy runs (`computeEasyRunTime`) and long runs (`computeLongRunTime`) consume it — at the easy-band midpoint, exactly the 9:35/mi the review reverse-engineered. Tempo/fartlek/hills get method-constant durations contributing nothing; cross-training minutes are invented separately | `weekPlan.ts:339-450`; `generatePlan.ts:260-327, 394, 817`; `extraDays.ts:49-55` |
| 2c. "30–90 min" weeks 4–5 | The build-phase 5-day pattern has exactly **one** easy day, so all non-long miles land on Wednesday; when that exceeds the method's 90-min clamp, the range inverts (`hi < lo`) and the code returns the raw JSON fallback `{min:30, max:90}` verbatim | `generatePlan.ts:276-284`; `roche_swap.json:105, 400-408` |
| 2d. Zone contradictions | Two generators, two formulas: Hyrox block hardcodes %maxHR strings (`Z3 = 0.75–0.85 × maxHR` → "150–170"); method engine computes Z3 from %LTHR (→ "144–155"). After splicing, `rezoneZoneString` renormalizes headers but `rezoneDetailString` has no rule for `Z<n> (lo–hi)` inside descriptions — header and body of the same card disagree by construction. The 155–162 gap is the uncovered span between `aerobic_threshold` top (0.88×LTHR) and `lactate_threshold` floor (0.92×LTHR); `computeZones` copies zone bounds independently with no contiguity check, and `getZoneForHR` returns `null` inside the gap | `src/utils/planGenerator.ts:210-213, 574`; `generatePlan.ts:186-228`; `rezone.ts:60-71` vs `:82-98`; `utils/zones.ts:5-15` |

### F3. Broken taper

| Mechanism | Location |
|---|---|
| `remapRaceWeekSchedule` rigidly shifts the authored race week by `raceDow − 7` and drops days pushed off the front: for a Saturday race, Monday's rest is deleted, hill strides land Wednesday, and the "short shakeout" lands **Friday** | `generatePlan.ts:156-164`; authored week `roche_swap.json:469-477` |
| The Friday shakeout is then sized by `computeEasyRunTime` against the taper week's entire non-long mileage → 66–80 min the day before the race. Its `volumeModifier: "short"` and "Shakeout 15-20 min" note are ignored | `generatePlan.ts:260-285` |
| Taper volume is anchored to **peak** (0.70×) while base weeks are anchored to **start** (0.55× peak) — so "Taper" exceeding base weeks is structurally guaranteed, and nothing compares them | `weekPlan.ts:393-395`; `roche_swap.json:458, 467` |

### F4. Hyrox never reaches race spec

| Mechanism | Location |
|---|---|
| Station distances are a hardcoded array at exactly **half** race spec; it is the only source of station volumes in the codebase. Official race spec (1000 m erg, 50 m sleds, 80 m BBJ, 200 m carry, 100 m lunges, 100 wall balls) is not represented anywhere machine-readable | `src/utils/planGenerator.ts:29-46` |
| The "full race distance" wording in the simulation prescription is prose glued next to the same half-distance list | `planGenerator.ts:604` |
| The only progression is a 3-bucket phase constant (wall-ball reps 50/75/100, sim station count 4/8); distances and loads never ramp; sled load is a string | `planGenerator.ts:106-162` |
| The two simulation session types that exist are gated exclusively on `phase === 'peak'` — and on clamped short plans `peakEnd = buildEnd`, so **peak is unreachable**: no simulation, ever, precisely when the Hyrox block is squeezed by a preceding race | `planGenerator.ts:260-262, 289-384, 604, 630` |
| No run→station transition (compromised-running/Roxzone) session type exists in the role vocabulary at all; the closest emit is the build-phase "long run + station finisher" (run-then-lift) | `planGenerator.ts:270-277, 627` |
| Weeks are pure functions of `(role, phase, isRecovery)` — any two weeks in the same phase are byte-identical (weeks 14/15); `deriveRecoveryWeeks` returns `[]` for ≤6-week plans, removing the last source of variation | `planGenerator.ts:99-104, 289-384, 487-636` |
| The layered "Hyrox prep" Monday is one static string with no week/phase/position parameter — 8 identical weeks; the only escalation is 1→2 doses/week at midpoint | `engines/season/layerSecondaryWork.ts:39-51, 88-100` |
| Division (Open/Pro) and loads are never modeled | (absent) |

### F5. Structural: two plans stapled together

| Mechanism | Location |
|---|---|
| The season splicer appends a **second, independently generated plan** after the anchor race; the two generators share no vocabulary layer (method plans: AeT/AnT, time-based; Hyrox: Z1–Z4 %maxHR, mileage-based) | `spliceSeason.ts:57-240, 433-442` |
| Block sizing gave the Hyrox build 17 days, clamped to 4 Monday-anchored weeks — the direct cause of the unreachable peak phase (F4) | `planSeason.ts:223-314` |
| Bridge content is a fixed 7-slot cyclic pattern — non-progressive by construction, station circuit has no volumes | `blockWeeks.ts:144-200` |
| `residualsCarried` (the Issurin decay math) is computed and stored but **never read** to select content — the quantitative residual profile is decorative | `residuals.ts:102-111`; `planSeason.ts:269`; `blockWeeks.ts:151` |

### F6. Citations and evidence hygiene

| Mechanism | Location |
|---|---|
| "(Hickson 1981)" is bound to the intensity-preserves-fitness claim in four places (athlete-facing in two); the correct reference is Hickson et al. 1985. The provenance comment binds the wrong PMID (7219129 = the reduced-frequency study) to the intensity claim | `blockWeeks.ts:139, 180`; `residuals.ts:12-14, 71, 83` |
| `engines/evidence.ts` exists precisely to prevent this (`TieredValue { value, tier, citation, url }`) but the season engine never imports it — citations are substrings inside prose, unreachable by any check | `evidence.ts:6-44`; importers: only `descent/eccentric.ts`, `terrain/minetti.ts`, `cycling/mim.ts` |

### F7. Missing athlete-safety features (v2 findings 3, 5)

| Gap | Current state |
|---|---|
| No benchmark/fitness test is ever scheduled; all zones are algebraic multiples of one self-reported easy pace (9:30/mi × intensity ratios reproduces 8:39–10:31 and 6:50–7:12 exactly); AnT bpm = %LTHR × (0.88 × estimated maxHR) — never measured. `feasibility.ts` prints a suggestion to time-trial but no code path creates the session. The `rpe_only` strategy on the VO2 zone is ignored, which is how 10-second hill strides got a meaningless per-mile pace band | `paceTargets.ts:44, 137-166, 194-210, 256-268`; `vdot.ts:104-140`; `feasibility.ts:122-130`; `roche_swap.json:74-85, 230` |
| Injury **area** ("knee") reaches prose only — greeting text and the LLM prompt. It selects no prehab exercises, no contraindications, no descent-load caution. Injury *status* correctly caps days/ramp | `generatePlan.ts:470-498`; `injuryRamp.ts:97-116`; `extraDays.ts` (fixed routine, phase/gym/menopause-aware only) |
| No plan-level QA: no `validatePlan`/lint anywhere in src, api, scripts, or tools. Feasibility checks pace/volume/calendar only — nothing on terrain, duration adequacy, day-before-race load, duplicate weeks, or mileage reconciliation | `feasibility.ts:108-222` |
| Tests lock in defects: `planAssert.test.ts:30` enshrines "a flat road plan prescribes zero vert" as the baseline guard; `layerSecondaryWork.test.ts:55-58` asserts dose counts only, not content variation; every R1 vert test hand-injects the `elevationGainFt` field no UI collects; the golden-persona harness has no climby and no dual-race persona | `src/__tests__/…`; `docs/onboarding-ground-truth.json` |
| PDF footer "Every workout carries full details and coaching in the app" is a hardcoded literal, not a verified claim | `pdfExport.ts:498` |

---

## 3. Root-cause themes

Five patterns explain nearly everything above. Fixes should target the pattern, not just the instance.

**T1 — Critical inputs are not collected, so capable engines idle.** The terrain/descent/vert stack is built, evidence-tiered, and tested — and gated on a number (`elevationGainFt`) that no UI collects. The same shape recurs: `volumeModifier` authored in method JSON but never read; `residualsCarried` computed but never consumed; `racePacing` ETA computed but never fed to long-run sizing; `injuryArea` collected but never acted on. *The wiring between subsystems is the weakest layer of the product.*

**T2 — No single source of truth per quantity.** Session duration exists twice (header string vs segments), weekly volume exists twice (top-down target vs implied day sum), zones exist twice (%maxHR vs %LTHR), and renormalization (`rezone`) patches only one surface. Every contradiction the review found is two representations disagreeing.

**T3 — Content is keyed on coarse categoricals, not continuous progression.** `phase` ∈ {base,build,peak,taper} (identical weeks within a phase), a no-parameter static template (8 identical Mondays), `i % 7` (cyclic bridge). Progressive overload is impossible under this shape. Boundary math on the categorical (`peakEnd = buildEnd`) silently deletes entire capability classes (simulations).

**T4 — Generation has no output QA.** Plans go straight from generator to UI/PDF/watch. The v2 workbook demonstrated that ~18 cheap, mechanical checks catch every headline defect (dual durations, day-before-race load, duplicate weeks, race-spec coverage, mileage reconciliation).

**T5 — Evidence and safety are prose, not data.** Citations are substrings; injury area is a display string; the "coaching" footer is a literal. Anything that lives only in prose drifts.

---

## 4. The plan

### Phase 0 — Correctness hotfixes (ship first; small, independent PRs)

Goal: the generator never again emits a self-contradictory document. No new features.

| # | Item | Fix sketch | Acceptance |
|---|---|---|---|
| 0.1 | **Single duration per session.** Scale `PlannedWorkout` segments to the computed session time; make `day.time` derived from segments (one source of truth), not parallel to them. Honor `volumeModifier`. Garmin export inherits the fix for free | `workouts.ts` gains a scaling pass parameterized by the resolved session minutes; `buildPlannedDay` derives the header from the scaled segments | v2 QA #8: no session carries a step total ≠ header time (tolerance ±10%); property test across all methods × phases × day counts |
| 0.2 | **Weekly totals = sum of days.** Compute `week.miles` (and minutes) bottom-up from every session including quality and long; keep the top-down number as the *planning target* internally, never as the displayed total. Display both run-miles and total-hours | New `summarizeWeek(days)`; `generatePlan.ts:817` uses it | Stated weekly mileage reconciles with day sum to ±0.2 mi on every generated plan; quality sessions included |
| 0.3 | **Kill the 30–90 fallback.** When the computed easy-run window exceeds the method clamp, redistribute miles across available days (or widen the clamp with an advisory) instead of returning the raw JSON range; never emit a range wider than 1.5× | `generatePlan.ts:276-284` | v2 QA-style check: no session's duration range exceeds 1.5× ratio; weeks 4/5 repro case reconciles |
| 0.4 | **Race-week remap rewrite.** Replace the rigid shift with placement by *days-before-race* semantics (rest D-1, shakeout ≤25 min D-1 or D-2, sharpener ≥3 days out, never drop a rest day). Size the shakeout from its own note/modifier, not residual weekly miles | `generatePlan.ts:156-164` + shakeout sizing exemption | v2 QA #10: run volume on D-1 ≤ 25 min for every race day/weekday combination (property test over all 7 race weekdays) |
| 0.5 | **Taper sanity.** Taper percentages anchored so taper week 1 < last build week and < peak; explicit invariant test | `weekPlan.ts:393-395` + method JSON review | Taper weeks monotonically decrease and never exceed any build week |
| 0.6 | **One zone system per plan.** Zones become plan-level structured data; both generators and all detail strings render from it (extend `rezoneDetailString` to `Z<n> (lo–hi)` as an interim patch). Enforce contiguity in `computeZones` — fill or fail on gaps/overlaps | `generatePlan.ts:186-228`; `rezone.ts:82-98`; `utils/planGenerator.ts:210-213` | No bpm value between Z1 floor and Z5 ceiling is unclassifiable; header and detail zone strings agree on every card |
| 0.7 | **Citation fixes.** Hickson 1981→1985 in the four sites; route season-engine claims through `evidence.ts` `TieredValue` so citations are data | `blockWeeks.ts`, `residuals.ts` | Grep-level test: no athlete-facing citation string exists outside an `evidence.ts` construct |

### Phase 1 — Plan QA gate (the structural guarantee)

Goal: a generated plan cannot reach the athlete, the PDF, or the watch without passing a validator. The v2 workbook's QA tab is the seed spec.

- **1.1 `validatePlan(plan, race, config)` engine** — pure, fast, runs at generation time and after every plan edit. Initial rule set (from v2 QA #1–18 + this review):
  - calendar: contiguous dates, exactly 7 days/week, race days on race dates, ≥1 rest day/week;
  - consistency: header/segment duration agreement (0.1), weekly totals reconcile (0.2), zone contiguity (0.6), no duration range >1.5×;
  - load: D-1 run ≤25 min, taper < build, ramp caps respected, no both-time-and-vert spike >35%;
  - progression: no two non-recovery weeks with identical session content (hash the week's workouts); layered sessions must vary across ≥3 dimensions over their run;
  - specificity: if race vert density >100 ft/mi → plan must contain vert targets and descent sessions; peak long run ≥ configurable fraction of predicted finish time; Hyrox plans must schedule ≥1 full simulation ≥10 days out and hit all-8-stations-at-spec in ≥1 week (post-P3);
  - safety: benchmark test scheduled when zones are estimate-only (post-P4); prehab present when injury history set (post-P4).
- **1.2 Severity model:** `error` blocks generation (regenerate with adjusted params or surface a fix-it), `warn` becomes a plan advisory (existing advisory pipeline). Never silently ship an `error`.
- **1.3 CI harness:** run the validator over the golden-persona matrix; **add the two missing personas** — a climby trail persona (Oakland-Hills-shaped: 13.3 mi, 2,900 ft, knee history) and the dual-race season persona already encoded in `fieldSeasonReanchor.integration.test.ts`. Every defect in this review becomes a red test first.
- **1.4 Fix the tests that pin defects:** replace the `planAssert` "flat road = zero vert" baseline guard with a terrain-conditional assertion; extend `layerSecondaryWork` tests to assert content progression; make R1 tests exercise the real input path once P2 lands.

### Phase 2 — Race intelligence (fixes the wrong-race failure class)

Goal: the plan is built for the race that actually exists.

- **2.1 Structured race inputs (the one-line root cause):** add elevation gain (ft/m), surface (road/trail/mixed), technicality (1–5), and true distance (numeric, replacing enum-snap; keep the enum as a quick-pick) to onboarding *and* to SeasonPanel race entry. Wire to `elevationGainFt` — the engine gate already works, as the hand-injecting tests prove.
- **2.2 Race enrichment assist:** on race-name entry, offer an assisted lookup (LLM-backed via the existing `api/coach` infrastructure or a race-DB integration) that proposes distance/vert/terrain/logistics (cupless, bib pickup) **for user confirmation — never silent**. The v2 review demonstrated the value: name mismatch ("Half Marathon" vs "Trail Run") and cupless logistics were both discoverable from the organizer's page. Confidence-tagged, editable, and stored on `RaceInfo` (the fields already exist: `course`, `gear`, `cutoff`, `landmarks`).
- **2.3 GPX import for user races:** revive the parked generic-course synthesis (`resolveCourse.ts` TODO) from an uploaded GPX — unlocks `racePacing`, course cards, elevation profile, and `workoutCourseMatch` ("your training matches X% of the climb") for any race.
- **2.4 Finish-time-aware long-run sizing:** compute predicted finish (racePacing for course/GPX races; VDOT + vert-density heuristic otherwise — `computeRaceProjection` already has an elevation penalty) and add a duration-adequacy term to `longRunMilesFor`: peak long effort targets a % of predicted duration (trail default ~60–70%, capped by injury policy), with grade-adjusted time display.
- **2.5 Terrain-aware method selection & display:** feed vert density into `inferTerrain`/method scoring; when terrain governs, demote pace to reference (effort/HR primary) exactly as v2 does — "on this course, grade sets the pace."
- **2.6 Logistics surfacing:** cupless/gear/bib-pickup fields render into race week (gear-day checklist, D-1 pickup on the calendar) and the PDF.

### Phase 3 — Hyrox engine rebuild

Goal: a Hyrox athlete arrives having done every station at spec and at least one full simulation.

- **3.1 Race-spec as data:** an official spec table (8 stations × distance × division loads, Open/Pro/Doubles/relay) with an evidence-tier citation to the rulebook; division selected in onboarding. All prescriptions render from it — no hardcoded half-distance strings.
- **3.2 Continuous progression model:** station volume/load as a function of week index and runway (e.g. ramp 50%→110% of spec across the block, deload-aware), replacing the phase-bucket constants. Reuse the `generalFitness` progression machinery (`cardioFactor`, A/B alternation) that already exists in-repo.
- **3.3 Guaranteed key sessions under any runway:** schedule backwards from race day — full simulation at ~14 days out, half simulation ~21 days out, all-stations-at-spec week before that — then fill forward. This removes the `peak`-unreachable failure: key sessions are placed by date arithmetic, not phase membership. Validator rule 1.1 enforces it.
- **3.4 Compromised-running session type:** add `run→station→run` brick sessions to the role vocabulary (the defining race demand; currently absent), plus grip-endurance accessory work.
- **3.5 Layered-track progression:** `hyroxLayeredDay` takes `(pos, totalEligible, spec)` and emits a progressing prescription (volumes ramp toward spec, exercises rotate); season-block clamping feeds the progression model so short blocks compress the ramp instead of cloning weeks.
- **3.6 Week de-duplication:** thread week index through `getHyroxWorkoutByRole`; validator rule (no identical non-recovery weeks) backstops it.

### Phase 4 — Athlete calibration & safety

Goal: zones are earned, injuries shape the plan, and the athlete can trust the numbers.

- **4.1 Benchmark scheduling:** when the fitness anchor is estimate-grade (easy-pace or age-derived maxHR), schedule a 20-min field test in week 1–2 (v2's Day-2 pattern), tag zones "estimated → calibrate on <date>", and run `recalibration` against the result to re-anchor zones (extend it beyond its current ±3% pace nudge; let it update HR zones).
- **4.2 Honor `rpe_only` zones:** never print a pace band for RPE-anchored work (fixes the 6:50–7:12 hill-stride absurdity); `formatZoneString` prefers the method's declared strategy.
- **4.3 Injury-area-driven prehab:** map `injuryArea` → prehab block (knee: eccentric heel drops, Spanish squats, hip abduction, Copenhagens — v2's exact block) injected into strength/cross days; knee/lower-leg history additionally caps descent-session dose and adds the "cut vert first" advisory. This turns an input we already collect into the best-evidenced injury-prevention lever we currently ignore.
- **4.4 Load-spike guard:** week-over-week time and vert deltas checked jointly (v2's rule: flag only when both spike >35%), surfaced as advisory and validator rule.

### Cross-cutting — Season coherence (runs alongside P2–P4)

- **S.1 Shared prescription vocabulary:** one zone/pace/duration rendering layer consumed by the method generator, the Hyrox generator, and bridge/recovery streams — eliminates the two-dialect plan (F5) at the root rather than patching strings.
- **S.2 Consume residuals:** `bridgeDayStream` reads `residualsCarried` to sequence bridge emphasis quantitatively (the math already exists; it's just unread), and bridge patterns progress rather than cycle mod-7.
- **S.3 Runway-aware splicing:** when the second block is clamped hard (17 days → 4 weeks), borrow from the recover/bridge allocation or surface a feasibility advisory ("your Hyrox build is 2.5 weeks; here's what we prioritized") instead of silently degrading.

---

## 5. Sequencing, effort, and the Witchel check

Recommended order and rough effort (1 eng, focused):

| Phase | Effort | Depends on | Why this order |
|---|---|---|---|
| P0 (0.1–0.7) | ~1.5 wk | — | Every current user sees these bugs in every plan; each is a small, testable PR |
| P1 QA gate | ~1 wk | P0 | Locks P0 in; every later phase adds rules to it; converts this review into regression tests |
| P2 race intelligence | ~2 wk | P1 | Highest athlete-outcome impact (wrong-race class); 2.1 alone (one form field + wiring) delivers most of the value in days |
| P3 Hyrox rebuild | ~2 wk | P1 | Second-biggest outcome gap; self-contained engine |
| P4 calibration & safety | ~1.5 wk | P1 | Depends on QA gate to enforce; 4.3 is small and high-value, can be pulled forward |
| S.1–S.3 | ~1 wk | P0.6 | Vocabulary layer lands naturally while touching both generators |

Witchel 3-rule check (per project convention):

- **Massive market:** P0/P1 affect 100% of generated plans; P2 affects every non-curated race — which is every race except three Broken Arrow events; P3 affects every Hyrox athlete (fast-growing segment). ✔
- **Visceral solve:** "My plan told me to run 42 minutes and 150 minutes on the same day," "I trained for a road race and met a mountain," "I met wall balls for the first time on race day" — all felt, all quoted from the review. ✔
- **Customer language:** the fixes surface in athlete words: one duration, real totals, "built for 2,900 ft of climbing," "you'll have done the full race once before race day," "zones calibrated from your test on Aug 18." ✔

Items that struggled under the filter and are deliberately **cut or deferred**: full race-DB integration (2.2 ships as assisted lookup with confirmation, not a crawler); Pro-division auto-detection; heat/travel planning beyond the existing advisories; any LLM involvement in generation itself (generation stays deterministic — the LLM proposes, the validator disposes, the user approves).

## 6. Success criteria

1. **The Mike scenario regenerates clean:** the dual-race persona (13.3 mi/2,900 ft trail half Oct 24 + Hyrox Dec 5, knee history, 9:30 easy pace) produces a plan that passes all validator rules and matches v2's structural properties — vert-loaded build with descent progression, peak long run ≥140 min, D-1 ≤25 min, all 8 stations at spec by the final build week, one full simulation ≥10 days out, one benchmark test in week 1–2, prehab in every strength/cross day, zero identical non-recovery weeks, weekly totals that reconcile.
2. **Validator green is a release gate:** CI fails on any persona plan with a validator `error`; the persona matrix includes climby, dual-race, short-runway-Hyrox, and all 7 race weekdays.
3. **No orphaned inputs:** every collected field (`elevationGainFt`, `volumeModifier`, `injuryArea`, `residualsCarried`, `rpe_only`) is either consumed by the engine or removed — enforced by a wiring audit test where practical.
4. **Evidence integrity:** all athlete-facing citations flow through `evidence.ts`; the Hickson/Issurin strings match their PMIDs.

## 7. Open questions for the team

1. **Race enrichment source (2.2):** LLM-assisted lookup via existing `api/coach` infra vs. a licensed race database vs. organizer-page fetch-and-parse — cost/accuracy/maintenance trade-off needs a spike.
2. **Validator failure UX:** when `error`-severity rules fire on a user's *edited* plan (not just generated), do we block the edit, warn, or auto-fix? Proposal: warn + one-tap fix, never block edits.
3. **Method JSON debt:** several defects live partly in `roche_swap.json` (single-easy-day build pattern, taper anchored to peak). Do we audit all 9 method files now (adds ~3 days to P0) or let the validator flush them out per-method? Proposal: validator-driven, fix on red.
4. **Hyrox division data:** Open loads are in the rulebook; confirm we also want Doubles/Relay/Pro at launch of P3 or Open-singles-only first.
