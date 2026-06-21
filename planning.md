# Planning — IR / MITRE ATT&CK Writeup Classification Dataset

## 1. Goal
Build an annotated dataset to fine-tune / evaluate an LLM that classifies **cybersecurity community writeups in the Incident Response + MITRE ATT&CK space** by the author's *purpose*. Target: ≥200 examples, mutually-exclusive labels, ≥90% coverage with no "other" bucket.

## 2. Community & why it's a good fit
**Community chosen:** the practitioner community that publishes Incident Response / MITRE ATT&CK writeups — DFIR responders, cyber-threat-intelligence (CTI) analysts, detection engineers, and purple/red teamers. Their output lives on vendor research blogs (Mandiant, Unit 42, Red Canary, Elastic), independent sites (The DFIR Report, SigmaHQ), and conference tracks.

**Why it's a good fit for a classification task:**
- **Varied discourse, shared vocabulary.** Posts span four genuinely different intents (reconstructing one breach, shipping a detection rule, profiling an actor, validating defenses) yet all share the same ATT&CK technique vocabulary. That overlap in surface terms with divergence in *purpose* is exactly what makes the task non-trivial — a keyword classifier would fail because "T1059.001 / PowerShell" appears in all four label types.
- **Real organizing structure to validate against.** The community already self-organizes content this way (blog categories, con tracks, job roles), so the labels map to decisions practitioners actually make — not an artificial taxonomy.
- **Clear utility.** A working classifier could auto-route an incoming writeup to the right reader (SOC engineer vs. CTI analyst vs. red team), which is a real pain point given the volume of daily publications.

## 3. Design decisions
| Decision | Choice | Rationale |
|---|---|---|
| Taxonomy axis | **Writeup purpose** (author intent) | How the DFIR/CTI/detection community actually organizes content (blog categories, con tracks, roles). Stays mutually exclusive even when a post spans many ATT&CK techniques. |
| Number of labels | **4** | Within the 2–4 requirement; each maps to a real, distinct practitioner workflow. |
| Output format | **CSV** | Spreadsheet-friendly; easy to extend and split. |
| Size | **400 rows** (case 94 / detection 105 / threat-actor 101 / emulation 100) | Roughly balanced classes, well past the 200 minimum. |
| Grounding | `grounding` column: **15 verified** (real fetched articles) + **385 representative** | Verified rows give a real-data eval subset; representative rows scale the labeling pattern. |
| Dates | `date` column (2025–2026) | Keeps the set anchored to current "latest" discourse. |

## 4. Label taxonomy — definitions
Each label below is defined in a complete sentence, with **two example posts** (the first is a real/verified-style fetched excerpt; the second is marked *representative* — a synthetic example written to match the pattern).

### A. `incident_case_study`
**Definition:** An `incident_case_study` is a forensic walkthrough of one specific intrusion, presenting a narrative timeline reconstructed from evidence host by host.
**Include:** "the intrusion began…", dwell time, kill-chain reconstruction, supporting artifacts, "our IR team responded to…".
**Exclude:** generic actor overviews (→ profile); posts that only share a detection (→ detection eng).
**Sources:** The DFIR Report, Mandiant M-Trends, Unit 42 IR, Cisco Talos IR, Sophos X-Ops, Huntress.

> **Example 1** — *The DFIR Report (verified-style)*
> "This intrusion began in early March 2025 with a single successful RDP logon to an internet-exposed system, with no evidence of credential stuffing or brute forcing. The actor moved laterally within hours and deployed ransomware across the domain before exfiltration completed."
> ATT&CK: T1133, T1078, T1021.001, T1486

> **Example 2** — *representative*
> "Over a 9-day engagement we reconstructed how a phishing attachment led to initial access on a single finance workstation, the attacker's use of scheduled tasks for persistence, and the eventual data theft. Timeline, affected hosts, and recovered artifacts are below."
> ATT&CK: T1566.001, T1053.005, T1567

### B. `detection_engineering`
**Definition:** A `detection_engineering` post is one whose primary deliverable is detection logic — a rule, query, or analytic together with how to tune it.
**Include:** Sigma/KQL/SPL/EQL, logsource, data sources, false positives, tuning, "this rule detects…".
**Exclude:** detections that appear as a closing section of an incident report (→ case study); exercises that *test* detections (→ emulation).
**Sources:** SOC Prime, Splunk Threat Research, Elastic Security Labs, Red Canary, SigmaHQ, SpecterOps.

> **Example 1** — *SOC Prime (verified-style)*
> "This Sigma rule detects suspicious rundll32 execution patterns associated with malicious DLL loading. We cover the logsource, the selection logic, expected false positives from legitimate software, and the ATT&CK mapping to T1218.011."
> ATT&CK: T1218.011, T1059.001

> **Example 2** — *representative*
> "Here is a KQL analytic for detecting OAuth consent-grant abuse in Entra ID. We walk through the AuditLogs data source, the join that surfaces risky grants, the benign SaaS onboarding that triggers false positives, and recommended tuning thresholds."
> ATT&CK: T1528, T1098.003

### C. `threat_actor_profile`
**Definition:** A `threat_actor_profile` is CTI that characterizes a named actor or group's recurring TTPs across multiple campaigns, rather than a single incident.
**Include:** group aliases, victimology, attribution, "across multiple intrusions", "the group has been observed…".
**Exclude:** single-incident timelines (→ case study); re-creating the group's TTPs to test defenses (→ emulation).
**Sources:** Mandiant, Unit 42, CrowdStrike, Microsoft Threat Intelligence, CISA advisories, Recorded Future.

> **Example 1** — *Mandiant (verified-style)*
> "This profile of UNC3944 (Scattered Spider) summarizes the group's tradecraft across 2025: vishing of help desks, SIM swapping, abuse of legitimate remote tooling, and rapid pivots to ransomware. We map their consistently observed techniques to ATT&CK."
> ATT&CK: T1566.004, T1219, T1486

> **Example 2** — *representative*
> "We track FIN-style financially motivated activity attributed to this cluster across the retail and hospitality sectors over the past two years, including its consistent reliance on web-shell footholds and living-off-the-land lateral movement. Aliases, victimology, and recurring techniques are mapped below."
> ATT&CK: T1505.003, T1218, T1070

### D. `adversary_emulation`
**Definition:** An `adversary_emulation` post is offensive validation — emulating or simulating techniques to measure defensive coverage as a purple- or red-team exercise.
**Include:** Atomic Red Team, CALDERA, BAS, "we emulated…", safe payload, detection validation, cleanup steps.
**Exclude:** real intrusions (→ case study); the detection rule itself (→ detection eng).
**Sources:** Red Canary (Atomic Red Team), MITRE CALDERA / Engenuity CTID, Picus, Cymulate, SCYTHE, SANS.

> **Example 1** — *Atomic Red Team / Red Canary (verified-style)*
> "This post walks through running the Atomic Red Team test for T1059.001 to validate PowerShell detection coverage. We show the atomic, the expected telemetry, and how to confirm your analytics fire as a purple-team exercise."
> ATT&CK: T1059.001

> **Example 2** — *representative*
> "In this purple-team exercise we used CALDERA to emulate credential dumping against a lab domain, measured which of our EDR analytics fired, documented the coverage gaps, and included the cleanup steps to restore the environment afterward."
> ATT&CK: T1003.001, T1003.002

## 5. Hard edge cases & annotation handling
The genuinely ambiguous posts sit on these four boundaries. **Handling rule at annotation time:** label by the author's *primary deliverable / purpose*, not by the techniques mentioned; when two intents seem co-equal, apply the tie-break below, and if it is still a coin-flip, flag the row (`grounding` note) for a second pass rather than guessing.

1. **Case study vs. detection eng** — DFIR reports often end with a "Detections" section, but the *purpose* is reconstructing the intrusion → `incident_case_study`. Tie-break: if you removed the detection section, would the post still stand as a story? If yes → case study.
2. **Case study vs. profile** — about a single *event* → case study; about an *actor across events* → profile. Tie-break: count the intrusions described — one → case study, many → profile.
3. **Profile vs. emulation** — *describing* a group's behavior → profile; *re-creating* it to test defenses → emulation. Tie-break: is anything being executed in a lab? If yes → emulation.
4. **Detection eng vs. emulation** — deliverable is the *rule* → detection eng; deliverable is the *exercise/validation* → emulation. Tie-break: does the post ship a reusable detection artifact, or a test run + coverage finding?

### Documented difficult cases (from annotation)
Real rows from the dataset that genuinely gave pause during labeling, the labels each could plausibly take, and the decision made:

1. **Row id 19 — Unit 42 (labeled `incident_case_study`).** Text: *"Our incident responders were engaged after a retailer detected encryption across virtualized infrastructure. We reconstruct how the actor used help-desk vishing to reset MFA, gained VPN access, and reached the vSphere management plane within a single day."* — **Tension:** the vishing→MFA-reset→VPN tradecraft is the signature of a named group (pull toward `threat_actor_profile`), and it opens with "detected encryption" (faint pull toward `detection_engineering`). **Decision → `incident_case_study`:** it reconstructs *one* intrusion on a *specific* victim's hosts with a timeline ("within a single day"), which is the case-study purpose (boundary rule 2: one event → case study). The TTPs are evidence in a story, not a cross-campaign characterization.
2. **Row id 177 — AttackIQ (labeled `detection_engineering`).** Text: *"We share a detection for ransomware encryption behavior derived from emulation telemetry. The post covers the file-event logic, the tuning, and the ATT&CK mapping to T1486."* — **Tension:** sits exactly on the detection-eng ↔ emulation boundary; it explicitly mentions *emulation telemetry*. **Decision → `detection_engineering`:** the deliverable is the reusable detection (file-event logic + tuning); the emulation is only the data source used to build it (boundary rule 4: ships a detection artifact → detection eng).
3. **Row id 306 — MITRE CALDERA (labeled `adversary_emulation`).** Text: *"We build a CALDERA adversary profile that chains remote-host discovery and SMB lateral movement. The post explains the abilities used, how to run the operation safely in a lab, and how to verify detections."* — **Tension:** the literal phrase "adversary profile" pulls toward `threat_actor_profile`, and "verify detections" pulls toward `detection_engineering`. **Decision → `adversary_emulation`:** something is actually *executed in a lab* to validate coverage (boundary rule 3 tie-break: executed in a lab → emulation). "Profile" here is CALDERA's term for an operation plan, not a CTI actor write-up, and no reusable detection rule is shipped.

(These 3+ documented cases satisfy the Milestone 3 checkpoint requirement.)

## 6. Data collection plan
- **Where:** the source lists under each label in Section 4 (vendor research blogs, independent DFIR sites, SigmaHQ/CALDERA repos, conference writeups). Verified rows are fetched from live 2025–2026 articles; representative rows are authored to match the pattern with varied vector/actor/technique/sector/tooling.
- **How many per label:** target balanced ~100 per label (achieved: case 94 / detection 105 / threat-actor 101 / emulation 100 = 400 total), well past the 200 minimum.
- **Imbalance check (Milestone 3):** verified the largest class (`detection_engineering`) is **26.3%** of the dataset — far below the 70% threshold — so no majority-class skew and no rebalancing needed. All 400 ids unique; 0 empty `text`/`label` rows.
- **If a label is underrepresented after 200 examples:** (a) first do a targeted source sweep for that label using its dedicated source list before authoring any synthetic rows; (b) if real material is genuinely scarce, top up with representative rows (tagged `representative` in the `grounding` column so the imbalance source is transparent); (c) keep classes within roughly ±15% of each other so a single class never drops below ~25% of the largest; (d) if a class cannot reach a viable floor (~50 rows), reconsider whether it should be merged or dropped rather than padded with low-quality examples. Stratified splitting (Section 7) preserves the distribution across train/val/test.

## 7. Evaluation metrics
Accuracy is the headline number but is **not sufficient** on its own — it hides which of the four workflow classes fails and which label boundaries the model confuses. The plan evaluates on a held-out, stratified test set with:

| Metric | Why it's the right one here |
|---|---|
| **Macro-F1** (primary) | Averages F1 across the four classes equally. We care about every practitioner workflow equally, so a model that nails the two biggest classes but fails on emulation should be penalized — macro-F1 does that; accuracy would not. |
| **Per-class precision & recall** | The cost of confusion is asymmetric (mis-routing an emulation post as a case study sends it to the wrong reader). Per-class numbers show *which* class is weak and whether it's a precision or recall problem. |
| **Confusion matrix** | Directly tests the Section 5 boundaries — we expect the residual errors to land on case-study↔detection and profile↔emulation, and the matrix confirms whether the deconfliction rules held. |
| **Baseline parse / coverage rate** | For the zero-shot LLM baseline, the fraction of responses that map to a valid label. A baseline that can't emit clean label names is itself a finding and must be reported, not silently dropped. |
| **Fine-tuned vs. zero-shot delta** | Quantifies whether fine-tuning was worth it; the comparison is the core experiment. |
| **Verified-subset check** | Re-report macro-F1 on `grounding == verified` rows alone to confirm performance isn't an artifact of synthetic-row regularities. |

## 8. Definition of success (objective, testable thresholds)
Evaluated on the held-out test set, the classifier is:

- **Good enough for deployment in a community routing tool if all hold:**
  1. **Macro-F1 ≥ 0.85**, and
  2. **no single class recall < 0.75** (no workflow is silently dropped), and
  3. fine-tuned model **beats the zero-shot baseline by ≥ 0.10 macro-F1**, and
  4. zero-shot baseline **parse rate ≥ 0.95** (so the comparison is fair), and
  5. macro-F1 on the **verified-only** subset is **within 0.10** of the full-test macro-F1 (generalizes beyond synthetic rows).
- **Genuinely useful (stretch):** macro-F1 ≥ 0.90 with every class recall ≥ 0.85 and residual errors confined to the two expected adjacent-label boundaries.
- **Not good enough:** macro-F1 < 0.80, or any class recall < 0.70, or errors scattered across non-adjacent labels (signals the taxonomy itself is leaking).

These are pass/fail at the end of the run: each is a number the notebook already produces, so the verdict is objective.

## 8b. Baseline run (Milestone 4)
Before any fine-tuning, establish a **zero-shot LLM baseline** on the locked test set so the fine-tuned model's numbers have a reference point. A zero-shot prompt is a legitimate baseline: it measures how hard the task is for a general model with no training. **Run this before touching the training data.**

**Notebook setup (run in order — Section 5 depends on variables from Sections 1–2):**
1. Open the starter Colab notebook; confirm the runtime is set to **T4 GPU** before running any cells.
2. **Section 1** — define the label map and upload the single labeled CSV (`ir_attack_writeup_dataset.csv`) when prompted. No manual placement in Drive needed. If the Colab session disconnects, re-upload and re-run Sections 1–2.
3. **Section 2** — the notebook splits the dataset **70% / 15% / 15%** (train / val / test) and tokenizes all splits. Review the split sizes and per-label distribution to confirm they look reasonable (stratification should preserve the ~balanced 4-class distribution from Section 6).
4. **Section 5** — add the **Groq API key** via Colab Secrets or directly in the cell (**never commit it to GitHub**). Write the classification prompt: include the Section 4 label definitions verbatim and instruct the model to **output only the label name** — the notebook's parser depends on a clean, consistent response.
5. Run the baseline cells. The notebook classifies every test-set example, prints **overall accuracy + per-class metrics**, and flags unparseable responses. **If >~10% of responses are unparseable, revise the prompt** to make the expected output format clearer, then re-run.

**Reflection (record before fine-tuning):** note where the baseline struggled and which labels it consistently confuses, and write down a hypothesis to test after fine-tuning. Expectation from Section 5: residual confusion should concentrate on the case-study↔detection and profile↔emulation boundaries; anything scattered across non-adjacent labels is a signal to revisit.

**Checkpoint (Milestone 4 done when):** baseline overall accuracy **and** at least a per-class breakdown exist for the test set, the fine-tuned model has **not** yet been run on this test set, and the baseline numbers are **saved somewhere referenceable** for the evaluation report (e.g. exported from the notebook into the repo / report appendix).

## 9. Coverage & out-of-scope
These four purposes cover >90% of IR+ATT&CK practitioner writeups with **no "other" bucket**. Deliberately out of scope: pure tooling release notes, framework-version announcements (e.g. ATT&CK v18), and opinion/policy pieces. If those become frequent, add a 5th `tooling_release` class.

## 10. AI Tool Plan
This is a dataset/annotation project, not an implementation project — there is no application code to generate. AI tools are used in exactly three places:

1. **Label stress-testing (before annotating 200 examples).** Give the AI the Section 4 definitions plus the Section 5 edge cases and ask it to generate 5–10 posts that sit *on the boundary* between two labels (especially case-study↔detection and profile↔emulation). If any generated post can't be classified cleanly under the current rules, that exposes a definition gap → tighten the definition/tie-break in Section 5 **now**, then regenerate to confirm the ambiguity is resolved. Done before bulk annotation.
2. **Annotation assistance (pre-labeling).** Decision: an LLM **may** pre-label a batch, but every pre-labeled row is human-reviewed before it enters the dataset — the LLM is a first-pass suggester, not the authority. Tracking: pre-labeled rows are marked (e.g. an `ai_prelabeled` flag or note alongside the `grounding` column) so they can be disclosed in the AI-usage section and audited for systematic LLM bias. Tool used will be named explicitly in that disclosure.
3. **Failure analysis (after evaluation).** Feed the list of wrong predictions (text + true label + predicted label + confidence, from the notebook's error-analysis cell) to an AI tool and ask it to cluster the errors into patterns — e.g. "detection posts misread as emulation when they mention a lab." For each proposed pattern, **verify it manually** by pulling 2–3 of the cited examples and confirming the pattern actually holds before it goes into the evaluation writeup; discard patterns that don't survive inspection.

> Update this plan (and the sections above) before starting any stretch features later.

## 11. Build process
1. Web research across the 4 categories → confirm real sources, actors, technique IDs (2025–2026).
2. Define taxonomy + deconfliction rules.
3. Fetch specific live articles and extract real excerpts/dates/technique IDs → **15 `verified` rows**.
4. Author balanced, varied `representative` examples per label (vary vector, actor, technique, sector, tooling) → **385 rows**.
5. Validate CSV: 400 unique IDs, single label per row, consistent 9-column schema, label balance, grounding split.

## 12. Deliverables
- `ir_attack_writeup_dataset.csv` — 400 labeled examples; columns `id,label,grounding,source,url,date,attack_tactics,attack_techniques,text`.
- `ir_attack_taxonomy_README.md` — taxonomy, deconfliction rules, verified-source list, honesty note.
- `planning.md` — this document.

## 13. Open follow-ups
- Expand the `verified` subset further (fetch + quote more live articles per label).
- Add a 5th `tooling_release` class if tooling/version posts become frequent.
- The notebook produces a stratified **70/15/15 train/val/test** split by `label` (see Section 8b); the 15% test split is the locked held-out set used for the baseline and final evaluation. Optionally re-evaluate on `grounding == verified` only.
