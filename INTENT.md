# INTENT.md — J1-PIPELINE Phase -1 (ORACLE)

**Repository:** `ghostwright/ghost-os`
**Analysis Date:** 2026-07-05
**Analyst:** J1-PIPELINE ORACLE (read-only)
**Status:** Intent Reconstructed

---

## What This System Does

Ghost OS is an **MCP (Model Context Protocol) server** — a Swift binary that gives AI agents the ability to see and operate any macOS application through the macOS accessibility (AX) tree and, when needed, a local vision language model (ShowUI-2B). It exposes **29 MCP tools** across 7 categories:

| Category | Tools | Purpose |
|----------|-------|---------|
| **Perception** (7) | `ghost_context`, `ghost_state`, `ghost_find`, `ghost_read`, `ghost_inspect`, `ghost_element_at`, `ghost_screenshot` | Read what's on screen — structured element data, not pixels |
| **Annotate** (1) | `ghost_annotate` | Set-of-Marks labeled screenshots with numbered interactive elements and click coordinates (zero ML) |
| **Actions** (10) | `ghost_click`, `ghost_type`, `ghost_press`, `ghost_hotkey`, `ghost_scroll`, `ghost_hover`, `ghost_long_press`, `ghost_drag`, `ghost_focus`, `ghost_window` | Click, type, scroll, press keys, manage windows — any app |
| **Wait** (1) | `ghost_wait` | 6 polling conditions: urlContains, titleContains, elementExists, elementGone, urlEquals, titleEquals |
| **Recipes** (5) | `ghost_recipes`, `ghost_run`, `ghost_recipe_show`, `ghost_recipe_save`, `ghost_recipe_delete` | JSON workflow storage and execution with parameter substitution |
| **Vision** (2) | `ghost_ground`, `ghost_parse_screen` | VLM-based element grounding via ShowUI-2B (Python sidecar) |
| **Learning** (3) | `ghost_learn_start`, `ghost_learn_stop`, `ghost_learn_status` | CGEvent tap recording of user actions for recipe synthesis |

### Operational Role

Ghost OS is installed as a **Homebrew formula** (`brew install ghostwright/ghost-os/ghost-os`) and runs as a **stdio-based MCP server** that any MCP-compatible client (Claude Code, Cursor, VS Code, Claude Desktop) connects to. The user runs `ghost setup` once to grant Accessibility, Screen Recording, and Input Monitoring permissions, then their AI agent can:

- **See** what's on screen (structured element data from the AX tree)
- **Operate** any macOS app (click, type, drag, scroll, hotkeys)
- **Learn** workflows by watching the user perform them once
- **Replay** saved workflows as parameterized JSON recipes
- **Fall back** to vision grounding when the AX tree can't find elements (web apps, dynamic content)

The system is **local-first** — no data leaves the machine. The vision model (ShowUI-2B, ~3.0 GB) runs locally via Apple MLX.

---

## Why This Was Built

### Real Problem

AI agents (Claude Code, Cursor, etc.) can write code, run tests, search files, and execute shell commands — but they **cannot interact with GUI applications**. They live inside a chat box. An agent can generate a perfect email, but it cannot click "Compose," fill in the recipient field, and press Send. It can write a Slack message, but it cannot navigate to a channel and type it. This gap between "can generate" and "can execute" is the fundamental barrier to AI agents doing real work on a user's computer.

### Why Existing Tools Were Insufficient

Before Ghost OS, the landscape of computer-use tools was:

| Tool | Approach | Limitations |
|------|----------|-------------|
| **Anthropic Computer Use** | Screenshot-based pixel analysis | Slow, expensive (vision API calls), no structured data, cloud-dependent, closed source |
| **OpenAI Operator** | Screenshot-based pixel analysis | Browser-only, cloud-dependent, closed source, no native app support |
| **OpenClaw** | Browser DOM scraping | Browser-only, no native macOS app support |
| **AppleScript / Automator** | Scriptable macOS automation | Brittle, app-specific, no AI integration, no vision fallback |
| **Hammerspoon / yabai** | macOS window management | Window management only, no element-level interaction, no AI integration |

The common failure mode of screenshot-based approaches: they guess what's on screen, they're slow (3-5s per vision call), they're expensive (API costs), they don't work for native macOS apps, and they have no structured understanding of the UI — they can't tell a button from a label without running a vision model.

### What Triggered Development

Ghost OS was built as a **v2 rewrite** (initial commit: `015335e "Phase 0: project setup with AXorcist dependency, MCP server, and CLI"`). The project was triggered by the convergence of:

1. **MCP protocol standardization** (late 2024/2025) — The Model Context Protocol created a standard way for AI agents to connect to tools, making a general-purpose computer-use server viable.
2. **AXorcist library maturity** — Peter Steinberger's [AXorcist](https://github.com/steipete/AXorcist) provided a modern Swift 6.2-concurrent wrapper around the macOS accessibility API, eliminating the need to write raw AX code.
3. **Local VLM capability** — Apple MLX + ShowUI-2B made it possible to run vision grounding entirely on-device, avoiding cloud API costs and latency.
4. **The "Ghostwright" vision** — Ghost OS is the first component of a three-part ecosystem: Ghost OS (eyes and hands), Shadow (memory and intelligence), Specter (deployment infrastructure). The project was designed from the start as part of a broader platform for AI agents that can truly operate computers.

### Ecosystem Fit

Ghost OS is the **perception and execution layer** of the Ghostwright ecosystem:

```
Ghostwright Ecosystem
│
├── Ghost OS  ← THIS REPO
│   └── Eyes and hands for AI agents on macOS
│       ├── Perception (AX tree reading)
│       ├── Actions (click, type, drag, etc.)
│       ├── Recipes (saved workflows)
│       ├── Vision (ShowUI-2B VLM grounding)
│       └── Learning (CGEvent tap recording)
│
├── Shadow (github.com/ghostwright/shadow)
│   └── Memory and intelligence
│       ├── 14-modality capture
│       ├── Proactive suggestions
│       ├── Episode generation
│       └── On-device LLM inference
│
└── Specter (github.com/ghostwright/specter)
    └── Deployment infrastructure
        ├── Persistent AI agents on dedicated VMs
        ├── Automatic DNS, TLS, systemd hardening
        └── Interactive TUI dashboard
```

Ghost OS is the foundational layer — without it, agents cannot interact with the user's desktop at all. Shadow adds memory and intelligence on top. Specter provides the infrastructure to run agents persistently.

---

## Operational Classification

**Classification: PRODUCTION**

Evidence:
- **9 tagged releases** (v2.0.0 through v2.2.1) with semantic versioning
- **Homebrew formula** published at `ghostwright/ghost-os/ghost-os` — standard macOS distribution channel
- **CI/CD**: CodeQL analysis workflow (`.github/workflows/codeql.yml`) running on push/PR to main
- **Dependabot**: Weekly dependency updates for GitHub Actions (`.github/dependabot.yml`)
- **Security policy** (`SECURITY.md`) with 90-day disclosure timeline and dedicated reporting email
- **Issue templates** (bug report + feature request) and **PR template** with checklist
- **CODE_OF_CONDUCT.md** (Contributor Covenant v2.1)
- **CONTRIBUTING.md** with detailed development setup, code style, and recipe authoring guide
- **500+ GitHub stars** and multiple external contributors (including international contributions from @junshi5218, @mcheemaa)
- **`ghost doctor` diagnostic tool** — comprehensive health check covering permissions, processes, MCP config, recipes, AX tree, vision model, and sidecar
- **`ghost setup` wizard** — interactive first-run configuration handling permissions, MCP config, recipe installation, and vision model download
- **3-way version consistency** enforced at build time (Types.swift + server.py + ghost-vision must match)
- **Codesigned binary** in release tarball (re-signed after copy to preserve adhoc signature)
- **Security audit commit** in git history (`ae397fe audit(ghost-os): sanitize email references`)
- **Structured error responses** with `suggestion` field telling the agent what to do next
- **Timeout hardening** — global AX messaging timeout (5s), per-tool timeout (60s), vision sidecar timeouts (2s health, 30s ground, 60s first-ground)

---

## Key Architectural Decisions

1. **MCP-first, single-threaded synchronous** — No persistent daemon, no socket IPC, no async state model. Every MCP tool call queries the AX tree fresh. This eliminates state management complexity and makes the server trivially restartable. The trade-off: deep AX tree walks (Chrome) can take 10-20s.

2. **stdout is sacred** — All MCP protocol data flows through stdout. The server captures the real stdout FD at init, then redirects stdout → stderr. A single stray `print()` or `Swift.print()` corrupts the protocol. All logging goes to stderr via a custom `Log` type.

3. **AX-native first, synthetic fallback, VLM last** — The action loop: (1) try AX-native via AXorcist's `PerformAction` command, (2) fall back to synthetic CGEvent input at the element's position, (3) if that fails, use VLM vision grounding (ShowUI-2B) to find the element visually. This cascade maximizes reliability while minimizing vision model usage (which is 10x slower).

4. **Semantic depth tunneling** — Empty layout containers (AXGroup with no content) cost 0 depth. A budget of 25 reaches content at 30+ DOM levels. This solves the Chrome/Electron problem where every web element is wrapped in deep AXGroup hierarchies.

5. **Self-learning via CGEvent tap** — The learning subsystem is the one exception to single-threaded: a CGEvent tap runs on a dedicated background thread with `os_unfair_lock` and CFRunLoop. This records user actions (clicks, keystrokes, app switches) enriched with AX context, then a frontier model synthesizes the raw observations into a parameterized, replayable recipe.

6. **Transport auto-detection** — First byte determines Content-Length (Claude Code) vs NDJSON (Claude Desktop) protocol. Response format matches input format. This single design choice supports both major MCP clients without configuration.

7. **Vision sidecar as separate process** — The ShowUI-2B VLM runs as a Python HTTP server (localhost:9876) launched on demand by the Swift binary. This isolates Python dependency management (mlx-vlm, transformers, mlx-lm) from the Swift build and allows the model to stay warm in memory between calls.

8. **3-way version consistency** — Version is defined in `Types.swift` and verified against `server.py` and `ghost-vision` at build time. Mismatch blocks the release. This prevents the common failure mode where the Swift binary and Python sidecar drift apart.

9. **Focus management with save/restore** — Perception tools work from background (no focus needed). Click/type auto-save and restore focus. Press/hotkey/scroll/hover/long_press/drag leave the target focused. Recipe execution saves focus at start and restores at end. This prevents the agent from "losing its place" after interacting with another app.

10. **Recipes as JSON, not code** — Workflows are stored as JSON files with a defined schema (schema_version 2). They are transparent (read every step before running), auditable, shareable, and executable by any MCP client. A frontier model figures out the workflow once; a small model runs it forever.

---

## Repository Structure

```
ghost-os/
├── Sources/
│   ├── GhostOS/                    # Library (MCP server logic)
│   │   ├── MCP/                    # MCPServer, MCPTools, MCPDispatch, WaitManager
│   │   ├── Perception/             # ghost_context, ghost_find, ghost_read, etc.
│   │   ├── Actions/                # ghost_click, ghost_type, ghost_hotkey, etc.
│   │   ├── Recipes/                # RecipeEngine, RecipeStore, RecipeTypes
│   │   ├── Screenshot/             # ScreenCaptureKit wrapper
│   │   ├── Vision/                 # VisionBridge, VisionPerception, CDPBridge
│   │   ├── Learning/               # LearningRecorder, LearningDispatch, etc.
│   │   └── Common/                 # Logger, Types, LocatorBuilder
│   └── ghost/                      # CLI entry point
│       ├── main.swift              # ghost mcp, setup, doctor, status, version
│       ├── SetupWizard.swift       # Interactive first-run setup
│       └── Doctor.swift            # Diagnostic health check
├── Tests/
│   └── GhostOSTests/
│       └── LocatorBuilderTests.swift  # Unit tests (Swift Testing)
├── vision-sidecar/                 # Python VLM sidecar
│   ├── server.py                   # HTTP server (ShowUI-2B grounding)
│   ├── requirements.txt            # Pinned Python dependencies
│   └── ghost-vision                # Shell launcher script
├── recipes/                        # Bundled JSON recipes
│   ├── gmail-send.json
│   ├── slack-send.json
│   ├── arxiv-download.json
│   └── finder-create-folder.json
├── scripts/
│   └── build-release.sh            # Release tarball builder
├── docs/
│   └── progress.md                  # Development progress tracker
├── .github/
│   ├── workflows/codeql.yml         # CodeQL analysis
│   ├── dependabot.yml               # Weekly dependency updates
│   ├── ISSUE_TEMPLATE/              # Bug report + feature request templates
│   └── PULL_REQUEST_TEMPLATE.md     # PR checklist
├── Package.swift                    # Swift Package Manager (Swift 6.2, macOS 14+)
├── Package.resolved                 # Dependency lockfile
├── README.md                        # Primary documentation
├── GHOST-MCP.md                     # Agent instructions (13 rules)
├── CLAUDE.md                        # Developer guide
├── CONTRIBUTING.md                  # Contribution guide
├── CODE_OF_CONDUCT.md               # Contributor Covenant v2.1
├── SECURITY.md                      # Security policy
├── LICENSE                          # MIT
├── .gitignore
├── logo.svg
├── logo-animated.svg
├── logo-animated.gif
├── demo.gif
├── demo-recipes.gif
├── demo-new-tools.gif
└── demo-slack-finder.gif
```

**Notes:**
- No empty directories or submodule issues found.
- Repo name (`ghost-os`) matches README brand (`Ghost OS`) — consistent.
- No Docker/compose files — this is a native macOS app, not containerized.
- Dependabot configured for `github-actions` ecosystem only — appropriate (no npm/Docker ecosystems to mismatch).
- Single test file (LocatorBuilderTests.swift) — test coverage is minimal, focused on the LocatorBuilder utility.

---

## Notes

- **Security audit in git history**: Commit `ae397fe` ("audit(ghost-os): sanitize email references") is a recent security sanitization pass — a positive maturity signal.
- **Vision sidecar fragility**: The CLAUDE.md notes that Python dependency management has caused 3 consecutive patch releases (issues #1, #3, #7). The sidecar requires pinned versions of mlx-vlm, transformers, and mlx-lm. This is a known maintenance burden.
- **`ghost_parse_screen` is a stub**: The YOLO detection endpoint is not yet implemented. For detecting all interactive elements without ML, the README directs users to `ghost_annotate` instead (AX-powered Set-of-Marks).
- **Test coverage is thin**: Only one test file (LocatorBuilderTests.swift) with 5 test cases. E2E testing is manual. This is a gap for a PRODUCTION-classified system.
- **No Docker/containerization**: Appropriate — this is a native macOS app that requires Accessibility permissions, Screen Recording, and Input Monitoring. Containerization would not make sense.
- **The repo is part of the "Ghostwright" organization** on GitHub, not OneByJorah. It is a public-facing open-source project with MIT license, not an internal JorahOne tool.
