# sn_zoom_incident_rooms — Dependency Report

**Author:** Vladimir Kapustin  
**Date:** 2026-05-28  
**Release Target:** AUSTRALIA (ServiceNow)

## 1. ServiceNow Platform Dependencies

### Required Plugins
| Plugin ID | Name | Version | Purpose |
|-----------|------|---------|---------|
| `com.snc.incident_management` | Incident Management | ≥ 10.0 | Core incident table and P1 routing |
| `com.glide.rest` | REST API Framework | Built-in | REST endpoint registration |
| `com.snc.oauth` | OAuth 2.0 Framework | Built-in | Zoom Server-to-Server auth |
| `com.snc.sla` | SLA Engine | ≥ 11.0 | Optional: room creation SLA tracking |

### System Tables Referenced
| Table | Access | Purpose |
|-------|--------|---------|
| `incident` | Read/Write | Trigger on insert/update; write `zoom_meeting_url` field |
| `sys_user` | Read | Resolve assignment group members |
| `sys_user_grmember` | Read | Group membership lookup |
| `sys_choice` | Read | Status field values |
| `cmn_schedule` | Read | On-call schedule lookup |
| `cmn_schedule_span` | Read | Active on-call windows |

### System Properties
| Property | Default | Description |
|----------|---------|-------------|
| `x_sn_zoom_incident_rooms.auto_create` | `true` | Enable/disable auto-room creation |
| `x_sn_zoom_incident_rooms.default_duration` | `60` | Default meeting duration in minutes |
| `x_sn_zoom_incident_rooms.max_active_rooms` | `50` | Concurrent room cap |
| `x_sn_zoom_incident_rooms.archive_after_hours` | `24` | Auto-close rooms after N hours of incident resolution |

## 2. External Dependencies

### Zoom API v2
| Endpoint | Method | Purpose | Auth |
|----------|--------|---------|------|
| `/v2/users/me/meetings` | POST | Create meeting | OAuth 2.0 Bearer |
| `/v2/meetings/{meetingId}` | GET | Get meeting status | OAuth 2.0 Bearer |
| `/v2/meetings/{meetingId}` | DELETE | End meeting | OAuth 2.0 Bearer |
| `/v2/meetings/{meetingId}/registrants` | GET | Participant list (optional) | OAuth 2.0 Bearer |

### Network Requirements
| Source | Destination | Port | Protocol |
|--------|-------------|------|----------|
| ServiceNow instance | `api.zoom.us` | 443 | HTTPS |
| ServiceNow instance | `zoom.us` (OAuth token) | 443 | HTTPS |

## 3. Development & Test Dependencies

### Python 3.10+ Runtime
| Package | Version | Purpose |
|---------|---------|---------|
| `requests` | ≥ 2.28 | HTTP client for Zoom API calls |
| `pytest` | ≥ 7.0 | Test framework |
| `pytest-mock` | ≥ 3.10 | Mocking support |
| `responses` | ≥ 0.23 | HTTP mock library (optional) |

### Node.js Test Dependencies (for GlideRecord mocks)
| Package | Purpose |
|---------|---------|
| Node.js ≥ 18 | JavaScript test runtime |
| `assert` | Built-in assertions |

## 4. Role Requirements

| Role | Access | Purpose |
|------|--------|---------|
| `x_sn_zoom_incident_rooms.admin` | Full CRUD on config/log/oncall tables | App administration |
| `x_sn_zoom_incident_rooms.incident_manager` | Execute create_room/close APIs | Incident response automation |
| `x_sn_zoom_incident_rooms.viewer` | Read-only on log/status | Audit and reporting |

## 5. Upgrade/Migration Notes

- **Zurich → Australia:** `sys_rest_message` API unchanged; REST endpoints continue to work
- **OAuth framework:** Australia uses `sn_generative_ai_cfg_provider` for AI; this app uses standard `oauth_credential` — no conflict
- **BYOK providers:** Not applicable (no AI/LLM calls)
- **Now Assist:** Not applicable (no AI dependencies)

## 6. Risk Matrix

| Dependency | Risk Level | Mitigation |
|------------|-----------|------------|
| Zoom API outage | P1 — Critical | Queue retry with exponential backoff (max 3 attempts) |
| OAuth token expiry | P2 — Medium | Auto-refresh 5 minutes before expiry |
| On-call schedule gaps | P3 — Low | Fallback to assignment group manager |
| REST endpoint rate limit | P2 — Medium | Batch processing with 100ms inter-request delay |
