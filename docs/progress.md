# Ghost OS v2 Progress

## Current State: Phases 0-7 COMPLETE. All 29 tools functional.

## What Works (tested end-to-end)
- **29/29 MCP tools** functional (7 Perception, 1 Annotate, 10 Actions, 1 Wait, 5 Recipes, 2 Vision, 3 Learning)
- **gmail-send recipe**: 5/5 success via ghost_run
- **arxiv-download recipe**: 3/3 success (agent-created, 8 steps)
- **Screenshots**: stable on multi-monitor, inline display in Claude Code
- **Cmd+L navigation**: works consistently (no flicker)
- **Scroll**: works on second monitor via element-based scroll
- **All perception tools**: context, state, find, read, inspect, element_at
- **All action tools**: click, type (into field + at focus), press, hotkey, scroll, hover, long_press, drag
- **All wait conditions**: urlContains, elementExists, elementGone, titleContains
- **Recipe CRUD**: list, show, save (with decode error details), delete
- **Annotate**: ghost_annotate with Set-of-Marks labels
- **Vision**: ghost_ground with ShowUI-2B VLM grounding
- **Learning**: ghost_learn_start/stop/status with CGEvent tap recording

## Key Numbers
- 26 Swift files, ~8,950 lines
- ~30+ commits on main
- GitHub: ghostwright/ghost-os

## What's Next

### Phase 8: Testing & Hardening
- Expand unit test coverage beyond LocatorBuilder
- Add CI test workflow to verify builds on push/PR
- Add integration tests for MCP server
- Add Python tests for vision sidecar
- Recipe reliability tests (run each 5x, track success rate)
- Stress test (MCP server up 24 hours)

## Known Items (not blocking)
- Field-finding takes ~11s for deep web apps (exhaustive tree walk)
- elementExists uses contains matching (document in GHOST-MCP.md)
- into:"To" works but into:"To recipients" is more reliable for recipes
