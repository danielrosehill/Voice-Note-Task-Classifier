# Spec

Status: draft, 2026-10-16. Taxonomy v0.2.0. Nothing in this repo is built yet.

## Where it sits

```
voice app ─► ASR ─► transcript ─► [ task classifier ] ─► router ─► one subagent per route label
 (capture)  (any)   (raw text)       this repo                     each: role prompt + one MCP tool
                                          │
                                          └─► tags (metadata only, stored with the note)
```

The classifier decides **where** a note goes. It does not extract fields (recipient,
date, items) and it does not act. Each subagent receives the whole transcript, its
own role instruction and its own MCP tool, and extracts only what concerns it.

## I/O contract

**Input:** one transcript, as produced by ASR. Must tolerate raw ASR output: no
guaranteed punctuation or casing, fillers, false starts, self-corrections,
misrecognised words, run-ons, and lengths from three words to several minutes.

```json
{
  "transcript": "ok so two things email the landlord about the boiler and uh dentist thursday at four thirty",
  "recorded_at": "2026-10-16T08:12:00Z",
  "source": "voicenotes"
}
```

Only `transcript` is required.

**Output:**

```json
{
  "model": "danielrosehill/Voice-Note-Task-Classifier",
  "model_version": "0.2.0",
  "taxonomy_version": "0.2.0",
  "routes": ["email_draft", "calendar_event"],
  "tags": [],
  "scores": {
    "shopping_list": 0.02, "todo": 0.31, "calendar_event": 0.91, "email_draft": 0.88,
    "content_draft": 0.01, "meeting_notes": 0.03, "ai_prompt": 0.06, "dev_spec": 0.01,
    "note_to_self": 0.07, "idea": 0.01, "brainstorm": 0.02
  },
  "thresholds_version": "0.2.0-cv5",
  "abstain": false
}
```

| Field | Type | Meaning |
|---|---|---|
| `routes` | array of route-label enum, may be empty | Route labels whose score cleared their threshold. The router dispatches once per element. |
| `tags` | array of tag-label enum, may be empty | Tag labels that cleared their threshold. Never dispatched. |
| `scores` | object, every label in the taxonomy | Independent sigmoid scores, 0–1. Not a distribution: they do not sum to 1. |
| `abstain` | bool | `true` when the model is unsure. See below. |
| `taxonomy_version` | semver | The label set the model was trained on. A router built against another major version must refuse the prediction rather than misroute. |

### Nullability and multi-label semantics

- **Multi-label.** Each label is an independent yes/no (one sigmoid per label, no
  softmax). "Add bread to the list and the dentist is Thursday at four" is
  `shopping_list` + `calendar_event`. Splitting compound notes into segments
  before classification was considered and rejected for the POC: it adds a model
  whose errors compound, and each subagent can ignore the irrelevant part cheaply.
- **Nullable.** `routes: []` is a valid, confident answer: the note has nothing to
  dispatch (a musing, a note to self) and is filed to a default inbox with its
  tags. It is not an error.
- **Abstain is not null.** `abstain: true` means the model cannot tell. It is set
  when no route clears its threshold **and** at least one route sits in an
  uncertainty band below it (band width calibrated on real notes). The router
  then falls back to the rung-0 LLM router or asks the user. Abstaining is cheaper
  than a confident wrong route.
- **Tags do not affect abstain.** A note can be `routes: []`, `tags: ["brainstorm"]`,
  `abstain: false`.
- **Per-label thresholds**, calibrated on real notes by cross-validation, not a fixed
  0.5. Shipped as a versioned file next to the weights so they can be re-tuned
  without retraining.

## Labels

Source of truth: [`taxonomy/labels.json`](../taxonomy/labels.json) (definitions,
confusable-pair rules, candidates).

| id | Kind | Downstream | Example (invented, ASR-style) |
|---|---|---|---|
| `shopping_list` | route | lists.shopping → list MCP | "need to get uh dish soap paper towels and batteries double a" |
| `todo` | route | tasks → to-do MCP | "today i need to buy bananas tomorrow apples" |
| `calendar_event` | route | calendar → calendar MCP | "dentist thursday the twentieth at four thirty" |
| `email_draft` | route | email.draft → email MCP (draft only) | "email to priya subject the invoice hi priya just following up…" |
| `content_draft` | route | documents.draft → drive | "blog post outline why i stopped using notion first point…" |
| `meeting_notes` | route | documents.meeting → drive | "notes from the call with the contractor they can start in march…" |
| `ai_prompt` | route | ai.prompt → agent runner | "go through this repo and replace every hard coded api url with an env variable" |
| `dev_spec` | route | documents.spec → drive project folder | "the idea is a small model that sits behind the transcription and decides…" |
| `note_to_self` | tag | — | "the spare key is in the blue box on the top shelf" |
| `idea` | tag | — | "idea a browser extension that summarises terms of service" |
| `brainstorm` | tag | — | "so i'm thinking about the backup thing two options one is rsync but then versioning…" |

**Why tags.** One pass gives both the routing decision and loose categorisation.
Tags are deliberately looser: lower precision is tolerable because nothing acts on
them. A tag becomes a route (minor version bump) once a subagent exists for it.

**Candidates** not adopted yet: `message_draft`, `reminder`, `journal`,
`research_query`, `expense_log`, `contact_info`. Adoption rule (from v0.1.0): a
candidate becomes a route if it covers a meaningful share of real notes **and**
goes somewhere no existing route goes; otherwise it is a tag or a sub-type.

**Not labels.** Priority, "contains names", "work vs personal", time-sensitivity.
Those are attributes for a subagent to extract, or the job of a separate classifier
(see [development-plan.md](development-plan.md#beyond-the-poc)).

### Versioning

- `taxonomy_version` is semver on `labels.json`. Adding a label, or promoting a tag
  to a route, is **minor**. Removing or redefining a label, or demoting a route to a
  tag, is **major**: it breaks routers and invalidates existing labels.
- `model_version` is semver on the released model; each release names the taxonomy
  it was trained on.
- Pre-1.0 the taxonomy is a draft and v0.1.0 → v0.2.0 is treated as minor although
  it changes label kinds. 1.0.0 freezes the contract.

## Decisions

Carried from v0.1.0 and still held:

- **Multi-label with per-label thresholds and abstain.** Above.
- **Misroutes are bounded by the subagents.** Subagents create drafts, not sends: an
  email is a draft, an event can be reviewed. A wrong route produces a visible,
  reversible artefact. This is what makes a small, fast, sometimes-wrong classifier
  acceptable in front of real services.
- **Text, not audio.** Routing depends on what was said; the transcript already
  exists; text-in keeps the classifier independent of the ASR model. An audio-token
  model with a transcription head and a classification head is deferred past the
  POC.
- **English only** for the POC. Hebrew and English–Hebrew code-switching later.
- **Synthetic training data, real evaluation data.** Real routing notes carry real
  names ("send ⟨name⟩ an email"), so training data is generated, seeded from a few
  examples, with invented names. Real notes judge the model; a model scored only on
  synthetic test data measures how well it learned the generator.

### Decision: does MCP make this unnecessary?

**The question.** An agent that receives transcripts with a prompt like "you are a
task-routing agent; identify the purpose of this note and invoke the right MCP"
needs no separate classifier. As MCP tooling has matured, that agent is easy to
build. So why train a model?

**Decision (2026-10-16): build both, and let the numbers choose.** The prompted LLM
router is **rung 0**: built first, run on the same gold eval, and kept permanently as
the fallback for `abstain`. The trained classifier is pursued because it may make
routing more robust, and robustness is testable:

| Property | Prompted LLM router (rung 0) | Trained classifier |
|---|---|---|
| Accuracy on gold eval | Measured in phase 1 | Measured in phase 4 |
| Latency per note | Seconds (one LLM call) | Milliseconds on CPU |
| Cost per 1,000 notes | Tokens × price, measured | ~0 marginal |
| Determinism | Same note can route differently on two calls | Same input, same output |
| Calibrated scores / abstain | Self-reported confidence, uncalibrated | Per-label sigmoid, thresholds tuned on real data |
| Versioned contract | Prompt text; drift is silent | `taxonomy_version` on every prediction; retrain is a checkpoint |
| Offline / on-device | No | Yes (ONNX) |
| Novel intents, no training data | Yes: edit the prompt | No: needs data and a retrain |
| Subagent isolation | Router sees every tool | Router sees no tools; each subagent sees one |

The last row is the architectural argument: with a classifier in front, no agent
ever holds the whole tool catalogue. Each subagent is narrow (one role, one tool),
which is where tool-selection failures are rarest — tool-retrieval errors are a
large share of agent failures in published MCP benchmarks
([research.md](research.md#5-small-classifier-vs-prompted-llm-and-tool-selection)).

**Decision rule, applied in phase 4:**

1. If the best trained rung is within **0.05 micro F1** of rung 0 on the gold eval
   (or better), ship the classifier as the primary router, rung 0 as the abstain
   fallback.
2. If it is more than 0.05 behind but its high-confidence predictions (above
   threshold, no abstain) are at least as precise as rung 0, ship it as a **cascade**:
   classifier first, LLM for everything it abstains on. Report the abstain rate.
3. Otherwise the LLM router is the answer, the classifier is shelved as a finding,
   and the model card says so.

The 0.05 margin is a judgement call for a POC with ~65 gold notes, where a single
note moves per-label F1 noticeably; revisit when the gold set is larger.

## Model candidates

Sizes confirmed from the model cards on 2026-10-16 ([research.md](research.md#3-small-encoders-for-multi-label-classification)).

| Candidate | Params | Context | Role | Concern |
|---|---|---|---|---|
| `BAAI/bge-small-en-v1.5` frozen + logistic | 33.4M | 512 | Rung 1 retrain; SetFit base (rung 2) | Truncates long notes |
| `BAAI/bge-base-en-v1.5` frozen + logistic | ~110M (card: "0.1B") | 512 | Rung 1; the v0.1.0 recipe | Frozen tops out ~0.55 in v0.1.0 |
| `answerdotai/ModernBERT-base` fine-tune | 149M | 8,192 | **Rung 3, main candidate** | At the top of the 150M budget |
| `jhu-clsp/ettin-encoder-68m` / `-150m` fine-tune | 68M / 150M | 8K | Rung 3 alternative; 68M if ModernBERT-size is not needed | Newer, less community evidence |
| `microsoft/deberta-v3-small` fine-tune | 44M backbone + 98M embeddings (~142M) | 512 | Rung 3 alternative | 512 tokens |
| `fastino-ai` GLiNER2 (small 74M … large 340M) | 74M–340M | — | Zero-shot comparison; later, labels + fields in one pass | No published CPU latency |
| `jhu-clsp/mmBERT-small` | ~140M (42M non-embedding) | 8,192 | Only if Hebrew/code-switching enters scope | Multilingual overhead for English-only |

Size budget: ≤150M parameters (CPU-server now, on-device via ONNX later). Long
context matters: real notes had a median of ~103 words in v0.1.0, and long specs and
prompts run to minutes, which is past 512 tokens. Assumption to test in phase 3: the
routing signal sits in the first 512 tokens. If it does, the 512-token models are
viable; if not, ModernBERT or Ettin.

## Evaluation

**Gold set:** hand labels (multi-label, taxonomy v0.2.0) on every note in
[`danielrosehill/Voice-Notes-Screened`](https://huggingface.co/datasets/danielrosehill/Voice-Notes-Screened)
(65 rows on 2026-10-16), growing as more notes are approved. Labels for routes and
tags are recorded separately.

**Metrics** (reported for routes and tags separately; routes are the headline):

- Micro and macro precision / recall / F1; per-label P/R/F1 with support.
- Exact-match rate of the route set.
- **Null handling:** share of no-route notes returned as `routes: []` without
  abstaining; abstain rate overall.
- **Misroute rate:** share of notes dispatched to at least one wrong route. This is
  the user-facing cost.
- Latency p50/p95 on the laptop CPU, and cost per 1,000 notes (for rung 0).
- Bootstrap 95% confidence intervals: with 65 notes they will be wide, and the
  README must show them.

**Bars to clear:**

| Bar | Value | Source |
|---|---|---|
| v0.1.0 baseline | micro F1 **0.56**, macro F1 0.48 | 5-fold CV on 226 screened real notes, silver labels ([lineage.md](lineage.md#v010-results-summary-aggregate-only)) |
| Rung 0 | measured in phase 1 | Prompted LLM router, same gold set |

The 0.56 was measured on silver labels over a different set and the old 9-label
taxonomy, so it is not directly comparable. To make it comparable, the v0.1.0
model is re-scored on the new gold set over the labels the two taxonomies share
(`shopping_list`, `todo`, `calendar_event`, `email_draft`, `content_draft`,
`ai_prompt`, `note_to_self`, `idea`, `brainstorm`). A new model must beat **that**
re-scored number on the shared labels, and the rung-0 rule above on all labels.

## Training data row

```json
{
  "id": "syn-v3-000123",
  "text": "uh need to grab batteries and um the car's booked in friday at nine",
  "routes": ["shopping_list", "calendar_event"],
  "tags": [],
  "source": "synthetic",
  "seed": "hand-written",
  "generator": "<model id>",
  "annotator": "<model id>",
  "taxonomy_version": "0.2.0"
}
```
