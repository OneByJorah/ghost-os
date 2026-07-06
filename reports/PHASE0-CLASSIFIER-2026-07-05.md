# PHASE 0: CLASSIFIER — Project Classification

**Repository:** `ghostwright/ghost-os`
**Analysis Date:** 2026-07-05
**Analyst:** J1-PIPELINE CLASSIFIER

---

## Classification

**PROJECT_CLASS: `Swift`, `macOS`, `MCP`, `AI`, `Agent`, `CLI`**

### Primary Class: `Swift` / `macOS`

This is a native macOS application written in Swift 6.2, targeting macOS 14+. It is a Swift Package Manager project with a library target (`GhostOS`) and an executable target (`ghost`).

### Secondary Classes

| Class | Rationale |
|-------|-----------|
| **MCP** | Core protocol: implements the Model Context Protocol (MCP) as a stdio-based server with 29 tools. Transport auto-detects Content-Length vs NDJSON. |
| **AI** | Integrates with AI agents (Claude Code, Cursor, VS Code, Claude Desktop) via MCP. Includes a VLM sidecar (ShowUI-2B) for vision grounding. |
| **Agent** | Provides computer-use capabilities to AI agents — perception, actions, recipes, learning. |
| **CLI** | Ships as a command-line tool (`ghost mcp`, `ghost setup`, `ghost doctor`, `ghost status`, `ghost version`). |

### Excluded Classes

| Class | Why Excluded |
|-------|-------------|
| Docker | No Docker/Compose files — this is a native macOS app requiring Accessibility/Screen Recording/Input Monitoring permissions. Containerization is not applicable. |
| Web | No web frontend. The MCP server communicates via stdio, not HTTP. |
| Python | The vision sidecar is Python, but it's a supporting component (~623 lines), not the primary project. |
| Library | While a `GhostOS` library target exists, the primary deliverable is the `ghost` CLI executable. |
| Documentation | Documentation is thorough but the project is a functional tool, not a documentation repo. |
| Template | Not a template or starter project. |

---

## Classification Evidence

1. **Swift Package Manager** — `Package.swift` with `swift-tools-version: 6.2`, targets `macOS .v14`
2. **MCP protocol** — `MCPServer.swift`, `MCPDispatch.swift`, `MCPTools.swift` implement the full MCP stdio server
3. **AI agent integration** — `GHOST-MCP.md` provides 13 rules for AI agents using the tools
4. **VLM sidecar** — `vision-sidecar/server.py` runs ShowUI-2B for vision grounding
5. **CLI entry points** — `main.swift` dispatches to `mcp`, `setup`, `doctor`, `status`, `version`
6. **macOS-specific APIs** — AXorcist (accessibility), ScreenCaptureKit, CGEvent taps, NSWorkspace
7. **Homebrew distribution** — Published as `ghostwright/ghost-os/ghost-os` Homebrew formula

---

## Pipeline Implications

- **No Docker hardening checks** needed (Phase 3 GUARDIAN skips Docker-specific items)
- **No container deployment** (Phase 5 DEPLOYER skips Compose/Swarm/K3s checks)
- **AI/Agent review** (Phase 11 AI REVIEW) is applicable — prompt quality, tool-calling correctness
- **GitHub curation** (Phase 12) is applicable — public-facing open-source project
- **Community standards** (Phase 15) are applicable — public repo with external contributors
