# SOP: ServiceNow Zoom Incident Rooms (sn_zoom_incident_rooms)

**License:** AGPL-3.0-only | **Copyright:** 2026 Vladimir Kapustin  
**Release:** AUSTRALIA | **Scope:** `x_sn_zoom_incident_rooms`

## Test Suite (12 tests)

| ID | Test |
|----|------|
| T01 | `test_T01_basic_room_creation` — P1 incident triggers Zoom meeting |
| T02 | `test_T02_oncall_invitation` — on-call engineer auto-invited |
| T03 | `test_T03_duplicate_prevention` — merge duplicate rooms within 5 min |
| T04 | `test_T04_room_close` — meeting archived on incident resolution |
| T05 | `test_T05_oauth_refresh` — expired token auto-refreshed |
| T06 | `test_T06_zoom_timeout_retry` — exponential backoff (2s/4s/8s) |
| T07 | `test_T07_zoom_permanent_failure` — max retries → FAILED |
| T08 | `test_T08_empty_oncall` — fallback to group manager |
| T09 | `test_T09_reopen_within_window` — rejoin existing room (<24h) |
| T10 | `test_T10_reopen_beyond_window` — new room after archive (>24h) |
| T11 | `test_T11_concurrent_cap` — queue at 50 active rooms |
| T12 | `test_T12_non_p1_incident` — P3 incidents skip room creation |

## Execution

```bash
pytest tests/ -v
# Expected: 12/12 PASS
```

## Artifacts

- `src/engine.py` — ZoomRoomManager core engine
- `src/cli.py` — offline testing / CI/CD CLI
- `src/sys_app.xml` — scoped app manifest
- `tests/test_engine.py` — 12 test scenarios
- `memory/checkpoints/` — architecture, dependencies, risks, execution plan
- `Validation/TEST CASES/sn_zoom_incident_rooms/` — SOP, regression, edge cases, checklist
