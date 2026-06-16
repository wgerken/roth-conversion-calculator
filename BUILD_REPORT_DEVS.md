# Build Report — Roth Conversion Educational Calculator

**Builder:** DEVS · **Date:** 2026-06-16 · **Deliverable:** `index.html` (single self-contained file)
**Spec:** `Team Inbox/Handoff_2026-06-15/ROTH_CONVERSION_CALCULATOR_HANDOFF.md`

## What I built

A single `index.html` — HTML + CSS + vanilla JS, no build step, no npm, no CDN, no network calls.
Opens by double-click on Windows and runs fully offline. Three cleanly separated layers inside the
file:

1. **`CONFIG_2026`** — the `TaxYearConfig`, visibly delimited and commented "REPLACE YEARLY." The
   `irmaa`, `fpl`, and `aca` objects are pasted **verbatim** from ARIA's verified supplement.
2. **Calculation engine** — pure functions, money in **cents** internally (`toC`/`toD`), no DOM
   access. This is the part that must match the spec formulas.
3. **Wizard/UI layer** — 9-step flow, "show the math" expanders, glossary tooltips, cliff warnings,
   keyboard-navigable step pips, responsive, print/export.

## Spec §7 function → code map

| Spec | Function in code |
|---|---|
| 7.1 progressive ordinary tax | `tax(incomeC, brackets)` (cents) + `marginalRate`, `bracketTopC`, `topOfBracketC` helpers |
| 7.2 taxable income + marginal rate + headroom | computed in `computeConversion` (`ordinaryTaxableBefore`, `mRateBefore`, `headroom`) + `seniorAddl` |
| 7.3 conversion sizing (amount / fillBracket) | `computeConversion` conversion block using `topOfBracketC` |
| 7.4 incremental federal tax | `fedTaxAfter − fedTaxBefore` via `tax()` |
| 7.5 SS taxation delta (no double count) | `taxableSS(...)` run before & after; folded into one ordinary-tax delta; `ssTaxationExtraTax` isolated as a transparency sub-component (already inside the federal figure — not double-added) |
| 7.6 LTCG stacking | `ltcgStackTax(ordinaryTaxableC, gainsC, ltcgBrackets)` before & after |
| 7.7 NIIT | `niitBefore/After = rate · min(NII, max(0, MAGI − threshold))` |
| 7.8 IRMAA cliff delta + distance-to-boundary | `irmaaTier(...)`; `irmaaAnnualSurchargeDelta`; boundary-crossing warning + `distToNext` |
| 7.9 ACA subsidy delta + 400% FPL cliff | `acaSubsidy(...)`, `acaApplicablePct(...)` (linear interp within band); cliff warning |
| 7.10 total cost + blended rate | `totalIncrementalCost`, `blendedEffectiveRate` (null when C=0) |
| 7.11 after-tax comparison + break-even | `comparison` block; two modes via `conversion.payTaxFrom` — taxable-pays (Roth vs. IRA-plus-after-tax-side-fund) and IRA-pays (`C_net = C·(1−blended)`, no side fund); mode-aware break-even loop |
| 7.12 RMD projection | `rmd` block (grow to start age 73, divide by Uniform Lifetime factor) |

**Compute ordering** follows §7: conversion size → recompute taxable SS with conversion in
provisional income → AGI/MAGI after → federal/LTCG/NIIT/IRMAA/ACA all keyed off the *after* MAGI.
The cascade is respected (MAGI is computed once per state and reused by the threshold modules).

## Decisions

- **MFS:** **Included but gated.** MFS is selectable in the filing-status dropdown. When selected,
  Step 1 shows a persistent "MFS IRMAA tiering pending verification" notice, the cost table tags the
  IRMAA line `(MFS pending)`, and the engine emits a warning whenever the IRMAA module runs for MFS.
  It uses ARIA's compressed 3-tier MFS table (middle boundary ~$394k flagged pending). No unverified
  MFS number is used silently. Single/MFJ/HoH are fully verified and unflagged.
- **Oregon config:** Loaded **2025** Oregon brackets + standard deduction as a clearly-commented
  `verify:true` placeholder — exact 2026 OR indexed brackets were **not** in the provided source
  docs. The config name string and the Summary assumptions list both surface "2025 brackets — verify
  2026." I did not block the build on it (per instructions). The state layer is proven config-driven
  by the `none` (no-income-tax) option, which yields state tax = 0 from empty brackets.
- **SS torpedo display:** §7.5 says fold taxable-SS into one tax delta to avoid double counting. I do
  exactly that; the "of which Social Security" line is a *decomposition* of the federal figure, not an
  additional add. It is **not** summed again into total cost.
- **OBBBA senior deduction:** modeled as an optional checkbox with the 6%-over-MAGI phaseout.

## Test results

In-page self-test panel (Summary step → "Run self-tests"); also runs on load to the JS console.
Verified by extracting the engine and running under Node:

**17 passed, 0 failed** at original build. (Now **22 passed, 0 failed** after the 2026-06-16
remediation — see the Remediation section below for the 5 added assertions.)

Includes: `tax()` bracket-edge value and +$1 marginal (22%), `tax()` monotonicity property test,
LTCG stacking straddling 0%→15%, NIIT threshold, **IRMAA tier boundary** (exact-boundary dollar
stays in lower tier; +$1 trips the cliff), **ACA 400% FPL cliff** (subsidy >0 just under, =0 just
over), the **§10 worked example** (MFJ both 62, Oregon, fill-the-22%-bracket), C=0 divide-by-zero
guard, and the conversion-≤-taxDeferred cap.

**§10 worked example — PASS.** Fills to top of MFJ 22% bracket: C = $148,600 (211,400 − 62,800).
Federal incremental matches hand calc `tax(211,400) − tax(62,800)`. No IRMAA (age 62), no ACA (not
marketplace). Blended effective rate ≈ **28.2%** (federal portion that crosses 12%→22%, **plus**
Oregon marginal ~8.75–9.9% on the converted dollars). The advantage-sign property holds: with this
example's blended rate (28.2%) above the input's 24% future rate, advantage is negative (don't
convert) — the engine then confirms advantage **> 0** once `expectedFutureMarginalRate ≥
blendedEffectiveRate` (test raises tFuture above blended and re-asserts). This is the spec's
conditional, and it passes.

### One bug found & fixed during verification
Initial `irmaaTier` used `magiMin ≤ magi < magiMax`, which put a MAGI sitting **exactly** on a tier
boundary into the *higher* tier. IRMAA's rule is "cross by $1" — the boundary dollar stays in the
**lower** tier. Fixed to `magiMin < magi ≤ magiMax` (tier 0 lower-bounded at −∞). Re-ran: 17/0.

## Known limitations / TODOs

- **Oregon 2026 brackets unverified** — using 2025 figures as a flagged placeholder. Needs a 2026 OR
  bracket pull before any Oregon number is relied upon.
- **MFS IRMAA middle boundary pending** (~$394k vs. $391k across secondary sources) — gated, not
  silent. Close with one CMS primary pull (ARIA's Pending item #1).
- **IRMAA Part B/D dollar figures** are one step removed from a directly-fetched CMS page (ARIA's
  Pending item #2) — arithmetically constrained and high-confidence, but the non-advice disclaimer
  stays prominent.
- **ACA**: defaults to 48-contiguous-states FPL; Alaska/Hawaii increments exist in ARIA's data but
  are not wired into a UI region toggle. SLCSP benchmark premium is a user input (the tool can't know
  it) — subsidy delta is $0 until the user enters a benchmark.
- **Multi-year planner** holds income/brackets/growth flat (no inflation indexing) — matches §13's
  "future enhancement," surfaced in a UI note. Caps displayed years at 15.
- **RMD table** is a subset (ages 73–100); extend `rmd.uniformLifetime` for older ages if needed.
- AGI/MAGI are modeled at the precision this educational tool needs (MAGI ≈ AGI + tax-exempt
  interest); not every line-item AGI adjustment is captured — appropriate for an estimate, not a
  return.

## Remediation 2026-06-16 (post-audit patch — ARIA F1, RHETT B1/B2 + publication conditions)

Focused patch to `index.html` after ARIA's correctness audit and RHETT's Gate 9 pass. No rebuild —
everything both audits PASSED (config transcription, bracket/tax math, AGI/MAGI defs, SS-torpedo
decomposition, NIIT base, IRMAA/ACA cliff detection) is untouched. Engine functions stay pure,
money stays in cents, all new behavior is config/input-driven. Every fix verified with a number under
Node before/after.

### B1 (BLOCKER, ARIA F1) — leftover standard deduction not applied to LTCG → gains over-taxed. **FIXED.**
- **Root cause:** ordinary taxable income was floored at 0 (`clamp0`) and the *full* gains amount was
  passed to `ltcgStackTax`, dropping the leftover deduction (deduction − ordinary income) when ordinary
  income is below the deduction.
- **Fix:** in `computeConversion`, compute the un-floored ordinary figure in both states,
  `leftoverDed = clamp0(−ordinaryRaw)`, and pass `clamp0(gains − leftoverDed)` as the gains argument.
  Stack base stays the floored ordinary taxable income. The *after* state recomputes leftover with the
  conversion included (conversion raises ordinary income, shrinking/removing the leftover).
- **Worked verification (Single, 2026): ordinary $10,000, std deduction $16,100, gains $55,000.**
  Leftover deduction $6,100 → taxable gains $48,900, under the Single 0% ceiling of $49,450.
  **Before fix: $832.50 LTCG tax (phantom). After fix: $0.00 EXACT.** Both reproduced under Node.
- **No-regression:** Single ordinary $90,000 (taxable $73,900 > 0% top), gains $20,000 → all $20k @ 15%
  = **$3,000 EXACT**, unchanged. Pinned as a self-test.

### B2 (BLOCKER, RHETT B1 / ARIA F2) — no-convert side fund grew tax-free; spec §7.11 requires after-tax. **FIXED.**
- **Root cause:** the §7.11 side fund (`C·blendedRate`, the tax you'd otherwise have paid) grew at the
  full pre-tax/Roth factor `(1+g)^n`, over-crediting the no-convert side and biasing the verdict against
  converting every run.
- **Fix (option a — clean input):** added `taxableDragRate` to the `projection` inputs (default `0.15`,
  exposed as a new UI field in Step 6). Side fund now grows at `g_afterTax = g·(1 − taxableDragRate)` in
  BOTH `noConvertAfterTaxValue` and the break-even loop. Convention documented in the Step 6
  "show the math" expander.
- **Worked verification (§10 case: MFJ, OR, C=$148,600, g=5%, n=20, tFut=24%, blended 28.19%, drag 15%
  → g_afterTax = 0.0425):** no-convert value **$395,964**, advantage **−$1,684** (was **−$16,560**
  pre-fix). Matches RHETT's predicted ~$395,989 / ~−$1,709 (~$15k swing corrected). Node-reproduced.

### B3 (BLOCKER, RHETT B2) — break-even was a tautology. **REPAIRED (not removed).**
- After B2, the side fund grows slower than the Roth, so the `(1+g)^y` factor no longer cancels and a
  genuine crossover exists. **Verified:** at tFut 27% (just under blended 28.19%), advantage starts
  **−$1,772 at y=0**, crosses zero near **year 7**, and grows monotonically positive thereafter
  (**+$61,047 at y=40**). Break-even sweep produces real varying years as tFut rises
  (24%→23 yrs, 26%→12, 27%→7, 28%→1, 30%→0). KPI kept. Two self-tests pin the crossover.

### C1 (publication) — naked point-estimate verdict. **DONE.**
Step 6 now renders a sensitivity band: the verdict (advantage $ + convert/wait) across `tFut ±5 points`
(−5, −2.5, 0, +2.5, +5), re-running the engine at each rate. A close-call box fires when the verdict
FLIPS inside the band. Verified live: default §10 case shows wait→wait→wait→convert→convert across
19%–29% and correctly flags the flip.

### C2 (publication) — frozen-law disclosure. **DONE.**
Step 7 note rewritten to state the projection freezes 2026 brackets/law (no TCJA-successor sunset, no
inflation indexing, static balance) and names the bias direction: flat nominal brackets **understate**
how much can be converted at a given rate in later years; upward bracket reversion would make converting
now look even better. Adds "verify current tax law" and "illustration, not a plan." Summary assumptions
list updated to match.

### C3 (publication) — future-side state tax. **DONE.**
The `expectedFutureMarginalRate` field label now says "ALL-IN" and its tooltip instructs the user to
include expected future **state** tax (the cost side already carries current state tax). No separate
future-state model — guidance only, per the condition.

### Ship-flagged items — confirmed rendering.
Oregon 2025 placeholder (`verify:true`, surfaced in config name + Step 1/Summary) and MFS IRMAA pending
notice both render on-screen. No change in this patch. (Print/export carriage of these flags was
tightened separately in the 2026-06-16 pre-publication housekeeping pass — see H2 below: the MFS notice
was added to the Summary so it survives into the exported PDF even when the IRMAA module is off.)

### F6 (housekeeping) — self-test for B1 path + exact assertions. **DONE.**
Added 5 assertions (17 → **22 total**): B1 spillover = **$0 EXACT**, B1 no-regression = **$3,000 EXACT**,
B2 after-tax rate = **0.0425 EXACT**, B3 crossover (adv<0 at y=0 & adv>0 at y=40), B3 finite break-even
(0 < be < 60). Two pin exact hand-computed numbers rather than ranges.

### F3/F4 (housekeeping) — MFS `taxableSS` tautology. **DONE.**
Replaced `Math.min(0.85*ssGross, 0.85*ssGross)` with `return 0.85*ssGross;` plus a note pointing to the
MFS-lived-apart edge for when MFS is productionized. Cosmetic; behavior identical.

### Test results — Remediation
Re-extracted the engine and ran under Node: **22 passed, 0 failed.** `node --check` on the full
`<script>` block: clean. Loaded in-browser via local static server: self-tests log **22/0** on load,
no console errors; Step 6 (drag input, ALL-IN label, sensitivity band, break-even 23 yrs, advantage
−$1,684) and Step 7 (frozen-law disclosure) render correctly. **§10 worked example still passes**
(C=$148,600, blended 28.19%) — unchanged by the patch.

### Not changed / out of scope for this patch
F5 (marginal rate display at exact bracket floor — cosmetic, tax owed correct) left as-is per audit
disposition. RHETT N2 (IRA-pays-the-tax toggle) not implemented — the external-payment assumption is
surfaced at the Step 6 verdict via the existing tooltip; full toggle was not in the approved scope.

---

## Pre-publication housekeeping 2026-06-16 (post re-verification — DEVS)

Cleanup-only pass before publishing the folder for review. **No engine or calculation logic changed;
no features added (the N2 toggle was explicitly NOT implemented — see below).** Both housekeeping
edits are string/disclosure or comment changes in the UI/doc layer. Self-tests re-run green after each.

### H1 — `engine.js` disposition: KEPT as a dev-only test artifact, regenerated & marked DO-NOT-PUBLISH.
`engine.js` is the headless Node extract of the engine + §12 self-tests; it is part of the verification
workflow (`node engine.js` is how the suite runs outside a browser), so I kept rather than deleted it.
`index.html` does **not** reference it (the tool is fully self-contained), so it never affected runtime —
but a stale copy in a published folder would be a trap. Resolution:
- **Regenerated from the CURRENT patched `index.html`.** The engine + self-test region is now
  **byte-equivalent** to `index.html` (verified by diff: only the DOM-only `$()` helper and in-page
  comment banners are omitted; `module.exports` + a CLI runner are appended). It carries the F1/B1
  leftover-deduction LTCG fix and the B2 after-tax side-fund drag fix — i.e. it matches the shipped
  engine exactly. No pre-patch/buggy code remains anywhere in the folder.
- **Marked dev-only / exclude-from-publish:** a `*** DEV-ONLY TEST ARTIFACT — DO NOT PUBLISH ***`
  banner heads the file, and the README now has a "What's in this folder (and what gets published)"
  table flagging `index.html` as the only published file and `engine.js` as excluded.
- (A background housekeeping chip also targeted this; the end state above is authoritative.)

### H2 — Print/export flag carriage: Oregon already present; **MFS notice ADDED to the Summary.**
The audit conditioned publication on the Oregon-2025 and MFS-IRMAA-pending flags appearing in the
print/export output, not only the on-screen wizard (a reader may keep only the exported PDF). The
print/export path is the browser's native `window.print()` from the **Summary** step (no separate
export function); `@media print` hides chrome (`header.app`, `.nav`, `.steps`, `.no-print`) and prints
the visible Summary panel. Inspected `stepSummary()` for both scenarios:
- **Oregon (`state=OR`): already present.** The config name string `"Oregon (2025 brackets — VERIFY
  2026)"` renders in the Summary's opening line, and the assumptions list carries "Oregon brackets are
  2025 figures pending 2026 verification." No change needed.
- **MFS (`filingStatus=mfs`): was a GAP — now fixed.** Previously the MFS-pending disclosure reached
  the Summary *only* via `r.warnings`, which is populated only when the IRMAA module runs
  (`onMedicare || userAge≥63`). An MFS user under 63 and not on Medicare got **no** MFS disclosure in
  the printed Summary even though MFS was selected. **Fix:** added an MFS-pending `warnbox` to
  `stepSummary` keyed on `filingStatus==='mfs'` itself (not on the IRMAA warning). This is a disclosure
  string in the export rendering; engine untouched.
- **Verification (Node-rendered `stepSummary` markup):**
  - MFS + OR, age 58 (IRMAA off): Oregon flag ✔, MFS notice ✔ (previously ✘).
  - MFS only, no-tax state, age 58: MFS notice ✔, Oregon correctly absent.
  - MFJ + OR: Oregon flag ✔, MFS notice correctly absent (gated to MFS only).

### Out of scope (unchanged, tracked fast-follow)
- **RHETT N2 — IRA-pays-the-tax toggle: NOT implemented** at the time of the housekeeping pass
  (explicitly out of scope then). **NOW IMPLEMENTED in the 2026-06-16 final build — see the "Final
  build" section below.**

### Test results — Housekeeping
`node --check engine.js`: clean. `node engine.js`: **22 passed, 0 failed.** **§10 worked example
unchanged** (C=$148,600, blended 28.19%). Engine+self-test region confirmed still byte-identical
between `index.html` and `engine.js` after all edits. No engine logic, no config values, and no
features were changed — disclosure strings and comments/docs only.

---

## For RHETT / ARIA to sanity-check

1. **ARIA:** the MFS gating and the IRMAA "boundary dollar = lower tier" semantics — confirm that
   matches the CMS intent for your tiers, and confirm the Oregon-2025-placeholder flag is acceptable
   for v1 or should be pulled before release.
2. **RHETT:** the §7.11 after-tax comparison convention (paying tax externally ≡ sheltering it in
   Roth, modeled as a side fund grown at the same rate) and the SS-torpedo decomposition (shown but
   not double-counted) are the two places a reviewer's eyes should linger.
3. The blended-rate sign logic in the §10 example: because Oregon adds ~9% on top, the example's
   blended rate (28.2%) exceeds the spec's illustrative "≈ 22% fed + OR marginal" only because OR
   marginal is itself ~6–9% — worth a second read to confirm that matches the intended worked figure.

---

## Final build — 2026-06-16

Last build pass before publication: implemented the two remaining tracked items (N2, F5) and
integrated ARIA's confirmed config (closeout addendum
`Research Library/TAX_REFERENCE_2026_Closeout_Addendum_ARIA.md`).

### N2 — "IRA-pays-the-tax" toggle (spec §7.11) — IMPLEMENTED
- New input `conversion.payTaxFrom: "taxable" | "ira"` (default `"taxable"`). UI control is a
  two-radio "Where does the conversion tax come from?" block in Step 6, near the comparison.
- **Taxable mode is unchanged** from the already-verified behavior (additive feature). Engine branches
  only when `payTaxFrom==='ira'`.
- **IRA mode** (spec §7.11): tax reduces the converted principal first — `C_net = C·(1 − blended)` —
  so the Roth grows only `C_net·(1+g)ⁿ`. The "don't convert" side has **no side fund** (no outside
  dollars were spent on tax), so `noConvert = C·(1+g)ⁿ·(1 − tFuture)`. Advantage and break-even are
  re-derived consistently per mode (the break-even loop branches on the same flag).
- Lower efficiency is flagged prominently: a red (`warnbox cliff`) callout in Step 6, the C_net figure
  on the "Convert" KPI card, a mode-specific show-the-math block, and a mode line + callout carried
  into the printable Summary.
- **Worked number (default §10 scenario, engine-default tFuture 0.24):** taxable
  advantage **−$1,684** vs. IRA advantage **−$16,531** for the *same* scenario — IRA-pays is strictly
  worse, as required. (Raise tFuture to 0.30 and both modes go positive — taxable **+$21,972**, IRA
  **+$7,125** — but IRA-pays stays the lower of the two; the self-test asserts `iraAdvantage <
  taxableAdvantage`, the ordering property, which holds at 0.30.) Self-tests assert the C_net reduction,
  the no-side-fund noConvert formula, and the advantage-ordering property.

### F5 — marginal-rate-at-bracket-floor display (cosmetic) — FIXED
- Per ARIA's original audit F5: income sitting exactly on a bracket floor reported the *upper*
  bracket's rate (the engine's `[min, max)` convention) even though the last taxed dollar was the
  lower rate. Added a **display-only** `displayMarginalRate(incomeC, brackets)` that, at an exact
  boundary, returns the rate the income tops out at (the lower bracket whose `max == income`).
- Wired to the two federal display sites only: the Step 2 "Current marginal bracket" figure
  (`mRateBefore`, a pure display field) and the Step 4 "Running marginal rate (after)" card.
- **Tax math untouched:** the engine's `marginalRate` (used for headroom and state-marginal display)
  keeps its convention. Verified `tax(100800)` MFJ = **$11,600** unchanged; `displayMarginalRate`
  reports **12%** at the $100,800 floor while `marginalRate` still reports 22%. Self-tests pin all four.

### Config integration (ARIA closeout addendum) — APPLIED
- **MFS IRMAA 394000 → 391000 + UN-GATED.** Both the tier1 `magiMax` and tier2 `magiMin` now $391,000
  (ARIA confirmed to Federal Register 2025-20251). Removed `pending:true`, the in-engine "MFS IRMAA
  tiering is PENDING" warning, the `irmaaTier` `pending: fs==='mfs'` flag (now `false`), the Step 1
  MFS-pending warnbox, the Step 8 summary MFS-pending warnbox, and the cost-table "(MFS pending)" tag.
  MFS now ships fully-supported. Surcharge dollars confirmed unchanged (mid +$446.30 B/$83.30 D, top
  +$487.00 B/$91.00 D).
- **IRMAA Part B/D dollars:** no value change. Updated config-source comments to reflect direct
  federal-primary confirmation (Fed. Reg. 2025-20251 for Part B Single/MFJ/MFS; Part D via .gov SHIP).
  Removed the "one-step-removed / pending" caveat language for IRMAA.
- **Oregon:** `standardDeduction` → `{single:2910, mfj:5820, hoh:5820, mfs:2910}` (DOR 150-206-436,
  2026 primary). `retirementSubtraction:0` kept (Roth conversion = OR ordinary income). **Kept
  `verify:true` and the 2025 placeholder bracket boundaries** — DOR has not published 2026 indexed
  charts (404 as of 2026-06-16). Updated the OR `name`/label, dropdown option, config comments, and
  the Summary assumption line to be precise (2026 std ded final; boundaries 2025 placeholders pending
  ~Dec 2025/Jan 2026). Confirmed HoH mirrors MFJ (Chart J) and MFS mirrors Single (Chart S).

### Test results — Final build
`node --check engine.js`: clean. `node engine.js`: **38 passed, 0 failed** (was 22). In-page suite
(run in a live browser preview): **38 passed, 0 failed**, no console errors. **§10 worked example
unchanged** (C=$148,600, blended 28.19%). Engine+self-test region re-confirmed **byte-identical**
between `index.html` and `engine.js` after regeneration. The N2 toggle, MFS un-gating, and OR label
were exercised live in-browser. No prior-audited behavior regressed (B1/B2/B3, AGI/MAGI, SS-torpedo,
NIIT, IRMAA/ACA cliffs, sensitivity band, disclosures all green).

**Pre-deploy NIT cleanup (2026-06-16):** Two non-blocking final-verification NITs cleaned before the
public deploy — RHETT's IRA-mode break-even tooltip (now mode-aware: side-fund wording in taxable mode,
future-rate-vs-blended wording in IRA mode) and ARIA's prose mislabel (the −$1,684/−$16,531 figures are
the tFut 0.24 case, not 0.30). Wording/docs only; no engine, math, or config change; self-tests still
38/38 green and §10 unchanged.

### Tracked items — status
All previously-tracked items are now **CLOSED** except one: **Oregon bracket-boundary indexing**
remains `verify:true` because the external data (DOR 2026 indexed OR-40 charts) is not yet published
(404 as of 2026-06-16; expected ~Dec 2025/Jan 2026). This is an external-data dependency, not a code
gap; the in-app and printable "VERIFY 2026" caveat stays until DOR posts the charts. Re-pull then.
