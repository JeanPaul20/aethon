# Aethon Training Pipeline

> **Privacy-first:** All training data stays on your PC by default. Nothing is uploaded anywhere unless you explicitly opt in.

This document explains how Aethon collects, exports, and uses training data to help you fine-tune your own small language model (SLM) on your personal coding style and decision patterns.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Data Collection (Automatic)](#2-data-collection-automatic)
3. [Export Formats](#3-export-formats)
4. [Running an Export](#4-running-an-export)
5. [Privacy & Data Location](#5-privacy--data-location)
6. [Optional: HuggingFace Push](#6-optional-huggingface-push)
7. [Optional: Experiment Trackers](#7-optional-experiment-trackers)
8. [Git Hook Auto-Export](#8-git-hook-auto-export)
9. [Fine-Tuning Your Own Model](#9-fine-tuning-your-own-model)
10. [Data Schema Reference](#10-data-schema-reference)

---

## 1. Overview

```
┌─────────────────────────────────────────────────────────────-──┐
│                    Your Development Workflow                    │
│                                                                │
│  @aethon refine → /finalize → /propose → /evaluate → /export  │
│       │              │            │            │          │     │
│       ▼              ▼            ▼            ▼          ▼     │
│   ┌───────────────────────────────────────────────────────┐     │
│   │              memory.jsonl (auto-logged)               │     │
│   └───────────────────────────────────────────────────────┘     │
│                              │                                  │
│                         /export                                 │
│                              │                                  │
│              ┌───────────────┼───────────────┐                  │
│              ▼               ▼               ▼                  │
│        training-       training-       training-                │
│        alpaca.jsonl    sharegpt.jsonl   dpo.jsonl               │
│                                                                 │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ STAYS ON YOUR PC ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│                                                                 │
│               (optional, explicit opt-in only)                  │
│                         │                                       │
│              ┌──────────┼──────────┐                            │
│              ▼          ▼          ▼                             │
│         HuggingFace  MLflow    W&B                              │
│         (private)   (local)  (cloud)                            │
└─────────────────────────────────────────────────────────────────┘
```

Every interaction you have with `@aethon` is logged locally as structured JSONL. When you run `/export`, that raw data is transformed into standard ML training formats. The files stay on your machine — cloud uploads are entirely optional and require explicit confirmation.

---

## 2. Data Collection (Automatic)

Aethon logs every interaction to `memory.jsonl` automatically. This is stored in two places:

| Store | Location | Purpose |
|-------|----------|---------|
| **Primary** | `.aethon/memory/memory.jsonl` (per-workspace or central folder) | Project-specific training data |
| **Global backup** | `~/.aethon/global/memory.jsonl` | Cross-project backup (always written) |

### What gets logged

Each entry captures:

| Field | Example | Description |
|-------|---------|-------------|
| `timestamp` | `2026-02-16T14:30:00Z` | When the interaction happened |
| `sessionId` | `aethon-abc123` | Groups interactions within one VS Code session |
| `project` | `my-app` | Workspace name |
| `command` | `refine`, `propose`, `evaluate` | Which Aethon command was used |
| `userPrompt` | `"build user auth with sessions"` | What you typed |
| `refinedPrompt` | `"## Task: Implement User Auth..."` | The AI-refined version |
| `domain` | `auth`, `api`, `database`, `ui`, ... | Auto-detected from content |
| `modelId` | `gpt-4o` | Which AI model was used |
| `durationMs` | `2340` | How long the AI call took |
| `result` | `success` | Outcome of the interaction |

### Additional data sources

Beyond `memory.jsonl`, the exporter also reads:

- **Task files** (`task-*.md`) — Finalized task descriptions
- **Proposal files** (`proposal-*.md`) — AI-generated code proposals
- **Evaluation files** (`evaluation-*.md`) — Comparative evaluations of proposals
- **Knowledge files** (`knowledge/*.md`) — Your coding principles

These are all local files in your `.aethon/` folder.

---

## 3. Export Formats

The `/export` command produces training data in three standard ML formats:

### 3.1 Alpaca (Instruction Tuning)

Best for: teaching a model to follow instructions in your style.

```json
{
  "instruction": "Refine this rough idea into a clean, structured AI task prompt.",
  "input": "build user auth with sessions",
  "output": "## Task: Implement User Authentication System\n\n### Requirements:\n1. Session-based auth..."
}
```

**Training pairs created from:**
- Prompt refinements: raw prompt → refined prompt
- Task → Proposal pairs (matched by task ID)
- Task → Evaluation pairs
- Knowledge files → Q&A pairs

### 3.2 ShareGPT (Multi-Turn Chat)

Best for: training conversational models that handle multi-step workflows.

```json
{
  "conversations": [
    {"from": "system", "value": "You are Aethon, an AI coding orchestrator..."},
    {"from": "human", "value": "build user auth with sessions"},
    {"from": "gpt", "value": "## Task: Implement User Authentication System..."},
    {"from": "human", "value": "add OAuth support too"},
    {"from": "gpt", "value": "## Updated Task: User Auth with OAuth..."}
  ]
}
```

**Training pairs created from:**
- Multi-turn refinement sessions (grouped by session ID)
- Task → Proposal as 2-turn conversations
- Task → Proposals → Evaluation as multi-turn dialogues

### 3.3 DPO (Preference Alignment)

Best for: teaching a model which outputs you prefer over others.

```json
{
  "prompt": "Implement the following task:\n\n## Task: Build Auth System...",
  "chosen": "(winning proposal content)",
  "rejected": "(losing proposal content)"
}
```

**Training pairs created from:**
- Evaluation comparisons: winner = `chosen`, loser = `rejected`
- Refined prompts (`chosen`) vs raw prompts (`rejected`)
- Later evaluation versions (`chosen`) vs earlier versions (`rejected`)
- Thumbs-up/down feedback from Copilot Chat

---

## 4. Running an Export

In Copilot Chat:

```
@aethon /export
```

You'll be prompted to select which formats to export (Alpaca, ShareGPT, DPO). The files are saved to:

```
.aethon/training_export/
├── training-alpaca-20260216.jsonl
├── training-sharegpt-20260216.jsonl
└── training-dpo-20260216.jsonl
```

After export, you'll see a summary table with entry counts and file sizes.

---

## 5. Privacy & Data Location

### Default behavior: everything stays local

| What | Where | Leaves your PC? |
|------|-------|------------------|
| `memory.jsonl` | `.aethon/memory/` | **No** |
| Exported JSONL files | `.aethon/training_export/` | **No** |
| Task/proposal/evaluation files | `.aethon/` | **No** |
| Global backup | `~/.aethon/global/` | **No** |

### Anonymization

If you use cloud AI providers for `/propose` or `/evaluate`, the `AnonymizationService` sanitizes outbound data:

- **API keys & secrets** — Patterns like `sk-...`, `AKIA...`, private keys are replaced with `<SECRET>`
- **File paths** — Replaced with `<PATH>` (configurable via `aethon.anonymizeFilePaths`)
- **Company names** — Replaced with `<NAME>` (configurable via `aethon.anonymizeCompanyNames`)
- **Custom patterns** — Add your own regex patterns via `aethon.anonymizationPatterns`
- **Max payload size** — Enforced via `aethon.anonymizationMaxChars` (default: 8000 chars)

> **Important:** Anonymization applies to cloud AI calls, not to the locally stored training data. Your local JSONL files contain the full, un-anonymized data — which is what makes them valuable for fine-tuning.

---

## 6. Optional: HuggingFace Push

> **Disabled by default.** Your data is never uploaded unless you explicitly confirm.

If you configure HuggingFace credentials, you'll be offered the option to push after an export. This requires:

| Setting | Description |
|---------|-------------|
| `aethon.hfToken` | Your HuggingFace write token |
| `aethon.hfDatasetRepo` | Target dataset repo (e.g. `JeanPaul20/aethon-training-data`) |

### What happens when you push

1. A **modal warning dialog** explains that your data will leave your PC
2. You must explicitly click "Upload to HuggingFace (Private)" to proceed
3. The dataset repository is always created as **private**
4. JSONL files + a dataset card (README.md) are uploaded

### Privacy considerations

- Even with a private repo, **HuggingFace has access** to your data on their servers
- Your training data may contain code snippets, project names, and implementation details
- HuggingFace's [Terms of Service](https://huggingface.co/terms-of-service) and [Privacy Policy](https://huggingface.co/privacy) apply
- **Recommendation:** Use local fine-tuning unless HuggingFace offers full end-to-end encryption or you're comfortable with their data handling

### If you don't want cloud uploads

Simply don't configure `aethon.hfToken` and `aethon.hfDatasetRepo`. The export will produce local files only, with no prompts about uploading.

---

## 7. Optional: Experiment Trackers

After export, you're also offered the option to log metrics to:

| Tracker | Privacy | Setting |
|---------|---------|---------|
| **MLflow** | Self-hosted (local) | `aethon.mlflowTrackingUri` (e.g. `http://localhost:5000`) |
| **Weights & Biases** | Cloud service | `aethon.wandbApiKey` + `aethon.wandbProject` |

MLflow can run entirely on your own machine. W&B is a cloud service — the same privacy considerations as HuggingFace apply.

---

## 8. Git Hook Auto-Export

You can install a pre-commit hook that automatically exports training data on every `git commit`:

```
Command Palette → Aethon: Install Git Hook
```

This creates a `.git/hooks/pre-commit` script that:
1. Runs the Aethon training-data export
2. Tags the commit with the latest task ID
3. Stages the exported JSONL files

The exported data stays local — the hook does not push to HuggingFace or any cloud service.

---

## 9. Fine-Tuning Your Own Model

Once you've exported training data, you can fine-tune a small language model locally. Here are recommended tools:

### Alpaca format (instruction tuning)

| Tool | GPU Required | Link |
|------|-------------|------|
| **Unsloth** | Yes (4GB+) | [github.com/unslothai/unsloth](https://github.com/unslothai/unsloth) |
| **Axolotl** | Yes (8GB+) | [github.com/OpenAccess-AI-Collective/axolotl](https://github.com/OpenAccess-AI-Collective/axolotl) |
| **HuggingFace TRL** `SFTTrainer` | Yes | [huggingface.co/docs/trl](https://huggingface.co/docs/trl) |

### ShareGPT format (multi-turn chat)

| Tool | GPU Required | Link |
|------|-------------|------|
| **Axolotl** | Yes (8GB+) | [github.com/OpenAccess-AI-Collective/axolotl](https://github.com/OpenAccess-AI-Collective/axolotl) |
| **LLaMA-Factory** | Yes (8GB+) | [github.com/hiyouga/LLaMA-Factory](https://github.com/hiyouga/LLaMA-Factory) |

### DPO format (preference alignment)

| Tool | GPU Required | Link |
|------|-------------|------|
| **HuggingFace TRL** `DPOTrainer` | Yes | [huggingface.co/docs/trl](https://huggingface.co/docs/trl) |
| **Unsloth DPO** | Yes (4GB+) | [github.com/unslothai/unsloth](https://github.com/unslothai/unsloth) |

### Example: Fine-tuning with Unsloth (local, no cloud)

```python
from unsloth import FastLanguageModel
from datasets import load_dataset
from trl import SFTTrainer

# Load your local training data
dataset = load_dataset("json", data_files="training-alpaca-20260216.jsonl")

# Load a small base model
model, tokenizer = FastLanguageModel.from_pretrained(
    model_name="unsloth/Phi-3.5-mini-instruct",
    max_seq_length=2048,
    load_in_4bit=True,
)

# Add LoRA adapters
model = FastLanguageModel.get_peft_model(model, r=16, target_modules=["q_proj", "v_proj"])

# Train
trainer = SFTTrainer(
    model=model,
    train_dataset=dataset["train"],
    dataset_text_field="output",
    max_seq_length=2048,
)
trainer.train()

# Save locally
model.save_pretrained("my-aethon-model")
```

After training, you can load your fine-tuned model in Ollama and use it as your Aethon AI provider — creating a fully local, self-improving coding assistant.

---

## 10. Data Schema Reference

### Memory entry (memory.jsonl)

```typescript
interface MemoryEntry {
  timestamp: string;       // ISO 8601
  sessionId: string;       // Groups interactions in one VS Code session
  project: string;         // Workspace name
  command: string;         // "refine" | "finalize" | "propose" | "evaluate" | ...
  userPrompt: string;      // What the user typed
  refinedPrompt?: string;  // AI-refined version
  domain?: DomainTag;      // "auth" | "api" | "database" | "ui" | "security" | ...
  modelId?: string;        // Which AI model was used
  models?: string[];       // For /propose — which models were selected
  result?: string;         // "success" | "partial" | "error"
  outputFile?: string;     // Path to saved file
  durationMs?: number;     // AI call duration
  tokenEstimate?: number;  // Rough char-based estimate
  metadata?: Record<string, unknown>;
}
```

### Domain auto-detection

Domains are auto-tagged based on keyword matching in the prompt text:

| Domain | Example keywords |
|--------|-----------------|
| `auth` | login, jwt, oauth, session, password |
| `api` | endpoint, rest, graphql, middleware |
| `database` | sql, query, schema, migration, orm |
| `ui` | component, css, layout, responsive |
| `security` | xss, csrf, injection, encrypt |
| `testing` | test, mock, jest, coverage, e2e |
| `deployment` | docker, ci/cd, kubernetes, terraform |
| `performance` | cache, optimize, lazy-load, bundle |
| `general` | (fallback when no domain matches) |

---

## Summary

| Aspect | Default | Optional |
|--------|---------|----------|
| Data logging | Local `memory.jsonl` | — |
| Export | Local JSONL files | — |
| HuggingFace push | **Off** (explicit opt-in) | Private repo only |
| MLflow | Not configured | Self-hosted (local) |
| W&B | Not configured | Cloud service |
| Git hook export | Not installed | Local files only |
| Fine-tuning | Use local tools (Unsloth, etc.) | — |

**Your data, your model, your machine.**
