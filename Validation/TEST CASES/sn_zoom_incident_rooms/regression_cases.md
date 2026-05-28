# Regression Cases: sn_zoom_incident_rooms

**Author:** Vladimir Kapustin | **License:** AGPL-3.0-only  
**Date:** 2026-05-28

## Purpose

Verify that previously fixed bugs and known fragile behaviors remain resolved across releases. Each regression case is traceable to a specific bug report or QA finding. Cases use RXX identifiers.

## Regression Case Register

| ID | Severity | Case | Bug Origin | Reproduction Steps | Expected Behavior | Auto-test Coverage |
|----|----------|------|------------|-------------------|-------------------|-------------------|
| R01 | P0 | **Duplicate rooms on rapid P1 flood** | v0.1 — race condition: 3 P1 incidents for same group within 1s created 3 Zoom meetings | Submit 3 P1 incidents simultaneously via REST API with same assignment group | Exactly 1 room created; 2 incidents reference same `join_url`; log shows 1 CREATED + 2 MERGED | `test_T03_duplicate_prevention` |
| R02 | P0 | **OAuth token expiry during multi-room batch** | v0.2 — token expired mid-batch of 5 rooms; only first 2 were created | Set token to expire in 1 second; trigger 5 room creations | Token refreshed between room 1 and room 2; all 5 rooms succeed; 0 FAILED entries | `test_T05_oauth_refresh` |
| R03 | P1 | **Incident resolved before room creation completes** | v0.3 — resolve→close race: user resolved incident while Zoom API was still pending | Trigger room creation; immediately resolve incident via REST while creation in-flight | Room completes creation; then close process runs; log shows CREATED → CLOSED (not stuck at CREATING) | Manual — requires timing control beyond unit test mock |
| R04 | P1 | **Zoom meeting deleted externally** | v0.4 — admin deleted Zoom meeting from Zoom web portal; ServiceNow log still showed ACTIVE | Create room normally; DELETE meeting via Zoom web dashboard; run reconciliation job | Reconciliation job detects 404 on meeting ID; updates log to CLOSED with `closed_reason=EXTERNAL_DELETE` | `test_T04_room_close` (partial) |
| R05 | P2 | **Unicode characters in incident short description** | v0.5 — Japanese characters in incident title caused Zoom API to return 400 Bad Request | Create P1 incident with `short_description = "データベース障害 — 本番環境"` | Room created successfully; Zoom topic contains sanitized ASCII-safe version; original text logged | `test_T01_basic_room_creation` (extended) |
| R06 | P2 | **Assignment group renamed mid-session** | v0.6 — group "Network Ops" renamed to "Network Operations"; on-call mapping broke | Create room for group "Network Ops"; rename group via `sys_user_group`; create new incident for renamed group | Both incidents map to correct on-call via `sys_id` (not group name); name change is transparent | `test_T02_oncall_invitation` (extended) |
| R07 | P1 | **Zoom API returns HTML error page instead of JSON** | v0.7 — Zoom CDN/proxy returned HTML 502 page; JSON parser crashed with `JSONDecodeError` | Mock `requests.post` to return `text/html` body with 502 status | Graceful error handling: retry once; if still HTML, treat as permanent failure, log `FAILED`, alert admin | `test_T07_zoom_permanent_failure` (extended) |
| R08 | P3 | **sys_created_on mismatch between incident and log** | v0.8 — log entry `sys_created_on` was 2s before incident `sys_created_on` due to clock skew | Compare timestamps in `x_sn_zoom_incident_rooms_log` against `incident.sys_created_on` | Difference < 5 seconds (acceptable due to transaction sequencing) | `test_T01_basic_room_creation` (assertion) |

## Regression Execution

```bash
# Full regression suite
pytest tests/test_regression.py -v --tb=long

# Quick smoke (P0 only)
pytest tests/test_regression.py -v -k "R01 or R02"
```

## Minimum Pass Threshold

- **P0 regression (R01–R02):** 2/2 MUST pass
- **P1 regression (R03–R04, R07):** 3/3 MUST pass  
- **P2 regression (R05–R06):** 2/2 MUST pass
- **P3 regression (R08):** 1/1 SHOULD pass
- **Total:** 8/8 ALL PASS before release
