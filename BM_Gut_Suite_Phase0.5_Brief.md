# BM Gut Suite — Phase 0.5 Comparability Fix Brief

**Scope:** Make the dysbiosis index *comparable* — patient-to-patient and visit-to-visit — without touching question content or the validated instruments. Three independent changes: (§1) a consistent recall window on every active-symptom item, (§2) completion-gating + missing-data handling, (§3) a score-confidence indicator surfaced alongside the Index%. No new domains, no new questions, no re-weighting (that stays deferred with Phase 0 §6).

**Why this phase exists:** Phase 0 makes the score *honest* (red-flags, GI floor, history exclusion). Phase 0.5 makes it *mean the same thing each time it's measured* — the precondition for the visit-to-visit deltas the database already displays, and for any later anchoring against a validated PROM.

**Relationship to Phase 0:** Independent of §1/§2/§4. Has a soft dependency on Phase 0 §3 (history exclusion) — see §1 note on which items get a recall window. Recommended sequencing: do Phase 0 first, then §1 → §2 → §3 here.

**Recommended sequencing:** §1 → §2 → §3.

---

## §1 — Consistent recall window on active-symptom items (highest priority)

**Problem:** The questionnaire mixes recall timeframes, which makes a re-test non-comparable. Most GI/BG/ME items use `DEFAULT`/`FREQ` scales with **no stated window** (`scales.js` → `SCALES.DEFAULT` = `['Never / Absent', 'Mild / Occasional', 'Moderate / Frequent', 'Severe / Always']`), while others are explicitly bounded — `im_infections` ("past 12 months"), `bg_headache`/`gi_irregular_bowel` (`FREQ`), `hx_abx_5yr` (5 years). When a patient re-takes the tool in 8 weeks, an unbounded "moderate bloating" cannot reliably show change because there is no defined period being compared. This is the single largest threat to longitudinal comparability, and the database (`ui-database.js` visit cards + delta colouring) already presents those deltas as if they were meaningful.

**Target standard:** GI PROMs in the literature standardise on a short window (GSRS = past 1 week; many IBS instruments = past 2 weeks). Use **"in the past 2 weeks"** for all active-symptom items. Two weeks (not one) because the clinic re-test cadence is multi-week and a 2-week window is less noisy for intermittent symptoms.

**Which items get a window (and which must NOT):**
- **Active-symptom items → yes.** All GI items, the symptomatic IM items (`im_histamine`, `im_allergies`, `im_skin_issues`, `im_joint_pain`), all BG items, the symptomatic ME items (`me_weight_gain`, `me_sugar_crav`, `me_thyroid_sym`).
- **Status / lab / diagnosis items → no.** `im_autoimmune`, `im_crp`, `me_nutrients`, `me_cholesterol`, `me_fatty_liver` describe a standing fact, not a 2-week symptom. Leave their `STATUS`/`STATUS_DIAG` scales untouched.
- **History / exposure items → no.** All `HX` items are lifetime/multi-year by construction. (After Phase 0 §3 these are excluded from `total` anyway, so they cannot affect the comparable severity score regardless.)
- **SK items → yes for symptomatic, but keep `RECUR` semantics.** `sk_oral_thrush`/`sk_vaginal_yeast` use `RECUR` (recurrence over time) — leave as-is; `sk_nails`/`sk_skin_rash` are present-state and take the 2-week window.

**Location:** Two coordinated edits, both in the *presentation* layer only — the scoring math does not change.
1. `scales.js` → add a per-scale or per-question recall-window prefix. Cleanest is a single shared constant plus a per-question opt-out, rather than editing 25 question strings by hand.
2. `ui-patient.js` → render the window once per question (or once per section header) so it is visible at the point of answering.

**Proposed structure (option A — section-level banner, lowest churn):**
Render a single line under each scored section header in the patient view:
```js
// ui-patient.js, in the section-card header render for scored domains GI/IM/BG/ME/SK
const RECALL_NOTE = 'Answer for how things have been over the past 2 weeks.';
// shown for GI, BG, and the symptomatic-only sections; suppressed for HX (history)
```
**Proposed structure (option B — per-question, more precise, more churn):**
Add an optional `recall: '2wk'` flag on active-symptom questions in `schema.js` and have `ui-patient.js` append "(past 2 weeks)" to `patientText` when present. Status/history items omit the flag and render unchanged.

**Recommendation:** Option A for Phase 0.5 (one constant, one render line, immediately ships the comparability win); migrate to Option B only if/when per-item windows diverge.

**Scoring impact:** None. `computeScores`, `BANDS`, `RAW_MAX`, `EFFECTIVE_MAX` are untouched. This is a wording/administration change. Old saved visits remain valid; their stored `answers` are unaffected. The comparability gain is prospective (applies to visits taken after the change).

**Acceptance test:** Patient view of the GI section shows "past 2 weeks" guidance; the HX section does **not** show it; saving a visit produces identical `total`/`secScores`/`severity` to the same answers before the change (pure presentation diff).

---

## §2 — Completion-gating + missing-data handling

**Problem:** `scoring.js` → `computeScores()` treats every unanswered item as 0 (`const get = ... => answers[id] ?? 0`). Two consequences that break comparability:
1. **Downward bias.** A patient who skips half the items scores artificially "Minimal" — unanswered is silently read as asymptomatic.
2. **Non-comparable denominators.** A patient who answered 40 items and one who answered 25 are scored against the same `RAW_MAX`/`EFFECTIVE_MAX`, so their Index% values are not on the same footing.

There is currently no completion check anywhere before a result is shown or a visit is saved (`app.js` → `saveVisit` accepts whatever `snapshot.answers` contains).

**Location:** `scoring.js` → `computeScores()` (add completeness + proration); `ui-patient.js` (results gating / nudge); optionally `app.js` → `saveVisit` (warn-on-incomplete).

**Proposed fix — make missingness explicit and prorate:**
1. Distinguish *unanswered* from *answered-zero*. Today both are `0`. Change the patient UI to leave an item with **no** selected option as `undefined`/absent in the answers map, rather than defaulting it to 0. (Array-form answers keep using a sentinel — `null` — for unanswered; `computeScores` already tolerates `?? 0`, so the sentinel must be read before that coalesces.)
2. In `computeScores`, compute per-domain **answered counts** and prorate:
```js
function computeScores(answers) {
  const raw = Array.isArray(answers)
    ? (id) => answers[QID[id]]
    : (id) => answers[id];
  const answered = (v) => v != null;            // 0 counts as answered; null/undefined do not

  const secScores = {}; const bands = {}; const secAnswered = {};
  let total = 0, answeredItems = 0;
  for (const sec of SECTIONS) {
    let sc = 0, n = 0, nMax = 0;
    for (const q of QUESTIONS) if (q.section === sec.id) {
      nMax++;
      const v = raw(q.id);
      if (answered(v)) { sc += v; n++; }
    }
    secScores[sec.id] = sc;
    secAnswered[sec.id] = { answered: n, of: nMax };
    bands[sec.id] = sectionBand(sc, sectionMax(sec.id));
    if (sec.id !== 'HX') { total += sc; answeredItems += n; }  // HX excluded per Phase 0 §3
  }
  const scoredItems = QUESTIONS.filter(q => q.section !== 'HX').length;
  const completeness = Math.round(answeredItems / scoredItems * 100);
  return { secScores, bands, secAnswered, completeness,
           total, severity: severityOf(total, secScores), index: indexPct(total) };
}
```
> Note: this keeps `total` as a raw sum of *answered* items (no upscaling), so the severity band stays conservative on an incomplete form rather than extrapolating a patient up a band. Proration is reported via `completeness` and used by §3, not baked into `total`. (If a future phase wants a prorated index for cross-patient comparison at equal completeness, add `indexProrated = round(total / answeredMax * scale)` as a *separate* field — do not overwrite `total`/`index`, which the stored `code` and old reports depend on.)

3. **Gate the result presentation** (`ui-patient.js`): below a completeness threshold (suggest **< 80%** of scored items), show the severity result with a visible "provisional — N items unanswered" caveat rather than a clean band. Do not hard-block (clinician may want a partial read), mirroring the Phase 0 §1 red-flag decision to inform-not-block.
4. **Warn on save** (`app.js` → `saveVisit`): if `completeness < 80`, the existing `dialogConfirm` message gains a line — "This visit is N% complete; saved results will be marked provisional." No new dialog, just augmented copy.

**Dependency note:** The HX exclusion above assumes Phase 0 §3 has landed (`total` no longer includes HX). If Phase 0 §3 has *not* landed yet, drop the `sec.id !== 'HX'` guards here and re-add them when §3 lands — the completeness math is otherwise independent.

**Migration note:** Old visits stored unanswered-as-0 and have no way to recover true missingness — they are treated as 100% complete (all-answered), which is the existing behaviour, so nothing regresses. `completeness`/`secAnswered` are computed live (not stored), consistent with how `total`/`severity` are derived in `storage.js`.

**Acceptance test:**
- All 40 answered → `completeness = 100`, `total`/`severity` identical to today.
- GI fully answered, all else unanswered (`null`) → `total` = GI sum only, `completeness` ≈ 25–30%, result renders with the provisional caveat, no false "Minimal" from coalesced zeros.
- A genuine all-zero complete form → `completeness = 100`, "Minimal" (must remain distinguishable from the incomplete case above).

---

## §3 — Score-confidence indicator on the result

**Problem:** A clinician reading "Index 38% · Significant" has no signal for how much to trust it. After Phase 0 §2 (GI floor) and §2 above (completeness), two things now materially change the reliability of the headline number but are invisible:
1. **Completeness** (from §2) — a 60%-complete form is a weaker signal.
2. **GI-floor applied** (Phase 0 §2) — when the band was capped because GI burden was low, the headline severity is deliberately not the raw band, and the clinician should see that the cap fired.

**Location:** `ui-patient.js` (patient results hero), `ui-clinician.js` (dashboard severity card), `report.js` (printed report hero). One shared descriptor so all three agree — same pattern as `modifiableFactors()` in `scales.js`.

**Proposed structure — one shared helper, three render sites:**
Add to `scoring.js` (it already owns `severityOf`/`indexPct`, and Phase 0 §2 already threads `secScores` through `severityOf`):
```js
// scoring.js — single source for the confidence chip every surface renders.
function scoreConfidence(scoreObj) {
  const c = scoreObj.completeness ?? 100;
  const capped = scoreObj.severity && scoreObj.severity.capped === true; // set by Phase 0 §2 floor
  let level, note;
  if (c < 60)      { level = 'Low';      note = `Provisional — only ${c}% of items answered.`; }
  else if (c < 80) { level = 'Moderate'; note = `${c}% of items answered — interpret with care.`; }
  else             { level = 'High';     note = 'All key items answered.'; }
  if (capped) note += ' Severity capped: direct gut symptoms are low relative to other domains.';
  return { level, note, completeness: c, capped };
}
```
> Requires Phase 0 §2 to tag its capped return with `capped: true` (currently it returns a `desc` override but no boolean). Add `capped: true` to the `Object.assign` in the floor branch of `severityOf` so this helper doesn't have to string-match the desc. If Phase 0 §2 isn't in yet, ship §3 with completeness-only and add the cap clause when §2 lands.

**Render:** a small chip under the Index% in the hero (`.pill`-style, reuse existing `.muted`/`.sub` classes — no new CSS required): e.g. `Confidence: Moderate · 72% complete`. Clinician card shows the full `note`; patient hero shows the short `level` + completeness; report prints the `note` line in the hero block.

**Scoring impact:** None — purely a derived, displayed descriptor. No change to `total`/`index`/`code`/bands.

**Acceptance test:**
- 100% complete, floor not applied → chip reads "High · 100% complete", no cap note.
- 72% complete → "Moderate · 72% complete" on all three surfaces (patient/clinician/report), text identical (shared helper).
- Floor applied (Phase 0 §2 capped result) → cap clause appended to the note on the clinician card and report.

---

## Deferred (unchanged from Phase 0)

- **External anchoring / concurrent validity.** Making the Index% interpretable against a validated PROM (GSRS / IBS-SSS / Rome IV) and deriving an MCID is a *data-collection* task, not a code change — but §2's `completeness` and a clean recall window (§1) are the preconditions that make that pilot data usable. Add to the Phase 0 §5 recalibration TODO.
- **Item de-duplication / re-weighting.** `gi_bloating`+`gi_gas`, `im_skin_issues`+`sk_skin_rash`, and the double-barrelled `im_histamine` overlap and cross-load. Holding with Phase 0 §6 — any manual re-weighting now would be overridden by empirical factor loadings later.
- **Domain-mean index** (scoring as a mean of domain scores rather than a sum of item scores, to remove item-count weighting bias). Larger change to `computeScores`/bands/`code`; sequence it *with* the recalibration, not before.

---

## Summary table for Claude Code

| § | File(s) | Function(s) | Type | Depends on |
|---|---|---|---|---|
| 1 | `scales.js`, `ui-patient.js` | render only (recall note) | presentation | Phase 0 §3 (soft — which items) |
| 2 | `scoring.js`, `ui-patient.js`, `app.js` | `computeScores()`, results gate, `saveVisit` | logic + presentation | Phase 0 §3 (HX exclusion) |
| 3 | `scoring.js`, `ui-patient.js`, `ui-clinician.js`, `report.js` | new `scoreConfidence()`, 3 render sites | derived + presentation | §2; Phase 0 §2 (cap flag) |
