# tenor: design document

Status: draft, written before any code. Figures in [brackets] are placeholders to be filled from the data before the split is frozen. Everything else is fixed; changes after this point are logged in the changelog at the bottom.

## 1. Use case and optimization metric

The feature scores the formality of a sentence from a learner's writing and flags it when it reads as informal against a formal target register. Formality is scored as a gradient; the flag is a comparison against the target, which in this writeup is fixed at formal. The product action is binary (flag or don't), so the headline metric is defined on the binary decision.

**Gold labels.** A sentence is *informal* if its mean annotator rating is below 0 on the Pavlick and Tetreault scale (-3 to 3), *formal* otherwise. Zero is the scale's neutral anchor, which annotators were instructed to treat as neither formal nor informal.

On the mirror's train split (n = 9,274), 50.4% of sentences have a mean rating below 0. The distribution is unimodal with its peak in [0.6, 1.2) and a broad plateau from -0.6 to 1.2 that holds 47% of sentences, so 0 sits inside the dense middle rather than in a trough between two populations. Zero is kept as the cut because it is the scale's neutral anchor, not because the data separate there. Sentences with a mean rating of exactly 0 (n = [x]) are labeled formal under the "below 0" rule; they fall inside the dead zone and are excluded from the secondary number. The test split (n = 2,000) has a slightly higher mean (-0.02 against -0.04) and a below-0 share near 46%. [Per-genre sentence pending: expected ordering news > blog > email > answers on mean rating, to be confirmed.]

**Dead zone.** Sentences with mean rating in [-0.5, 0.5) are kept in the main evaluation. A second set of numbers is reported with that band excluded, so a reader can see how much of each system's error rate is label noise near the cut. Roughly 25% of sentences fall in this band (estimated from 0.6-wide histogram bins; exact count [x] once the split is loaded), so the dead-zone-excluded results describe a materially easier subset, not a minor adjustment, and are reported second.

**Error costs.** A false flag tells a learner that an acceptable sentence is wrong. The product assumption is that once this happens the learner discounts later flags, so one false flag degrades the whole feature. A miss leaves one informal sentence uncorrected. False flags are the worse error. This is a stated product assumption, not a measured result.

**Optimization metric.** Precision on the informal class.

**Monitoring metrics**, reported but not optimized:

- Recall on the informal class, with an initial floor of 0.70. A system below the floor is disqualified regardless of precision. The floor is revisited after the feature model baseline is scored.
- Spearman correlation between system score and mean annotator rating, to confirm the gradient model is sensible underneath the threshold.
- Per-genre precision and recall.

All three systems are evaluated on the same held-out set at the same cut. For the hybrid, the routing band (the range of feature-model probabilities that gets sent to the LLM) is swept on the dev set, and precision, recall, and API calls per 1,000 sentences are reported at each band width.

## 2. Evaluation set and sample size

**Data.** Pavlick and Tetreault (2016), sentence-level formality annotations in four genres: answers (Yahoo Answers), blog, email (Jeb Bush gubernatorial email archive), news. News and blog sentences originate in Lahiri (2015). Accessed via the Hugging Face mirror `osyvokon/pavlick-formality-scores`, which detokenizes the answers and email genres. License as tagged on the mirror: CC-BY-3.0, to be confirmed against the original release. Both papers are cited.

The mirror ships a train split (9,274 sentences: answers 4,083, news 2,287, blog 1,476, email 1,428) and a test split (2,000 sentences: answers 894, news 488, blog 345, email 273). Genre proportions match across the two. Total 11,274.

**Split.** The mirror's test split is used as the frozen test set unchanged; it is the size this document targets, and using the published split removes any question of whether test was cut favorably. Dev is drawn once from the mirror's train split with a fixed random seed, stratified by genre:

- Test: the mirror's 2,000-sentence test split. Frozen. Every result in this writeup comes from this set, scored once per final system.
- Dev: 1,500 sentences from train. Used for LLM prompt iteration and the hybrid routing-band sweep.
- Remainder: 7,774 sentences from train. Training data for the feature model.

**Sizing the test set.** The headline metric is precision on the informal class, a proportion over the sentences a system flags. The relevant sample size is the number of flagged sentences, not the number of test sentences. Target: a 95% confidence interval no wider than ±0.03 on precision. Under the normal approximation, n ≈ 1.96² × p(1-p) / h²; at expected precision 0.85 and h = 0.03 this gives about 544 flagged sentences. With an informal base rate near 0.46 on test, recall near the 0.70 floor, and precision near 0.85, roughly 38% of test sentences are flagged, so 2,000 test sentences yields about 760 flags. That leaves room for the dead-zone subset, which removes about a quarter of the set. Reported intervals come from a bootstrap over test sentences (10,000 resamples); the approximation is used only to size the set.

**Comparing systems.** Systems are compared with a paired bootstrap on the difference in precision over the same test sentences. Power depends on the number of sentences the two systems disagree on, not on the total. If two systems disagree on fewer than about 150 test sentences, the difference is reported with its interval and no claim of significance is made. The 150 figure is a rule of thumb: at that many paired disagreements and a true gap of 0.03, the bootstrap interval on the difference is roughly ±0.03 to ±0.04, which is about where a 0.03 difference becomes distinguishable from zero.

**Per-genre results.** The smallest genre, email, has 273 test sentences and will produce roughly 100 flags, a precision interval near ±0.07. Too wide to optimize against, so per-genre numbers are monitoring only.

**Leakage.** The LLM prompt and the routing band are tuned on dev, so dev numbers are optimistic for both the LLM and the hybrid. Dev numbers are not reported as results. The feature model is trained on the remainder and its threshold is not tuned, so it never sees dev; no system has seen test.

## 3. Systems

All three systems take one sentence and return a formality score and a binary label at the fixed cut. They are scored on the same 2,000 test sentences.

### 3.1 Feature model

A logistic regression over hand-built linguistic features, trained on the 7,774 remainder sentences. The linguistic content is in the features; the weights are learned.

Features, computed with spaCy or regex:

- contraction rate
- first- and second-person pronoun rate
- informal lexicon hits (slang, intensifiers such as *totally*, discourse markers such as *lol*, *btw*)
- formal lexicon hits (Latinate verbs, nominalizations)
- exclamation marks, ellipses, repeated punctuation
- all-caps tokens
- sentence length in tokens
- mean dependency depth
- passive voice present
- subordinate clause present

Lexicons are fixed before dev or test are inspected and are committed to the repo with the date. Output is a calibrated probability that the sentence is informal; label is informal at p ≥ 0.5. Coefficients are reported so each feature's contribution is inspectable.

### 3.2 LLM zero-shot

Claude Haiku [exact model string and run date pinned in the repo], temperature 0, one sentence per call. The prompt gives the -3 to 3 scale definition from Pavlick and Tetreault and asks for a single number. No examples from dev or test appear in the prompt. Label is informal if the returned score is below 0, the same cut as the gold labels. The prompt is iterated on dev only. Non-numeric responses are logged and counted as errors, not dropped.

### 3.3 Hybrid

The feature model runs on every sentence. If its probability is outside the band [0.5 - w, 0.5 + w], its label stands and no API call is made. If inside, the sentence goes to the LLM and the LLM's label stands. One system owns each decision, so every error is attributable.

The band width w is swept on dev. At each w the writeup reports precision, recall, share of sentences routed, API calls per 1,000 sentences, dollar cost per 1,000, and p50 latency. The w used on test is chosen on dev before test is scored.

### 3.4 What touches dev

The feature model never sees dev. The LLM prompt and the band width are tuned on dev. Dev numbers are not reported.

## 4. Cost and latency

For each system: API calls per 1,000 sentences, dollars per 1,000 at list price on the run date, p50 and p95 latency per sentence measured over the test run. The feature model is 0 calls; the LLM is 1,000; the hybrid is 1,000 × share routed. Hybrid latency is reported for the routed and unrouted paths separately, since a single p50 would hide the bimodal shape.

## 5. Error analysis

Done last, on test, after the headline numbers are fixed. Errors are broken down by genre and by error type (false flag vs. miss), then sampled and tagged with a short cause taxonomy written after reading the first 50 errors, not before. The taxonomy explains the metrics; it does not replace them. For the hybrid, every error is tagged with the system that owned the decision, so the writeup can say whether the band is too narrow (feature-model errors near the edge) or the LLM is the weak link inside it.

## 6. Robustness

The feature model is retrained four times, each time leaving out one genre, and scored on the held-out genre. The drop from the in-domain number is reported per genre next to the LLM's precision on that genre. Answers is the genre nearest to informal learner writing, so the answers-held-out result is the one discussed in limitations. Monitoring only; no system is selected on it.

## 7. Known limitations, stated up front

- The dataset contains no learner writing. All text is native-speaker. Learner-specific register signals (article dropping, L1 transfer, over-formal textbook vocabulary) are not represented and cannot be learned from this data. The feature model's lexicons are built for native text. Any claim about learner writing is a claim about transfer from native text, and the obvious follow-up is a small annotated learner set using the same 7-point protocol.
- Answers make up roughly half the data, so a pooled metric is mostly a Yahoo Answers metric. Stratification keeps the test set representative of the dataset, not of learner writing.
- The feature model is trained and tested on the same distribution, so its in-domain numbers overstate what it would do elsewhere. Section 6 measures how much.
- The dead-zone subset inflates every system's numbers. It is reported second, not first.

## 8. Deliverables

- This document, unchanged after the split is frozen except for the changelog.
- `README.md` written as a study writeup: question, method, results with intervals, error analysis, limitations.
- Evaluation notebook with figures: score distributions by genre, precision/recall with bootstrap intervals per system, the routing-band sweep, paired-difference intervals, leave-one-genre-out table.
- Cost and latency table.

## Changelog

- 2026-10-29: initial draft.
- 2026-10-29: filled split sizes, base rate, and dead-zone share from the mirror's column statistics; adopted the mirror's test split as the frozen test set.
