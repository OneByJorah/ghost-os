# PHASE 2: ARCHITECT — Architecture Review

**Repository:** `ghostwright/ghost-os`
**Analysis Date:** 2026-07-05
**Analyst:** J1-PIPELINE ARCHITECT

---

## Architecture Score: 88/100 — OPERATIONAL

---

## Overview

Ghost OS is an MCP (Model Context Protocol) server that gives AI agents the ability to see and operate macOS applications through the accessibility (AX) tree. It is a single-threaded, synchronous Swift binary with a Python VLM sidecar for vision grounding.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                   AI Agent (MCP Client)                  │
│          Claude Code / Cursor / VS Code / Claude Desktop │
└────────────────────────┬────────────────────────────────┘
                         │ MCP Protocol (stdio)
                         │ Auto-detects Content-Length vs NDJSON
                         ▼
┌─────────────────────────────────────────────────────────┐
│              ghost mcp (Swift, @MainActor)               │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────┐ │
│  │Perception │  │ Actions  │  │ Recipes  │  │Learning │ │
│  │ (7 tools) │  │(10 tools)│  │ (5 tools)│  │(3 tools)│ │
│  └─────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬────┘ │
│        │              │              │              │      │
│  ┌─────┴──────────────┴──────────────┴──────────────┘    │
│  │                    AXorcist                            │
│  │         (macOS Accessibility API Wrapper)              │
│  └────────────────────────┬───────────────────────────────┘
│                           │
│  ┌────────────────────────┴───────────────────────────────┐
│  │              ScreenCaptureKit (Screenshots)             │
│  └────────────────────────────────────────────────────────┘
│                                                          │
│  ┌────────────────────────────────────────────────────────┐
│  │           Vision Bridge (HTTP to sidecar)              │
│  └────────────────────────┬───────────────────────────────┘
└───────────────────────────┼───────────────────────────────┘
                            │ HTTP (localhost:9876)
                            ▼
┌─────────────────────────────────────────────────────────┐
│           Vision Sidecar (Python, ShowUI-2B)              │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐ │
│  │ /health     │  │ /ground     │  │ /detect, /parse  │ │
│  │ (status)    │  │ (VLM coord) │  │ (stubs)          │ │
│  └─────────────┘  └─────────────┘  └──────────────────┘ │
│                                                          │
│  ┌──────────────────────────────────────────────────────┐│
│  │  ShowUI-2B (MLX) — Local VLM, ~3.0 GB, lazy-loaded  ││
│  └──────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

---

## Key Architectural Decisions

### 1. MCP-first, Single-Threaded Synchronous ✅

**Decision:** No persistent daemon, no socket IPC, no async state model. Every MCP tool call queries the AX tree fresh.

**Assessment:** Excellent for a computer-use tool. Eliminates state management complexity, makes the server trivially restartable, and avoids race conditions. The trade-off (deep AX tree walks taking 10-20s for Chrome) is acceptable — the alternative (stateful daemon) would introduce far more bugs.

### 2. stdout is Sacred ✅

**Decision:** All MCP protocol data flows through stdout. The server captures the real stdout FD at init, redirects stdout → stderr. A single stray `print()` corrupts the protocol.

**Assessment:** Correct and well-implemented. The `Log` struct enforces stderr-only output. This is a common failure mode in MCP servers and Ghost OS handles it properly.

### 3. AX-native First, Synthetic Fallback, VLM Last ✅

**Decision:** Action cascade: (1) AX-native via AXorcist, (2) synthetic CGEvent at element position, (3) VLM vision grounding.

**Assessment:** Optimal strategy. AX-native is instant and reliable for native apps. Synthetic input works when AX fails. VLM is 10x slower and used only as last resort. This cascade maximizes reliability while minimizing latency.

### 4. Semantic Depth Tunneling ✅

**Decision:** Empty layout containers (AXGroup with no content) cost 0 depth. Budget of 25 reaches content at 30+ DOM levels.

**Assessment:** Clever solution to the Chrome/Electron problem where every web element is wrapped in deep AXGroup hierarchies. Well-documented in code and CLAUDE.md.

### 5. Self-Learning via CGEvent Tap ✅

**Decision:** Learning subsystem runs on a dedicated background thread with `os_unfair_lock` and CFRunLoop. Records user actions enriched with AX context.

**Assessment:** Well-architected. The CGEvent tap is the one exception to single-threaded design, and the isolation is properly handled. Password redaction is built in. The ephemeral (in-memory only) design is a good privacy choice.

### 6. Transport Auto-Detection ✅

**Decision:** First byte determines Content-Length (Claude Code) vs NDJSON (Claude Desktop) protocol. Response format matches input format.

**Assessment:** Elegant solution. Supports both major MCP clients without configuration. Single design choice eliminates a whole class of configuration issues.

### 7. Vision Sidecar as Separate Process ✅

**Decision:** ShowUI-2B VLM runs as a Python HTTP server (localhost:9876) launched on demand.

**Assessment:** Good isolation. Separates Python dependency management from Swift build. Allows the model to stay warm in memory. The idle timeout (600s default) auto-shuts down the sidecar to save resources.

### 8. 3-Way Version Consistency ✅

**Decision:** Version defined in `Types.swift` and verified against `server.py` and `ghost-vision` at build time.

**Assessment:** Essential for a system with multiple components. Prevents the common failure mode where the Swift binary and Python sidecar drift apart. The build script enforces this at release time.

### 9. Focus Management with Save/Restore ✅

**Decision:** Perception tools work from background. Click/type auto-save and restore focus. Other actions leave target focused. Recipe execution saves focus at start and restores at end.

**Assessment:** Well-thought-out UX. Prevents the agent from "losing its place" after interacting with another app. The save/restore pattern is clean.

### 10. Recipes as JSON, Not Code ✅

**Decision:** Workflows stored as JSON files with schema_version 2. Transparent, auditable, shareable.

**Assessment:** Excellent design. JSON is safer than code (no arbitrary execution), transparent (read every step before running), and portable across MCP clients. The schema is well-documented in GHOST-MCP.md.

---

## Architecture Strengths

1. **Clean separation of concerns** — 7 modules (MCP, Perception, Actions, Recipes, Screenshot, Vision, Learning) with clear responsibilities
2. **Minimal dependencies** — Only 1 direct Swift dependency (AXorcist) + 6 Python packages
3. **Defensive design** — Error responses include `suggestion` field, timeouts at every level, focus save/restore
4. **Privacy-first** — Local-only, no data leaves the machine, password redaction in learning
5. **Self-documenting tools** — MCP tool descriptions are comprehensive and include usage patterns
6. **Graceful degradation** — Action cascade ensures fallback at every level

---

## Architecture Concerns

### ⚠️ Concern 1: No Async/Await for Long Operations

The single-threaded synchronous model means deep AX tree walks (Chrome with 30+ DOM levels) block the MCP server for 10-20 seconds. While acceptable for the current use case, this could become a problem with:
- Multiple concurrent MCP clients
- Timeout-sensitive agents
- Complex multi-step operations

**Recommendation:** Consider structured concurrency (Swift async/await) for long-running operations, while keeping the synchronous model for fast operations.

### ⚠️ Concern 2: Vision Sidecar Fragility

The CLAUDE.md explicitly notes that Python dependency management has caused 3 consecutive patch releases (issues #1, #3, #7). The `--no-deps` install for `mlx-vlm` and the `transformers<4.49` pin are workarounds for upstream dependency conflicts.

**Recommendation:** Consider containerizing the vision sidecar (even a simple Docker image) to isolate Python dependencies. Alternatively, investigate if ShowUI-2B can be replaced with a more stable VLM.

### ⚠️ Concern 3: CDPBridge Usage Unclear

`CDPBridge.swift` (311 lines) implements Chrome DevTools Protocol integration, but it's unclear from the code whether this is actively used or a vestigial component. The action cascade mentions CDP but the README and GHOST-MCP.md don't document it.

**Recommendation:** Audit CDPBridge usage. If unused, remove it. If used, document it.

### ⚠️ Concern 4: No Graceful Shutdown

The MCP server has no signal handling for graceful shutdown. A SIGTERM or SIGINT during an AX tree walk could leave the accessibility API in an inconsistent state.

**Recommendation:** Add signal handlers for SIGTERM/SIGINT to allow in-progress operations to complete or abort cleanly.

### ⚠️ Concern 5: No Rate Limiting

The MCP server has no rate limiting or backpressure mechanism. An aggressive agent could flood the server with requests, overwhelming the AX API.

**Recommendation:** Consider adding a simple rate limiter or concurrent request limit.

---

## Architecture Score Breakdown

| Category | Score | Notes |
|----------|-------|-------|
| Separation of Concerns | 95 | Clean module boundaries |
| Dependency Management | 90 | Minimal deps, well-pinned |
| Error Handling | 95 | Suggestion field, timeouts |
| Performance Design | 80 | Sync model limits throughput |
| Security Architecture | 90 | Local-first, password redaction |
| Extensibility | 85 | Module structure supports growth |
| Documentation of Architecture | 90 | CLAUDE.md, README, GHOST-MCP.md |
| **Overall** | **88** | |

---

## Conclusion

The architecture is well-designed for its purpose — a single-threaded MCP server for macOS computer-use. The key decisions (MCP-first, AX-native cascade, JSON recipes, transport auto-detection) are sound and well-executed. The main concerns are around scalability (sync model for long operations) and the vision sidecar's dependency fragility.
