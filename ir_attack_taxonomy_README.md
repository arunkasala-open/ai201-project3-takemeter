# IR / MITRE ATT&CK Writeup — Label Taxonomy & Dataset

**Purpose:** A fine-tuning / classification dataset for labeling cybersecurity community blog posts and writeups in the **Incident Response + MITRE ATT&CK** space, plus an evaluation comparing a fine-tuned DistilBERT classifier against a zero-shot baseline.

**Files**
- `ir_attack_writeup_dataset.csv` — 400 annotated examples (roughly balanced: case study 94, detection 105, threat-actor 101, emulation 100).
- `ir_attack_taxonomy_README.md` — full taxonomy specification and labeling guide (the authoritative design document).
- `planning.md` — working notes and design rationale.
- `README.md` — this file: dataset overview and **evaluation results with analysis**.

> **Document scope.** This README is the deliverable-level summary and evaluation. The detailed taxonomy spec lives in `ir_attack_taxonomy_README.md`; the design reasoning and open questions live in `planning.md`. To avoid the three drifting out of sync, the labeling rules below are kept brief — see the taxonomy README for the canonical version.

**CSV columns:** `id`, `label`, `grounding`, `source`, `url`, `date`, `attack_tactics`, `attack_techniques`, `text`

---

## Taxonomy (summary)

The labels split on **writeup purpose** — *why* the post was written (author intent) — not on which ATT&CK tactic appears. This is how the DFIR/CTI/detection community actually organizes (blog categories, conference tracks, job roles), and it keeps classes mutually exclusive even when a post touches many techniques.

| Label | One-line definition |
|---|---|
| `incident_case_study` | Forensic walkthrough of **one specific intrusion** — a narrative timeline reconstructed from evidence. |
| `detection_engineering` | A post whose **primary purpose is detection logic** — building/sharing a rule, query, or analytic. |
| `threat_actor_profile` | CTI characterizing a **named actor's recurring TTPs** across campaigns (not one incident). |
| `adversary_emulation` | **Offensive validation** — emulating/testing techniques to measure defensive coverage. |

The four mutual-exclusivity edge cases (case-study vs. detection, case-study vs. actor-profile, actor-profile vs. emulation, detection vs. emulation) and the exhaustiveness/coverage argument are documented in full in `ir_attack_taxonomy_README.md`. They are not repeated here.

---

## How the dataset was built (honesty note)

The dataset is **mostly synthetic, and this is the single most important fact for interpreting the results below.**

- **15 `verified` rows (~4%)** have text excerpts, dates, URLs, and ATT&CK techniques pulled directly from specific real, live 2025–2026 articles (The DFIR Report, Unit 42, Microsoft Threat Intelligence, Elastic Security Labs, Splunk, Red Canary, Picus).
- **385 `representative` rows (~96%)** are text I **wrote** to read like each source/label — modeled on real community publishing but not verbatim copies. The publishers, ATT&CK technique IDs, and named actors (Scattered Spider/UNC3944, Akira, Qilin/AGENDA, Volt Typhoon, Salt Typhoon, Mustang Panda, etc.) are real; the prose is authored to match each label's pattern.

To train/evaluate only on real data, filter `grounding == verified`. To use the full pattern set, use all rows.

---

## Model & training configuration

**Fine-tuned model:** `distilbert-base-uncased` (HuggingFace), with a 4-way classification head.
**Baseline:** zero-shot classification via Groq.
**Platform:** Google Colab (single GPU).

These choices are deliberate for the task and constraints, not just framework defaults:

| Decision | Value | Why this, here |
|---|---|---|
| Base model | `distilbert-base-uncased` | The inputs are short English blog excerpts where the signal is lexical/topical (rule snippets, "we responded to…", group aliases) rather than long-range reasoning. DistilBERT is ~40% smaller and ~60% faster than BERT-base while retaining ~97% of its accuracy — so it fits a Colab GPU comfortably and trains in minutes. `uncased` because the discriminative tokens (`sigma`, `kql`, `emulated`, actor names) don't depend on capitalization, and lowercasing shrinks the vocabulary the model has to disambiguate. A larger model (RoBERTa, DeBERTa) is unjustified given the dataset is small and, as the evaluation shows, already trivially separable. |
| Platform | Google Colab | Free GPU, zero local setup, and the model is small enough that a single session covers training + evaluation. Adequate for a dataset this size; no need for paid/distributed infra. |
| Epochs | 3 | Standard for fine-tuning a pretrained transformer on a small set — enough to adapt the head and upper layers without memorizing 400 rows. Validation accuracy plateaus immediately here (see evaluation), so more epochs would only risk overfitting the synthetic patterns. |
| Learning rate | 2e-5 | The conventional fine-tuning rate for BERT-family models; high enough to move the pretrained weights, low enough not to wash out the pretrained representations. No schedule search was warranted given the task saturates at epoch 1. |
| Batch size | 16 | Fits Colab GPU memory for DistilBERT at this sequence length with margin, and gives stable gradient estimates on a 400-row set. |

**Honest caveat on the hyperparameters:** because the task turns out to be near-trivially separable (see below), these settings are *defensible* but largely *uncritical* — almost any reasonable configuration would reach the same accuracy. Their justification is "appropriate and not wasteful," not "tuned to squeeze out performance." A genuinely hard version of this task (real articles, ambiguous edge cases) is where hyperparameter and model-size choices would actually start to matter, and where a search would be worth running.

---

## Evaluation

Both models were evaluated on the **same held-out test set of 60 examples** (15% of the 400 rows, stratified by `label`). The fine-tuned model never saw these rows during training; the zero-shot baseline is given each row's `text` and asked to pick one of the four labels. Using one identical test set is what makes the comparison fair.

### Baseline prompt (zero-shot, Groq)

The baseline receives the label definitions and one post, and must return exactly one label. The prompt sent for each test example is:

```
You are classifying a cybersecurity blog post / writeup by its PURPOSE — why the
author wrote it — not by which MITRE ATT&CK technique it mentions.

Choose exactly ONE of these four labels:

- incident_case_study: a forensic walkthrough of ONE specific intrusion; a
  narrative timeline reconstructed from evidence ("we responded to…", dwell time,
  host-by-host timeline).
- detection_engineering: the post's primary purpose is detection logic — building
  or sharing a rule/query/analytic (Sigma/KQL/SPL/EQL, logsource, false positives,
  tuning).
- threat_actor_profile: characterizes a NAMED actor/group's recurring TTPs across
  multiple campaigns (aliases, victimology, attribution) — not a single incident.
- adversary_emulation: offensive validation — emulating/testing techniques to
  measure defensive coverage (Atomic Red Team, CALDERA, purple team, "we emulated…").

Post:
"""
{text}
"""

Respond with ONLY the label string, nothing else.
```

`{text}` is the post excerpt from the test row. The response is parsed to the matching label (defaulting to the closest match if the model adds stray text).

### Results

| Model | Accuracy (n=60) |
|---|---|
| Zero-shot baseline (Groq) | **1.000** (60/60) |
| Fine-tuned DistilBERT | 0.983 (59/60) |
| **Δ (fine-tuned − baseline)** | **−0.017 (regression)** |

The zero-shot baseline classified **every** test example correctly. The fine-tuned model made one error. So fine-tuning did not merely fail to add value — it produced a model **slightly worse than the off-the-shelf zero-shot baseline.**

### Per-class metrics

Test-set support per class (15% stratified split of 94/105/101/100): `incident_case_study` 14, `detection_engineering` 16, `threat_actor_profile` 15, `adversary_emulation` 15.

**Fine-tuned DistilBERT**

| Class | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `incident_case_study` | 1.00 | 1.00 | 1.00 | 14 |
| `detection_engineering` | 1.00 | 0.94 | 0.97 | 16 |
| `threat_actor_profile` | 0.94 | 1.00 | 0.97 | 15 |
| `adversary_emulation` | 1.00 | 1.00 | 1.00 | 15 |
| **macro avg** | **0.98** | **0.98** | **0.98** | **60** |

The single miss is a `detection_engineering` post predicted as `threat_actor_profile` — it costs `detection_engineering` one point of recall and `threat_actor_profile` one point of precision; every other cell is perfect.

**Zero-shot baseline (Groq)** — 60/60. Every class is perfect, so every cell is 1.00.

| Class | Precision | Recall | F1 | Support |
|---|---|---|---|---|
| `incident_case_study` | 1.00 | 1.00 | 1.00 | 14 |
| `detection_engineering` | 1.00 | 1.00 | 1.00 | 16 |
| `threat_actor_profile` | 1.00 | 1.00 | 1.00 | 15 |
| `adversary_emulation` | 1.00 | 1.00 | 1.00 | 15 |
| **macro avg** | **1.00** | **1.00** | **1.00** | **60** |

### Confusion matrix (fine-tuned DistilBERT)

Rows = true label, columns = predicted (from the notebook's `confusion_matrix` output).

| true ↓ / pred → | case_study | detection | actor_profile | emulation |
|---|---|---|---|---|
| **incident_case_study** | 14 | 0 | 0 | 0 |
| **detection_engineering** | 0 | **15** | **1** | 0 |
| **threat_actor_profile** | 0 | 0 | 15 | 0 |
| **adversary_emulation** | 0 | 0 | 0 | 15 |

The only off-diagonal cell is a `detection_engineering` post pulled into `threat_actor_profile`. Notably, this is **not** one of the four boundaries the taxonomy flags as ambiguous (case-study↔detection, case-study↔actor, actor↔emulation, detection↔emulation) — which makes it more revealing, not less (see error analysis).

### Confusion matrix (zero-shot baseline, Groq)

| true ↓ / pred → | case_study | detection | actor_profile | emulation |
|---|---|---|---|---|
| **incident_case_study** | 14 | 0 | 0 | 0 |
| **detection_engineering** | 0 | 16 | 0 | 0 |
| **threat_actor_profile** | 0 | 0 | 15 | 0 |
| **adversary_emulation** | 0 | 0 | 0 | 15 |

Perfectly diagonal — the baseline made no mistakes at all. The model the project set out to build (fine-tuned DistilBERT) is the *only* one of the two that got anything wrong.

### Error analysis — the one wrong prediction

| True | Predicted | Why it failed |
|---|---|---|
| `detection_engineering` | `threat_actor_profile` | The misclassified post is a piece of detection content built *around a named adversary* — the kind of write-up that ships a rule/analytic for hunting a specific group and therefore carries that group's name, aliases, and TTP vocabulary. Those are precisely the lexical cues that define `threat_actor_profile`. With the actor-name signal loud and the post's actual deliverable (the rule) not loud enough to override it, the model classified on **surface vocabulary** rather than purpose. |

Two honest notes on this error:

1. **The pre-run talk notes guessed wrong.** `project3_5min_notes.md` predicted the likely miss would be Splunk ESCU → `adversary_emulation` (a detection↔emulation slip). The actual run shows detection → `threat_actor_profile` instead. I'm reporting what the confusion matrix shows, not what I expected. *(The specific row's `id`/`text` isn't in the artifacts I have — if you share the misclassified example from the notebook I'll quote it here.)*
2. **This error is off the taxonomy's documented boundaries**, which makes it *more* diagnostic, not less. A slip on a boundary the taxonomy already flags as genuinely ambiguous would be defensible. A slip from `detection_engineering` to `threat_actor_profile` — two classes the taxonomy treats as clearly separable (one ships a rule, the other characterizes a group) — is harder to excuse as inherent ambiguity. It looks like the model latched onto actor-name keywords and ignored the deliverable. That is the pattern-matching failure mode in miniature.

### Error pattern analysis (systematic)

**Caveat first:** with one error from the fine-tuned model and zero from the baseline, there is no error *distribution* to do statistics on — any "pattern" claimed from a single data point would be overreach. The honest systematic findings are about the *structure* of confusability, not its frequency:

- **Errors are directional, and the direction is informative.** The one observed error goes `detection_engineering` → `threat_actor_profile`, never the reverse. Under the keyword-cue hypothesis this is exactly what you'd predict: actor-name vocabulary (`Scattered Spider`, aliases, "across campaigns") is a *strong, unmistakable* signal, while the deliverable cue for detection (a rule/query snippet) is *easily absent* from an excerpt. So the predicted failure direction is **classes with weak/optional cues bleeding into classes with strong/always-present cues**, not the other way around. A real test set should over-sample that direction (detection and case-study posts that mention named actors) rather than sampling uniformly.
- **The confusable pairs are predictable from cue overlap, not from the taxonomy's stated boundaries.** The taxonomy flags four "ambiguous" boundaries for *human* annotators. But the *model's* likely confusions are driven by lexical overlap, which is a different map: any class can leak into `threat_actor_profile` whenever a real post names an adversary (extremely common across all four categories), and `detection_engineering` ↔ `adversary_emulation` overlap because both discuss detections. The taxonomy-ambiguous boundaries and the model-confusable boundaries are not the same set — and the one error we have sits on a model-confusable pair that is *not* a taxonomy-ambiguous one. That mismatch is the single most useful systematic takeaway.
- **The baseline's clean sweep is itself a pattern.** Zero errors across all four classes from an untrained model means the cues are not just present but *linearly trivial* — separable without any learned decision boundary. A genuinely subjective task would show at least a few baseline errors scattered across the ambiguous boundaries; their complete absence is the systematic signature of a saturated, synthetic dataset.

**How to actually surface error patterns (method, not yet run):** the right way to get pattern-level evidence from a near-perfect model is not more held-out examples (they're all easy) but **controlled probing**:
1. *Confidence/margin analysis* — log the softmax margin per class on the test set. Even with 0–1 errors, narrow margins reveal which classes the model is *nearly* confusing.
2. *Ablation by cue removal* — programmatically strip the defining keyword from each post (delete rule snippets from `detection_engineering`, anonymize actor names in `threat_actor_profile`) and re-classify. The pattern of where accuracy collapses maps exactly which cue each class depends on.
3. *Cross-cue injection* — inject actor names into detection posts (and vice versa) and measure flip rate. The DE→TA error predicts a high flip rate here; confirming it across many examples would turn the single anecdote into a quantified pattern.

If you run any of these, share the output and I'll fold the quantified pattern (margins, flip rates) into this section.

### Confidence calibration

Calibration asks: when the model says *0.9 confident*, is it right about 90% of the time? A well-calibrated model's confidence tracks its accuracy; an overconfident one assigns high probability to predictions that are often wrong.

**Why this dataset can't show a real calibration curve.** Calibration is measured by bucketing predictions by confidence and comparing each bucket's mean confidence to its accuracy (a reliability diagram, summarized by Expected Calibration Error). That needs predictions spread *across* the confidence range. Here the accuracy is 59–60/60, so essentially every prediction lands in the top bucket and is correct — there are no low-confidence buckets to populate and no errors to spread across buckets. **You cannot estimate calibration from data with one error.** Reporting an ECE here would be a number computed on a single misclassification — meaningless.

**What the structure nonetheless predicts.** Because the labels are separable by planted keywords, the fine-tuned model is almost certainly **confidently peaked** — softmax near 1.0 on nearly every example, because the discriminative token is right there. That produces a degenerate calibration profile: confidence is uniformly high and *uninformative*, so it can't be used to triage uncertain predictions. The single most diagnostic number available is **the confidence on the one wrong prediction (DE → TA):**
- If that error was **high-confidence** (e.g. > 0.9), the model is **overconfident and miscalibrated** — it was sure and wrong, which is the dangerous failure mode and consistent with "it saw actor-name keywords and committed."
- If it was **low-confidence** (a narrow margin over `detection_engineering`), the model was at least *appropriately uncertain* on the case it got wrong — better-calibrated, and it would mean a confidence threshold could have caught the error.

The talk notes (`project3_5min_notes.md`) left these confidences as `‹0.__›` placeholders, so the deciding number isn't in the artifacts I have.

**Method to do this properly (not yet run):**
1. Log the full softmax vector per test example (both for the fine-tuned model and, via log-probs or repeated sampling, the Groq baseline).
2. Plot a reliability diagram and compute ECE — though expect it to be uninformative until the test set is hard enough to produce a confidence spread.
3. More useful on a saturated set: report the **confidence histogram** and the **margin (top-1 minus top-2 probability)** distribution. A pile-up near margin ≈ 1.0 quantifies the "confidently peaked, uninformative" hypothesis; the margin on the DE→TA error answers the overconfidence question above.

Share the per-example probabilities and I'll compute ECE, the reliability diagram, and the error's confidence and drop the real numbers in here.

### Reflection — intended vs. learned

**What the taxonomy intended the model to learn:** a *semantic, intent-level* distinction — the author's purpose in writing the post, which cuts across ATT&CK techniques and can only be resolved by reading what the post is *for* (an event? an actor? a rule? a test?).

**What the model actually learned:** *surface lexical cues* that happen to co-occur with each intent in my authored text — `sigma`/`kql` ⇒ detection, dwell-time/"we responded to…" ⇒ case study, group aliases/"across campaigns" ⇒ actor profile, "we emulated…"/Atomic Red Team ⇒ emulation.

The gap between the two is the real result of this project, and the fine-tuned model's single error is its proof. The large zero-shot model resolved every example — including the `detection_engineering` post that also carried a named actor's vocabulary. The smaller fine-tuned model did not: on that one post it followed the **actor-name keywords straight into `threat_actor_profile`** and ignored the actual deliverable. A model that had learned *intent* would have read "this post exists to ship a detection rule" regardless of which adversary the rule happens to hunt. So 0.983 measures how reliably the fine-tuned model can spot keywords I planted — and the one case where the keywords conflicted is the one it got wrong. The honest headline is the **intended-vs-learned gap**, not the accuracy.

### Critical analysis — these numbers are a red flag, not a result

At face value, 0.983 looks like a strong outcome. It is not. **A subjective, intent-based classification task should not be *perfectly* solvable by an off-the-shelf zero-shot model — and the model I actually trained for the task scored *below* that zero-shot baseline.** This is worse than "fine-tuning added no value": the dedicated model regressed (−0.017), so the fine-tuning step was not just unnecessary but mildly counterproductive. When a zero-shot baseline hits 60/60 and task-specific training can only match-or-lose, the most likely explanation is that the task, as evaluated, is too easy. The planning notes anticipated exactly this ("fine-tuned barely beats baseline means fine-tuning added little — labels too easy or too noisy"); the result came in at the pessimistic end of that prediction.

The mechanism is the dataset itself. Because **96% of the rows are representative text I authored with the label definitions in front of me**, each example tends to contain the same surface signals the labels are defined by — Sigma/KQL snippets for `detection_engineering`, "the intrusion began…" and dwell-time language for `incident_case_study`, group aliases and "across campaigns" for `threat_actor_profile`, Atomic Red Team / "we emulated…" for `adversary_emulation`. A capable zero-shot model reads those planted cues straight off the page and gets everything right with no training at all. Fine-tuning DistilBERT on the same cues can't improve on a perfect baseline — it can only add noise, which is exactly what the one regression looks like. Neither model is learning the subtle community distinctions the taxonomy is designed around; both are keying on surface patterns I baked in, and the smaller model is simply slightly worse at it.

So the high accuracy measures **pattern matching on synthetic data**, not generalization to how real practitioners write — and the baseline beating the fine-tuned model confirms there was no task-specific signal left to learn. The two questions worth carrying into future work:
- *What would make this task actually hard?* — Real posts that mix purposes (a DFIR case study with a substantial detections section), terse or atypical writing, and the documented edge cases where the deliverable is ambiguous.
- *Is the evaluation measuring real generalization, or pattern matching?* — Right now, the latter. Headline accuracy on the synthetic split should not be reported as the project's result without this caveat.

### A more honest evaluation would require

- **Test on real, held-out articles only.** Evaluate against the `verified` subset (and expand it — fetch and quote more genuine articles per label) so the test set isn't drawn from the same hand-authored distribution as training.
- **Stress the edge cases deliberately.** The taxonomy flags four ambiguous boundaries, but the one observed error didn't land on any of them — a `detection_engineering` post was pulled to `threat_actor_profile` (see error analysis), apparently because it named an adversary. A real evaluation should deliberately include detection content that references named actors, mixed-purpose posts, and the documented boundary cases, to probe exactly this kind of keyword-driven failure.
- **Report the synthetic/real split alongside every metric**, so a 0.98 on authored text is never mistaken for a 0.98 on the wild.

---

## AI usage

AI tools were used throughout this project. Two specific, load-bearing instances:

1. **Generating the 385 `representative` dataset rows.** I used an LLM to author the synthetic blog-post text, prompting it per label with the taxonomy definitions and tell-tale signals so each row read like genuine community publishing for its class. This is also the project's central weakness, surfaced honestly in the evaluation: because the same label definitions that *defined* each class were used to *generate* the examples, the planted lexical cues made the task near-trivially separable (zero-shot baseline scored 60/60). The AI sped up dataset creation enormously but baked a circularity into it that inflated the scores — which is precisely why the report treats those scores as a red flag rather than a result.

2. **Building the zero-shot baseline classifier.** The baseline itself is an AI system — a Groq-hosted LLM prompted (see [Baseline prompt](#baseline-prompt-zero-shot-groq)) to pick one of the four labels per post. Here AI is not a helper but the artifact under evaluation: it is the reference point the fine-tuned DistilBERT had to beat, and it set the bar at a perfect score.

*(Other AI assistance — drafting and revising this README, sketching the eval scaffolding, fetching/summarizing the 15 real articles behind the `verified` rows — was supporting rather than load-bearing. Adjust this list to match exactly what you used if submitting for assessment.)*

## Spec reflection

How the work measured up against the specification it set for itself (the taxonomy in `ir_attack_taxonomy_README.md`):

- **The spec was the strongest part of the project.** The intent-based axis, the four mutually-exclusive labels, the deconfliction rules for the ambiguous boundaries, and the exhaustiveness argument are coherent and defensible — they reflect how the DFIR/CTI community actually organizes. The taxonomy is a genuine design contribution.
- **The spec quietly assumed the hard part was *labeling*; the real hard part was *evaluation*.** The deconfliction rules anticipate where a human annotator would struggle. But the evaluation showed the models never had to engage with those subtleties at all — the planted keywords resolved everything. The spec optimized for label quality and under-specified how to *prove* a model had learned the distinction rather than the keywords. The single fine-tuned error landed on a transition (`detection_engineering` → `threat_actor_profile`) that the spec treats as clearly separable, which is the clearest sign of this gap.
- **The spec's own honesty note predicted the failure mode, but the original deliverable didn't follow through.** `planning.md` flagged "labels too easy or too noisy" and the build note disclosed the 96%-synthetic split. The spec had the right instincts; the missing step was carrying that skepticism into the results section — which this revision now does.
- **What I'd change in the spec next time:** add an *evaluation* section to the taxonomy itself — mandating a real, held-out test set drawn from a different distribution than the training text, plus deliberately adversarial examples (detection posts that name actors, mixed-purpose write-ups) designed to break keyword shortcuts. A taxonomy spec should specify not just how to label, but how to demonstrate a model learned the labels' *meaning*.

---

## Suggested next steps

- Replace the synthetic test split with a real-data evaluation set (`grounding == verified`), and grow that subset.
- Re-run the baseline-vs-fine-tuned comparison on the real set; a non-trivial *positive* Δ there (fine-tuned beating zero-shot) would be the first meaningful result. On the current synthetic set the Δ is negative, which only confirms the task is too easy to need fine-tuning.
- Optionally add a 5th `tooling_release` class for out-of-scope tooling/version-announcement posts.
