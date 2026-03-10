<p align="center">
  <img src="media/aethon-icon.png" alt="Aethon" width="96" height="96" />
</p>

# Aethon Mission Control — Technical Architecture

**Version:** 1.3.0 · **VS Code:** ^1.109.0 · **TypeScript:** strict mode

---

## 1. What Is Aethon?

Aethon Mission Control is a **VS Code extension** that acts as a multi-agent AI coding orchestrator. It is NOT a language model — it is a meta-layer that sits on top of existing LLMs (GitHub Copilot / VS Code LM models, Ollama-hosted models, Infomaniak Swiss AI, and any OpenAI-compatible provider) and orchestrates them through a structured workflow to produce higher-quality results than any single model call.

**The core workflow:**
1. **Refine** a rough idea into a clean task prompt (knowledge files injected as guardrails)
2. **Propose** — fan out the refined prompt to multiple models simultaneously for competing proposals
3. **Evaluate** — a judge model reads all proposals, scores them, declares a winner, synthesizes an enhanced version
4. **Orchestrate** — execute the winning plan step-by-step with AI oversight, verification, and correction
5. **Export** — all interactions accumulate as structured training data (Alpaca / ShareGPT / DPO JSONL)

Aethon also provides:
- **Aethon Studio Panel** — a full-screen model catalog + interactive playground for every connected AI
- **RAG `/ask`** — semantic search over notes, brainstorm files, or Digest JSON indexes
- **`/fix`** — zero-token static diagnostic fixer with fallback to streaming Copilot fix
- **Setup Wizard** — first-run auto-discovery of all local AI providers (Ollama, LM Studio, Jan.ai, VS Code LM)
- **MCP server** — exposes Aethon tools to external LM clients via Model Context Protocol

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│              AETHON MISSION CONTROL  (VS Code Extension)        │
├──────────────────┬──────────────────┬───────────────────────────┤
│  Chat (@aethon)  │  Editor UI       │  Webview Panels           │
│  ├─ /refine      │  ├─ Lightbulb    │  ├─ AethonStudioPanel     │
│  ├─ /propose     │  │   quick-fix   │  │   ├─ Model Catalog      │
│  ├─ /evaluate    │  │   (/fix)      │  │   ├─ Playground         │
│  ├─ /orchestrate │  └─ Code lens    │  │   └─ Orchestrator tab   │
│  ├─ /export      │                  │  ├─ DashboardView          │
│  ├─ /ask         │                  │  ├─ SettingsView           │
│  ├─ /fix         │                  │  └─ ActiveAgentsView       │
│  ├─ /setup       │                  │                            │
│  ├─ /catalog     │                  │                            │
│  ├─ /digest      │                  │                            │
│  ├─ /improve     │                  │                            │
│  ├─ /release     │                  │                            │
│  ├─ /agents      │                  │                            │
│  └─ /code        │                  │                            │
├──────────────────┴──────────────────┴───────────────────────────┤
│                     COMMAND ROUTER                              │
│          chatParticipant.ts → CommandRouter.ts → handlers/      │
├─────────────────────────────────────────────────────────────────┤
│                     CORE SERVICES                               │
│  OrchestratorService   MemoryLogger      KnowledgeManager       │
│  EmbeddingService      VectorStore       BrainstormDigestService │
│  CredentialStore       SetupWizard       ReleaseService         │
│  CodebaseAnalyzer      SecurityScanner   AnonymizationService   │
│  TrainingDataExporter  AgentStateMachine AutoUpdateService      │
│  AgentRegistry (infra) CircuitBreaker    EventBus               │
├─────────────────────────────────────────────────────────────────┤
│                     AI PROVIDERS                                │
│  VS Code LM (Copilot, AI Toolkit, OpenRouter…)                 │
│  Ollama (multi-endpoint, OllamaDiscovery)                       │
│  Infomaniak Swiss AI (InfomaniakClient)                         │
│  OpenAI / Perplexity / Kimi (NamedProviders)                    │
│  Any OpenAI-compatible (OpenAICompatibleClient)                 │
│  ResilientProviderChain (fallback chain)                        │
├─────────────────────────────────────────────────────────────────┤
│              LM TOOLS  (MCP + vscode.lm.registerTool)          │
│  readFile  editFile  createFile  deleteFile  listFiles          │
│  renameFile  runTerminal  getDiagnostics  searchWorkspace       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. The Workflow Pipeline

```
Human types rough idea
        │
        ▼
┌───────────────────────┐
│  PROMPT REFINEMENT    │  ← Copilot/any LM rewrites the idea into a clean task spec
│  /refine              │  ← Knowledge files (.md) injected into system prompt
└──────────┬────────────┘
           │  Human iterates; each turn logged to memory.jsonl
           ▼
┌───────────────────────┐
│  /finalize            │  ← Saves refined prompt as versioned task-YYYYMMDD-NNN.md
└──────────┬────────────┘
           ▼
┌───────────────────────┐
│  /propose             │  ← Fans out task to N providers simultaneously
│                       │  ← PRIVACY GATE: source code only sent to local/private providers
│                       │  ← Each proposal saved as <model>-<taskId>-proposal.md
└──────────┬────────────┘
           ▼
┌───────────────────────┐
│  /evaluate            │  ← Judge model reads all proposals + task
│                       │  ← Scores 1-10 per criteria, declares winner
│                       │  ← Synthesizes enhanced version combining best parts
│                       │  ← Saved as evaluation-<taskId>-NNN.md (versioned)
└──────────┬────────────┘
           ▼
┌───────────────────────┐
│  /orchestrate         │  ← Takes winning evaluation, plans atomic steps
│                       │  ← Assigns each step to best AI by evaluation score
│                       │  ← Executes, verifies (git diff + AI review + tests)
│                       │  ← Up to 3 correction attempts per step on failure
└──────────┬────────────┘
           ▼
┌───────────────────────┐
│  /export              │  ← Reads all accumulated data
│                       │  ← Exports Alpaca / ShareGPT / DPO JSONL
│                       │  ← Optional push to HuggingFace dataset
└───────────────────────┘
```

Every step is logged to `memory/memory.jsonl` — training data accumulates automatically as you work.

---

## 4. Data Architecture

### 4.1 Storage Root Priority

1. **`aethon.centralFolder` setting** — explicit user-chosen path
2. **VS Code `globalStorageUri`** — default, auto-created on first activation
3. **Per-workspace `.aethon/`** — legacy fallback only

### 4.2 Folder Layout

```
<root>/
├── task-20260210-001.md
├── agents/
│   ├── gpt-4o-task-20260210-001-proposal.md
│   ├── ollama-task-20260210-001-proposal.md
│   └── evaluation-task-20260210-001-001.md
├── knowledge/
│   ├── 01_Coding_Principles_GLOBAL.md
│   └── 02_UI_Design_Principles_GLOBAL.md
├── memory/
│   └── memory.jsonl
├── orchestrator/
│   └── orchestrator-state.json
├── rag/
│   └── rag-index-<hash>.json
└── training_export/
    ├── training-alpaca-20260210.jsonl
    ├── training-sharegpt-20260210.jsonl
    └── training-dpo-20260210.jsonl
```

### 4.3 Memory JSONL Schema

```json
{
  "timestamp": "2026-02-10T14:30:00.000Z",
  "sessionId": "20260210-abc123",
  "project": "my-app",
  "command": "refine",
  "userPrompt": "build a user auth system",
  "refinedPrompt": "## Task: Implement User Authentication...",
  "modelId": "gpt-4o",
  "result": "success",
  "outputFile": "task-20260210-001.md",
  "durationMs": 3400,
  "tokenEstimate": 1250,
  "metadata": {}
}
```

---

## 5. Aethon Studio Panel

A full-screen WebviewPanel (`src/views/studioPanel/`) with three tabs:

### Model Catalog tab
- Auto-loads all connected AI providers on open (no manual refresh)
- **VS Code LM models** — live via `vscode.lm.selectChatModels()` (Copilot, AI Toolkit, OpenRouter, GitHub Models, Azure Foundry, etc.)
- **Ollama models** — from all configured endpoints; embedding models detected by name pattern → shown as "API Only"
- **Infomaniak models** — loaded via live API call + fallback to 10 known-good models
- **Named providers** — OpenAI, Perplexity, Kimi (shown only when API key is present)
- Cards show: provider icon, availability dot, context window, privacy tag, model class chip (EMBEDDING / AUDIO / IMAGE)
- "▶ Open in Playground" on every active chat model

### Playground tab
- Model dropdown grouped by provider; only active chat models shown
- Optional collapsible system prompt
- Multi-turn conversation history (user right / assistant left)
- **Streaming** for VS Code LM models (`playground:chunk` protocol); buffered for Ollama/Infomaniak

### Orchestrator tab
- Live orchestration progress view

**CSP rule:** All buttons use `data-action` / `data-model-id` attributes + one delegated click listener inside the nonce-protected `<script>` block. `onclick="..."` inline attributes are silently blocked by `script-src 'nonce-...'`.

---

## 6. RAG System (`/ask`)

| File | Role |
|------|------|
| `src/services/EmbeddingService.ts` | Ollama embeddings: `embed()`, `embedBatch()` (array batch, 32 texts/request, serial fallback), `cosine()` |
| `src/services/VectorStore.ts` | Build + query vector index (folder mode or BrainstormDigest JSON mode); `Chunker` splits at sentence boundaries |
| `src/handlers/askHandler.ts` | `/ask` command: `index`, `folder`, `<question>` subcommands |

```
/ask <question>
    ├─ Folder source → chunk files → embed → store in rag-index-<hash>.json
    ├─ BrainstormDigest JSON → one embedding per DigestIdea (rich metadata preserved)
    ├─ Freshness check (mtime-based)
    ├─ Cosine similarity ranking (keyword fallback when Ollama unavailable)
    └─ Top-N chunks injected into Copilot system prompt; sources shown with relevance %
```

---

## 7. Fix System (`/fix`)

```
Diagnostic (Error/Warning) in editor
        │
        ▼
KnownFixesRegistry.lookup(source, code)
        │
        ├─ Found → apply static fix (zero tokens, instant)
        └─ Not found → streaming Copilot fix
               └─ getRuleDescription(source, code) injects official rule text into prompt
```

Language coverage: Markdown (markdownlint), TypeScript (ESLint + @typescript-eslint + TS compiler), JavaScript, Python (flake8/pylint/ruff/mypy), PHP (phpcs/Intelephense), JSON, YAML.

Adding a new language: `node scripts/add-fix-language.js <language>` — scaffolds the file and auto-patches `fixRules/index.ts`.

---

## 8. Setup Wizard

`src/services/SetupWizard.ts` — auto-fires on activation when `needsSetup()` is true (no VS Code LM models found and no cloud API keys configured). Also available as `@aethon /setup`.

Parallel discovery probes: VS Code LM · Ollama :11434 · all `aethon.ollamaEndpoints` · LM Studio :1234 · Jan.ai :1337 · GPT4All :4891 · all `AETHON_*` env/SecretStorage keys.

Optional LAN scan via `OllamaDiscovery.scanNetwork()`. Cloud provider keys collected via `CredentialStore` (SecretStorage — never stored in settings).

---

## 9. Infrastructure Layer

| Module | Purpose |
|--------|---------|
| `result.ts` | `Result<T>`, `ok()`, `fail()`, `attempt()`, `unwrap()`, `map()` |
| `CircuitBreaker.ts` | 3 failures → 30 s cooldown; wraps all Ollama + Infomaniak calls |
| `Logger.ts` | Structured log: `[HH:MM:SS.mmm] LEVEL [component] message` |
| `EventBus.ts` | Typed pub/sub; `extensionEventBus` singleton; 8 event types |
| `AgentRegistry.ts` | Per-model perf tracking; `bestFor(taskType)`; `leaderboard()` |
| `httpClient.ts` | Shared fetch wrapper |
| `FileSystemWalker.ts` | Recursive workspace file scanner |
| `ShellExecutor.ts` | Binary allowlist (`git`, `npm`, `npx`, `node`, `tsc`, `eslint`, `prettier`); `execute()` (sync, `execFileSync`); `executeAsync()` (async, `promisify(execFile)`); `tryExecute()` (non-throwing); `shell: false` always on non-Windows; Windows `.cmd` scripts get `shell: true` only for `npm`/`npx` when invoked via the resolved full path |

---

## 10. AI Providers

| Level | Providers | Code access |
|-------|-----------|-------------|
| `local` 🟢 | Ollama (any endpoint) | Full source ✅ |
| `private` 🔵 | Infomaniak Swiss AI | Full source ✅ |
| `standard` 🟡 | Copilot, OpenAI, Perplexity, Kimi, custom | Task only 🛡️ |
| `unknown` 🔴 | Unclassified custom providers | Task only 🛡️ |

`OllamaDiscovery` validates all user-supplied endpoint URLs via `validateEndpointUrl()` before sending any HTTP request: link-local ranges (`169.254.*`), cloud metadata endpoints (GCP `metadata.google`, Oracle `metadata.internal`, AWS IMDS), and non-http(s) protocols are rejected. Network scan (`scanNetwork()`): port 11434 is probed on every host in the local `/24` subnet; ports 1234/1337/4891/8080 are probed on localhost only (∼260 total probes, down from ∼2 032). Two-stage probe per target: Ollama `/api/tags` first, OpenAI `/v1/models` fallback (covers LM Studio :1234 and Jan.ai :1337).

`AnonymizationService` sanitises all cloud-bound payloads: built-in secret patterns (OpenAI `sk-*`, AWS `AKIA*`, Google `AIza*`, PEM private keys), file paths, and organisation names. User-supplied redaction patterns are verified by `isSafePattern()` before compilation: patterns >200 chars are rejected; patterns containing catastrophic nested quantifiers (`(a+)+`, `(a*)*`, etc.) are rejected; any pattern that takes >50 ms to match a 50-character probe string is rejected. Company name masking requires ≥6 lowercase letters to prevent false positives on code identifiers such as `TypeScript`, `Promise`, or `Array`.

`RunTerminalTool` (AI agent file tool) always invokes commands with `shell: false`. Only allowlisted binaries may be run. On Windows, `.cmd` wrappers (`npm.cmd`, `npx.cmd`, `tsc.cmd`, `eslint.cmd`, `prettier.cmd`) are located via PATH and invoked as `cmd.exe /d /s /c <resolved-path> [args…]` so each argument remains a separate token — shell metacharacters in arguments are never interpreted.

---

## 11. Orchestrator Layer

Six modules in `src/services/orchestrator/`:

| Module | Role |
|--------|------|
| `OrchestratorPlanner.ts` | Reads ALL proposals, parses per-AI scores, assigns steps to best AI, pairs code↔test steps |
| `OrchestratorExecutor.ts` | Sends each step to its assigned AI |
| `OrchestratorVerifier.ts` | git diff + AI review + test runner; test failures override AI review pass |
| `OrchestratorProviderGateway.ts` | Routes tasks to Copilot / Ollama / Infomaniak / Custom |
| `OrchestratorService.ts` | Thin coordinator; manages state persistence and session resume |
| `OrchestratorTypes.ts` | Shared types: `OrchestratorStep`, `OrchestratorState`, `AIScoreCard`, `TestResult` |

Execution: Plan → Execute → Verify → Correct (up to 3 attempts, same AI) → log per-AI performance to MemoryLogger (persists cross-session).

---

## 12. Training Data Export

**Alpaca** — `{instruction, input, output}` triples from refine interactions and proposal/evaluation pairs.

**ShareGPT** — multi-turn `{conversations}` from session groups and task→proposal→evaluation chains.

**DPO** — `{prompt, chosen, rejected}` pairs from winning vs losing proposals; later evaluation versions chosen over earlier.

HuggingFace push (`/export --hf`): creates repo, uploads JSONL files, generates dataset card. Requires `aethon.hfToken` + `aethon.hfDatasetRepo`.

---

## 13. Error System

`src/services/AethonError.ts` — every catch block uses `handleError()` or `AethonError.wrap()`. Each error carries `code`, `source` (file → class → method), `details`, `cause`, `timestamp`.

---

## 14. Configuration Reference

| Setting | Default | Description |
|---------|---------|-------------|
| `aethon.centralFolder` | `""` | Custom storage root (empty = VS Code globalStorage) |
| `aethon.proposalProvider` | `"copilot"` | Provider for `/propose` |
| `aethon.evaluationProvider` | `"copilot"` | Provider for `/evaluate` |
| `aethon.orchestratorProvider` | `"ollama"` | Planner AI for orchestrator |
| `aethon.orchestratorCoderProvider` | `"copilot"` | Coder AI for orchestrator |
| `aethon.ollamaEndpoint` | `"http://localhost:11434"` | Primary Ollama URL |
| `aethon.ollamaEndpoints` | `[]` | Additional endpoints (label + url objects) |
| `aethon.ollamaModel` | `"qwen3-coder:480b-cloud"` | Default Ollama model |
| `aethon.infomaniakProductId` | `""` | Infomaniak AI product ID |
| `aethon.infomaniakApiKey` | `""` | Infomaniak API key |
| `aethon.customProviders` | `[]` | OpenAI-compatible provider config array |
| `aethon.hfToken` | `""` | HuggingFace write token |
| `aethon.hfDatasetRepo` | `""` | HuggingFace dataset repo |
| `aethon.autoUpdate.enabled` | `true` | Check GitHub for new releases every 6 hours |
| `aethon.cloudEscalationMode` | `"ask"` | `"local-only"` \| `"ask"` \| `"cloud-allowed"` |

---

## 15. File Inventory

```
src/
├── extension.ts                          ← Activation → bootstrap()
├── chatParticipant.ts                    ← @aethon chat participant + routing
├── bootstrap/bootstrap.ts                ← Composition root: all services + commands
├── chat/
│   ├── commandRegistration.ts
│   ├── followups.ts
│   └── sessionGraph.ts
├── config/
│   ├── settings.ts                       ← getSettings(), validateSettings()
│   └── promptConfig.ts
├── handlers/
│   ├── CommandRouter.ts
│   ├── refineHandler.ts                  ← /refine /finalize /tasks
│   ├── proposeHandler.ts
│   ├── propose/                          ← proposalJobs, proposalCollectors, types
│   ├── evaluateHandler.ts
│   ├── orchestrateHandler.ts             ← /orchestrate /oversight
│   ├── exportHandler.ts
│   ├── askHandler.ts                     ← /ask — RAG query
│   ├── fixHandler.ts                     ← /fix chat command
│   ├── fixDiagnosticHandler.ts           ← static rules → streaming fallback
│   ├── digestHandler.ts
│   ├── improveHandler.ts
│   ├── releaseHandler.ts
│   ├── agentHandler.ts                   ← /agents /code
│   ├── statusHandler.ts
│   ├── sessionHandler.ts
│   ├── customizeHandler.ts
│   ├── forkHandler.ts
│   ├── shared.ts
│   ├── shared/
│   │   ├── privacy.ts                    ← getBuiltInPrivacyLevel, getPrivacySafeContext
│   │   ├── copilot.ts, editorContext.ts, prompts.ts
│   │   ├── settings.ts, taskFiles.ts, workspaceContext.ts
│   ├── types.ts
│   └── validation.ts
├── infrastructure/
│   ├── result.ts                         ← Result<T>
│   ├── CircuitBreaker.ts
│   ├── Logger.ts
│   ├── EventBus.ts
│   ├── AgentRegistry.ts
│   ├── httpClient.ts
│   ├── FileSystemWalker.ts
│   └── ShellExecutor.ts
├── services/
│   ├── AethonError.ts
│   ├── CredentialStore.ts                ← SecretStorage wrapper
│   ├── SetupWizard.ts                    ← First-run provider discovery
│   ├── EmbeddingService.ts               ← Ollama embeddings
│   ├── VectorStore.ts                    ← RAG index
│   ├── KnowledgeManager.ts
│   ├── MemoryLogger.ts                   ← JSONL dual-store writer
│   ├── BrainstormDigestService.ts        ← .md/.txt/.docx → DigestIdea[]
│   ├── TrainingDataExporter.ts
│   ├── training/                         ← formatters, hfClient, types
│   ├── fixRules/                         ← index, markdown, typescript, javascript,
│   │                                         python, php, json, yaml, types
│   ├── KnownFixesRegistry.ts
│   ├── MarkdownRuleCatalog.ts
│   ├── OllamaClient.ts
│   ├── OllamaDiscovery.ts
│   ├── infomaniak/InfomaniakClient.ts
│   ├── NamedProviders.ts
│   ├── OpenAICompatibleClient.ts
│   ├── ResilientProviderChain.ts
│   ├── orchestrator/                     ← Service, Planner, Executor, Verifier,
│   │                                         ProviderGateway, Types
│   ├── analysis/                         ← SecurityScanner, StyleAnalyzer,
│   │                                         DependencyReader, GitChangeTracker, ProjectSummarizer
│   ├── paths/                            ← CentralPathResolver, PathStringUtils,
│   │                                         SpecialFolderResolver, WorkspaceResolver
│   ├── update/                           ← AutoUpdateService, GitHubReleaseClient,
│   │                                         UpdateScheduler, VsixInstaller
│   ├── AnonymizationService.ts
│   ├── AgentStateMachine.ts
│   ├── CachedCodebaseAnalyzer.ts
│   ├── CodebaseAnalyzer.ts
│   ├── ExperimentExporter.ts
│   ├── GitHookService.ts
│   ├── MigrationService.ts
│   ├── ProposalScanner.ts
│   ├── ReleaseService.ts
│   ├── ResponseValidator.ts
│   ├── SessionMemoryService.ts
│   ├── TaskReadmeGenerator.ts
│   └── TestRunnerService.ts
├── tools/
│   ├── AethonTools.ts
│   ├── ReadFileTool.ts, EditFileTool.ts, CreateFileTool.ts
│   ├── DeleteFileTool.ts, RenameFileTool.ts, ListFilesTool.ts
│   ├── RunTerminalTool.ts, GetDiagnosticsTool.ts, SearchWorkspaceTool.ts
│   ├── SafePathValidator.ts
│   ├── ToolRegistry.ts
│   └── withToolErrorHandling.ts
├── views/
│   ├── AethonStudioPanel.ts              ← Entry point → studioPanel/
│   ├── studioPanel/
│   │   ├── AethonStudioPanel.ts          ← Singleton WebviewPanel controller
│   │   ├── catalogModels.ts
│   │   ├── playground.ts
│   │   ├── orchestratorTab.ts
│   │   ├── html.ts
│   │   ├── types.ts
│   │   └── index.ts
│   ├── DashboardView.ts
│   ├── SettingsView.ts
│   ├── AethonChatView.ts
│   ├── ActiveAgentsView.ts
│   ├── AgentDebugView.ts
│   └── webviewNonce.ts
├── providers/
│   └── AethonCodeActionProvider.ts       ← Lightbulb: Fix + Explain quick-fixes
├── mcp/
│   └── AethonMcpServer.ts
├── container/ServiceContainer.ts
├── domain/                               ← Interfaces: ICodebaseAnalyzer, IAethonClient, etc.
├── gateway/                              ← FilePromptGateway, HttpAethonGateway
├── application/prompts/PromptRegistry.ts
└── types/protocol.ts

media/knowledge/                          ← Bundled coding principles (auto-copied on first run)
scripts/
├── add-fix-language.js                   ← Scaffold new fixRules language
└── create-issue.js                       ← Programmatic GitHub issue management
.github/workflows/release.yml             ← Auto-release: push to main or workflow_dispatch
```

---

## 16. Release Pipeline

Triggered by: push to `main` (non-md paths) or manual `workflow_dispatch`.

Steps: checkout → read version from `package.json` → skip if tag exists → validate changelog → `npm ci --ignore-scripts` → compile → `npm install -g @vscode/vsce` → `vsce package --no-dependencies` → extract changelog section → create GitHub Release + upload `.vsix`.

`.vscodeignore` excludes: `src/`, `.github/`, `scripts/`, `.aethon/`, `claude.md`, `idea.md`, `*IMPROVEMENT_REPORT*`, `infomaniak_api_*.json`, and all other private/dev files. The `.vsix` contains only: `out/`, `media/`, `package.json`, `readme.md`, `changelog.md`, `LICENSE`, `AGENTS.md`.
