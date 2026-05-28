# Test Suite SOP: sn_zoom_incident_rooms

**Author:** Vladimir Kapustin | **License:** AGPL-3.0-only  
**Release:** AUSTRALIA | **Scope:** `x_sn_zoom_incident_rooms`  
**Date:** 2026-05-28

## Overview

This SOP defines the complete test suite for SN Zoom Incident Rooms. All scenarios use TXX identifiers (T01–T12). Tests cover core functionality (Zoom room creation, on-call resolution), error handling (auth failures, API timeouts), edge cases (duplicate incidents, re-opened incidents), and performance (concurrent room creation).

## Test Environment

- **ServiceNow PDI:** `dev362840.service-now.com` (or any AUSTRALIA instance)
- **Python:** 3.10+ with `pytest`, `requests`, `unittest.mock`
- **Zoom API mock:** All Zoom calls mocked via `unittest.mock.patch` — no real Zoom account needed for unit tests
- **GlideRecord mock:** Simulated via Python dicts in `tests/conftest.py`

## Test Scenarios

| ID | Priority | Scenario | Pre-conditions | Expected Result | Mock Strategy |
|----|----------|----------|---------------|-----------------|---------------|
| T01 | P0 | **Basic room creation** — P1 incident triggers automatic Zoom meeting | Incident with `priority=1`, `assignment_group=Network Ops`, valid Zoom OAuth config | Zoom meeting created via API, `zoom_meeting_id` + `join_url` written to `x_sn_zoom_incident_rooms_log`, status=`CREATED` | Mock `ZoomAPIClient.create_meeting()` returns `{"meeting_id":"123","join_url":"https://zoom.us/j/123"}` |
| T02 | P0 | **On-call user invitation** — correct on-call engineer auto-invited | `cmn_schedule` returns `john.doe@company.com` for current shift | `john.doe@company.com` added as Zoom meeting host; audit log records invited user | Mock `OnCallResolver.get_current()` returns `["john.doe@company.com"]` |
| T03 | P0 | **Duplicate incident prevention** — two P1 incidents for same group within 5 minutes | First incident already has active room; second incident triggers within 5 min | Second incident posts existing `join_url` to work notes; no new meeting created; audit log shows `status=MERGED` | Query `x_sn_zoom_incident_rooms_log` with `status=ACTIVE AND assignment_group=X` returns existing room |
| T04 | P0 | **Room close on incident resolution** — meeting archived when incident resolved | Incident resolved (`state=6`), room is active | Zoom meeting ended via DELETE API; audit log updated to `status=CLOSED`; `closed_on` timestamp set | Mock Zoom DELETE returns 204 |
| T05 | P1 | **OAuth token refresh** — expired token auto-refreshed | Current token expires in 2 minutes; room creation triggered | Token refreshed before API call; new token used; audit log records token refresh event | Mock OAuth endpoint returns new token with 60-minute expiry |
| T06 | P1 | **Zoom API timeout** — retry on transient failure | Zoom API returns 503 Service Unavailable (attempt 1) | Automatic retry after 2s, then 4s, then 8s; 3rd attempt succeeds; total latency < 15s | Mock `requests.post` raises `Timeout` on calls 1-2, succeeds on call 3 |
| T07 | P1 | **Zoom API permanent failure** — max retries exhausted | Zoom API returns 500 on all 3 attempts | Room status set to `FAILED`; admin notification sent; incident work notes post "Zoom room creation failed — manual action required" | Mock all `requests.post` calls raise `HTTPError(500)` |
| T08 | P2 | **Empty on-call schedule** — no one on shift | `cmn_schedule_span` returns no active spans for current time | Fallback to assignment group manager; second fallback to incident assignee; audit log shows `status=CREATED` with `fallback_reason=NO_ONCALL` | Mock `OnCallResolver` returns `[]`, mock `getGroupManager()` returns `manager@company.com` |
| T09 | P2 | **Re-opened incident** — incident resolved then re-opened | Room was CLOSED 30 minutes ago; incident re-opened with `state=2` | Existing room re-opened (within `archive_after_hours=24`); join URL re-posted to work notes; status updated to `REOPENED` | Query log shows room closed 30 min ago (less than 24h threshold) |
| T10 | P2 | **Re-opened after archive window** — incident reopened >24h after close | Room was CLOSED 26 hours ago; incident re-opened | New Zoom meeting created (old room archived); new log entry with `status=CREATED`; old log still shows `status=CLOSED` | Query log shows room closed 26h ago (exceeds 24h threshold) |
| T11 | P1 | **Concurrent room creation (50 active)** — max active rooms reached | 50 rooms currently `ACTIVE`; new P1 incident triggered | Room creation queued with `status=QUEUED`; processed when an ACTIVE room closes; admin notification sent | Mock log query returns 50 ACTIVE rooms |
| T12 | P3 | **Non-P1 incident** — P3 incident does not trigger room creation | Incident with `priority=3` | No Zoom API call made; no log entry created; Business Rule exits early | Verify `ZoomRoomManager` is never instantiated |

## Execution

```bash
# Run all tests
pytest tests/ -v --tb=short

# Expected output
# test_engine.py::test_T01_basic_room_creation PASSED
# test_engine.py::test_T02_oncall_invitation PASSED
# test_engine.py::test_T03_duplicate_prevention PASSED
# test_engine.py::test_T04_room_close PASSED
# test_engine.py::test_T05_oauth_refresh PASSED
# test_engine.py::test_T06_zoom_timeout_retry PASSED
# test_engine.py::test_T07_zoom_permanent_failure PASSED
# test_engine.py::test_T08_empty_oncall PASSED
# test_engine.py::test_T09_reopen_within_window PASSED
# test_engine.py::test_T10_reopen_beyond_window PASSED
# test_engine.py::test_T11_concurrent_cap PASSED
# test_engine.py::test_T12_non_p1_incident PASSED
# 
# ========================== 12 passed in 2.34s ==========================
```

## Minimum Pass Threshold

- **P0 tests (T01–T04):** 4/4 MUST pass — no release without these
- **P1 tests (T05–T07, T11):** 4/4 MUST pass — core robustness
- **P2 tests (T08–T10):** 3/3 MUST pass — edge case handling
- **P3 tests (T12):** 1/1 SHOULD pass — completeness
- **Total minimum:** 12/12 ALL PASS before Git push

## Failure Handling

| Failure Pattern | Debug Step 1 | Debug Step 2 | Fix Approach |
|----------------|-------------|-------------|--------------|
| Zoom mock not called | Check `unittest.mock.patch` decorator path matches import | Verify `requests.post` is the patched call (not `requests.request`) | Fix import path in test |
| On-call resolver returns wrong user | Check mock data for `cmn_schedule_span` time overlap | Verify `GlideDateTime` comparison in resolver | Adjust mock dates to current time window |
| Room deduplication false positive | Check log query filter: `status=ACTIVE AND group=X AND sys_created_on > now-5min` | Verify sys_created_on in mock data is within window | Increase window or adjust mock timestamps |
| OAuth token not refreshed | Verify `expires_at` timestamp comparison | Check `token_expiry_buffer_ms` config value | Reduce buffer or adjust mock expiry |
