# Roth Conversion Educational Calculator

A single-file, self-contained web tool that teaches what a Roth conversion is and walks a user,
step by step, through estimating the **true** cost of converting tax-deferred dollars to Roth in
tax year **2026** — including the secondary threshold effects (IRMAA, Social Security "tax
torpedo," NIIT, LTCG stacking, the ACA 400% FPL cliff) that change the real marginal cost.

## How to open it

Double-click **`index.html`**. It runs in any modern browser on Windows.

- **No build step, no npm, no server, no internet.** Everything (HTML + CSS + vanilla JS) is in the
  one file. It works fully offline.
- **Nothing is stored or transmitted.** All computation is client-side; there are no network calls,
  no analytics, no storage.
- Export your result with the **Print / Save as PDF** button on the Summary step (uses the browser's
  native print, which is fully client-side). The remaining ship-flagged caveat is carried into this
  printed/exported output, not just the on-screen wizard: the **Oregon bracket-boundary "VERIFY 2026"**
  notice appears whenever Oregon is the selected state (its 2026 standard deduction is final, but the
  indexed bracket boundaries are still 2025 placeholders) — so a reader who keeps only the exported PDF
  still sees the disclosure. *(As of the 2026-06-16 final build, MFS IRMAA is fully verified and ships
  un-gated — the former "MFS IRMAA pending" notice is gone.)*

## What's in this folder (and what gets published)

| File | Published? | Notes |
|---|---|---|
| `index.html` | **YES — this is the tool** | Single self-contained file; the only thing a user opens. Does not reference any other file. |
| `README.md` / `BUILD_REPORT_DEVS.md` | Optional (docs) | Reference for reviewers; ship if publishing the folder. |
| `engine.js` | **NO — dev-only test artifact** | Headless Node copy of the engine + §12 self-tests, used only to run the suite via `node engine.js`. It is **byte-equivalent** to the engine inside `index.html` (regenerated from the final patched file 2026-06-16) and is marked DO-NOT-PUBLISH in its header. `index.html` does **not** load it. Exclude it from any published bundle; if you publish the folder, leave the header banner intact so the exclusion is unambiguous. |

To run the engine self-tests headless: `node engine.js` → expect **38 passed, 0 failed**.

## What it does (wizard, 9 steps)

0. Intro & disclaimer · 1. About you (filing status, age, state, Medicare/ACA flags) ·
2. Pre-conversion income · 3. Retirement balances · 4. Conversion amount (specific amount **or**
fill-the-bracket) · 5. Cost of converting this year (itemized, with blended effective marginal
rate) · 6. Should you? (after-tax value convert vs. don't, break-even, **pay-tax-from toggle**
(taxable vs. IRA), **sensitivity band** showing the verdict across ±5 points of your future-rate
guess, RMD reduction) · 7. Multi-year plan · 8. Summary & printable export.

### Step 6 inputs

- **Horizon (years)** and **Annual growth rate** — the projection window and pre-tax growth.
- **Expected FUTURE marginal rate at withdrawal — ALL-IN** — the rate you expect to pay later.
  Enter it **all-in (federal + future state)**: the cost side already includes your current state
  tax, so the future rate must include state too or the comparison is apples-to-oranges.
- **Annual tax drag on taxable growth** (default `0.15`) — the fraction of each year's taxable-account
  return lost to tax (your LTCG + NIIT + state on annual gains). The "don't convert" side fund (the
  tax dollars you'd otherwise have paid) lives in a **taxable** account, so per spec §7.11 it grows at
  an after-tax rate `g·(1 − drag)`, not the tax-free Roth rate. Setting it to `0` treats the side fund
  as tax-free (overstates not-converting). This after-tax asymmetry is what produces a genuine
  **break-even year** (tax-free Roth growth eventually overtakes the taxed side fund).
- **Where does the conversion tax come from?** (default *taxable*) — a toggle for the two cases per
  spec §7.11. **Taxable (outside) funds** is the efficient case (the full conversion lands in the
  Roth; the no-convert side fund applies). **The IRA / converted funds** is materially less efficient:
  the tax is withheld from the conversion, so only `C_net = C·(1 − blended)` reaches the Roth and there
  is **no side fund** (no outside dollars were spent), making `noConvert = C·(1+g)ⁿ·(1 − tFuture)`. The
  UI flags the lower efficiency prominently and the advantage is correspondingly lower (or negative).

Every computed figure has a **"show the math"** expander exposing the formula and the inputs used.
Bold glossary terms have inline hover/focus tooltips. Cliff warnings (IRMAA, ACA, SS) render
prominently.

## Configuration — swapping in a future tax year

The tool is **config-driven**. The only thing that changes year to year is the `CONFIG_2026`
object near the top of the `<script>` block, clearly delimited and commented **"REPLACE YEARLY."**
The calculation engine never hard-codes a threshold.

To produce, e.g., a 2027 version:
1. Open `index.html`, find the `CONFIG_2026` block.
2. Replace every value (federal brackets, standard deduction, LTCG brackets, NIIT thresholds, SS
   provisional thresholds, the `irmaa` object, `fpl`, `aca` schedule, RMD table, `stateConfig`)
   with the new year's verified figures. Rename to `CONFIG_2027` and update the single reference
   (`computeConversion(CONFIG_2026, ...)` → `CONFIG_2027`) — or just keep the name and swap values.
3. No engine code changes. Re-run the in-page self-tests (Summary step → "Run self-tests").

The 2026 IRMAA and ACA config objects were pasted **verbatim** from ARIA's verified supplement
(`Research Library/TAX_REFERENCE_2026_IRMAA_ACA_Supplement_ARIA.md`), then closed out against ARIA's
**closeout addendum** (`Research Library/TAX_REFERENCE_2026_Closeout_Addendum_ARIA.md`): MFS IRMAA
$391k boundary + un-gating, IRMAA Part B/D primary sourcing, and the Oregon 2026 standard deduction.

## Final build — 2026-06-16

- **N2 — "pay tax from" toggle (spec §7.11):** added the IRA-pays case alongside the existing
  taxable-pays case. Taxable-pays behavior is unchanged (additive). IRA-pays reduces the Roth
  principal to `C_net = C·(1 − blended)` and removes the no-convert side fund; the lower efficiency is
  flagged in the UI and the show-the-math.
- **F5 — marginal-rate-at-bracket-floor display (cosmetic):** the "current marginal bracket" display
  now reports the rate of the last taxed dollar at an exact bracket floor (e.g. 12% at the MFJ
  $100,800 boundary, not 22%). Tax math is unchanged; the engine's `marginalRate` convention is
  untouched (a new display-only `displayMarginalRate` is used for the two display sites).
- **Config integration:** MFS IRMAA boundary **394000 → 391000** + MFS un-gated (no longer pending);
  IRMAA Part B/D sourcing upgraded to federal primary; Oregon 2026 standard deduction integrated.
- **Self-tests:** 22 → **38 passed, 0 failed** (`node engine.js`). §10 worked example unchanged
  (C = $148,600, blended 28.19%). engine.js regenerated byte-equivalent from the final index.html.
- **One remaining flagged item:** Oregon **bracket boundaries** stay `verify:true` (2025 placeholders)
  until DOR publishes its 2026 indexed OR-40 charts — external data not yet available, not a code gap.

## Verification status of each module

| Module | Status |
|---|---|
| Federal ordinary tax, std deduction, senior add'l, OBBBA | **Verified** (TAX_REFERENCE_2026) |
| LTCG stacking | **Verified** |
| NIIT | **Verified** |
| Social Security taxation (two-threshold worksheet) | **Verified** |
| IRMAA — Single / MFJ / HoH | **Verified — federal primary.** Part B Single/MFJ confirmed to Federal Register notice 2025-20251; Part D via authoritative .gov (Illinois SHIP reproducing CMS). |
| IRMAA — **MFS** | **Verified — federal primary, un-gated (2026-06-16 final build).** Middle boundary confirmed to **$391,000** (was a pending ~$394k placeholder; ARIA confirmed to the Federal Register). MFS now ships as a fully-supported filing status with no pending notice. |
| ACA premium tax credit + 400% FPL cliff | **Verified** (IRS Rev. Proc. 2025-25, HHS 2025 FPL). Defaults to 48-contiguous-states FPL; AK/HI not wired into the UI. |
| RMD projection | **Verified** structure (SECURE 2.0 age 73; Uniform Lifetime Table subset, ages 73–100). |
| **Oregon state tax** | **Partially closed (2026-06-16).** 2026 **standard deduction** ($2,910 S / $5,820 M) and `retirementSubtraction:0` are **confirmed-primary** (DOR 150-206-436). Rates and the $125k/$250k statutory top boundaries are final. The **lower bracket boundaries remain 2025 placeholders** — Oregon DOR has not yet published its 2026 indexed OR-40 charts (404 as of 2026-06-16; expected ~Dec 2025/Jan 2026). `verify:true` and the in-app/print "VERIFY 2026" caveat stay until those publish. (Oregon uses two charts: single & MFS share Chart S; MFJ, HoH, QSS share Chart J.) |
| No-income-tax state | **Verified** (empty brackets → state tax 0; proves the state layer is config-driven). |

## Derives from

- **Spec:** `Team Inbox/Handoff_2026-06-15/ROTH_CONVERSION_CALCULATOR_HANDOFF.md`
- **Base config:** `Team Inbox/Handoff_2026-06-15/TAX_REFERENCE_2026.md`
- **IRMAA + ACA supplement:** `Research Library/TAX_REFERENCE_2026_IRMAA_ACA_Supplement_ARIA.md`

## Not tax advice

Educational estimate only. Confirm every figure with a CPA before acting. See `BUILD_REPORT_DEVS.md`
for engineering detail, the spec-function-to-code map, test results, and known limitations.
