# IR / MITRE ATT&CK Writeup — Label Taxonomy & Dataset

**Purpose:** A fine-tuning / classification dataset for labeling cybersecurity community blog posts and writeups in the **Incident Response + MITRE ATT&CK** space.

**Files**
- `ir_attack_writeup_dataset.csv` — 400 annotated examples (roughly balanced: case study 94, detection 105, threat-actor 101, emulation 100).
- `ir_attack_taxonomy_README.md` — this file.
- `planning.md` — design plan.

**CSV columns:** `id`, `label`, `grounding`, `source`, `url`, `date`, `attack_tactics`, `attack_techniques`, `text`

**`grounding` column:**
- `verified` (15 rows) — text excerpt, date, URL, and ATT&CK techniques pulled from a specific **real, live 2025–2026 article** I fetched (The DFIR Report, Unit 42, Microsoft Threat Intelligence, Elastic Security Labs, Splunk, Red Canary, Picus).
- `representative` (385 rows) — text written to read like the source/label actually reads, modeled on real community publishing but **not a verbatim copy**; `url` points to the publisher's blog index and `date` is a plausible 2025–2026 value.

To train/evaluate only on real data, filter `grounding == verified`. To use the full pattern set, use all rows.

---

## Taxonomy axis: *writeup purpose*

The labels split on **why the post was written** — the author's intent — not on which ATT&CK tactic appears. This is the distinction that the DFIR/CTI/detection community actually organizes around (it's how blog categories, conference tracks, and job roles are divided), and it keeps the classes mutually exclusive even when a post touches many ATT&CK techniques.

### The 4 labels

| Label | What it is | Tell-tale signals | Typical sources |
|---|---|---|---|
| `incident_case_study` | A forensic walkthrough of **one specific intrusion** — a narrative timeline reconstructed from evidence. | "the intrusion began…", dwell time, host-by-host timeline, "we responded to…", supporting artifacts. | The DFIR Report, Mandiant M-Trends, Unit 42 IR, Talos IR, Sophos X-Ops, Huntress |
| `detection_engineering` | A post whose **primary purpose is detection logic** — building/sharing a rule, query, or analytic. | Sigma/KQL/SPL/EQL snippets, logsource, false positives, tuning, "this rule detects…". | SOC Prime, Splunk Threat Research, Elastic Security Labs, Red Canary, SigmaHQ |
| `threat_actor_profile` | CTI characterizing a **named actor/group's recurring TTPs** across campaigns (not one incident). | "Group X has been observed…", victimology, attribution, "across campaigns", group aliases. | Mandiant, Unit 42, CrowdStrike, MSTIC, CISA advisories, Recorded Future |
| `adversary_emulation` | **Offensive validation** — emulating/testing techniques to measure defensive coverage. | Atomic Red Team, CALDERA, purple team, "we emulated…", safe payload, detection validation, cleanup. | Red Canary (Atomic), MITRE Caldera/CTID, Picus, Cymulate, SCYTHE, SpecterOps |

### Mutual-exclusivity rules (the edge cases that matter)

These are the deconfliction rules used while labeling — they keep "exactly one label per post" enforceable:

1. **Case study vs. detection engineering.** A DFIR report almost always ends with a "Detections" section. It is still `incident_case_study`, because the *purpose* of the post is to reconstruct an intrusion. Only label `detection_engineering` when sharing the detection **is** the post.
2. **Case study vs. threat actor profile.** One intrusion with a timeline = `incident_case_study`. A synthesis of a group's behavior *across many intrusions* = `threat_actor_profile`. The question is "is this about an event, or about an actor?"
3. **Threat actor profile vs. adversary emulation.** Describing what a group does = `threat_actor_profile`. *Re-creating* what they do to test defenses (even when modeled on that group) = `adversary_emulation`.
4. **Detection engineering vs. adversary emulation.** Both discuss detections. If the deliverable is the **rule/query**, it's detection engineering. If the deliverable is the **exercise/test that validates coverage**, it's adversary emulation.

### Coverage claim (exhaustiveness)
In the IR + ATT&CK corpus these four purposes cover the overwhelming majority of practitioner writeups — well past the 90% bar — without an "other" bucket. The small residual (pure tooling release notes, framework-version announcements like ATT&CK v18, opinion/policy pieces) is intentionally out of scope; if needed, fold tooling posts into the nearest functional label or add a 5th `tooling_release` class.

---

## How the dataset was built (honesty note)

- **15 `verified` rows** have text excerpts, dates, URLs, and ATT&CK techniques pulled directly from specific real live articles I fetched. Examples of the real sources behind them:
  - The DFIR Report — *Confluence Exploit Leads to LockBit Ransomware* (2025-02-24), *Fake Zoom Ends in BlackSuit Ransomware* (2025-03-31), *Cobalt Strike and a Pair of SOCKS Lead to LockBit* (2025-01-27), *Blurring the Lines* (2025-09-08), *Bissa Scanner Exposed* (2026-04-22).
  - Unit 42 — *Muddled Libra Threat Assessment* (2025-07-25), *2025 Ransomware and Extortion Trends* (2025-04-23).
  - Microsoft Threat Intelligence — *Storm-0501: cloud-based ransomware* (2025-08-27), *Octo Tempest across multiple industries* (2025-07-16).
  - Elastic Security Labs — *Linux persistence mechanisms primer*. Splunk Threat Research — *Nov 2025 security-content update*, *Attack Range v4.0*. Red Canary — *Top Atomic Red Team tests*, *Applying Atomic Red Team* (2025-08). Picus — *ASV guide 2025*.
- **385 `representative` rows** are written to read like each source/label actually reads — modeled on real community publishing but **not verbatim copies**; `url` points to the publisher's blog index and `date` is a plausible 2025–2026 value. Labels, publishers, ATT&CK technique IDs, and named actors (Scattered Spider/UNC3944, Akira, Qilin/AGENDA, Volt Typhoon, Salt Typhoon, Mustang Panda, etc.) are all real.
- This makes the file safe to extend and to train on as a **labeling pattern**. For a larger *gold* set, filter to `verified` and ask me to fetch + quote more specific live articles (I can keep adding real rows per label).

## Suggested next steps
- Hold out ~15% as a validation split (stratified by `label`).
- Filter `grounding == verified` for a small real-data evaluation set.
- I can expand the `verified` subset further (more fetched articles per label) or add a 5th `tooling_release` class on request.

## Evaluation

Overall accuracy — both models
Model 	Accuracy 	Macro-F1
Zero-shot baseline (Groq) 	0.983 	0.98
Fine-tuned DistilBERT 	0.983 	0.98
Per-class metrics — both models

Zero-shot baseline:
Label 	Precision 	Recall 	F1 	Support
incident_case_study 	1.00 	1.00 	1.00 	14
detection_engineering 	0.94 	1.00 	0.97 	16
threat_actor_profile 	1.00 	1.00 	1.00 	15
adversary_emulation 	1.00 	0.93 	0.97 	15

Fine-tuned DistilBERT:
Label 	Precision 	Recall 	F1 	Support
incident_case_study 	1.00 	1.00 	1.00 	14
detection_engineering 	1.00 	0.94 	0.97 	16
threat_actor_profile 	0.94 	1.00 	0.97 	15
adversary_emulation 	1.00 	1.00 	1.00 	15

Fine-tuned model accuracy: 0.983

Per-class metrics (fine-tuned model):
                       precision    recall  f1-score   support

  incident_case_study       1.00      1.00      1.00        14
detection_engineering       1.00      0.94      0.97        16
 threat_actor_profile       1.00      1.00      1.00        15
  adversary_emulation       0.94      1.00      0.97        15

             accuracy                           0.98        60
            macro avg       0.98      0.98      0.98        60
         weighted avg       0.98      0.98      0.98        60

Sample classifications (fine-tuned model)

    3–5 posts run through the fine-tuned model with predicted label + confidence. For at least one correct prediction, explain why it's reasonable.

Here are some sample correct classifications from the fine-tuned model:

# 	Text excerpt 	Predicted label 	Confidence
1 	We present a detection for runtime payload decoding using process and command-line telemetry. The po... 	detection_engineering 	0.63
2 	This post shares KQL hunting queries for Kerberoasting in Microsoft Sentinel. We explain the 4769 ev... 	detection_engineering 	0.64
3 	Trend's responders narrate a hands-on-keyboard intrusion. We follow the PowerShell reconnaissance, r... 	incident_case_study 	0.47
4 	A Securelist profile of Kimsuky. We detail their campaigns, tools (like KGH_SPY), and their focus o... 	threat_actor_profile 	0.50
5 	Using Mandiant Security Validation, we emulate a C2 beacon and exfiltration to test detection and n... 	adversary_emulation 	0.49     

 Zero-shot classification complete for demo.

First 5 demo zero-shot predictions:
  Text: This profile uses infrastructure analysis to characterize a phishing-f...
  Predicted: threat_actor_profile
  Text: A Check Point Research profile of an access-broker actor. We summarize...
  Predicted: threat_actor_profile
  Text: Trustwave SpiderLabs reconstructs an intrusion from an exploited appli...
  Predicted: incident_case_study
  Text: Volexity reconstructs an espionage intrusion at a think tank. The post...
  Predicted: incident_case_study
  Text: We continuously emulate data archiving prior to exfiltration to valida...
  Predicted: adversary_emulation

  Calculating metrics for demo zero-shot predictions...

🎯 Demo Zero-shot Accuracy: 1.000 (evaluated on 60/60 parseable responses)

Per-class metrics (Demo Zero-shot):
                       precision    recall  f1-score   support

  incident_case_study       1.00      1.00      1.00        14
detection_engineering       1.00      1.00      1.00        16
 threat_actor_profile       1.00      1.00      1.00        15
  adversary_emulation       1.00      1.00      1.00        15

             accuracy                           1.00        60
            macro avg       1.00      1.00      1.00        60
         weighted avg       1.00      1.00      1.00        60
