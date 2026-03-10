# Aethon Mission Control — Public Changelog

This changelog tracks all released versions and acknowledges user-requested features.

> 💡 **Feature requests** are tracked as [GitHub Issues](https://github.com/JeanPaul20/aethon/issues) on this repository.
> When a user-requested feature is implemented, it is marked here with 📬 and linked to the original issue.

---

## [1.3.4] — 2026-03-09

### Added
- **Training Data tab** in Aethon Studio — export Alpaca, ShareGPT, and DPO formats from within the panel
- **Settings tab** in Aethon Studio — full provider configuration inside the panel
- **Run Backend button** — start the local backend server directly from the UI
- New `media/forms/` architecture — each Studio tab is now a separate JS module

### Fixed
- Auto-update now performs a clean uninstall before installing the new version, preventing service worker corruption
- Startup no longer warns about the optional `SELF_IMPROVEMENT_PROMPT.md` file

---

## [1.3.3] — 2026-02-25

### Added
- Aethon Studio Panel with Model Catalog, Playground, Coding, and Orchestrator tabs
- Multi-provider catalog: GitHub Copilot, Ollama, Infomaniak, OpenAI, Perplexity, Kimi, Custom providers
- Model assignment system — assign models to Proposal, Evaluation, Orchestrator, Coder, Embeddings roles
- Setup credential modal — configure API keys directly from the catalog card
- Privacy-tiered provider classification: Local / Private / Cloud / Built-in

---

## How User Requests Are Tracked

When a feature request submitted via the extension is implemented:

```
📬 [#12] Feature: <title>
   Requested by: user (GitHub issue #12, submitted 2026-03-10)
   Implemented in: v1.4.0
```

---

*Aethon is developed by [JeanPaul20](https://github.com/JeanPaul20). Source code is private. Releases and feedback are managed via this public repository.*
