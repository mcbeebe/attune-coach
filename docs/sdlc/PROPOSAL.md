# SDLC Uplevel — Proposal

**Date:** 2026-08-29 · **Status:** Open — awaiting Mike's review
**Scope:** attune.coach (source: `mcbeebe/Broken-Arrow-Training`, deploy: `mcbeebe/attune-coach`), `mcbeebe/Waypoint`, `mcbeebe/LEGO-Sorter`
**Artifacts:** this proposal → per-repo application PRs (on approval) → 30-day retrospective
**How this was produced:** eight fresh-context subagents read the three repos independently (delivery pipeline, verification, document record, standing instructions), an adversarial panel then attacked this draft, and the surviving text is what you're reading. Every claim below traces to a file path.

---

## 1. The diagnosis

Your three repos are far above typical solo-developer practice: attune.coach has 212 frontend test files with property-invariant "laws," evidence-tiered constant registries, and expert-review packets; Waypoint runs a dated analysis→decision chain with a supersession banner and locked-decision tables; LEGO-Sorter has a 550-test offline suite where even the 60-pieces-per-minute throughput target is an executable acceptance test. The raw material is excellent.

But two failure modes recur in all three repos, and they are exactly the ones the artifact-chain ideas predict:

**1. Rules that live in prose decay; rules that live in a gate hold.** The evidence is unusually clean:

| Rule as written | Where it lives | What actually happened |
|---|---|---|
| "Include the Witchel check in PR descriptions" | attune CLAUDE.md | Silently abandoned — zero mentions in PRs #330–#363; last use 2026-07-07 |
| "Keep the coach prompt and renderer in step" | attune CLAUDE.md | Manual convention; no parity test, though `test_sync_allowlist_parity.py` proves you know the pattern |
| "Ship when tsc/vitest/eslint pass" | Waypoint CLAUDE.md | Self-policed: eslint isn't even in ci.yml; no branch protection |
| "Commit only when pytest passes" | LEGO CLAUDE.md | Self-reported in 32 commit bodies; zero CI; push-to-main auto-deploys to Render |
| Dependency pins after the double outage | attune `test_requirements_pins.py` | **Held.** Machine-checked since PR #329 |
| Sync allowlist client/server parity | attune `test_sync_allowlist_parity.py` | **Held.** Born from a weeks-long silent P0, hasn't regressed |
| Plan-generator invariants P1–P12 | attune `property-invariants.test.ts` | **Held.** Every publish is gated on them |

The pattern is perfect: every prose rule decayed, every machine rule held. This is the advisory-vs-deterministic distinction made empirical in your own repos.

**2. The artifact chain exists, but the loop never closes.** Each repo has a real intent→analysis→build→audit lifecycle — and in each repo it dies the same death:

- **attune:** the doc→PR pipeline worked beautifully through July (audit doc → R0–R4 roadmap → PRs #306–#311 within 24 hours), then the three largest August initiatives (Hyrox P0–P6, PRD-101–110, Adaptive engine 1–9) shipped with *no committed plan doc at all* — their numbering survives only in squash-commit subjects. `PROJECT_PLAN.md` (72KB, April) still prescribes sprint tags and a CHANGELOG that never existed; two design docs still say "do not start coding until approved" for features that shipped in June; the one ADR cites a `specs/` directory that was never committed.
- **Waypoint:** the one archive pass (commit `51b5e57`, March 15) was exemplary — typed subdirectories, a commit message enumerating keep-vs-archive decisions — and never happened again. ~30 binary .docx/.xlsx files invalidated by the August pivot sit live at root with no supersession marker. CLAUDE.md's repository-structure section names two directories that don't exist and twice instructs deleting a nested `.git` that was removed in March, though the file was edited on 2026-08-28. The August pivot's own decision inputs (`WaypointProductProposal.docx`, the 62-requirement backlog, the prototype screens) were never committed — two decision records cite them clause-by-clause and the record is incomplete without them.
- **LEGO:** CLAUDE.md was maintained as-built through M8 (honest "(simulated)" annotations included), then frozen on July 10 while ~60 commits built a different product; README still says "Read CLAUDE.md first — it is the full software specification" and "milestones (M1–M4)". Physically tuned servo angles exist only on the flashed Arduino board — unrecoverable after a re-flash. The build guide wires the feeder to D6 while the firmware and README say D5.

Nothing here is a discipline problem — the July evidence shows the discipline is real. It's an *encoding* problem: the rituals live in your head and in chat scrollback, so they survive exactly as long as your attention does. The uplevel is to move them into files the agent reads and gates the machine enforces.

---

## 2. The six mechanisms

Each mechanism below pairs one idea from the source material (the artifact-chain notes + the Anthropic newsletter practices) with the concrete form it takes in your repos. The drop-in files already exist under `docs/sdlc/templates/`.

### M1 — The artifact chain: one folder per initiative, dated commits, closed loops

*"Every stage commits something the next stage reads, and the chain of commits becomes the audit trail."*

Adopt one convention across all three repos:

```
docs/initiatives/
  README.md                 # the registry: ID · name · status · PR range · artifacts
  017-hyrox-expert-loop/
    intent.md               # written BEFORE analysis (template shipped)
    plan.md                 # the build plan, PR-numbered
    closeout.md             # or annotations in plan.md: "Shipped: PR #NNN (date)"
docs/archive/               # superseded docs move here, stamped, same commit as their successor
```

Three rules make it a chain instead of a graveyard:

1. **No PR series without a committed plan doc.** Even a rough one. This is the rule the August initiatives broke; `pr-a-sync-plan.md` states the reason in its own header: *"Keep this file in the repo so future sessions can recover the plan even when chat context has been compressed."*
2. **The registry assigns IDs from one global sequence per repo.** attune currently has two unrelated R-series, two P-series, and four restart-at-1 PR-numberings; commit subjects are ambiguous to any future session.
3. **Close-out is part of the merge, not a someday task.** When the last PR merges, in that same session: flip the status, append `Shipped: PR #NNN`, archive what got superseded. `onboarding-improvements-plan.md` (5 commits, all-✅ table, cites PR #250) proves you already do this when the doc is in front of you — the rule just makes the doc always be in front of you.

### M2 — Intent before analysis

*Ten minutes, five sections: problem, proposed outcome, affected parties, constraints, open questions.*

Template shipped at `docs/sdlc/templates/intent-template.md`. The test of a good intent doc is that someone could check the final work against it — which is also what makes it the anchor of the chain. Waypoint's `Assumptions-Audit-Aug2026.md` ("24 claims checked → 13 confirmed · 6 partly true · 3 could-not-verify · 2 contradicted") shows what downstream verification looks like when intent is explicit.

### M3 — Standing instructions encoded once, and kept true

*CLAUDE.md and skills are institutional knowledge as versioned files — but a stale standing instruction is worse than none, because the agent believes it.*

All three CLAUDE.md files currently contain falsehoods (attune's has no commands or topology at all; Waypoint's describes a five-months-gone repo layout; LEGO's describes a product that pivoted). Three fixes:

1. **One-time regeneration** of each CLAUDE.md: commands, deploy topology, the conventions you actually practice (attune's narrative-changelog commit style is consistent — codify it; delete the abandoned Conventional-Commits DoD). Fold in the knowledge currently buried in YAML comments: the `run-coach-eval` label, the Vercel 12-function cap, `ATTUNE_PUBLISH_ENABLED`, the scheduled-push CRON_SECRET pairing.
2. **The self-healing rule, written into each CLAUDE.md:** *"Any PR that changes commands, layout, deploy topology, or conventions updates this file in the same PR."* LEGO's own history proves the mechanism works: ROADMAP phase markers were flipped in their shipping PRs (the doc being read got updated); CLAUDE.md (no longer being read) froze.
3. **Portfolio skills**, dropped into `.claude/skills/` in each repo:
   - **`/recap`** (`templates/recap-skill.md`) — refresh stale state first, then report exactly four things: the goal in the requester's words; where things stand *with evidence* (unit tests passing doesn't count as proven — attune's own eval README distinguishes fixture-honesty from live behavior for exactly this reason); blocked-on-person vs blocked-on-technical; next steps each tagged `[Claude]` or `[Mike]`. No new work during a recap.
   - **`/adversary`** (`templates/adversary-skill.md`) — see M6.
   - Plus `.claude/settings.json` permission allowlists for each repo's known-safe loop (the `fewer-permission-prompts` skill generates this from transcripts).

### M4 — Verification is mechanical, or it isn't verification

*"Every figure traces to a cited source" beats "check the numbers."*

The generalization of your own best pattern: **any prose contract you actually rely on gets a machine check.** `test_sync_allowlist_parity.py` (regex-parse both sides, assert set equality) is the template — it was born from a P0 and has held ever since. Apply it to the contracts still held by vibes:

- attune: coach prompt ↔ markdown renderer (the CLAUDE.md "keep in step" rule); `onboarding-ground-truth.json` freshness (regenerate in CI, diff against committed).
- Waypoint: the classifier prompt duplicated verbatim in `ai.ts` and `prompt-regression.mjs`, guarded only by a "must mirror" comment.
- LEGO: promote `tools/segment_parity.py` (the JS-port-vs-cv2-oracle check) from manual script into pytest.

And the quantifiable-target discipline you already have — evidence tiers that "upgrade only with a citable source," 85%/80% eval gates, `ppm >= 60.0` as an assertion — extends naturally: attune's expert-packet verdicts land back as citations; Waypoint's weekly prompt regression gets teeth (below); LEGO's simulated 115.8 ppm gets a committed real-hardware benchmark log.

### M5 — Advisory vs deterministic: classify every control, then move the load-bearing ones

*A checklist is advisory. Anything that must hold needs a gate.*

The portfolio rule: **for each control, ask "what happens if the agent forgets?" If the answer is "production breaks" or "a paying user sees it," it must be a required check, a hook, or a physical impossibility — not a sentence in CLAUDE.md.** Concretely, the deterministic tier per repo:

- **attune:** branch protection requiring the `test` job + `coach-unit` before merge (they exist; nothing requires them); pytest gating the Vercel API deploy (today the API ships with *zero* tests on the deployed revision — the exact hole the #327/#328 outages went through); eslint + `tsc` in CI (lint currently runs nowhere); a post-deploy smoke check that `attune.coach/version.json` and `/api/version` serve the just-pushed SHA (turning the DeployDiagnostics panel from a manual runbook into an automated gate); fix the `workflow_dispatch` footgun that lets a manual dispatch publish any feature branch to production.
- **Waypoint:** put eslint into ci.yml so the self-merge rule's third gate actually exists; a CI check on Edge Functions (five production Deno functions currently deploy with zero tests and no typecheck); make the prompt-regression suite fail loudly when its secrets are unset instead of exiting 0 green (ROADMAP still lists creating the QA account as open ops — today "green" can mean "never ran"); a migration-drift guard so code that assumes migration N fails fast when N isn't applied (commit `e0bdcdd` is the shipped production bug this rule would have caught).
- **LEGO:** a ~15-line GitHub Actions workflow running the 8.7-second offline suite on every push/PR — this single file converts "commit only when pytest passes" from self-reported to enforced *and* gates the Render auto-deploy; pin `requirements.txt` (pulp is already emitting the deprecation warnings that precede the break); align the Dockerfile to Python 3.11 so the deployed interpreter is the tested one.

Everything else — style, formatting, product filters like the Witchel check — stays advisory, and honestly so: either re-scope the Witchel rule to what practice has been since July (planning docs only) or give it a PR-template checkbox so it fires where the description is written. A rule you've stopped following is a lie in your standing instructions; keep it only if you give it a mechanism.

### M6 — Separation of duties: the producer never approves

*The agent that produced the work has no route to approve it — keep that even for a one-person workflow.*

You already run the strongest version of this for training constants: the expert-review packets send your judgment layer to an outside coach whose verdicts land back as citations. Extend the same shape inward:

1. **`/adversary` before human review** (template shipped): before you look at any non-trivial PR, a fresh-context subagent that has *not* seen the producing conversation attacks the diff for correctness and simplicity; the resulting memo — findings, assumptions, design decisions, and the 2–4 spots to focus on — goes in the PR description. You read the memo and the hot spots instead of the whole diff cold. This is the practice from the Claude Code team, and it's the piece your current flow lacks: Waypoint's CLAUDE.md currently instructs the *producing session* to self-merge on green, which is the producer approving its own work.
2. **Named-human gates for consequential boundaries**, held by mechanism, not self-report: anything leaving your desk for the tax-equity-provider-equivalents of each project — App Store submission, the DDS/vendorization packet, payer-facing letters, anything touching paying users' money (`stripe-webhook`), and DB migrations — requires your explicit sign-off recorded in the artifact (Waypoint's ROADMAP Phase 4b already marks this boundary: "provisioning I shouldn't stand up unsupervised"; make that the norm, not the exception).
3. **Branch protection as the mechanical second signer.** Solo, you can't have a second human on every merge — but you can make the machine's approval non-optional. Required checks mean even a self-merged PR carries an independent verdict.
4. **`/recap` at session boundaries** so the human always re-enters with true state, not the agent's narrative of it.

---

## 3. Per-repo plans

Each item is sized for one Claude Code session or less. "Gate" items are M5's deterministic tier; they come first because they protect everything else.

### 3.1 attune.coach (source repo `mcbeebe/Broken-Arrow-Training`)

**Now — gates (≈3 sessions):**
1. CI: add eslint + a Python job (keyless pytest) to `deploy.yml`'s gate; fix the `workflow_dispatch` publish conditional (`&& github.ref == env.PUBLISH_REF` — one line).
2. Turn on branch protection / a ruleset requiring `test` + `coach-unit` (GitHub settings — **[Mike]**, five minutes).
3. Gate the Vercel API deploy on green CI (Vercel "Ignored Build Step" keyed on commit status), and add the post-deploy smoke job asserting `version.json` + `/api/version` serve the pushed SHA.
4. Prompt↔renderer parity test, cloned from `test_sync_allowlist_parity.py`.

**Next — chain (≈2 sessions):**
5. Create `docs/initiatives/` + registry; backfill stub entries for the August initiatives (Hyrox P0–P6 → PRs #296–#305, PRD-101–110 → #312–#319, Adaptive 1–9 → #344–#352) so their record exists while the commits are still fresh; extract the D1–D8 "locked decisions" and the menopause overlay decision into ADRs 0002+.
6. Close stale loops: stamp `PROJECT_PLAN.md` SUPERSEDED and archive it; flip the two "do not start coding until approved" design-doc headers; fix or annotate the dead references (ADR's `specs/` link, the four `BA_*_v3.html` citations, deploy.yml's out-of-repo plan-file citation — rescue that content into `docs/` before the session file is gone).
7. Commit the un-regenerable harnesses: the expert-review exhibit generator and the running-audit reproduction harness.

**Then — standing instructions & rituals:**
8. Regenerate CLAUDE.md (commands, topology, real conventions; resolve the Witchel contradiction one way); replace the Vite-template README; add `.claude/skills/` (recap, adversary) + settings allowlist; schedule the live coach report-card weekly with failure→issue so regressions don't wait for someone to remember the label.
9. Housekeeping that removes standing confusion: rename the repo `attune-coach-app` (Broken Arrow is legacy branding; GitHub redirects old URLs — verify the Vercel connection and `ATTUNE_DEPLOY_TOKEN` cross-repo push after rename), migrate trunk to `main` (deploy.yml is pre-wired for it), and start tagging deploys so "what was live yesterday" is a lookup.

### 3.2 Waypoint

**Now — gates (≈2 sessions):**
1. Add eslint to ci.yml; add a deno-check/test job for the five Edge Functions; make `prompt-regression.yml` fail red when secrets are unset, and finish the QA-account ops item so it has ever actually run — then decide what its 85%/80% gates *block* (at minimum: red = a GitHub issue assigned to you, not a silent Monday).
2. Kill the prompt duplication: one source of truth for the classifier prompt, imported by both `ai.ts` and `prompt-regression.mjs` (or a parity test if the runtimes can't share).
3. Migration-drift guard: a startup/health check that asserts the highest applied migration ≥ the highest the code requires; delete the dead `pages.yml` `dev` trigger.

**Next — chain & record (≈2 sessions):**
4. Second archive pass (the reader found the exact list): superseded PRD/persona binaries → `Archive/`, `SETUP-FOR-CLAUDE-CODE.md` → archive (all steps completed in March), the 300KB session transcript → `Conversation-Logs/` after distilling a 1–2 page Entity-Navigation-Matrix spec; delete the byte-identical KB draft duplicate; commit the missing August pivot inputs (proposal, requirements backlog, prototype screens) so the decision record's citations resolve.
5. Write the two-line supersession rule and the Archive bucket convention into CLAUDE.md + `Archive/README.md`; adopt the four-line decision-record header (Date / Status / Supersedes / Superseded-by) on `Roadmap/*.md`; markdown-first rule for documents of record (binaries become export deliverables, generated on demand).
6. Regenerate CLAUDE.md's repository-structure section + the self-healing rule; real README (what's live, what's being built, where the plan of record is).

**Standing decision to make — [Mike]:** gas-mvp is labeled "Active — Production" and deploys by copy-paste, yet `.clasp.json` has existed since March. Either wire clasp so production deploys are committed pushes, or declare gas-mvp frozen-legacy in CLAUDE.md. (Also note `/adversary`'s relevance here: CLAUDE.md's self-merge-on-green rule currently has the producer approving its own work — keep auto-ship for mechanical changes if you like the velocity, but route user-facing and money-touching changes through the adversary memo + your read.)

### 3.3 LEGO-Sorter

**Now — gates (1 session, the highest ratio in the portfolio):**
1. The 15-line CI workflow (offline suite, <10s) + pinned `requirements.txt`/lockfile + Dockerfile on 3.11 + pin pulp <4.0. This one session converts an ungated auto-deploying production app into a gated one.
2. Fix the build guide's feeder-pin contradiction (D6 → D5) — for a hardware novice this is the single most dangerous stale line in the portfolio.

**Next — record (1 session):**
3. `HARDWARE_STATE.md`: which phases are physically built, which sketch revision is flashed, the actual tuned angle constants — plus the habit "after any tuning session, paste final values here." Today those values live only on the flashed board.
4. Un-freeze the paper trail: five-line "Current state" header on CLAUDE.md pointing at `docs/ROADMAP.md` as the live plan; fix README's "M1–M4" and "to be built" lines; flip the two dead status markers (ROADMAP Phase 1, PATH_B M1) and add one line saying whether Path B is paused or superseded by the container deploy; resolve M4's ambiguity (software ✅ / feeder unbuilt).
5. Promote `segment_parity` into pytest; add the `check-apis` live-smoke command for pre-session contract checks; `.env.example` drift (add `APP_TOKEN`, remove or wire `REBRICKABLE_API_KEY`); tag milestones for rollback targets.
6. Add the self-healing rule + recap/adversary skills, sized to taste — this repo is a hobby; the gates matter, the ceremony can stay light.

---

## 4. Rollout

**Order.** Gates first (they protect everything else), then the chain conventions, then standing-instruction regeneration, then rituals. LEGO's session 1 is the best first move — smallest effort, whole-repo risk reduction, and a rehearsal of the pattern before applying it to the bigger repos.

**Who does what.** Claude Code can execute every numbered item above except: branch-protection/ruleset toggles, repo rename, Vercel/Render/Supabase dashboard settings, secret creation (QA account, Cloudflare token), and physical hardware steps. Those are **[Mike]** items — most are minutes each.

**The application PRs.** On your word, each repo gets one starter PR: `.claude/skills/` (recap + adversary), `docs/initiatives/` + registry, the CI gate file, and its regenerated CLAUDE.md — the per-repo plans above then proceed initiative by initiative, each with its own intent.md, as the convention prescribes. (This proposal — committed, dated, awaiting your sign-off before application — is initiative 001 of the convention it proposes.)

**The check.** In 30 days, run the audit that produced this document again (the reader prompts are reusable) and diff: are the new initiatives' folders present, did any status header go stale, did any gate get bypassed? The uplevel is working if the *mechanisms* — not your memory — caught whatever drifted.

---

## Appendix A — What's already strong (keep, and name as the standard)

- **Postmortem → permanent machine guard** (attune): requirements-pin tests, sync-parity test, `keep_files` after the blank-page P0. This is the house pattern; M4 just says "apply it before the incident, not only after."
- **Fixture honesty** (attune): keyless tests proving each eval fixture's claimed scenario actually appears in the built prompt — evals that can't drift into testing nothing.
- **Budget-safe eval architecture** (attune): `-m "not eval"` default; live model calls opt-in by label, never accidental.
- **Living plan-of-record with supersession** (Waypoint): ROADMAP.md's version header, locked-decisions table, dated status log — the model for every repo's registry.
- **The one perfect archive commit** (Waypoint `51b5e57`) and **the one fully closed loop** (attune `onboarding-improvements-plan.md`): existence proofs that the conventions in M1 are your own practices, promoted from events to rules.
- **Executable spec + honest as-built annotations** (LEGO): "M8 done (simulated): 115.8 ppm… real-hardware figure awaits the physical feeder build" — exactly the evidence discipline `/recap` asks for.
- **Deploy provenance** (attune): SHA stamped through gh-pages commit → version.json → bundle → API. M5's smoke check just closes the loop automatically.

## Appendix B — Source material

- Artifact-chain notes: every stage commits what the next stage reads; intent before analysis; standing instructions as versioned files; mechanical verification targets; advisory vs deterministic controls; producer never approves.
- Anthropic newsletter practices: the `/recap` skill (Richard Zhuang) — refresh stale state, then goal-in-their-words / status-with-evidence / blocked-on-person-vs-technical / owned next steps, no new work during recap; the adversarial pre-review subagent (Dickson Tsai) — fresh context, attack for correctness and simplicity, publish the assumptions/decisions memo so the human knows where to focus.
- Repo evidence: eight structured reader reports (2026-08-29) over `mcbeebe/Broken-Arrow-Training`, `mcbeebe/attune-coach`, `mcbeebe/Waypoint`, `mcbeebe/LEGO-Sorter`; file-level citations retained in the session record and reproducible with the same prompts.
