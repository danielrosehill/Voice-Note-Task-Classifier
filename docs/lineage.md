# Lineage

This repo is the **public successor** of the v0.1.0 voice-note intent classifier
work. The v0.1.0 work was done in an earlier private working repo, which stays
private and is not linked from here. Decided by the author on 2026-10-16: new
public repo, the private one is frozen with a pointer to this one.

Everything that is public from the v0.1.0 work is on Hugging Face and is linked from
the [README](../README.md).

## What was carried over

| Item | From v0.1.0 | Here |
|---|---|---|
| Problem framing | Text classifier between ASR and a router of task-specific subagents, each holding one MCP connection | Same. Restated from the 2026-10-16 voice note in [`context/`](../context/) |
| Output contract | Per-label sigmoid scores, labels above per-label thresholds, `abstain` flag, `taxonomy_version` on every prediction | Same, plus routes and tags reported separately. [spec.md](spec.md#io-contract) |
| Taxonomy | 9 flat labels, 8 candidates, confusable-pair rules | Adapted to v0.2.0: route/tag split, `meeting_notes` and `dev_spec` added. [labels.json](../taxonomy/labels.json) |
| Design decisions | Multi-label not single-label; per-label thresholds; abstain to an LLM fallback; misroutes bounded by draft-only subagents; text not audio; English only | All kept. [spec.md](spec.md#decisions) |
| Data approach | Synthetic training data, real screened notes for evaluation; generator and annotator from different model families, blind annotation, 10% audit of agreements, near-duplicate removal, template-leakage check | Kept. [development-plan.md](development-plan.md) |
| Results | v0.1.0 placeholder: frozen `BAAI/bge-base-en-v1.5` + per-label logistic head; micro F1 0.56 on real notes | The baseline to beat. [spec.md](spec.md#evaluation) |
| Experiment findings | Synthetic v1 too short (median 27 words vs 103 for real notes); adding real notes to training adds about 0.08 micro F1; a frozen encoder tops out around 0.55 | Inform phases 2–3 of the plan |
| Model candidates | SetFit on bge-small, ModernBERT-base fine-tune on Modal | Kept and extended. [spec.md](spec.md#model-candidates) |

## v0.1.0 results summary (aggregate only)

Evaluation: 5-fold cross-validation, out-of-fold, on 226 real notes that had passed a
privacy screen. Each fold's head was trained on 1,492 synthetic rows plus the other
four real folds. Labels on the real notes were **silver**: agreement between two LLM
families, not hand labels. The two LLMs gave identical label sets on only 63% of the
real notes, so treat these as a rough guide.

| Metric | v0.1.0 |
|---|---|
| Micro P / R / F1 | 0.50 / 0.64 / **0.56** |
| Macro F1 (labels present) | 0.48 |
| Exact match | 0.39 |
| No-intent notes correctly abstained | 10 / 26 |

| Label | F1 |
|---|---|
| `shopping_list` | 0.78 |
| `ai_prompt` | 0.76 |
| `email_draft` | 0.50 |
| `brainstorm` | 0.45 |
| `todo` | 0.41 |
| `idea` | 0.36 |
| `content_draft` | 0.35 |
| `note_to_self` | 0.23 |
| `calendar_event` | unscored (no real examples) |

Encoder comparison from the PoC experiments (same protocol, synthetic + real
training, thresholds tuned on real): `bge-small-en-v1.5` micro F1 0.52,
`bge-base-en-v1.5` 0.55. Synthetic-only training scored 0.43–0.48.

## What changed

- **Repo is public from the start.** No real voice-note text lives here. Real notes
  are only used via the public, author-approved
  [Voice-Notes-Screened](https://huggingface.co/datasets/danielrosehill/Voice-Notes-Screened)
  dataset.
- **Route vs tag.** v0.1.0 gave every label a route. v0.2.0 makes `note_to_self`,
  `idea` and `brainstorm` tag-only: they are emitted for categorisation but never
  dispatched.
- **New routes.** `meeting_notes` (was a candidate) and `dev_spec` (new; the voice
  note that started this repo is itself one).
- **Rung 0 is explicit.** v0.1.0 planned an LLM zero-shot baseline but never ran
  it. Here it is phase 1 and the deciding comparison — see
  [spec.md](spec.md#decision-does-mcp-make-this-unnecessary).
- **Retraining is a first-class requirement.** The label set will change; the plan
  makes a retrain from a changed `labels.json` a scripted, checkpointed step.
- **Name.** "Task classifier" rather than "intent classifier": the labels name the
  task the downstream agent performs.

## What was deliberately not carried over

- Any real voice-note text, examples drawn from real notes, per-note labels, or
  privacy-screen outputs.
- Corpus statistics that describe personal content categories.
- The private training data and generation logs. The synthetic sets from v0.1.0
  were generated with few-shot context that is not cleared for release, so new
  synthetic data is generated here from scratch.
- Data-handling scripts that read the private export. The scripts here will be
  rewritten against the public screened dataset only.
