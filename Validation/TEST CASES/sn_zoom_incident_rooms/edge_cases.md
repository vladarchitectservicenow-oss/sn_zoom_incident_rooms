# Edge Cases: sn_zoom_incident_rooms

**Author:** Vladimir Kapustin | **License:** AGPL-3.0-only  
**Date:** 2026-05-28

## Purpose

Document boundary conditions and unusual inputs that the system must handle gracefully. Each edge case identifies the breaking point and the expected system behavior.

## Edge Case Catalog

| ID | Category | Edge Case | Trigger Condition | Expected Behavior | Current Status |
|----|----------|-----------|-------------------|-------------------|----------------|
| E01 | **Input** | Incident short_description is empty string (`""`) | P1 incident created with no short description | Zoom topic defaults to `"[P1] Incident {sys_id} — No Description"`; room created normally | ✅ DESIGNED |
| E02 | **Input** | Incident short_description is 500+ characters | Zoom API limit is 200 chars for topic field | Topic truncated to 197 chars + `"..."`; full description logged to audit table | ✅ DESIGNED |
| E03 | **Auth** | Zoom OAuth client_secret contains special characters (`/`, `+`, `=`) | Standard base64-encoded Basic auth header construction | Proper URL-encoding for special chars; auth succeeds | ⚠️ NEEDS TEST |
| E04 | **Network** | Zoom API returns HTTP 429 Rate Limited with `Retry-After: 30` header | Burst of 10+ simultaneous room creations | Respect `Retry-After` header; queue remaining rooms with delay; log RATE_LIMITED event | ⚠️ NEEDS IMPLEMENTATION |
| E05 | **Data** | On-call schedule spans midnight UTC | Current time is 23:55 UTC; on-call shift ends at 00:00 | `cmn_schedule_span` correctly returns the user whose span includes 23:55 (not the one starting at 00:00) | ✅ DESIGNED |
| E06 | **Platform** | ServiceNow instance timezone ≠ UTC | Instance in America/New_York (-5); `sys_created_on` shows local time | Zoom meeting `start_time` uses UTC as required by Zoom API; `sys_created_on` stored as-is; conversion only for comparisons | ✅ DESIGNED |
| E07 | **Concurrency** | Two Business Rules fire simultaneously for same incident (before/after update) | Incident assigned to group AND priority changed in same transaction | Deduplication via `incident_sys_id` lock; second trigger sees ACTIVE room and skips; no duplicate creation | ✅ DESIGNED |
| E08 | **Data Loss** | `x_sn_zoom_incident_rooms_config` table is empty (first install) | No Zoom configuration exists | Return clear error: "Zoom not configured. Please set up x_sn_zoom_incident_rooms_config." Do NOT crash. | ✅ DESIGNED |
| E09 | **External** | Zoom user account deleted or deprovisioned | `zoom_user_email` in on-call map no longer exists in Zoom | Zoom API returns 404 "User not found"; escalate to assignment group manager; log WARNING; mark on-call map entry as INACTIVE | ⚠️ NEEDS IMPLEMENTATION |

## Test Execution Priority

| Priority | Edge Cases | Impact if Untested |
|----------|-----------|-------------------|
| P0 | E01, E02, E08 | System crashes on null/empty/missing inputs |
| P1 | E05, E06, E07 | Silent wrong behavior (wrong user invited, wrong time) |
| P2 | E03, E04 | Graceful degradation on API issues |
| P3 | E09 | Rare external event, manual workaround exists |
