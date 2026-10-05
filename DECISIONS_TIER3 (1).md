# Tier 3 — decision rules (fixed before any Tier 3 experiment is run)

_Revision note (before first commit, before any Tier 3 experiment on real data): F3(a) originally required exact score
reproduction. Testing the code on synthetic data showed that LightGBM's quantile binning changes scores slightly under
1e-6 feature perturbations, so exact equality is not a valid test of the CORAL implementation; F3(a) now tests the
transform and the metrics with stated tolerances._

Responds to the second internal review. Commit this file before running Tier 3; later changes must be dated amendments.
Common protocol, seeds (42, 7, 13, 101, 2024), features, 1% FPR operating point, incident definition, category-drop
definition (> 0.05) and validity guard (achieved FPR <= 1.5%) are unchanged from DECISIONS_TIER2.md.

## H6 — Is the host-disjoint collapse explained by attack-data loss rather than the attacker shortcut?

Problem: in CIC-IDS2017 one attacker host produced DoS, BruteForce, Web and PortScan, so host-disjoint training removes
both the host and most training attacks.

Control ("standard, size-matched"): standard (non-host-disjoint) folds in which the training attack flows are reduced by
seeded uniform random sampling to exactly the number of training attack flows of the host-disjoint fold with the same
held-out category and seed; validation attack flows are reduced by the same fraction; benign flows are unchanged.
Models: GBM-flow (balanced) and the reference MLP (cross-entropy only), both exactly as in Tiers 1-2.
Reported: training attack counts per fold (standard, host-disjoint, size-matched) and per category.

Metric: incident-level macro recall at 1% FPR (mean over seeds). For each model, with SM = size-matched, HD = host-disjoint:
- **"Shortcut interpretation supported"** if SM - HD >= 0.10 AND SM > HD in >= 4 of 5 seeds;
- **"Data-quantity interpretation supported"** if SM - HD <= 0.05;
- otherwise **"inconclusive"**.
The verdicts for the two models are reported separately. If they disagree, the paper reports both and does not claim
either interpretation in general.

## H7 — Does the logic add value beyond rule-derived data augmentation?

Augmentation model: the same MLP, data, epochs and seeds as the rule-axiom LTN model (H1); the LTN satisfiability term
is replaced by binary cross-entropy that labels the same rule-region points as Attack. In every batch, the same number of
points per rule (1024) is drawn from the same sets S_R (real rule-extreme training flows plus the same 5,000 synthetic
flows per rule); the loss is CE + 1.0 * mean over rules of BCE(points, Attack). Nothing else differs.

Primary comparison (host-disjoint, incident-level macro recall at 1% FPR):
- **"Logic adds value beyond augmentation"** if LTN - augmentation >= 0.05 AND LTN > augmentation in >= 4 of 5 seeds
  AND no category of the LTN model falls more than 0.05 below the augmentation model;
- **"Logic and augmentation equivalent"** if |LTN - augmentation| < 0.05;
- **"Augmentation better"** if augmentation - LTN >= 0.05 AND augmentation > LTN in >= 4 of 5 seeds;
- otherwise **"inconclusive"**.
Also reported, without a decision: augmentation vs. reference MLP (H1 criteria), and both LTN and augmentation vs. the
untrained rules (rules v1c), per category and per incident.

Consequence fixed in advance: unless H7 returns "logic adds value beyond augmentation", the paper describes the positive
result as "behavioural knowledge injected into training" and does not attribute it to logical reasoning.

## Fixes and checks (no hypotheses; reported as described)

F1. 2018 fusion with rules v1c (the pre-registered rule set) in place of v2; labelled exploratory in the paper.
F2. Size-balanced host-level benign split: benign source hosts are shuffled with the seed and assigned greedily to the
    split (train/val/test) with the largest remaining deficit against 70/15/15 of benign flows. Reported: achieved test
    share per seed and achieved FPR for GBM-flow, GBM-flow+win, autoencoder and fused detector. The original host split is
    reported as superseded.
F3. CORAL sanity checks: (a) identity check — CORAL with source and target statistics taken from the same 2017 sample
    must be an identity transform (max feature change < 1e-5) and must reproduce the unaligned model's held-out recall
    within 0.02 and benign FPR within 0.002 (2018, first held-out category, seed 42); (b) diagnostics on 2018 — number of distinct scores on 2018 benign
    and attack flows, and the share of flows at the threshold value. If (a) fails, the H5 result is withdrawn as an
    implementation error and rerun; if (a) passes, H5 stands and (b) is reported as the explanation.
F4. Pooled bootstrap intervals only: incident intervals are reported for pooled macro recall, not for categories with
    fewer than 5 incidents.
F5. Bot rise under host-disjoint GBM: report the benign training flows removed with the Bot hosts and the Bot recall of
    GBM-flow with only those benign flows removed (attacks unchanged), to test the stated explanation.

## Amendment 1 (recorded after the Tier 3 results, before any sensitivity run)

Axiom-weight sensitivity check, requested in review: the H7 comparison used one LTN configuration (axiom weight 1,
p = 2). The rule-axiom LTN model is rerun, host-disjoint, five seeds, at axiom weights 5 and 10 (p = 2, everything else
unchanged); weight 1 is rerun as a reproduction check. This is a sensitivity analysis without a decision rule.
Reporting fixed in advance:
- H7 keeps its committed verdict (weight 1).
- All three weights are reported next to the augmentation model, per seed and per category.
- If at any weight the LTN model's incident-level macro is within 0.05 of the augmentation model or above it (the H7
  equivalence margin), the paper states that the H7 result is sensitive to the axiom weight and does not claim that the
  logical formulation is less effective than rule-derived data.
- If at every weight the LTN model is more than 0.05 below the augmentation model, the paper states that the result held
  across the weights tested (1, 5, 10).
