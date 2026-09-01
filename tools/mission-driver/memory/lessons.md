# Mission-Driver Self-Memory — Procedural Lessons

Runtime-generated durable lessons about the mission-driver harness/loop (prompts,
flow, retries, Windows/tooling quirks). Maintained by `mission-driver analyze`
under a consolidate-don't-accumulate protocol. One rule per recurring failure;
each rule is imperative and points at a concrete file or command.

---

## L001 — Treat opencode empty-body + exit≠0 as a transient fault, not an onError failure
- **Rule:** When an agent step returns a header-only/empty body with `exitCode !== 0` and no stderr, classify it as a transient provider/CLI crash and retry on the independent transient budget. Never let it consume `onError.maxRetries` or trip `maxCycleVisits`.
- **Origin:** ENV/TOOL   **Severity:** SEV1   **count:** 1   **last_seen:** 2026-07-29
- **Evidence:** `docs/postmortems/2026-07-29-mission-driver-actionable-fixes-postmortem.md` F1 — 39 retries, ~1h10m aborted; `oc-EXECUTE-1785299653961-73hw8q.log` (0-char body), run-state `empty/short output, exit=1 (no stderr captured)`.
- **Fix target:** `src/engine.js` transient block (~`:1698-1744`) — extend the transient signature to include `body.length < PARSE_MIN_BODY_CHARS && exitCode !== 0 && !stderrTail`.

## L002 — Give execute/closure prompts an "already-landed" short-circuit
- **Rule:** Tell EXECUTE and CLOSURE_AUDIT agents: if the code and tests are already present (inconsistent state: work done, boxes unticked), tick the items, run the test gate ONCE, and emit the result marker immediately. Do not re-derive or re-run the full verification suite for already-landed work.
- **Origin:** PROMPT   **Severity:** SEV1   **count:** 1   **last_seen:** 2026-07-29
- **Evidence:** `docs/postmortems/2026-07-29-mission-driver-actionable-fixes-postmortem.md` F2 — agents burned whole turns re-verifying and were cut off mid-sentence; `CLOSURE_SCRIPT_CHECK` failed all visits ("17 unchecked items remain") despite green build.
- **Fix target:** `prompts/execute.md` (near top) and `prompts/closure-audit.md`.

## L003 — Gate rate-limit backoff on a real rate-limit signal, not on duration alone
- **Rule:** Never assume a short-duration failure is a rate limit. Only back off (30s+) when there is an actual provider rate-limit/overload signature in stderr or a transient flag. For empty-body no-stderr crashes, use a short fixed delay (2–5s) or none.
- **Origin:** FLOW   **Severity:** SEV2   **count:** 1   **last_seen:** 2026-07-29
- **Evidence:** `docs/postmortems/2026-07-29-mission-driver-actionable-fixes-postmortem.md` F3 — `backing off 30-90s before retry (previous attempt lasted 6s — likely rate-limited)` ×19; `src/engine.js:1780` keys on `durationMs < 60_000`.
- **Fix target:** `src/engine.js:1780-1792`.

## L004 — Do not reference {{forEachItem}} at a forEach-container step's scope (prompt OR flowArgs)
- **Rule:** `{{forEachItem}}` is bound per child subflow *iteration*, not at the forEach-container step scope. Remove it from the container step's prompt text AND from its top-level `flowArgs` (e.g. EXEC_PLANS `flowArgs.PLAN_FILE`) — the engine re-resolves `flowArgs` per iteration with `forEachItem` bound (`src/engine.js:1076-1079`), so the step-scope binding is dead and only produces a recurring `WARNING: unresolved template variable`. If you must reference the item at container scope, move that text into the per-item subflow prompt.
- **Origin:** FLOW   **Severity:** SEV3   **count:** 3   **last_seen:** 2026-08-04
- **Evidence:** `docs/postmortems/2026-07-29-mission-driver-actionable-fixes-postmortem.md` F4 (prompt text) ; `etd-age/tools/mission-driver/docs/postmortems/2026-08-03-onboarding-postmortem.md` F2 — `WARNING: unresolved template variable {{forEachItem}}` ×3 on EXEC_PLANS visits 1/2/3; root now at `flows/mission-driver.json:51` `"flowArgs": { "PLAN_FILE": "{{forEachItem}}" }` ; recurred again `etd-age/tools/mission-driver/docs/postmortems/2026-08-04-docs-deepening-and-optimization-proposals-postmortem.md` F2 (EXEC_PLANS visits 1+2; per-iteration PLAN_FILE still resolved correctly) — known fix still unapplied.
- **Fix target:** `flows/mission-driver.json` EXEC_PLANS step — drop the step-scope `flowArgs` line (per-iteration re-resolution covers it); alternatively suppress the warning for forEach-iteration vars (`forEachItem`/`forEachIndex`/`forEachTotal`) in the template resolver (`src/expression.mjs`).

---

## L005 — Draft and review must RUN every plan Proof/verification command, not write it by intuition
- **Rule:** Every `Proof` / verification grep/command in a drafted plan MUST be executed by the author against the live repo and its real output cited inline; the REVIEW_PLANS step MUST run each one and confirm the real output count matches the stated `expect:` before promoting `draft` → `active`. Two regex traps recur: char-class `[xy]` matches single chars only (use `(x|y)` for alternation); under `grep -E` the alternator is bare `|` while `\|` is a literal pipe.
- **Origin:** PROMPT   **Severity:** SEV2   **count:** 1   **last_seen:** 2026-08-03
- **Evidence:** `etd-age/tools/mission-driver/docs/postmortems/2026-08-03-onboarding-postmortem.md` F1 — drafted plan shipped `grep -nE '^\| 2026-07-2[07]'` (can't match day 30) and a `\|` alternation under `grep -E` (literal pipe); REVIEW_PLANS missed both (caught only an unrelated over-scoped grep); EXECUTE worked around them with corrected regexes and documented inline.
- **Fix target:** `prompts/draft-from-roadmap.md` (add run-and-cite Proof rule + the two regex traps) + `prompts/plan-review.md` (add Review Checklist item: run each Proof command; real output must match `expect:` or it is a Major).

## L006 — A template var wired into one renderer is unbound everywhere else; audit shared prompts for unresolvable tokens
- **Rule:** Before a prompt references `{{var}}`, confirm that var is injected by the renderer that actually serves that prompt's steps. `backlogDir` is injected only into the brief/draft pipeline (`src/main.js:403,467`), yet `prompts/draft-from-audit.md` (reused by the audit-loop steps CHECK_OPEN_AUDITS + SCAN_NEW_RESULTS) references `{{backlogDir}}` — so it emits `WARNING: unresolved template variable {{backlogDir}}` every audit round and ships a literal broken token to the model. Either reword the prompt to drop the unbound token, or inject the var into the run-loop step renderer so all prompts see it.
- **Origin:** FLOW/PROMPT   **Severity:** SEV3   **count:** 1   **last_seen:** 2026-08-04
- **Evidence:** `etd-age/tools/mission-driver/docs/postmortems/2026-08-04-docs-deepening-and-optimization-proposals-postmortem.md` F1 — `WARNING: unresolved template variable {{backlogDir}}` ×2/round (CHECK_OPEN_AUDITS + SCAN_NEW_RESULTS); `prompts/draft-from-audit.md:8` references it; `src/main.js` injects it only at :403/:467 (brief/draft); model recovered by routing `[P2]` to the roadmap `## Follow-up Backlog`.
- **Fix target:** `prompts/draft-from-audit.md:8` (reword to "in the mission roadmap's `## Follow-up Backlog` section"); alternatively inject `backlogDir` into the run-loop per-step `resolveTemplateVars` call in `src/main.js`. Mirror the existing `test/draft-path-consistency.test.js` Case A coverage for audit-loop prompts.

## L007 — Verify host RAM headroom before long DEEP_AUDIT runs; residents (IntelliJ/Docker) can starve the loop
- **Rule:** Before launching a mission whose roadmap includes DEEP_AUDIT, ensure ≥2 GB free RAM (close IntelliJ/Docker). The run loop logs `freeMemGB` per heartbeat — sustained values <0.5 GB risk slowdown/OOM on large audit corpora even though individual agent node processes (~0.6 GB RSS) fit. Treat low `freeMemGB` as a preflight signal, not a per-step failure.
- **Origin:** ENV/TOOL   **Severity:** SEV3   **count:** 2   **last_seen:** 2026-09-01
- **Evidence:** `etd-age/tools/mission-driver/docs/postmortems/2026-08-04-docs-deepening-and-optimization-proposals-postmortem.md` F3 — `sys-snapshot.csv` freeGB 0.2 at START and through all of DEEP_AUDIT (memPressure 48–53%); ideaMB ~2.6–3.3 GB + dockerMB ~1.6–1.9 GB dominant; agent node ~0.6 GB; no OOM only because each step fit. Recurred `app_starter/tools/mission-driver/docs/postmortems/2026-09-01-starter-postmortem.md` §5 — freeGB 0.2–0.6 through a 7h48m run (memPressure 42–50%), no OOM attribution.
- **Fix target:** `TROUBLESHOOTING.md` (add a ≥2 GB free-RAM preflight entry before DEEP_AUDIT); optionally `src/main.js` run launcher — warn (not abort) at run start when free RAM is below threshold.

---

## L008 — Give held (human-gated) plans a `held` terminal outcome; never let them ride the fail→retry cycle
- **Rule:** When a plan's frontmatter is `status: held` (or its next item is an ask-first human gate with no recorded decision), EXECUTE must emit `<AI_STEP_RESULT>held</AI_STEP_RESULT>` immediately — no re-verification battery — and the flow must terminate the subflow as `blocked` instead of retrying. Never dispatch BUILD_VERIFY's full command battery for a plan known to be held; never let BUILD_VERIFY's build-health pass mark a subflow `completed` while plan items remain unchecked.
- **Origin:** FLOW   **Severity:** SEV1   **count:** 1   **last_seen:** 2026-09-01
- **Evidence:** `app_starter/tools/mission-driver/docs/postmortems/2026-09-01-starter-postmortem.md` F1 — 20 EXECUTE fail-visits + 14 `max_retries` limit hits across 3 human gates (~2h cycling of a 7h48m run); `oc-BUILD_VERIFY-1788239388517-4eqygw.log` ran the full 4-command battery green then honestly failed for a held 0/28 plan; subflow 2-1 "completed" via BUILD_VERIFY pass with 17 unchecked items, deferring real completion to the next 3h40m round.
- **Fix target:** `flows/plan-execution.json` (add `"held": { "done": "blocked" }` to EXECUTE and BUILD_VERIFY) + `prompts/execute.md` / `prompts/build-verify.md` (emit `held` marker, skip battery) + `src/engine.js` (accept `held` in the marker vocabulary, surface subflow `blocked`).

## L009 — Fail fast (or warn loudly) when DEEP_AUDIT runs with no audit prompts configured
- **Rule:** Before launching a mission with `maxAuditRounds > 0`, verify `multiAuditPrompt`/`openAuditPrompt` are non-empty; if both are empty the deep-audit round audits NOTHING while still consuming a round and producing a clean-looking `audit_gate` exit. Warn at run start or refuse to count vacuous audit rounds.
- **Origin:** FLOW   **Severity:** SEV2   **count:** 1   **last_seen:** 2026-09-01
- **Evidence:** `app_starter/tools/mission-driver/docs/postmortems/2026-09-01-starter-postmortem.md` F3 — `events.jsonl` `step_skipped` MULTI_AUDIT + OPEN_AUDIT `"reason":"when condition false"`; `flows/deep-audit-loop.json` gates both on `multiAuditPrompt != ''` / `openAuditPrompt != ''`; `missions/starter.json` defines neither — audit round 1/3 was a 3 ms no-op and the roadmap closed on DRAFT_PLANS self-assessment alone.
- **Fix target:** `src/main.js` (run-start validation: `maxAuditRounds > 0` ∧ both audit prompts empty → loud warning or fail fast); alternatively ship default audit prompts in `missions/base.json`.

## L010 — Never run concurrent missions against a digest-pinned baseline sharing one Maven localRepository
- **Rule:** If a mission's gate pins digests of installed upstream artifacts (e.g. compliance R4), do not run another mission (or any `mvn install` of a different vintage) against the same `localRepository` concurrently — it silently overwrites pinned artifacts and red-lines the gate until a human re-adjudicates. Isolate per mission (`-Dmaven.repo.local`) or serialize missions on the host.
- **Origin:** ENV/TOOL   **Severity:** SEV2   **count:** 1   **last_seen:** 2026-09-01
- **Evidence:** `app_starter/tools/mission-driver/docs/postmortems/2026-09-01-starter-postmortem.md` F2 — `oc-CLOSURE_AUDIT-1788253482524-o8wytd.log`: "R4 fails with the exact same 7 digest mismatches … (external localRepository pollution by the parallel onerx mission)"; blocked M6-WI3's last 2 items ~55 min until human adjudication commit `da88420`.
- **Fix target:** `TROUBLESHOOTING.md` (concurrent-mission localRepository isolation entry); operationally, per-mission `-Dmaven.repo.local` in mission `commands`.
