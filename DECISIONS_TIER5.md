# DECISIONS_TIER5.md — committed before any Tier 5 run

Committed on: ____________ (fill in the date of the commit). No Tier 5 experiment had been run when this file was
committed. The thresholds below are those in `v2/tier5_verdicts.py` and may not change after results are seen.
Amendments go below the line, dated, and say whether they were written before or after the run they affect.

## Question

Do the knowledge-guided neural detectors (Tiers 2–4) transfer to a different network, CSE-CIC-IDS2018?
Tier 1 showed that the untrained rules transfer only when grounded on the target network's benign traffic
(0.480 grounded on 2018 vs 0.013 grounded on 2017). The knowledge-guided models build their rule-region points
from 2017 traffic, so the same limitation may apply; H13 tests re-grounding them on 2018 benign traffic.

## Protocol (as `transfer2018.py`, scenario B)

Train on corrected CIC-IDS2017 **without** category C (5-tuple split, all hosts); test on CSE-CIC-IDS2018's C.
Thresholds: 99th percentile of the 2018 benign **calibration** half. Metrics: the untouched 2018 benign **test** half
plus all 2018 flows of C. Seeds 42 / 7 / 13 / 101 / 2024. Nothing is tuned on 2018.
**Primary metric:** 2018 incident-level macro recall at 1% FPR (incident = source host × attack label; per seed,
mean over categories of the mean over that category's incidents).
**Shared constants** (DECISIONS_TIER2.md): MIN_GAIN = 0.05, MIN_SEEDS = 4 of 5, DROP = 0.05, FPR_GUARD = 1.5%.

All systems use the committed Tier 2–4 code unchanged (MLP, losses, benign cap, seeds). Per (seed, C) the data,
scaler and rule-region points are built once and shared, so systems differ only in their loss:
`ref` (cross-entropy), `ltn_w1` (LTN rule axioms, weight 1), `ltn_w5` (weight 5), `aug_w1` (rule-derived data,
weight 1), and the locally grounded `ltn_w1_local`, `aug_w1_local`.
- **2017-grounded rule-region points:** 2017 training flows with calibrated rule truth ≥ 0.99 (any label) plus
  5,000 synthetic points per rule from 2017 training benign — exactly as Tiers 2–4.
- **Locally grounded rule-region points:** 5,000 synthetic points per rule built from a seeded sample (≤ 500,000) of
  2018 benign **calibration** flows, with 2018 calibration percentiles defining each rule's extreme region.
  Label-free, and the same data the thresholds already use.
- **Comparator:** rules v1c grounded on the 2018 calibration half, computed as in `transfer2018.py`.

## R0 — reproduction (gate)

The comparator must reproduce Tier 1 to three decimals: flow-level macro 0.480, incident-level macro 0.415.
Otherwise the Tier 5 verdicts are not valid and the difference is investigated before anything is reported.

## Hypotheses (H1 criteria: mean paired gain ≥ 0.05, positive in ≥ 4/5 seeds, no category falling by more than
0.05 at incident level, FPR guard on both systems)

- **H11:** `ltn_w1` beats `ref` on 2018 (2017-grounded LTN knowledge transfers).
- **H12:** `aug_w1` beats `ref` on 2018 (2017-grounded rule-derived data transfers).
- **H13a:** `ltn_w1_local` beats `ltn_w1` (re-grounding the LTN's knowledge on 2018 benign helps).
- **H13b:** `aug_w1_local` beats `aug_w1` (re-grounding the rule-derived data on 2018 benign helps).

Reported without a decision: `ltn_w5` vs `ref`; all systems per category, flow and incident level; achieved FPR;
recall and achieved FPR at the nominal 0.1% threshold.

## Paired intervals (reporting rule, no decision)

Per-incident recall (mean over seeds) over the 2018 incidents; paired difference; 95% percentile bootstrap
(B = 10,000, generator seed 0). **Primary comparisons**, the only ones interpreted: (i) `ltn_w1` − `ref`;
(ii) `aug_w1` − `ref`; (iii) `ltn_w1_local` − rules; (iv) `aug_w1_local` − rules. Others are exploratory.
Stated limitation: incidents are treated as exchangeable although several share a source host.

## No further variants

H11–H13 are run once, at the settings above. If a hypothesis is not supported, no other weights, grounding
schemes or model settings are tried for the paper. A later variant would be a new, separately committed test.

## Consequences for the paper (fixed in advance)

- **R0 not reproduced:** nothing from Tier 5 is reported until the difference is explained.
- **H11 and H12 both not supported:** the paper states that knowledge injected from the training network did not
  transfer to a new network, in the abstract as well as the body.
- **H11 or H12 supported:** the paper reports cross-network benefit for that form of injection.
- **H13a or H13b supported:** the paper reports that re-grounding the injected knowledge on the new network's benign
  traffic (no attack labels) restores transfer — the neuro-symbolic analogue of the Tier 1 local-grounding result.
- **H13a and H13b both not supported:** the paper states that local re-grounding did not help the knowledge-guided
  models, unlike the untrained rules.
- In every case the paper places the knowledge-guided models next to the locally grounded rules (0.480, from this
  run) and the locally retrained autoencoder (0.803, Tier 2); it calls a locally grounded knowledge model better
  than the rules only if its primary interval excludes zero, and the autoencoder comparison is descriptive only.

---
## Amendments

(none)
