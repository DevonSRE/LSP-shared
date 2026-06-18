# Frontend → Backend API Requests

**Module:** Email Broadcast System
**Raised by:** Frontend team
**Date:** 2026-06-17
**Status:** Open
**Source documents:** `api/openapi.yaml` (Broadcasts / Broadcast Templates tags), `guide/PRD.md`

While integrating the documented `Broadcasts` and `Broadcast Templates` endpoints against the Email Broadcast System PRD, the frontend found gaps where the PRD's acceptance criteria cannot be met by the current `openapi.yaml` contract. Each item below proposes an OpenAPI fragment intended to be merged into `openapi.yaml` once implemented, plus what the frontend is shipping in the meantime so backend work can land independently without blocking v1.

---

## REQ-01 — Audience resolution / recipient count preview

**PRD reference:** §05 US-04 (AC05, AC07), §6.3 Audience Selection, §12.1 ("Get Audience Count")

**Problem:** `BroadcastRequest.recipients` only accepts an explicit, pre-resolved list of `{recipient_id, recipient_type}` pairs. The PRD requires filtering by User Role, Court (specific court / court type / state / judicial division), and Subscription Plan — individually or combined with AND logic — with a live estimated recipient count as filters are applied. None of this can be resolved from currently documented endpoints. There is also no list/search endpoint for `LITIGANT` recipients, so that recipient type in `BroadcastRecipientType` is currently unreachable from the UI.

**Requested endpoint:**

```yaml
/platform/broadcasts/audience/preview:
  post:
    tags: [Broadcasts]
    summary: Resolve audience filters to an estimated recipient count
    description: >
      Given one or more audience filters (combined with AND logic), returns
      the resolved recipient count and, optionally, the resolved recipient
      list so it can be submitted back via BroadcastRequest.recipients.
    requestBody:
      required: true
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/AudienceFilterRequest"
    responses:
      "200":
        description: Resolved audience
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/AudiencePreviewResponse"
```

**Requested schemas:**

```yaml
AudienceFilterRequest:
  type: object
  properties:
    everyone:
      type: boolean
    user_types:
      type: array
      items:
        $ref: "#/components/schemas/UserType"
    court_ids:
      type: array
      items: { type: integer, format: int64 }
    court_types:
      type: array
      items:
        $ref: "#/components/schemas/CourtType"
    states:
      type: array
      items: { type: string }
    subscription_plans:
      type: array
      items: { type: string }
      description: Depends on a subscription/plan data model existing — see REQ-03.
    include_litigants:
      type: boolean
      description: Depends on a litigant directory/search endpoint, which is not currently documented.

AudiencePreviewResponse:
  type: object
  properties:
    estimated_count:
      type: integer
      format: int64
    recipients:
      type: array
      items:
        $ref: "#/components/schemas/RecipientRequest"
```

**Frontend plan until this lands:** broadcast creation ships with only "Send to Everyone" and "Custom Selection" (manual user search via the existing `/platform/users` endpoint) targeting. Role/Court/Subscription-Plan filter UI is deferred.

---

## REQ-02 — Attachment support on Broadcast

**PRD reference:** §6.2 "Attachments", US-01 AC03

**Problem:** `BroadcastRequest` / `BroadcastResponse` only expose a `content` (HTML) field. The PRD requires attaching PDFs, images, and general documents to a broadcast, separate from images embedded inline in the rich-text body.

**Requested schema addition:**

```yaml
BroadcastAttachment:
  type: object
  properties:
    filename:
      type: string
    url:
      type: string
      format: uri
    content_type:
      type: string
    size_bytes:
      type: integer
      format: int64
```

Add to both `BroadcastRequest` and `BroadcastResponse`:

```yaml
attachments:
  type: array
  items:
    $ref: "#/components/schemas/BroadcastAttachment"
```

**Frontend plan until this lands:** v1 ships with inline images only, embedded directly into the Tiptap-authored `content` HTML via the existing `/utility/media/upload` endpoint. No separate attachment picker until this field exists.

---

## REQ-03 — Subscription plan data model (blocks REQ-01)

**PRD reference:** §6.3 "By Subscription Plan" (Basic / Standard / Essential / Pro)

**Problem:** No `subscription_plan` field exists on `CourtResponse`, `UserResponse`, or anywhere else in the current spec. Filtering broadcast audiences by plan (REQ-01) cannot be implemented until courts and/or users carry this data somewhere in the API.

**Frontend plan until this lands:** "By Subscription Plan" filter UI is deferred.

---

## REQ-04 — Platform-wide broadcast performance-summary endpoint

**PRD reference:** §6.1 Broadcast Dashboard, "Performance Metrics" (Total Sent, Delivered, Opened, Clicked, Failed)

**Problem:** The PRD's Broadcast Dashboard requires a "Performance Metrics" panel aggregating **Total Sent**, **Delivered**, **Opened**, **Clicked**, and **Failed** counts across *all* broadcasts. The only documented analytics endpoint, `GET /platform/broadcasts/{id}/analytics`, returns `BroadcastAnalytics` for a single broadcast only (and only once that broadcast is `SENT`/`FAILED`) — there is no endpoint that aggregates these fields platform-wide. The dashboard's "Summary Counters" (Total/Draft/Scheduled/Sent/Failed *broadcast* counts, also in §6.1) are a separate, simpler metric that the frontend can already derive by counting statuses from the existing `GET /platform/broadcasts` list response, so no new endpoint is needed for those.

**Requested endpoint:**

```yaml
/platform/broadcasts/performance-summary:
  get:
    tags: [Broadcasts]
    summary: Aggregate broadcast delivery & engagement metrics
    description: >
      Returns Total Sent, Delivered, Opened, Clicked, and Failed counts
      aggregated across all broadcasts (or within an optional date range),
      for the Broadcast Dashboard's Performance Metrics panel.
    parameters:
      - name: from
        in: query
        required: false
        schema:
          type: string
          format: date-time
      - name: to
        in: query
        required: false
        schema:
          type: string
          format: date-time
    responses:
      "200":
        description: Aggregate performance metrics
        content:
          application/json:
            schema:
              allOf:
                - $ref: "#/components/schemas/GenericResponse"
                - type: object
                  properties:
                    data:
                      $ref: "#/components/schemas/BroadcastPerformanceSummary"
```

**Requested schema:**

```yaml
BroadcastPerformanceSummary:
  type: object
  properties:
    total_sent:
      type: integer
      format: int64
    delivered:
      type: integer
      format: int64
    opened:
      type: integer
      format: int64
    clicked:
      type: integer
      format: int64
    failed:
      type: integer
      format: int64
```

**Frontend plan until this lands:** the Broadcast Dashboard ships with the "Summary Counters" only (Total/Draft/Scheduled/Sent/Failed broadcast counts, computed client-side from the broadcast list). The "Performance Metrics" panel (Total Sent/Delivered/Opened/Clicked/Failed email-level aggregates) is not built until this endpoint exists — there's no way to derive it from currently documented endpoints without fetching `{id}/analytics` for every single broadcast.
