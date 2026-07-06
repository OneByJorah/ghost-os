# PHASE 3: GUARDIAN — Security Review

**Repository:** `ghostwright/ghost-os`
**Analysis Date:** 2026-07-05
**Analyst:** J1-PIPELINE GUARDIAN

---

## Security Score: 88/100 — OPERATIONAL

---

## Summary

| Category | Status | Score |
|----------|--------|-------|
| Auth / Authz | ✅ N/A | N/A |
| HTTPS / TLS | ✅ N/A | N/A |
| CSP / Headers | ✅ N/A | N/A |
| Docker Hardening | ✅ N/A | N/A |
| Rootless Containers | ✅ N/A | N/A |
| Supply Chain | ✅ PASS | 90/100 |
| Secrets Management | ✅ PASS | 100/100 |
| SBOM | ⚠️ INFO | N/A |
| AppArmor / SELinux | ✅ N/A | N/A |
| Rate Limiting | ⚠️ INFO | N/A |
| Input Validation | ✅ PASS | 90/100 |
| Permission Model | ✅ PASS | 95/100 |
| Privacy | ✅ PASS | 95/100 |

---

## 1. Auth / Authz

**Status: ✅ N/A**

Ghost OS is a local-first MCP server that communicates over stdio. There is no network authentication or authorization needed — the MCP client (AI agent) runs on the same machine and communicates via stdin/stdout pipes.

The vision sidecar listens on `127.0.0.1:9876` (localhost only) — no network exposure.

---

## 2. HTTPS / TLS

**Status: ✅ N/A**

No network services exposed. The MCP server uses stdio. The vision sidecar uses localhost HTTP only.

---

## 3. CSP / Headers

**Status: ✅ N/A**

No web frontend. No CSP or security headers needed.

---

## 4. Docker Hardening

**Status: ✅ N/A**

No Docker/containerization. This is a native macOS app.

---

## 5. Rootless Containers

**Status: ✅ N/A**

Not applicable.

---

## 6. Supply Chain

**Status: ✅ PASS (90/100)**

### Swift Dependencies
- **AXorcist** (0.1.0) — from `github.com/steipete/AXorcist.git`, pinned to version
- **Commander** (0.2.1) — transitive via AXorcist
- **swift-log** (1.10.1) — transitive via AXorcist

### Python Dependencies
- All version-pinned with upper bounds in `requirements.txt`
- `mlx-vlm==0.1.15` installed with `--no-deps` (documented workaround)
- `mlx-lm<0.30.0` capped to prevent resolver conflicts

### Findings
- ✅ Dependencies are from trusted sources (Apple, @steipete, HuggingFace)
- ✅ Version pinning with upper bounds
- ✅ `Package.resolved` locks exact revisions
- ✅ Dependabot configured for weekly `github-actions` updates
- ⚠️ No automated dependency vulnerability scanning (Dependabot only covers GitHub Actions, not Swift or Python deps)
- ⚠️ No SBOM generation

---

## 7. Secrets Management

**Status: ✅ PASS (100/100)**

- No hardcoded secrets in source code
- No `.env` files committed
- Password field redaction in learning subsystem (`EventHandlers.swift` line 18)
- Password field detection list in `LearningTypes.swift` (lines 135-136)
- Security audit commit `ae397fe` confirms prior sanitization pass
- `.gitignore` covers all build artifacts, caches, and release tarballs

---

## 8. SBOM

**Status: ⚠️ INFO**

No SBOM (Software Bill of Materials) is generated. For a PRODUCTION-classified project with 500+ stars, generating an SBOM would be a best practice. However, the dependency tree is small (3 Swift + 6 Python packages), so the risk is low.

---

## 9. AppArmor / SELinux

**Status: ✅ N/A**

macOS-only application. AppArmor/SELinux are Linux security modules.

---

## 10. Rate Limiting

**Status: ⚠️ INFO**

No rate limiting or backpressure mechanism in the MCP server. An aggressive agent could flood the server with requests. However, since the MCP server is single-threaded and synchronous, it naturally serializes requests — the agent must wait for each response before sending the next request. This provides implicit rate limiting.

**Risk:** Low — the stdio protocol is inherently serial.

---

## 11. Input Validation

**Status: ✅ PASS (90/100)**

### MCP Tool Parameters
- All 29 tools have typed parameter schemas defined in `MCPTools.swift`
- Required parameters are enforced
- Parameter types are validated (string, number, boolean, array)
- Error responses include `suggestion` field

### Vision Sidecar
- Request body size limited to 50 MB (`MAX_BODY_SIZE`)
- JSON parsing with try/catch
- Base64 image decoding with error handling
- File path traversal not applicable (all paths are resolved from known locations)

### Findings
- ✅ Typed parameter schemas for all tools
- ✅ Required parameter enforcement
- ✅ Structured error responses with suggestions
- ⚠️ No explicit input sanitization for string parameters (e.g., recipe names, app names) — relies on AXorcist to handle invalid input gracefully

---

## 12. Permission Model

**Status: ✅ PASS (95/100)**

Ghost OS requires three macOS permissions:
1. **Accessibility** — Required for AX tree access (AXorcist)
2. **Screen Recording** — Required for screenshots (ScreenCaptureKit)
3. **Input Monitoring** — Required for learning mode (CGEvent tap)

### Findings
- ✅ `ghost setup` wizard handles permission configuration interactively
- ✅ `ghost doctor` checks all permissions and reports status
- ✅ `ghost status` checks Accessibility permission
- ✅ Permissions are checked at runtime before attempting operations
- ✅ Learning mode explicitly requires Input Monitoring (separate from Accessibility)
- ✅ Password fields are automatically redacted in learning recordings
- ✅ Recordings are ephemeral (in-memory only, never written to disk)

---

## 13. Privacy

**Status: ✅ PASS (95/100)**

- **Local-first:** All processing happens on-device. No data leaves the machine.
- **Vision model:** ShowUI-2B runs locally via Apple MLX. No cloud API calls.
- **Learning:** Recordings are ephemeral (in-memory only). Password fields redacted.
- **Recipes:** JSON files stored locally in `~/.ghost-os/recipes/`.
- **No telemetry:** No analytics, crash reporting, or usage tracking.

### Findings
- ✅ No cloud dependencies for core functionality
- ✅ No telemetry or analytics
- ✅ Password redaction in learning
- ✅ Ephemeral recordings
- ⚠️ The vision sidecar downloads the ShowUI-2B model from HuggingFace on first use — this is a one-time download, not continuous data exfiltration

---

## Security Findings Summary

### CRITICAL Items
- None

### DEGRADED Items
- None

### INFO Items
1. **No SBOM generation** — Low risk given small dependency tree
2. **No automated dependency vulnerability scanning** — Dependabot only covers GitHub Actions
3. **No explicit input sanitization** for string parameters — low risk, relies on AXorcist
4. **Vision sidecar model download** from HuggingFace on first use — one-time, documented

---

## Security Recommendations

1. **Add Dependabot for pip ecosystem** — Currently only configured for `github-actions`. Adding `pip` would catch Python dependency CVEs.
2. **Consider SBOM generation** — `cyclonedx-swift` or similar tool could generate an SBOM for the Swift dependencies.
3. **Add input sanitization** for string parameters that are used in file paths or shell commands (recipe names, app names).
4. **Document the security model** in SECURITY.md — currently SECURITY.md only covers vulnerability reporting, not the security architecture.
