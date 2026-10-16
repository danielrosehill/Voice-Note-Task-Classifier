# Development plan

Proof-of-concept scale: one developer, a CPU-only laptop, GPU time on Modal, a few
dollars of LLM batch calls per phase. Picks up where v0.1.0 left off (see
[lineage.md](lineage.md)).

## Current position

**2026-10-16 — phase 0 (spec).** Repo created; spec, taxonomy v0.2.0 and this plan
written. No code, no data, no model yet in this repo. The v0.1.0 placeholder on
Hugging Face
([danielrosehill/Voice-Note-Intent-Classifier](https://huggingface.co/danielrosehill/Voice-Note-Intent-Classifier),
micro F1 0.56) is the baseline.

**Next step:** phase 1 — hand-label the public screened notes against taxonomy
v0.2.0 and run the rung-0 LLM router on them.

## Phases

| # | Phase | Output | Exit criterion |
|---|---|---|---|
| 0 | Spec | `docs/spec.md`, `taxonomy/labels.json` v0.2.0 | Route/tag split agreed; each route has a named subagent target |
| 1 | Gold eval + rung 0 | Hand labels for the public screened notes; rung-0 LLM router scores | Every note in `Voice-Notes-Screened` (65 rows on 2026-10-16) hand-labelled; rung-0 per-label P/R/F1, latency and cost recorded |
| 2 | Synthetic data v3 | Public synthetic dataset on HF | Quality controls below passed; length distribution matches real notes (median within ±20%) |
| 3 | Train the ladder | Rungs 1–3 trained against v0.2.0 | Each rung's results JSON in `results/` |
| 4 | Decide | Comparison table rung 0 vs rungs 1–3 | Decision recorded in `docs/spec.md` per the rule in [spec.md](spec.md#decision-does-mcp-make-this-unnecessary) |
| 5 | Release v0.2.0 | HF model `danielrosehill/Voice-Note-Task-Classifier`: safetensors + ONNX + card | Card reports gold-eval scores, not synthetic-test scores; v0.1.0 card points here |
| 6 | Retrain path | One command: changed `labels.json` → regenerate deltas → retrain → eval → checkpoint | A label added end-to-end in under an hour of wall time |
| 7 | Integrate | Router consuming the contract in [spec.md](spec.md#io-contract) | A real voice note becomes the right draft in the target service |

### Phase 1 — gold eval and rung 0

v0.1.0's real labels were silver (two-LLM agreement, 63% identical), so its 0.56 is
soft. Hand labels fix that. Because the screened notes are already public and
approved, their hand labels can be published too, alongside the notes or as a
separate labels file keyed by `id` (decide at the time; a labels file avoids
touching the screened dataset).

- 65 notes is small. Report per-label support and confidence intervals, and grow the
  set as the author approves more notes into `Voice-Notes-Screened`.
- `calendar_event` had zero real examples in v0.1.0. If it is still absent, it is
  scored on a small hand-written held-out set and reported separately, flagged as
  synthetic.
- Rung 0 is a prompted LLM router: the taxonomy (definitions + confusable pairs)
  in a system prompt, JSON-schema output matching the contract. Run it with at
  least one small model (Haiku-class) and one cheap open model via OpenRouter.
  Record F1, p50/p95 latency and cost per 1,000 notes.

### Phase 2 — synthetic data v3

Generated from scratch; v0.1.0's synthetic sets are not reused (see
[lineage.md](lineage.md#what-was-deliberately-not-carried-over)).

- **Seeded, then extrapolated.** A few examples per label — written by hand, or
  taken from the public screened notes — seed the generator; the bulk is
  extrapolated. Names, places and organisations in the output are invented, so the
  model learns `email_draft` from "send Priya the invoice", not from a real contact
  list. This is the reason for synthetic data: real routing notes carry real names.
- **Length-matched.** v0.1.0's synthetic v1 had a median of 27 words against 103 for
  real notes, and the gap cost accuracy. Sample target lengths from the real
  distribution, including multi-minute notes.
- **ASR style.** Fillers, missing punctuation, self-corrections, numbers as words,
  misheard words.
- **Compound notes** with two or three labels, in either order, and **no-label
  notes**, so empty output is learned, not defaulted to.
- **Hard negatives** built on every confusable pair in `labels.json`, especially the
  new `dev_spec` boundaries.

Quality controls (from v0.1.0, which took them from the Hebrew-Latin-Token-Classifier
pipeline):

- Generator and annotator from **different model families**; the annotator labels
  blind. Keep a row only where the annotator agrees with the requested labels.
- **10% random audit of agreements** by a human. Disagreement rate says nothing
  about how wrong the agreeing majority is.
- **Near-duplicate removal** by embedding similarity before splitting.
- **Template-leakage check:** no label-revealing phrase ("shopping list") in more
  than a set share of a class, or the model learns the cue.
- **Word-boundary matching** in any automatic label check, never substring.

Dataset row shape is in [spec.md](spec.md#training-data-row).

### Phase 3 — the ladder

| Rung | Model | Where | Why |
|---|---|---|---|
| 1 | Frozen encoder + per-label logistic head (v0.1.0 recipe, retrained on v0.2.0) | Laptop CPU, seconds | Isolates the taxonomy + data change from the model change |
| 2 | SetFit on a small sentence encoder | Laptop CPU or Modal | Contrastive fine-tune; cheap step up from frozen |
| 3 | Full fine-tune of ModernBERT-base (or a same-size alternative) | Modal GPU | The main candidate |

Train each rung on synthetic only and on synthetic + real (cross-validated), since
v0.1.0 found real data adds about 0.08 F1. Thresholds are tuned per label on real
notes, never on the test fold.

### Phase 6 — checkpointed retraining

The label set will change as subagents are added. Requirements:

- Every checkpoint records `taxonomy_version`, the training-set hash and the
  thresholds file.
- Adding a label generates synthetic rows for the new label plus hard negatives
  against its confusable neighbours, and **re-annotates existing rows for the new
  label** (an existing row may now also be a `meeting_notes`); it does not regenerate
  the whole set.
- Rung 1 retrains in seconds and can ship as an interim checkpoint while rung 3
  retrains.

## Beyond the POC

Recorded so they are decisions, not omissions. None is in scope until phase 5 ships.

- **Per-route extractors.** Each subagent could get its own small model that turns
  the transcript into the structured payload for its MCP tool, e.g. a `todo` note
  "today I need to buy bananas, tomorrow apples" →
  `[{"date": "today", "task": "buy bananas"}, {"date": "tomorrow", "task": "buy apples"}]`.
  The last leg is then plain structured output to an MCP call, as close to
  fail-safe as it gets. Until then, the subagents use an LLM with a JSON schema.
- **Parallel classifiers on the same transcript.** PII detection and a sensitivity
  level, so routing can also decide where a note is allowed to go.
- **Audio-token classifier head.** A speech model with a transcription head and a
  classification head. Deferred: routing depends on what was said, the transcript
  already exists, and it couples the router to one ASR model.
- **Hebrew and English–Hebrew code-switched notes.** After the English model works.
