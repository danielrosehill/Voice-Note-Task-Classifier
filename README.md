# Voice-Note-Task-Classifier

A small text classifier that reads a voice-note transcript and decides which
task-specific agent should handle it: email draft, calendar event, shopping list,
to-do, blog outline, meeting notes, AI prompt, development spec. It is the routing
layer at the top of a voice-productivity stack, between the ASR output and a set of
subagents that each hold one MCP tool.

**Status (2026-10-16): planning.** Spec and taxonomy v0.2.0 written; no model trained
in this repo yet. Next: hand-label the public screened notes and run the prompted-LLM
baseline ([plan](docs/development-plan.md#current-position)).

```
voice app ─► ASR ─► transcript ─► [ task classifier ] ─► routes: ["email_draft", "calendar_event"]
                                                         tags:   ["note_to_self"]
                                                            │  one dispatch per route
              ┌──────────────┬──────────────┬───────────────┼──────────────┐
              ▼              ▼              ▼               ▼              ▼
          shopping        to-do         calendar          email      dev spec / AI prompt …
          subagent       subagent       subagent         subagent       subagents
              │              │              │               │              │
          list MCP       tasks MCP     calendar MCP     email MCP     drive / agent runner
```

## Why a classifier and not just an agent with tools

An MCP-equipped agent prompted as a task router can do this without a separate
model. This project tests whether a small classifier makes routing more **robust**
— deterministic, calibrated, versioned, milliseconds on a CPU, offline-capable —
and keeps the subagents narrow: each gets one role prompt and one tool. The prompted
LLM router is built first as rung 0 and is the bar the classifier has to clear. See
[the decision](docs/spec.md#decision-does-mcp-make-this-unnecessary).

## Labels (taxonomy v0.3.0)

Multi-label and nullable: a note can carry several labels, or none.

| Kind | Labels |
|---|---|
| **Route** — dispatched to a subagent | `shopping_list` `todo` `calendar_event` `email_draft` `content_draft` `meeting_notes` `ai_prompt` `dev_spec` `contact_info` (0.3.0) |
| **Tag** — categorisation only, never dispatched | `note_to_self` `idea` `brainstorm` |

Definitions and boundary rules: [`taxonomy/labels.json`](taxonomy/labels.json).

## Lineage and Hugging Face

This is the public successor of the v0.1.0 work, which was done in an earlier
private working repo. What was carried over: [docs/lineage.md](docs/lineage.md).

| Resource | What |
|---|---|
| Model [`danielrosehill/Voice-Note-Intent-Classifier`](https://huggingface.co/danielrosehill/Voice-Note-Intent-Classifier) | v0.1.0 placeholder: frozen `bge-base-en-v1.5` + logistic head, micro F1 0.56 on real notes. The baseline to beat. |
| Dataset [`danielrosehill/Voice-To-Action`](https://huggingface.co/datasets/danielrosehill/Voice-To-Action) | Synthetic voice notes, one per label or label pair, read aloud by the author (recording started 2026-11-16; the dataset is created on the first sync). `prompt_kind` is the primary label, `prompt_note` the full label list. Staging data for testing the MCP routing app end to end. |
| Dataset [`danielrosehill/Voice-Notes-Screened`](https://huggingface.co/datasets/danielrosehill/Voice-Notes-Screened) | Real voice-note transcripts, privacy-screened and approved by hand by the author. The evaluation source. |

Both verified public on 2026-10-16. Releases from v0.2.0 on will go to a new model
repo, `danielrosehill/Voice-Note-Task-Classifier`, created at the first release.

## Docs

- [Spec](docs/spec.md) — I/O contract, label semantics, model candidates, evaluation, the MCP question
- [Development plan](docs/development-plan.md) — phases from where v0.1.0 left off
- [Research](docs/research.md) — prior art: routers, intent datasets, small encoders, voice-note apps
- [Lineage](docs/lineage.md) — what came from v0.1.0 and what changed
- [`context/`](context/) — the voice note that briefed this repo (`voice-note.mp3`) and its raw transcript

## Whose use case

Built around one person's dictation habits: long spoken prompts to AI agents, project
specs, shopping lists, email drafts. To adapt it, edit `taxonomy/labels.json` to your
routes, regenerate synthetic data for the labels you changed, and retrain. The plan
makes that a single step ([phase 6](docs/development-plan.md#phase-6--checkpointed-retraining)).
