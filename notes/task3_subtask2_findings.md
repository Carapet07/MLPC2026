# Task 3 — Subtask 2: Annotations — Findings & Methodology

## Dataset overview

- 3656 recordings, 15 sound classes
- Annotations stored per recording as `[T, C, A]` arrays: T time segments, C=15 classes, A annotators
- Segments: 1s window, 0.5s hop (overlapping)
- Annotation values are continuous `[0, 1]` overlap fractions — fraction of the 1s window the sound occupies
- Values are **independent per class** (not shared/distributed across classes — sum across classes can exceed 1)
- ~98% of values are exactly 0.0 (silence dominates)
- 1.6% of values are fractional (not strictly 0 or 1)
- Annotator count per recording: A=1 (625), A=2 (1750), A=3 (1235), A=4 (45), A=5 (1)
- 447 recordings have no owner annotator

---

## Subtask 2a — Annotator Agreement

### Naive approach (flawed)

Computed pairwise agreement as `1 - mean|a_i - a_j|` over all segments per (recording, class, annotator pair).

**Figure 1** — bar chart: mean naive agreement per class, sorted ascending.

Key values:
- `footsteps`: 0.929 (lowest)
- `bell_ringing`: 0.994 (highest)
- Overall mean: 0.980

**Why this is misleading:** ~98% of annotation values are 0. Two annotators trivially agree on silence, inflating the metric. The overall mean of 0.980 is barely better than what two random annotators would achieve by chance alone (also ~0.98 given the silence base rate). The mean of class means was also taken directly from already-grouped values — valid here only because all 15 classes have the same observation count, but poor practice.

### Revised approach — conditional agreement

Recomputed agreement using only segments where `(a_i > 0) | (a_j > 0)` — at least one annotator marked the sound as present. This excludes trivial mutual-silence cases.

**Figure 2** — bar chart: conditional mean agreement per class.

Overall conditional mean: **0.740** (vs 0.980 naive) — a more honest reflection of actual annotation difficulty. Overall mean computed directly from all rows, not from class means.

---

## Subtask 2b — Converting Annotations to Labels

### Method (original)

Collapsed `[T, C, A]` → `[T, C]` binary label matrix per recording using owner-weighted mean + threshold at 0.5.

- Owner annotator (`is_own_recording=True`) gets weight **2**, others get weight **1**
- Aggregated value: `(2·a_own + Σa_k) / (2 + A - 1)` when owner is present, `Σa_k / A` otherwise (all weights = 1)
- Binarized at 0.5

**Rationale for 0.5 threshold:** chosen as the natural majority threshold — if the aggregated value leans toward present, label it 1. The assumption was that annotation values would be mostly 0 or 1 (sound absent or fully present in a window), making 0.5 a clean separator. The 1.6% of fractional values were acknowledged as edge cases from sounds partially overlapping a window boundary and not treated as a design concern.

**Rationale for owner weighting — knowledge advantage:** the recording collector has direct knowledge of their own recording and is most likely to annotate it accurately. Weight=2 protects the owner's vote from being overridden by a single external annotator. It is the boundary case at A=3 — one disagreeing external annotator cannot override the owner, but two can. At A≥4 a sufficient majority of external annotators can outvote the owner.

**Rationale for owner weighting — careless annotator resilience:** a careless annotator contaminates both their own recordings (as owner) and other people's recordings (as non-owner). Falling back to a plain mean would not escape this — their votes are present either way. The owner weighting ensures that on a given recording, the person with the most context is not easily outvoted by someone who may be annotating carelessly across the board.

See notebook markdown cell (beafaeba) for full methodology, edge cases, and conditions under which labels can be incorrect.

### Sanity check — target class coverage

For each recording, every class in `target_classes` should have at least one positively labeled segment.

**Original labels:** 967 / 10214 (recording, target class) pairs flagged (9.5%)

Flagged breakdown (original):
| Class | Flagged |
|---|---|
| light_switch | 485 |
| footsteps | 109 |
| door_open_close | 85 |
| wardrobe_drawer_open_close | 50 |
| cutlery_dishes | 46 |
| (others) | 192 |

### Investigation — why light_switch fails (65.6% flag rate)

`light_switch` had 485 flags out of 739 total target recordings (65.6%) — a systematic failure, not noise.

**Annotation value analysis (Figure 3 — 6-class value distributions):**
`light_switch` has a spike at 1.0 (like sustained classes `keyboard_typing`, `toilet_flushing`) but also a significant local accumulation between 0.2 and 0.4 that is absent in sustained classes. The 1.0 spike corresponds to segments where the annotated window fully covers the 1s frame — likely where annotators drew wider windows. The 0.2–0.4 cluster corresponds to edge segments where the brief click only partially overlaps the window, explained by the overlapping window geometry: a brief sound spreads across 2–3 adjacent windows with small overlap fractions in each.

**Temporal misalignment check:**
Of flagged `light_switch` recordings with ≥2 annotators noticing it (363 total):
- 56.5% peaked in the same segment
- 43.5% peaked in different segments

→ Misalignment exists but is not the dominant cause.

**Span analysis:**
Active annotators marked a median of 3 segments with mean value ~0.275 within those segments. This is consistent with the window geometry — a brief sound spreads across ~3 overlapping segments with low per-segment overlap fractions.

Contiguity check: 80% of annotator spans were contiguous (single coherent block), confirming annotators were drawing deliberate windows, not random marks.

**Owner/non-owner breakdown for flagged light_switch:**
- 70.9% of flags: both owner and non-owner noticed it but values were still too low
- 13.4%: only owner noticed
- 2.9%: only non-owner noticed
- 9.5%: no owner annotator
- 3.3%: neither noticed

→ The dominant failure is not disagreement on presence but values being structurally too low to survive the 0.5 threshold.

### Investigation — why footsteps fails (6.3% flag rate)

109 flagged out of 1720 total (6.3%) — much less severe than light_switch.

**Figure 4** — annotation value distribution for `footsteps`: spikes at 1.0, similar to sustained classes. Duration is not the problem.

Owner/non-owner breakdown for flagged footsteps:
- 42.2%: neither annotator noticed anything
- 32.1%: non-owner noticed, owner did not
- 10.1%: owner noticed, non-owner did not
- 10.1%: no owner annotator
- 5.5%: both noticed but still flagged

→ Footsteps fails due to **annotation inconsistency**, not acoustic difficulty. Owners deliberately recorded and declared footsteps in their metadata — they knew the sound was there. The 42.2% "neither noticed" and 32.1% "non-owner noticed, owner didn't" cases are therefore surprising. Likely explanations: the owner annotated carelessly during the session, forgot to mark their own target class, or footsteps occur as multiple short bursts spread across the recording making complete annotation tedious. This is an annotation quality issue rather than anything fixable at the labeling stage.

### Annotation value distributions and span analysis — all 15 classes

**Figure 5** — bar chart: median annotation span per class (sorted ascending), with mean overlaid as scatter points.

Classes sorted by median span (segments):
`light_switch` (3) → `door_open_close`, `wardrobe_drawer_open_close` (4) → `window_open_close`, `bell_ringing`, `cutlery_dishes`, `keychain` (5–7) → `footsteps`, `phone_ringing` (8) → `toilet_flushing` (13) → `keyboard_typing` (14) → `running_water` (18) → `coffee_machine` (19) → `microwave` (25) → `vacuum_cleaner` (33)

**Key observation:** span length correlates directly with the physical brevity of the sound — transient classes occupy fewer contiguous segments than sustained ones. Mean > median for most classes indicates right-skewed span distributions.

**Figure 6** — 3×5 grid of non-zero annotation value histograms per class, sorted in the same order as Figure 5.

**Key observation:** all classes spike at 1.0. The difference is in the floor below 1.0 — briefer classes have a higher and more noticeable density there. `light_switch` is the only class with truly significant density below 1.0, with a distinct local peak between 0.2 and 0.4 alongside the 1.0 spike. This directly justifies the per-annotator binarization threshold of 0.1: genuine annotations clearly cluster well above 0.1 even for `light_switch`, confirming that any value ≥0.1 represents a deliberate annotation of sound presence.

### Revised method — per-annotator binarization (labels_v2)

**Problem with original method:** conflates two questions — *was the sound present?* and *how much of the window did it occupy?* For brief sounds, the second question systematically penalizes the first.

**Fix:** binarize each annotator's values independently at threshold 0.1 before aggregation, then apply owner-weighted mean on binary votes.

Threshold 0.1 chosen because:
- Physically meaningful: sound present for ≥100ms of window
- Non-zero annotation values cluster well above 0.1 for all classes (annotation interface requires deliberate drag gestures)
- Edge bleed at 0.1 adds at most one extra boundary segment — acceptable

```
votes      = (ann >= 0.1)              # [T, C, A] binary
aggregated = (votes * weights).sum(axis=2) / weights.sum()  # [T, C]
labels_v2  = (aggregated >= 0.5)       # [T, C] binary
```

Owner weighting (weight=2) is fully preserved.

In `labels_v2` the 0.5 threshold has a cleaner justification than in the original: it is now explicitly a weighted majority vote on binary presence decisions. A value ≥0.5 means the weighted majority of annotators agreed the sound was present in that segment — more principled than the original where 0.5 was a midpoint on a continuous overlap scale.

**Result:**
| Method | Flagged pairs | Flag rate |
|---|---|---|
| labels (original) | 967 | 9.5% |
| labels_v2 (revised) | 366 | 3.6% |

Improvement is concentrated in transient classes:
- `light_switch`: 485 → 36 (near-complete fix)
- `door_open_close`: 85 → 40
- `wardrobe_drawer_open_close`: 50 → 20
- `footsteps`: 109 → 89 (minimal change — genuine disagreement, not fixable by threshold)

All 366 remaining flags in `labels_v2` are accounted for:

| Category | Count | % of flags |
|---|---|---|
| Neither noticed (all < 0.1) | 164 | 44.8% |
| Non-owner noticed, owner did not | 104 | 28.4% |
| A=1 (no consensus possible) | 78 | 21.3% |
| No owner annotator | 18 | 4.9% |
| Owner noticed, outvoted by non-owners | 2 | 0.5% |

The 104 "non-owner noticed, owner did not" cases represent only 1.15% of all 9020 owner-present (recording, target class) pairs. Given the footsteps finding that owners can be inconsistent annotators, some of these may reflect genuine owner annotation errors — but at 1.15% this is an acceptable rate and does not undermine the owner weighting design.

---

## Key limitations to mention in report

1. **Single annotator recordings (A=1, 625 recordings):** no consensus possible, label entirely depends on one person
2. **No-owner recordings (447):** owner reliability advantage disappears, labels reduce to plain mean
3. **Footsteps annotation inconsistency:** owners deliberately recorded and declared footsteps in metadata yet still failed to annotate them in many cases — likely due to careless or incomplete annotation sessions rather than acoustic ambiguity. Unrecoverable at the labeling stage.
4. **Threshold edge cases:** sounds beginning/ending mid-window may be partially missed; very short sounds near the 0.1 threshold boundary are borderline
5. **Conditional agreement metric:** excludes mutual-silence which is the trivial case; the 0.740 figure reflects real disagreement on active segments only
