# sn_zoom_incident_rooms — Risk Report

**Author:** Vladimir Kapustin  
**Date:** 2026-05-28  
**Review Cadence:** Quarterly

## Risk Register

| ID | Severity | Category | Risk Description | Probability | Impact | Mitigation Strategy | Owner | Status |
|----|----------|----------|------------------|-------------|--------|---------------------|-------|--------|
| R01 | P0 | External API | Zoom API v2 is unavailable (outage, rate limit, deprecation) | Medium | Critical — no rooms created for P1 incidents | Retry queue with exponential backoff (3 attempts, 2s/4s/8s); fallback to posting Zoom link placeholder in work notes; alert to admin via email | DevOps | MITIGATED |
| R02 | P0 | Auth | OAuth token expires mid-operation | Low | Critical — all API calls fail | Token pre-refresh 5 minutes before expiry; health check every 15 minutes; admin alert on 401 response | App Admin | MITIGATED |
| R03 | P0 | Data | Room creation fails silently — no audit log entry written | Low | Critical — P1 incident has no war room | Transaction wrapper: room creation + log insert in same Business Rule execution; if log insert fails, roll back room creation | Dev Lead | MONITORED |
| R04 | P1 | Platform | On-call schedule has zero members for current shift | Medium | High — room created but empty | Fallback to assignment group manager; second fallback to incident assignee; log warning to `x_sn_zoom_incident_rooms_log` with status `NO_ONCALL` | App Admin | MONITORED |
| R05 | P1 | Concurrency | Two incidents trigger simultaneous room creation for same group | Low | High — duplicate Zoom meetings, confused responders | Deduplication check before creation: query `x_sn_zoom_incident_rooms_log` for active rooms on same assignment group within last 5 minutes; merge into existing room | Dev Lead | MITIGATED |
| R06 | P1 | Security | Zoom join URL leaked to unauthorized users (public incident comments) | Medium | High — unauthenticated access to war room | Meeting password auto-generated; waiting room enabled by default; post join URL only in work notes (NOT public comments); ACL on `zoom_meeting_url` field | Security Lead | MITIGATED |
| R07 | P2 | Performance | P1 incident flood (10+ simultaneous) exhausts Zoom API rate limit | Low | Medium — delayed room creation | Batch processing with 200ms inter-request delay; queue overflow → escalate oldest pending; admin dashboard shows queue depth | Dev Lead | MITIGATED |
| R08 | P2 | Data Integrity | Manually deleted Zoom meeting still referenced in ServiceNow log | Medium | Medium — stale references | Daily reconciliation job: compare `x_sn_zoom_incident_rooms_log.active` against Zoom API status; mark CLOSED if meeting deleted externally; run at 02:00 UTC | Dev Lead | PLANNED |
| R09 | P2 | Platform | Upgrade from Zurich to Australia changes REST endpoint registration format | Low | Medium — endpoints stop working | Compatibility test suite validates all 3 REST endpoints post-upgrade; CI/CD pipeline runs on AUSTRALIA PDI before production push | QA Lead | MITIGATED |
| R10 | P3 | UX | Wrong on-call user invited due to stale group membership | Low | Low — wrong person in room (can be corrected manually) | Nightly sync job: compare `sys_user_grmember` against Zoom account directory; flag mismatches; admin notification | App Admin | PLANNED |
| R11 | P3 | Compliance | Meeting recordings retained beyond GDPR retention policy | Low | Medium — compliance violation | Auto-delete Zoom cloud recordings after 30 days (configurable); log deletion to audit trail; GDPR data subject request: search logs by user email | Compliance | MITIGATED |
| R12 | P3 | Edge Case | Incident resolved and re-opened — new room vs rejoin existing | Medium | Low — confusion on which room to use | Re-open trigger: check if room exists and is < archive_after_hours old; if yes, post rejoin link; if no, create new room; log as REOPENED status | Dev Lead | PLANNED |

## Risk Summary

| Severity | Count | Mitigated | Monitored | Planned |
|----------|-------|-----------|-----------|---------|
| P0 (Critical) | 3 | 3 | 0 | 0 |
| P1 (High) | 3 | 2 | 1 | 0 |
| P2 (Medium) | 3 | 1 | 0 | 2 |
| P3 (Low) | 3 | 1 | 0 | 2 |
| **Total** | **12** | **7** | **1** | **4** |

## Top Action Items

1. **R08 reconciliation job** — Implement daily Zoom↔ServiceNow sync before Q3 2026 audit
2. **R10 on-call sync** — Nightly group membership validation against Zoom directory
3. **R12 re-open logic** — Incident re-open handling with room reuse heuristic
4. **Quarterly review** — All P0/P1 risks re-assessed every 90 days with updated Zoom API changelog review
