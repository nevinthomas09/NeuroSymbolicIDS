# Tier 2 — decision rules (fixed before any Tier 2 experiment is run)

_Revision note: H3 was revised once before this file was first committed, after reading the published
reference values; no Tier 2 experiment had been run at that point._

Commit this file to the public (anonymised) repository **before** implementing Tier 2.
The commit timestamp is the evidence that these rules preceded the results.
Any later change must be a new, dated section below; the original rules stay unchanged.

## Common protocol (inherited from Tier 1, unchanged)

- Data: corrected CIC-IDS2017 (Engelen et al.) and corrected CSE-CIC-IDS2018 (Liu et al.), Tier-1 cleaning.
- Seeds: 42, 7, 13, 101, 2024. Features: 79 flow + 22 causal window features (fingerprint features excluded).
- Operating point: threshold at the 99th percentile of validation-benign scores (nominal 1% FPR).
- **Primary metric:** macro over the five held-out categories of **incident-level** recall at 1% FPR,
  under **host-disjoint** category-level leave-one-out on CIC-IDS2017.
- Secondary metrics: flow-level macro recall (standard and host-disjoint); 2018 scenario B macro (flow and incident);
  achieved FPR.
- "Paired improvement" = the per-seed difference (new minus reference) of the primary metric.
- A "category drop" = any category's incident-level recall falling by more than 0.05 (mean over seeds).
- Validity guard: a result counts only if achieved test-benign FPR is <= 1.5% (mean over seeds).
  If the guard fails, the outcome is reported as "invalid operating point", not as support or rejection.

## H1 — Knowledge integrated into training (rule-axiom LTN)

Model: the same MLP as the reference, trained with cross-entropy plus an LTN satisfiability loss over axioms
derived from the frozen rule set v1c (no rule is added or edited):
for each rule R, `forall x: R(x) -> Attack(x)` evaluated on rule-extreme points (rule truth >= 0.99 against
training-benign percentiles), including synthetic points obtained by perturbing benign windows into the rule's
region; plus `forall x in benign: not Attack(x)`. Axiom weight and LTN aggregator settings are fixed in advance:
weight 1.0, p = 2 (as in Bizzarri et al. 2024). No tuning on held-out categories.

Reference: the identical MLP trained with cross-entropy only (same seeds, data, epochs).

**H1 is supported** only if ALL hold:
1. Primary metric improves by >= 0.05 on average AND the paired improvement is positive in >= 4 of 5 seeds;
2. No category drop;
3. Validity guard holds for both models.

Additionally reported, not part of the decision: 2018 scenario B with the same comparison; and whether adding the
LTN model as a fusion channel changes the fused detector (same drop tolerance 0.005 as Tier 1).

## H2 — Label axioms (Bizzarri et al. hybrid loss) under our protocol

Model: the same MLP trained with cross-entropy plus the satisfiability loss of the axioms
`forall x_b: P(x_b, benign)` and `forall x_a: not P(x_a, benign)` (weight 1.0, p = 2), as in Bizzarri et al. 2024,
applied to flow + window features (their payload-byte pipeline is not reproduced).

**H2 is supported** under exactly the H1 criteria (same reference model).
Expectation stated in advance: no improvement, because these axioms restate the training labels.
The outcome is reported either way.

## H3 — Reproduction of Hindy et al. (2020) and measured protocol inflation

Their code and their bidirectional-flow CICIDS2017 files are public
(github.com/AbertayMachineLearningGroup/zero-day-detection); no other data is needed.
Published reference values (text of the paper, Electronics 9(10):1684):
- input features after correlation filtering (threshold 0.9): 18;
- benign specificity at MSE thresholds 0.15 / 0.10 / 0.05: 95.19% / 90.47% / 81.13%;
- best attack detection: DoS GoldenEye 90.01%, DoS Hulk 98.43%, PortScan 98.47%, DDoS 99.67%
  (compared with our value at the best of thresholds 0.05 / 0.10 / 0.15, as "reaches" in the paper).

Step A (reproduction, their protocol): their preprocessing (drop IPs and first column, correlation filter 0.9 fitted
on Monday benign, StandardScaler fitted on Monday benign), their split (75/25 random row split of Monday benign,
random_state=1), their model (linear dense layers 15-9-15 plus linear output, L2 1e-4 on kernels, Adadelta with
learning rate 1.0, MSE, batch 1024, 50 epochs), their evaluation (fraction of each attack file with MSE above each
threshold). Re-implemented in PyTorch because their Keras version is obsolete; three seeds.
**Reproduction succeeds** if the feature count is 18 AND at least 6 of the 7 published values above are matched
within 5 percentage points (mean over seeds). Otherwise it is reported as failed, with all differences.

Step A2 (threshold choice only): same data, model and split as Step A, but the threshold is the 99th percentile of
the benign validation scores (1% FPR). Reported: per-class recall, next to Step A.

Step B (our protocol): the same architecture and training procedure on our 101 features, trained on our benign
training split, evaluated with the common protocol (corrected labels, category-level leave-one-out, standard and
host-disjoint, 1% FPR from validation benign, incident level).

Reported, no pass/fail: the per-class change from A to A2 (threshold choice) and from A2 to B (data, labels, held-out
categories, hosts); and Step B next to our autoencoder.

## H4 — Standard anomaly detectors

Isolation Forest and One-Class SVM (OC-SVM on a seeded 100k benign subsample), flow + window features, trained on
benign only, default hyperparameters (scikit-learn), evaluated with the common protocol. Reported without a
hypothesis, as baselines for the label-free autoencoder.

## H5 — Domain adaptation for the supervised detector (2018 transfer)

Model: GBM-flow+win trained on 2017 features aligned to 2018 by CORAL (covariance alignment of 2017 training
features to the 2018 benign calibration half; no 2018 attack data used).

**H5 is supported** if 2018 scenario-B flow macro recall improves by >= 0.05 over the unaligned GBM-flow+win
AND the paired improvement is positive in >= 4 of 5 seeds, with the validity guard holding.

## What happens with the outcomes

- H1 supported: "neuro-symbolic" returns to the paper's framing, scoped to knowledge-integrated training.
- H1 not supported: reported as a negative result; the paper keeps its current framing.
- H2: reported as the evaluation of the base paper's method under leakage control.
- H3: the paper reports the reproduction and the measured inflation alongside our results.
- H4, H5: baselines in the main results and transfer tables.
- No hypothesis, threshold, axiom, rule or metric above is changed after any Tier 2 result is seen.

## Amendment 1 (recorded before any Tier 2 experiment was run)

1. H4, OC-SVM: scikit-learn's default nu = 0.5 is not computationally feasible at this scale (half the training
   sample becomes support vectors; scoring ~1.7M flows would take days). OC-SVM therefore uses nu = 0.01 (the value in
   Hindy et al.'s code), rbf kernel, gamma = 'scale', fitted on a seeded 20k training-benign subsample after robust
   scaling. Isolation Forest is unchanged (defaults, <= 500k training-benign flows).
2. H1, "additionally reported" items (2018 scenario B with the LTN model; the LTN model as a fusion channel) are
   deferred to a later run and are not part of this run's report. The H1 decision itself (CIC-IDS2017, host-disjoint,
   incident level) is unchanged.
3. All MLP variants (reference, H1, H2) train on at most 500k training-benign flows plus all training attack flows,
   matching the GBM's benign cap; the reference and both LTN variants use identical data, epochs and seeds.
