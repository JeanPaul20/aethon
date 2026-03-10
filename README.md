<p align="center">
  <img src="media/aethon-icon.png" alt="Aethon" width="128" height="128" />
</p>

<h1 align="center">Aethon Mission Control</h1>

<p align="center">
  <em>AI-powered coding orchestrator for VS Code.</em><br/>
  <a href="https://github.com/JeanPaul20/aethon-mission-control">GitHub</a> · <a href="https://github.com/JeanPaul20/aethon-mission-control/issues">Issues</a> · <a href="AETHON_ARCHITECTURE.md">Architecture</a> · <a href="TRAINING_PIPELINE.md">Training Pipeline</a> · <a href="LICENSE">MIT License</a>
</p>

---

**AI-powered coding orchestrator for VS Code.** Aethon refines rough ideas into precise prompts, generates parallel code proposals from multiple AI models (GitHub Copilot, Ollama, Infomaniak), evaluates the best approach with automated security scanning, and exports training data for fine-tuning—all from the Copilot Chat sidebar.

## What is Aethon?

Aethon is a **VS Code extension** that implements multi-agent orchestration for AI-assisted coding. Unlike standard GitHub Copilot (which provides single suggestions), Aethon manages complete AI workflows: iteratively refining vague requirements into structured prompts, requesting parallel code proposals from heterogeneous AI models, evaluating implementations against security and quality criteria, and logging interactions for DPO (Direct Preference Optimization) training.

**Target users:** Developers who need to compare multiple AI models on identical coding tasks, create reproducible AI workflows with audit trails, or build private training datasets from AI coding interactions.

## When to Use Aethon

Use Aethon when you need to:
- **Compare multiple AI models** for the same coding task (e.g., GPT-4 vs Claude vs local Llama3)
- **Refine vague requirements** into structured, actionable prompts through iterative AI dialogue
- **Evaluate competing implementations** with automated scoring for security, style, and correctness
- **Build private training datasets** from AI coding workflows (Alpaca, ShareGPT, DPO formats)
- **Orchestrate complex coding agents** with state machines (plan → execute → verify → revise)

**Don't use Aethon if:** You want simple, single-shot code completions. Use standard GitHub Copilot for that.

## Key Capabilities

| Feature | Description | Privacy Level |
|---------|-------------|---------------|
| **`@aethon` Chat Participant** | Native Copilot Chat integration with command interface | N/A |
| **Prompt Refinement** | Iterative AI-guided clarification of rough ideas into precise task specifications | Local |
| **`/propose`** | Parallel code generation from multiple models (Copilot, Ollama, Infomaniak, custom OpenAI-compatible endpoints) | Configurable |
| **`/evaluate`** | Automated scoring and synthesis of best approach across providers | Configurable |
| **`/finalize`** | Persist refined tasks to `.aethon/` workspace folder with auto-generated documentation | Local |
| **`/export`** | Export training data to Alpaca, ShareGPT, or DPO formats (HuggingFace compatible) | Local-only default |
| **Security Scanner** | Auto-scan proposals for 14 security patterns (secrets, eval, SQL injection, XSS, path traversal) | Local |
| **Domain Auto-tagging** | Automatic categorization of tasks (auth, api, database, ui, security, testing, deployment) | Local |
| **Human Feedback Loop** | Capture Copilot Chat thumbs-up/down for DPO training datasets | Local |
| **Knowledge System** | Domain-filtered injection of coding principles based on task type | Local |
| **MCP Server** | Model Context Protocol server exposing Aethon tools to external LMs (Claude Desktop, etc.) | Local |
| **Git Hook Integration** | Pre-commit auto-export of training data | Local |

## How Aethon Fits In

Aethon is not a replacement for any of these tools — it runs *alongside* them and extends what they can do.

| Tool | Relationship | What Aethon adds |
|------|-------------|------------------|
| **GitHub Copilot** | Required dependency — Aethon uses the Copilot Chat API as its primary LM backend | Multi-model comparison, structured prompt refinement, training data export, automated evaluation. Copilot alone gives you one suggestion; Aethon lets you run the same task across Copilot + Ollama + Infomaniak and compare results. |
| **Ollama** | Supported local backend | Aethon discovers Ollama servers on your network automatically, routes tasks to the right model, and adds circuit-breaker resilience around Ollama calls. |
| **Cursor / Windsurf** | Different product category | Aethon stays in VS Code. If you prefer VS Code over an AI-native fork, Aethon offers similar agentic editing (`/code`) without switching editors. |
| **Continue.dev** | Complementary — both work inside VS Code | Continue focuses on inline completions and chat. Aethon focuses on *workflow orchestration*: refine → propose → evaluate → export. Different use cases, can coexist. |
| **Aider** | Complementary — terminal-based | Aider is excellent for terminal-driven AI editing. Aethon operates from the Copilot Chat sidebar with a visual workflow and webview panels. |

## Architecture & Concepts

Aethon implements a **multi-agent orchestration pattern** for AI coding:

- **Prompt Refinement Pipeline:** Chain-of-Thought iterative clarification converting natural language into structured task specifications
- **Multi-Model Ensemble:** Parallel inference across heterogeneous AI providers with automatic routing and aggregation
- **Evaluation Layer:** Multi-dimensional scoring (security, style, correctness, performance) with weighted synthesis
- **Memory System:** JSONL-based logging compatible with DPO (Direct Preference Optimization) and RLHF training pipelines
- **State Machine Agent:** Structured lifecycle (plan → execute → verify → revise) with automated diagnostic loops
- **MCP Integration:** Model Context Protocol server enabling external LM clients to access Aethon tooling

### Privacy Model

Aethon uses a tiered privacy architecture where source code access is strictly controlled by provider classification:

- **Local (Ollama):** Full source code access, completely offline, zero network egress
- **Private Cloud (Infomaniak):** Swiss-hosted, GDPR-compliant, ISO 27001 certified, contractual prohibition on training data usage
- **Public Cloud (Copilot, Groq, etc.):** Task descriptions only; source code anonymized (paths, secrets, company names masked) before transmission

See [Privacy & Security](#privacy--security) for detailed data handling policies.

## Tech Stack

- **Runtime:** Node.js 18+ / TypeScript 5.0+
- **Platform:** VS Code Extension API (1.85+)
- **AI Integration:** GitHub Copilot Chat API, Ollama REST API, OpenAI-compatible API clients
- **Protocols:** Model Context Protocol (MCP) stdio transport
- **Data Formats:** JSONL (training data), Markdown (task documentation), YAML (configuration)
- **Export Formats:** Alpaca, ShareGPT, DPO (HuggingFace datasets compatible)
- **Security:** Custom regex patterns + LLM-based scanning for vulnerability detection

## Installation

1. Install from VS Code Extensions marketplace (search "Aethon Mission Control")
2. Or download `.vsix` from [GitHub Releases](https://github.com/JeanPaul20/aethon-mission-control/releases)
3. Open Copilot Chat (Ctrl+Shift+I or Cmd+Shift+I)
4. Type `@aethon` to invoke the orchestrator

## Quickstart

```bash
# 1. Open Copilot Chat and invoke Aethon
@aethon I need a function to validate JWT tokens with rate limiting

# 2. Refine the prompt (Aethon will ask clarifying questions)
@aethon Add Redis caching and include refresh token rotation

# 3. Finalize the task description
/finalize

# 4. Get parallel proposals from multiple models
/propose

# 5. Evaluate and select best approach
/evaluate

# 6. Export training data (optional)
/export

```


## Usage Commands

All commands are invoked via the `@aethon` chat participant in Copilot Chat:

- **`@aethon <description>`** — Start prompt refinement session
- **`/finalize`** — Save refined task to `.aethon/` folder
- **`/propose`** — Request parallel code proposals from configured models
- **`/evaluate`** — Score proposals and synthesize best approach
- **`/orchestrate`** — Execute state-machine agent with plan/execute/verify loop
- **`/improve [area]`** — AI-powered codebase analysis (`architecture`, `security`, `performance`, …)
- **`/fix`** — Inline diagnostic fixer: static rules first (zero tokens), streaming Copilot fallback
- **`/ask <question>`** — Semantic search over workspace notes or Brainstorm Digest JSON (RAG)
- **`/digest`** — Read brainstorm files, classify ideas, build roadmap
- **`/agents`** — Show active agent statuses and performance leaderboard
- **`/code`** — Agent mode: read, edit, create, delete files autonomously
- **`/export`** — Export training data to Alpaca/ShareGPT/DPO formats
- **`/release [patch|minor|major]`** — Automated release manager with checklist tracking
- **`/setup`** — Run the Setup Wizard (discover LM servers, configure cloud keys)
- **`/catalog`** — Open the Aethon Studio Model Catalog
- **`/status`** — Privacy dashboard: provider status and data handling policies
- **`/tasks`** — List all saved task files in `.aethon/`
- **`/explain`** — Explain the current editor selection


### Aethon Studio

Open via **Aethon: Open Model Catalog** or **Aethon: Open Playground** in the command palette.

- **Model Catalog** — Live cards for every available AI model across all configured providers. Filter by category (Local / Private / Cloud), search by name, open any model directly in the Playground.
- **Playground** — Multi-turn streaming chat with any catalog model. Supports VS Code LM (GitHub Copilot, AI Toolkit), Ollama, Infomaniak, OpenAI-compatible providers.


### MCP Server

To expose Aethon tools to external LM clients (Claude Desktop, etc.):

1. Run **Aethon: Start MCP Server (stdio)** from the command palette
2. Configure the external client to use stdio transport
3. Available tools: `aethon_readFile`, `aethon_editFile`, `aethon_listFiles`, `aethon_runTerminal`, `aethon_getDiagnostics`, `aethon_searchWorkspace`


## Privacy & Security

### Data Handling by Provider Class

Aethon uses a tiered privacy architecture where source code access is strictly controlled by provider classification:

| Provider Class | Code Access | Data Residency | Training Usage | Anonymization |
|----------------|-------------|----------------|----------------|---------------|
| **Local (Ollama)** | Full source | Your machine | Never | N/A |
| **Private Cloud (Infomaniak)** | Full source | Switzerland 🇨🇭 | Contractually prohibited | None required |
| **Public Cloud (Copilot, Groq, etc.)** | Task description only | Provider-specific | Per provider ToS | Mandatory (paths, secrets, company names masked) |

**Key principle:** Source code is *never* sent to public cloud providers without explicit anonymization. Only task descriptions and context summaries leave your machine for cloud providers.

### Security Scanner

All `/propose` outputs are automatically scanned for:

- **Secrets:** Hardcoded API keys, tokens, passwords, private keys
- **Injection:** SQL injection, XSS, command injection, path traversal
- **Dangerous patterns:** `eval()`, `new Function()`, `exec()`, unsafe deserialization
- **Cryptography:** Weak randomness, hardcoded IVs, deprecated algorithms
- **Configuration:** Debug flags, CORS misconfigurations, verbose error messages

Findings are appended to proposal files with severity ratings (Critical/High/Medium/Low).

### Data Retention & Controls

- **Local data:** Stored in VS Code's global storage or user-configured paths; never transmitted to Aethon servers
- **Training data:** JSONL files stay local unless explicitly exported to HuggingFace (opt-in, private repos by default)
- **Memory logs:** Copilot Chat feedback (thumbs up/down) is logged locally for DPO training; not sent to external analytics
- **Anonymization:** Cloud-destined payloads are sanitized via regex patterns + LLM-based PII detection

### Compliance & Certifications

- **Infomaniak:** ISO 27001 certified, GDPR compliant, Swiss data protection act compliant
- **GDPR:** Right to erasure supported (delete `.aethon/` folder and memory files)
- **Zero-trust:** API keys can be provided via environment variables instead of settings storage

### User Control

- **Local-only mode:** Disable all cloud providers in settings for air-gapped operation
- **Consent flow:** Euria (Infomaniak) integration requires explicit opt-in per session
- **Audit trail:** All AI provider calls are logged with timestamp, provider, and anonymization status in `/status` dashboard


## What's New

> **Current version: v1.2.0** — see [CHANGELOG.md](CHANGELOG.md) for the full history.

### v1.2.0 — 2026-03-05
- **Multi-language fix rules** — zero-token static fixes + official rule catalog for Markdown, TypeScript, JavaScript, JSON, PHP, Python, YAML. Official descriptions injected into LM prompts for any supported linter.
- **/fix command** — inline diagnostic fixer: static rules first (no LM), streaming Copilot fallback.
- **scripts/add-fix-language.js** — scaffolds a new language into the fix rules system in one command.

### v1.1.0 — 2026-03-04
- **Aethon Studio Panel** — visual Model Catalog (live cards for VS Code LM / Ollama / Infomaniak / cloud) + streaming Playground.
- **Setup Wizard** — auto-discovers local LM servers (Ollama, LM Studio, Jan.ai, GPT4All) + cloud key onboarding.
- **/ask command** — RAG semantic search over workspace notes or Brainstorm Digest JSON.
- **CredentialStore** — encrypted key storage via VS Code SecretStorage; injected into process.env at bootstrap.

### v1.0.0 — 2026-03-01
- Initial production release: full command set, multi-model orchestration, evaluation, orchestrator agent, training data export, security scanner, MCP server, auto-update.

## Usage

1. Open Copilot Chat (Ctrl+Shift+I)
2. Type `@aethon` followed by your rough coding idea
3. Iterate on the refinement until satisfied
4. Use `/finalize` to save, `/propose` to get proposals, `/evaluate` to compare

## Settings

| Setting | Default | Description |
|---------|---------|-------------|
| **Storage** | | |
| `aethon.centralFolder` | *(globalStorage)* | Override root folder for all Aethon data |
| `aethon.knowledgeFolderPath` | | Custom path for knowledge files |
| `aethon.memoryFolderPath` | | Custom path for memory / training data |
| **Backend** | | |
| `aethon.backendPort` | `3123` | Port for the local orchestrator backend |
| `aethon.mockMode` | `false` | Use mock backend instead of live server |
| **Privacy / Anonymization** | | |
| `aethon.cloudEscalationMode` | `ask` | When to escalate to cloud (`always` / `ask` / `never`) |
| `aethon.alwaysAnonymizeBeforeCloud` | `true` | Strip sensitive data before cloud calls |
| `aethon.anonymizeFilePaths` | `true` | Mask file paths in cloud payloads |
| `aethon.anonymizeCompanyNames` | `true` | Mask company names in cloud payloads |
| `aethon.anonymizationPatterns` | `[]` | Extra regex patterns to mask |
| `aethon.anonymizationMaxChars` | `8000` | Max characters sent in a single cloud payload |
| **AI Providers** | | |
| `aethon.evaluationProvider` | `copilot` | Provider for `/evaluate` (`copilot`, `ollama`, `infomaniak`, or `custom`) |
| `aethon.proposalProvider` | `copilot` | Provider for `/propose` (`copilot`, `ollama`, `infomaniak`, `custom`, `both`, or `all`) |
| `aethon.copilotProposalModels` | `[]` | Pre-selected Copilot models for `/propose` (skip picker) |
| `aethon.ollamaProposalModels` | `[]` | Pre-selected Ollama models for `/propose` (skip picker) |
| `aethon.infomaniakProposalModels` | `[]` | Pre-selected Infomaniak models for `/propose` (skip picker) |
| `aethon.orchestratorProvider` | `ollama` | Provider for `/orchestrate` planner/verifier (`ollama`, `infomaniak`, `custom`, or `copilot`) |
| `aethon.orchestratorCoderProvider` | `copilot` | Provider for orchestrator coding steps (`copilot`, `ollama`, `infomaniak`, or `custom`) |
| `aethon.ollamaEndpoint` | `http://localhost:11434` | Primary Ollama server URL |
| `aethon.ollamaEndpoints` | `[]` | Additional Ollama servers (`[{ url, label, apiKey }]`). Use "Scan for Ollama Servers" to auto-discover. |
| `aethon.ollamaModel` | `codellama` | Default Ollama model |
| `aethon.ollamaApiKey` | | API key for primary Ollama (empty for local) |
| **Infomaniak AI** | | |
| `aethon.infomaniakProductId` | | Infomaniak AI Product ID |
| `aethon.infomaniakApiKey` | | Infomaniak API key (ai-tools scope) |
| `aethon.infomaniakModel` | `llm` | Default Infomaniak model |
| `aethon.infomaniakEndpoint` | `https://api.infomaniak.com` | Infomaniak API endpoint |
| **Custom Providers** | | |
| `aethon.customProviders` | `[]` | Array of OpenAI-compatible provider configs (see docs) |
| **Export / Tracking** | | |
| `aethon.hfToken` | | HuggingFace write token for dataset push |
| `aethon.hfDatasetRepo` | | HuggingFace dataset repo (e.g. `user/aethon-data`) |
| `aethon.mlflowTrackingUri` | | MLflow tracking server URL (e.g. `http://localhost:5000`) |
| `aethon.wandbApiKey` | | Weights & Biases API key |
| `aethon.wandbProject` | `aethon-training` | W&B project name |
| **Auto-Update** | | |
| `aethon.autoUpdate.enabled` | `true` | Check GitHub for new releases automatically |
| `aethon.autoUpdate.checkIntervalHours` | `6` | Hours between update checks (1–168) |
| `aethon.autoUpdate.includePrerelease` | `false` | Include pre-release versions |


**Keywords:** AI coding orchestrator, multi-model AI comparison, VS Code extension, prompt engineering, DPO training data, local LLM, GitHub Copilot alternative, AI agent orchestration, Model Context Protocol, Swiss AI hosting, privacy-first coding assistant
