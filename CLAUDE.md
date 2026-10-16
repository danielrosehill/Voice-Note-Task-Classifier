# CLAUDE.md

Public repo. A small multi-label text classifier that reads a voice-note transcript
and decides which task-specific subagent (each with its own MCP tool) should handle
it — the routing layer at the top of a voice-productivity stack.

Read in this order: `README.md` (status) → `docs/spec.md` (contract, labels,
decisions, eval bar) → `docs/development-plan.md` (current phase and next step) →
`docs/lineage.md` (what came from v0.1.0). The originating brief is
`context/voice-note.mp3`, transcribed verbatim in `context/transcript-raw.txt`
(AssemblyAI, 2026-10-16; raw ASR, do not edit).

## Rules

- **This repo is public. No real voice-note text here, ever** — not in docs, tests,
  fixtures, few-shot prompts or examples. The only real notes this project may use
  are those in the public HF dataset `danielrosehill/Voice-Notes-Screened`, read
  from the Hub at run time, not copied in. Examples in docs are invented.
- **The v0.1.0 work lives in an earlier private working repo. Do not link it, name
  it by URL, or copy from it.** `docs/lineage.md` records what was carried over.
  Refer to it only as "an earlier private working repo".
- Do not mention or link any private dataset or model. Linked HF resources are listed
  in `project.json` → `hf_resources`; re-verify visibility before adding one
  (`curl -s https://huggingface.co/api/datasets/<id>` → `"private": false`).
- `taxonomy/labels.json` is the source of truth for labels. Change it, bump
  `schema_version` (rules in `docs/spec.md#versioning`), and update `docs/spec.md`
  in the same commit. Every prediction carries `taxonomy_version`.
- Python via `uv` only. The laptop is CPU-only: GPU training runs on Modal, personal
  workspace (`MODAL_PROFILE=danielrosehill`).
- Registry: this project is tracked in the private
  `~/repos/github/ai-models/AI-Model-Development-Workspace/projects.json`. Keep its
  entry and this repo's `project.json` in step when the status changes.
