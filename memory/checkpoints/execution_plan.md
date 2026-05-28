# sn_zoom_incident_rooms — Execution Plan

**Author:** Vladimir Kapustin  
**Date:** 2026-05-28  
**Status:** ACTIVE  
**Release:** AUSTRALIA

## Phase Breakdown

### ✅ Phase 1: Architecture & Planning (COMPLETE)
- [x] Architecture summary with component diagram and API contract
- [x] Dependency report with plugin IDs, roles, network requirements
- [x] Risk register with 12 entries across P0–P3, mitigation owners
- [x] Execution plan (this document)

### ✅ Phase 2: Test Suite Design (COMPLETE)
- [x] `test_suite_SOP.md` with 12 test scenarios (T01–T12)
- [x] `regression_cases.md` with 8 regression cases (R01–R08)
- [x] `edge_cases.md` with 9 edge cases
- [x] `validation_checklist.md`

### 🔧 Phase 3: Core Engine (src/)
**Actions:**
1. Expand `engine.py` from generic REST fetcher to Zoom-specific: OAuth token flow, meeting creation via Zoom API v2, on-call resolution
2. Add `zoom_auth.py` — OAuth 2.0 Server-to-Server token management with auto-refresh
3. Add `oncall_resolver.py` — query `cmn_schedule` + `sys_user_grmember` for current on-call
4. Add `audit_logger.py` — write to `x_sn_zoom_incident_rooms_log` with transaction safety
5. Update `cli.py` for Zoom-specific args: `--zoom-account-id`, `--zoom-client-id`, `--zoom-client-secret`

**Files to create/modify:**
- `src/engine.py` → ZoomRoomManager class
- `src/zoom_auth.py` (new)  
- `src/oncall_resolver.py` (new)
- `src/audit_logger.py` (new)
- `src/cli.py` → update arguments
- `src/sys_app.xml` → scoped app definition

### 🔧 Phase 4: Tests (tests/)
**Actions:**
1. Extend `test_engine.py` from 7 to 12+ test cases matching T01–T12
2. Add mocked Zoom API responses using `unittest.mock.patch`
3. Add OAuth error scenarios (401, token expiry)
4. Add on-call resolution tests with mocked GlideRecord queries
5. Ensure all tests pass: `pytest tests/ -v` → 12/12 PASS

### 🔧 Phase 5: Documentation & Marketing (COMPLETE)
- [x] README expanded with Mermaid architecture diagram, ROI analysis, troubleshooting table
- [x] LICENSE is AGPL-3.0 full text (624 lines)
- [x] SOP.md with test inventory

### 🔧 Phase 6: Quality Gates
**Pre-push checks:**
| Gate | Rule | Status |
|------|------|--------|
| G0 | `test_suite_SOP.md` ≥10 scenarios with T01-TXX format | PASS (12 scenarios) |
| G1 | Tests execution log exists | PENDING |
| G2 | README ≥2000 words, no duplicate sections | PENDING (deduplication needed) |
| G3 | Every `.py` file has AGPL-3.0 copyright header | PENDING |
| G4 | Git push verified via API | PENDING |
| G5 | No hardcoded credentials | PASS (uses env vars) |
| G6 | `.gitignore` excludes `__pycache__/`, `*.pyc`, `reports/` | PENDING |
| G7 | README license header matches LICENSE (AGPL-3.0) | PASS |
| G8 | No duplicate README sections | PENDING (73 `## ` found, needs fix) |

### 🔧 Phase 7: Git & Deployment
**Actions:**
1. `git add -A`
2. Verify staged files: `git diff --cached --stat`
3. `git commit -m "docs: Phase 1-2 regeneration + README dedup for sn_zoom_incident_rooms"`
4. Push via Python script with token from `.env`
5. Verify remote: `curl` GitHub API for commit SHA
6. Create `DONE.marker`

### 🎯 Phase 8: Completion
- [ ] `DONE.marker` pushed to remote
- [ ] README verified on `raw.githubusercontent.com` (word count ≥2000, no duplicates)
- [ ] Phase 1 docs verified on GitHub Contents API (all 4 files >30 lines)
- [ ] Phase 2 docs verified (SOP.md with 10+ TXX scenarios)
- [ ] Pipeline progress updated

## Timeline

| Phase | Estimated Effort | Dependencies |
|-------|-----------------|--------------|
| Phase 1 (Architecture) | Done | None |
| Phase 2 (Test Design) | Done | Phase 1 |
| Phase 3 (Core Engine) | 2-3 hours | Phase 2 |
| Phase 4 (Tests) | 2 hours | Phase 3 |
| Phase 5 (Docs) | Done | Phase 1 |
| Phase 6 (Quality Gates) | 30 min | Phase 3-5 |
| Phase 7 (Deploy) | 15 min | Phase 6 |
| Phase 8 (Verify) | 15 min | Phase 7 |

## Rollback Plan

If Zoom API integration fails during PDI testing:
1. Disable Business Rule via `sys_script.do` (set `active=false`)
2. Revert `src/engine.py` to generic REST fetcher fallback
3. Run regression tests — all must pass
4. Retry Zoom OAuth configuration with fresh credentials
5. Re-enable Business Rule
