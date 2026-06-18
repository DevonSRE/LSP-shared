# Frontend Integration Status — Platform Admin API

**Module:** Platform Admin (Authentication, Courts, Court Configuration, Users, Analytics, Retraining, Broadcasts, Broadcast Templates)
**Raised by:** Frontend team
**Date:** 2026-06-17
**Status:** Living document — updated as integration progresses
**Source documents:** `api/openapi.yaml`, `guide/PRD.md`

This tracks, per documented endpoint, whether the JudicAI frontend (`platform` admin section) has implemented and wired up the integration, so the backend team has visibility into rollout progress and can flag spec drift early.

**Legend:** ✅ Integrated & matches spec · ⚠️ Integrated but path/contract differs from spec · ❌ Not integrated

---

## Authentication

| Endpoint (spec) | Status | Note |
| --- | --- | --- |
| `POST /auth/login` | ✅ | `PlatformAuthService.login` → `platformAuthConfig` (baseURL includes `/auth`) → resolves to `/auth/login`. Matches. |
| `POST /auth/request_reset_link` | ⚠️ | Frontend calls `/auth/password-request` (`PlatformAuthService.passwordRequest`). Same `/auth` prefix, different path segment — needs alignment with backend on whether `request_reset_link` replaced `password-request` or vice versa. |
| `POST /users/{id}/change_password` | ⚠️ | Frontend calls `/auth/password-reset` (`PlatformAuthService.changePassword`) with a single-field payload (no `old_password`, no user `id` in path) — this looks like a token-based "reset via emailed link" flow, not the spec's authenticated old/new password change. Needs a conversation: are these two different endpoints (reset-via-token vs. authenticated change), and if so, is the reset-via-token endpoint missing from `openapi.yaml`? |

## Courts

| Endpoint (spec) | Status | Note |
| --- | --- | --- |
| `GET /courts` | ✅ | `PlatformService.listCourts` |
| `POST /courts` | ✅ | `PlatformService.createCourt` |
| `GET /courts/constants` | ⚠️ | Frontend calls `/courts/constants?constant=court_types\|models`, spec defines no query param (implies both lists returned together). Functionally fine if backend ignores/accepts the param, but worth confirming. |
| `POST /courts/onboard` | ✅ | `PlatformService.onboardCourt` |
| `GET /courts/{id}` | ✅ | `PlatformService.getCourt` |
| `PATCH /courts/{id}` | ✅ | `PlatformService.updateCourt` |
| `DELETE /courts/{id}` | ✅ | `PlatformService.deleteCourt` |

## Court Configuration

| Endpoint (spec) | Status | Note |
| --- | --- | --- |
| `GET /courts/configurations` | ⚠️ | Frontend calls `/court_config` (`PlatformService.listCourtConfigs`) — different path entirely. Needs alignment. |
| `POST /courts/configurations` | ❌ | No create-configuration flow wired up in the frontend yet. |
| `GET /courts/configurations/{id}` | ❌ | Not wired up. |

## Users

| Endpoint (spec) | Status | Note |
| --- | --- | --- |
| `GET /platform/users` | ✅ | `PlatformService.listUsers` |
| `POST /platform/users` | ✅ | `PlatformService.addUser` |
| `DELETE /users/{id}` | ⚠️ | Frontend calls this (`PlatformService.deleteUser`) but it is **not documented** in `openapi.yaml` (no per-user endpoints exist there besides the list/create pair). Confirm whether this endpoint still exists server-side or needs adding to the spec. |
| `GET /users/user_types` | ⚠️ | Frontend calls this (`PlatformService.listUserTypes`) but it's undocumented in the spec — `UserType` is already an inline enum in `components/schemas`, so this call may be legacy/redundant. |

## Analytics

| Endpoint (spec) | Status | Note |
| --- | --- | --- |
| `GET /platform/analytics` | ✅ | `PlatformService.listAnalytics` |
| `GET /platform/analytics/group` | ✅ | `PlatformService.listCourtsAnalytics` |
| `GET /platform/analytics/{courtID}/metric` | ❌ | Not wired up yet. |

## Retraining

| Endpoint (spec) | Status | Note |
| --- | --- | --- |
| All `/platform/retrain/*` | ❌ | Not applicable — consumed by `PLATFORM_MODEL_RETRAINER` role, no UI for this in the current frontend. |

## Broadcasts

| Endpoint (spec) | Status | Note |
| --- | --- | --- |
| All `/platform/broadcasts*` | ❌ | Not yet integrated. Net-new module — build planned (see `shared/frontend/broadcast-api-requests.md` for open backend asks blocking parts of this). |

## Broadcast Templates

| Endpoint (spec) | Status | Note |
| --- | --- | --- |
| All `/platform/broadcast-templates*` | ❌ | Not yet integrated. Planned alongside Broadcasts. |
