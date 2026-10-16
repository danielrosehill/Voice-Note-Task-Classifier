# Research: prior art

Surveyed 2026-10-16 with web search and fetch. **Confirmed** = the page was opened
on 2026-10-16 and says so. **Inferred** = from a search snippet or memory, page not
opened. Hugging Face model existence, visibility and safetensors parameter counts
were re-checked against the HF API on the same day where marked "(HF API)".

## Summary

- Most "router" prior art routes a prompt to a **model** (cheap vs expensive LLM).
  Only semantic-router and a few small HF models route to **intents**, and all of
  them are single-label.
- Every public intent benchmark (CLINC150, MASSIVE, BANKING77, SNIPS) is short
  single-command utterances with one label each. None models long, rambling,
  multi-intent dictated notes. **That gap is this project's niche.**
- No voice-note app found auto-classifies a note's type and routes it to a
  type-specific action. They rely on user-picked rewrite styles, meeting
  action-item extraction, or user-configured Zapier/webhook rules.
- Published MCP/tool-use benchmarks put tool retrieval and selection at the centre
  of agent failure, and show that narrowing the tool set first helps. That supports
  a classifier gate in front of narrow subagents, though the measured failure
  curves are at hundreds of tools, not the ~10 routes here.

## 1. LLM routers and semantic routing

| Item | Routes between | Notes | |
|---|---|---|---|
| [RouteLLM](https://github.com/lm-sys/RouteLLM) (LMSYS) | 2 LLMs (strong vs weak) | Matrix factorisation, weighted Elo, BERT classifier or causal-LLM classifier. Claims up to 85% cost cut at 95% of GPT-4 quality on MT Bench. Not intents or tools. | Confirmed |
| [semantic-router](https://github.com/aurelio-labs/semantic-router) (Aurelio) | Named intent routes | Each route defined by example utterances; embedding similarity, optional hybrid sparse encoder. `fit()` sets a confidence threshold so it can return `None`. Pitched for tool-use decisions "in milliseconds". Single-route. Closest pattern to this project; a candidate rung-1 alternative. | Confirmed |
| [nvidia/prompt-task-and-complexity-classifier](https://huggingface.co/nvidia/prompt-task-and-complexity-classifier) | 11 prompt-task classes + complexity scores | DeBERTa-v3-base, 184M params (HF API). Primary + optional secondary task (NA below 0.1). Described as a NeMo Curator labelling tool. | Confirmed |
| [NVIDIA LLM Router blueprint](https://github.com/NVIDIA-AI-Blueprints/llm-router) | 3 LLMs | Default "intent-based" mode prompts Qwen 1.7B (no training); "auto-routing" uses CLIP embeddings + trained net. Deprecated in favour of NeMo Switchyard. | Confirmed |
| [Not Diamond](https://www.notdiamond.ai/) | LLMs across providers | Commercial model router. | Confirmed (homepage) |
| [Martian](https://withmartian.com/) | LLMs | Homepage now describes interpretability research; model-router positioning is from memory. | Inferred |
| [ENTUM-AI/AgentRouter](https://huggingface.co/ENTUM-AI/AgentRouter) | 10 agent intents | Fine-tuned MiniLM-L12-H384, 33.4M params (HF API). Appears single-label; claims <5 ms on GPU. | Confirmed |
| [tgupj/tiny-router](https://huggingface.co/tgupj/tiny-router) | 4 independent heads | deberta-v3-small with heads for relation_to_previous, actionability, retention, urgency. Close to this project's tag-only idea. No param count on card. | Confirmed |
| flowxai/intentrouter, Ttimms/sovereign-edge-intent-router, `tinyintent` (PyPI) | Small intent sets | ModernBERT-base 8 healthcare classes; DistilBERT 5 classes; frozen bge-large + linear head. | Inferred |

## 2. Intent benchmarks (all single-label)

| Dataset | Size | Labels | Relevance |
|---|---|---|---|
| [CLINC150](https://huggingface.co/datasets/clinc/clinc_oos) | `plus`: 15,250 / 3,100 / 5,500 | 150 intents, 10 domains, + `oos` | Its out-of-scope class is the nearest analogue to `routes: []`. Confirmed |
| [MASSIVE](https://huggingface.co/datasets/AmazonScience/massive) (Amazon) | >1M utterances, 52 languages (~19.5k each) | 60 intents, 55 slot types, 18 scenarios | One intent per utterance. Confirmed |
| [BANKING77](https://huggingface.co/datasets/PolyAI/banking77) | 13,083 (10,003 / 3,080) | 77 | Multi-class. Confirmed |
| [SNIPS](https://github.com/sonos/nlu-benchmark/tree/master/2017-06-custom-intent-engines) | >2,000 queries per intent | 7 | Confirmed |

None is usable as training data here (different label space, short commands), but
CLINC's `oos` handling is a reference for evaluating null output.

## 3. Small encoders for multi-label classification

| Model | Params | Context | Notes | |
|---|---|---|---|---|
| [SetFit](https://arxiv.org/abs/2209.11055) ([multi-label docs](https://huggingface.co/docs/setfit/how_to/multilabel)) | — | — | Contrastive fine-tune of a sentence transformer, then a head; no prompts. `multi_target_strategy`: one-vs-rest, multi-output, classifier-chain. Docs example uses bge-small. | Confirmed |
| [ModernBERT-base](https://huggingface.co/answerdotai/ModernBERT-base) / large | 149M / 395M | 8,192 | 22 / 28 layers. arXiv 2412.13663. | Confirmed |
| [deberta-v3-small](https://huggingface.co/microsoft/deberta-v3-small) | 44M backbone + 98M embeddings (~142M) | 512 | 6 layers, hidden 768, 128K vocab. | Confirmed |
| [bge-small-en-v1.5](https://huggingface.co/BAAI/bge-small-en-v1.5) | 33.4M | 512 | 384-dim. | Confirmed |
| [bge-base-en-v1.5](https://huggingface.co/BAAI/bge-base-en-v1.5) | ~0.1B (card) | 512 | 768-dim. The v0.1.0 encoder. | Confirmed |
| [Ettin encoders](https://huggingface.co/jhu-clsp/ettin-encoder-150m) | 17M, 32M, 68M, 150M, 400M, 1B | 8K | July 2025. | Confirmed |
| [mmBERT-small](https://huggingface.co/jhu-clsp/mmBERT-small) / base | ~140M / ~307M (42M / 110M non-embedding) | 8,192 | 1,800+ languages; relevant only if Hebrew enters scope. | Confirmed |
| [GLiClass modern-base v2.0](https://huggingface.co/knowledgator/gliclass-modern-base-v2.0-init) | 151M (HF API) | — | Zero-shot, ModernBERT backbone, multi-label mode with threshold. | Confirmed |
| [GLiNER2](https://github.com/fastino-ai/GLiNER2) | 74M … 340M | — | `classify_text` with `multi_label` and `cls_threshold`, plus entity extraction in one pass — a candidate for labels + fields later. Claims "CPU first", no benchmarks. | Confirmed |

## 4. Voice-note apps: do they classify or route?

None found does automatic note-type classification with routing to a type-specific
action.

| App | What it does with a note | |
|---|---|---|
| [Voicenotes](https://voicenotes.com/) | Summaries, action items, follow-up emails/to-dos on request. No auto-classification or push to calendar/tasks claimed. | Confirmed |
| [AudioPen](https://audiopen.ai/) | Rewrites in a user-picked style; routing via Zapier/webhooks. | Confirmed |
| [Letterly](https://letterly.app/) | User-chosen rewrite types; only language is auto-detected; Zapier/webhooks. | Confirmed |
| [Cleft](https://www.cleftnotes.com/) | Extracts main points and actions; sends action items to Apple Reminders, Things 3, Todoist. Nearest product, but one extraction type only. | Confirmed |
| [TalkNotes](https://talknotes.io/) | User picks the output style (meeting notes, task list, brainstorm). | Confirmed |
| [Superwhisper](https://ai.superwhisper.com/docs/modes/switching-modes) | Modes switch manually, by active app/website, or by deep link — not by content. | Confirmed |
| [Wispr Flow](https://wisprflow.ai/) | Dictation cleanup with per-app style; no action routing. | Confirmed |
| [Otter](https://otter.ai/) | Meeting action items captured and assigned; Asana/Jira/CRM sync. Meeting-only. | Confirmed |
| [Plaud](https://www.plaud.ai/) | Structured notes per use case; "Plaud Agent" not specified. | Confirmed |
| [Bee](https://www.bee.computer/) | Suggests to-dos from conversations; email/calendar only when asked. | Confirmed |
| [Limitless](https://www.limitless.ai/) | Acquired by Meta; no Pendant sales since 2025-12-05. Not relevant now. | Confirmed |
| [Apple Reminders (26)](https://support.apple.com/en-ae/guide/mac-help/mchl979b81e9/26/mac/26) | Suggests reminders from shared content; auto-categorises within a list. Not voice-note routing. | Confirmed |
| Apple Notes audio | Recordings get a summary, no routing. | Inferred |

## 5. Small classifier vs prompted LLM, and tool selection

| Source | Finding | |
|---|---|---|
| [Cost-Aware Model Selection for Text Classification](https://arxiv.org/abs/2602.06370) (Feb 2026) | Fine-tuned BERT-family encoders "competitive with, and often better than" zero/few-shot LLM prompting on macro F1, at one to two orders of magnitude lower cost and latency; recommends hybrids. Datasets are IMDB, SST-2, AG News, DBPedia — not intents. | Confirmed (abstract) |
| [Intent classification for agent routers](https://tianpan.co/blog/2026/04/16/intent-classification-agent-routers) (blog, 2026-04-16) | Cascade: keyword → embeddings → fine-tuned classifier → LLM fallback, confidence bands 0.8 / 0.5. Latency figures (embedding 16–100 ms, classifier 50–200 ms, LLM 1–5 s) are the blog's own. | Confirmed (secondary source) |
| [RAG-MCP](https://arxiv.org/abs/2505.03275) (May 2025) | Retrieving relevant MCPs before the LLM call raised tool-selection accuracy from 13.62% to 43.13% and cut prompt tokens by over 50%. | Confirmed |
| [MCP-Zero](https://arxiv.org/abs/2506.01056) | Agent requests tools via server-then-tool routing; 308 servers, 2,797 tools; 98% token reduction on APIBank. | Confirmed |
| [ToolRet](https://arxiv.org/abs/2503.01763) | 7.6k tasks over 43k tools; strong IR retrievers do poorly on tool retrieval, lowering agent pass rates. | Confirmed |
| [LiveMCPBench](https://arxiv.org/abs/2508.01780) | 95 tasks, 70 servers, 527 tools; Claude-Sonnet-4 78.95%, most of 12 models 30–50%; retrieval errors caused nearly half of failures. | Confirmed |
| ertas.ai fine-tune vs GPT-4o post | Fine-tuned 8B reportedly 94.1% vs 82.4%. Vendor selling fine-tuning. | Inferred |

## What this means for the spec

- The MCP question is real: with ~10 routes, a capable LLM with a routing prompt
  will likely do well on single-intent notes. The classifier's case rests on cost,
  latency, determinism, calibrated null output, multi-label fan-out and keeping
  subagents narrow — hence the rung-0 comparison in
  [spec.md](spec.md#decision-does-mcp-make-this-unnecessary).
- semantic-router is a cheap extra rung-1 comparison (example utterances per route,
  threshold to `None`), but it is single-route.
- GLiNER2 is the natural later candidate for doing routing and per-route field
  extraction in one model.

## Misses and partial data

- `cleft.ai` redirects (307) to `cleftnotes.com`.
- The tiny-router card gives no parameter count; bge-base's card gives only "0.1B";
  the GLiClass card gives both 0.2B and 151M (HF API safetensors total: 151,378,176).
- For arXiv 2602.06370 only the abstract was read.
- No Apple feature was found that extracts tasks from a recording.
