# Planning — IR / MITRE ATT&CK Writeup Classification Dataset

## 1. Goal
Build an annotated dataset to fine-tune / evaluate an LLM that classifies **cybersecurity community writeups in the Incident Response + MITRE ATT&CK space** by the author's *purpose*. Target: ≥200 examples, mutually-exclusive labels, ≥90% coverage with no "other" bucket.

## 2. Design decisions
| Decision | Choice | Rationale |
|---|---|---|
| Taxonomy axis | **Writeup purpose** (author intent) | How the DFIR/CTI/detection community actually organizes content (blog categories, con tracks, roles). Stays mutually exclusive even when a post spans many ATT&CK techniques. |
| Number of labels | **4** | Within the 2–4 requirement; each maps to a real, distinct practitioner workflow. |
| Output format | **CSV** | Spreadsheet-friendly; easy to extend and split. |
| Size | **400 rows** (case 94 / detection 105 / threat-actor 101 / emulation 100) | Roughly balanced classes, well past the 200 minimum. |
| Grounding | `grounding` column: **15 verified** (real fetched articles) + **385 representative** | Verified rows give a real-data eval subset; representative rows scale the labeling pattern. |
| Dates | `date` column (2025–2026) | Keeps the set anchored to current "latest" discourse. |

## 3. Label taxonomy — definitions

### A. `incident_case_study`
**Definition:** A forensic walkthrough of **one specific intrusion** — a narrative timeline reconstructed from evidence, host by host.
**Include:** "the intrusion began…", dwell time, kill-chain reconstruction, supporting artifacts, "our IR team responded to…".
**Exclude:** generic actor overviews (→ profile); posts that only share a detection (→ detection eng).
**Sources:** The DFIR Report, Mandiant M-Trends, Unit 42 IR, Cisco Talos IR, Sophos X-Ops, Huntress.

### B. `detection_engineering`
**Definition:** A post whose **primary deliverable is detection logic** — a rule, query, or analytic and how to tune it.
**Include:** Sigma/KQL/SPL/EQL, logsource, data sources, false positives, tuning, "this rule detects…".
**Exclude:** detections that appear as a closing section of an incident report (→ case study); exercises that *test* detections (→ emulation).
**Sources:** SOC Prime, Splunk Threat Research, Elastic Security Labs, Red Canary, SigmaHQ, SpecterOps.

### C. `threat_actor_profile`
**Definition:** CTI characterizing a **named actor/group's recurring TTPs across campaigns** — not a single incident.
**Include:** group aliases, victimology, attribution, "across multiple intrusions", "the group has been observed…".
**Exclude:** single-incident timelines (→ case study); re-creating the group's TTPs to test defenses (→ emulation).
**Sources:** Mandiant, Unit 42, CrowdStrike, Microsoft Threat Intelligence, CISA advisories, Recorded Future.

### D. `adversary_emulation`
**Definition:** **Offensive validation** — emulating or simulating techniques to measure defensive coverage (purple/red team).
**Include:** Atomic Red Team, CALDERA, BAS, "we emulated…", safe payload, detection validation, cleanup steps.
**Exclude:** real intrusions (→ case study); the detection rule itself (→ detection eng).
**Sources:** Red Canary (Atomic Red Team), MITRE CALDERA / Engenuity CTID, Picus, Cymulate, SCYTHE, SANS.

## 4. Deconfliction rules (exactly one label per post)
1. **Case study vs. detection eng** — DFIR reports end with a "Detections" section but the *purpose* is reconstructing the intrusion → `incident_case_study`.
2. **Case study vs. profile** — about an *event* → case study; about an *actor across events* → profile.
3. **Profile vs. emulation** — *describing* a group's behavior → profile; *re-creating* it to test defenses → emulation.
4. **Detection eng vs. emulation** — deliverable is the *rule* → detection eng; deliverable is the *exercise/validation* → emulation.

## 5. Examples (one per label)

**`incident_case_study`** — *The DFIR Report*
> "This intrusion began in early March 2025 with a single successful RDP logon to an internet-exposed system, with no evidence of credential stuffing or brute forcing. The actor moved laterally within hours and deployed ransomware across the domain before exfiltration completed."
> ATT&CK: T1133, T1078, T1021.001, T1486

**`detection_engineering`** — *SOC Prime*
> "This Sigma rule detects suspicious rundll32 execution patterns associated with malicious DLL loading. We cover the logsource, the selection logic, expected false positives from legitimate software, and the ATT&CK mapping to T1218.011."
> ATT&CK: T1218.011, T1059.001

**`threat_actor_profile`** — *Mandiant (Google Cloud)*
> "This profile of UNC3944 (Scattered Spider) summarizes the group's tradecraft across 2025: vishing of help desks, SIM swapping, abuse of legitimate remote tooling, and rapid pivots to ransomware. We map their consistently observed techniques to ATT&CK."
> ATT&CK: T1566.004, T1219, T1486

**`adversary_emulation`** — *Atomic Red Team (Red Canary)*
> "This post walks through running the Atomic Red Team test for T1059.001 to validate PowerShell detection coverage. We show the atomic, the expected telemetry, and how to confirm your analytics fire as a purple-team exercise."
> ATT&CK: T1059.001

## 6. Coverage & out-of-scope
These four purposes cover >90% of IR+ATT&CK practitioner writeups with **no "other" bucket**. Deliberately out of scope: pure tooling release notes, framework-version announcements (e.g. ATT&CK v18), and opinion/policy pieces. If those become frequent, add a 5th `tooling_release` class.

## 7. Build process
1. Web research across the 4 categories → confirm real sources, actors, technique IDs (2025–2026).
2. Define taxonomy + deconfliction rules.
3. Fetch specific live articles and extract real excerpts/dates/technique IDs → **15 `verified` rows**.
4. Author balanced, varied `representative` examples per label (vary vector, actor, technique, sector, tooling) → **385 rows**.
5. Validate CSV: 400 unique IDs, single label per row, consistent 9-column schema, label balance, grounding split.

## 8. Deliverables
- `ir_attack_writeup_dataset.csv` — 400 labeled examples; columns `id,label,grounding,source,url,date,attack_tactics,attack_techniques,text`.
- `ir_attack_taxonomy_README.md` — taxonomy, deconfliction rules, verified-source list, honesty note.
- `planning.md` — this document.

## 9. Open follow-ups
- Expand the `verified` subset further (fetch + quote more live articles per label).
- Add a 5th `tooling_release` class if tooling/version posts become frequent.
- Stratified 85/15 train/validation split by `label`; optionally evaluate on `grounding == verified` only.
