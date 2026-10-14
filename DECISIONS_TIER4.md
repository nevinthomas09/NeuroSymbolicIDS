# DECISIONS_TIER4.md — committed before any Tier 4 run

Committed on: ____________ (fill in the date of the commit). No Tier 4 experiment had been run when this
file was committed. Once committed, the thresholds below are those in `v2/tier4_verdicts.py` and may not
change after results are seen. Amendments go below the line, dated, and say whether they were written
before or after the run they affect.

Protocol unchanged from Tiers 1–3: corrected CIC-IDS2017, category-level leave-one-out, host-disjoint
training (the held-out category's source hosts removed from training and validation), seeds
42 / 7 / 13 / 101 / 2024, threshold at the 99th percentile of validation benign.
**Primary metric:** incident-level macro recall at 1% FPR (`tier2_verdicts.incident_table`).
**Shared constants** (from DECISIONS_TIER2.md): MIN_GAIN = 0.05, MIN_SEEDS = 4 of 5,
DROP = 0.05, FPR_GUARD = 1.5% (both systems' mean achieved FPR, else "invalid operating point").

## R0 — reproduction (gate)

Re-run systems must reproduce Tiers 1–3 to three decimals: reference MLP 0.004, LTN weight 1 0.353,
weight 5 0.443, weight 10 0.440, augmentation (weight 1) 0.445, rules v1c 0.258, GBM-flow (balanced)
0.269 (incident level); GBM-flow+window 0.197 (flow level). If any differs, the Tier 4 verdicts are
not valid and the difference is investigated before anything is reported.

## H8 — logic versus data at matched weight

Tier 3 compared the LTN at weights 5 and 10 against augmentation at weight 1 only. H8 runs
augmentation (`--mode augment --axiom-weight w`) at w = 5 and 10.
For each w, d = LTN_w − augmentation_w per seed:
- **logic better** if mean d ≥ 0.05 and d > 0 in ≥ 4/5 seeds;
- **augmentation better** if mean d ≤ −0.05 and d < 0 in ≥ 4/5 seeds;
- otherwise **no detectable difference**.

## H9 — benign flows inside the rule-region points

`S_R` contains every training flow with calibrated rule truth ≥ 0.99, about 1% of benign training flows
per rule, which the axiom (or augmentation) labels Attack. H9 removes the benign ones
(`--no-benign-rule-points`; attack flows and the 5,000 synthetic points per rule stay) for the committed
LTN (weight 1) and for augmentation (weight 1). The random-number sequence is unchanged; because `S_R` is
smaller, the points drawn from it in each batch differ.
d = without-benign − with-benign per seed:
- **benign rule points needed** if mean d ≤ −0.05 and d < 0 in ≥ 4/5 seeds;
- **benign rule points harmful** if mean d ≥ 0.05 and d > 0 in ≥ 4/5 seeds;
- otherwise **robust (no detectable difference)**.

## H10 — the same knowledge in a gradient-boosted model

GBM-flow+window, host-disjoint, with the same `S_R` appended as Attack-labelled training rows; their
total weight equals the total weight of the original training rows, split equally across the five
rules (the GBM analogue of the augmentation loss at weight 1). Everything else as
`baseline_gbm --views flow window --host-disjoint` (unbalanced weights, same benign draw).
Comparator: GBM-flow+window host-disjoint. Decision rule: **H1 criteria** — mean paired gain ≥ 0.05,
positive in ≥ 4/5 seeds, no category falling by more than 0.05, FPR guard.
**SUPPORTED** means rule knowledge helps a non-neural supervised detector too.

## No further variants

H8–H10 are run once, at the settings above. If a hypothesis is not supported, no other weights,
rule-point sets or model settings are tried for the paper. Any later variant would be a new, separately
committed test, reported as such.

## F6 — paired incident-level intervals (reporting rule, no decision)

For each knowledge model (LTN weights 1/5/10, augmentation weights 1/5/10, both H9 variants) against
rules alone, the reference MLP and GBM-flow (balanced), and for LTN_w against augmentation_w and
GBM+rules against GBM-flow+window: per-incident recall (mean over seeds), paired difference over the
18 incidents, 95% percentile bootstrap (B = 10,000, generator seed 0). "Distinguishable" iff the interval
excludes 0. Reported as stated; no multiplicity correction, no verdict attached.

**Primary comparisons** (the only intervals the paper interprets): (i) augmentation weight 1 vs. rules
alone; (ii) LTN weight 1 vs. rules alone; (iii) GBM+rules vs. GBM-flow+window. All other intervals are
reported in full as exploratory.

**Stated limitation:** the bootstrap treats incidents as exchangeable, but 12 of the 18 incidents share
one source host, so the intervals are optimistic; the paper says so wherever they are used.

## F7 — 0.1% operating point (reporting rule, no decision)

Macro recall and achieved FPR at the nominal 0.1% threshold for every system in this run.

## Consequences for the paper (fixed in advance)

- **R0 not reproduced:** nothing from Tier 4 is reported until the difference is explained.
- **H8:** "no detectable difference" at both weights → the paper states that logical and data-based
  injection are indistinguishable at matched weight. Either side better at either weight → the paper
  reports which, at which weight, and drops the "indistinguishable" claim.
- **H9:** "robust" for both models → the paper states that removing the benign rule-region flows does not
  change the result. "Needed" for either → the paper states that the knowledge result depends on
  benign flows labelled Attack, in the abstract as well as the body. "Harmful" → reported as found.
- **H10:** supported → the paper reports that rule knowledge also improves a non-collapsed supervised
  detector. Not supported → the paper states that the knowledge rescued a collapsed neural detector but
  did not improve gradient boosting, in the abstract as well as the body.
- **F6:** only the three primary comparisons are interpreted, with the clustering limitation stated.

---
## Amendments

(none)
