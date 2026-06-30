# Near Real-Time Audit Logs (Early Release)

> [!IMPORTANT]
> This feature is in **early release**. The event contract described below is
> stable for the fields listed, but additional fields may be added over time
> and a small number of known gaps remain (see [Known Limitations](#known-limitations)).
> Reach out to your Contentful contact to be onboarded.

Near Real-Time Audit Logs (NRAL) deliver a stream of audit events describing
API activity against your Contentful organization within minutes of the
underlying request, rather than via the 24-hour batch export available today.

Events are emitted in the [OCSF](https://schema.ocsf.io/) (Open Cybersecurity
Schema Framework) format, version **1.3.0**, as
[API Activity](https://schema.ocsf.io/1.3.0/classes/api_activity) events
(`class_uid = 6003`). This makes the feed directly ingestible by any SIEM that
supports OCSF.

## Contents

- [What you get](#what-you-get)
- [Event shape](#event-shape)
- [Sample event](#sample-event)
- [Field reference](#field-reference)
- [General conventions](#general-conventions)
- [Experimental: enrichment events](#experimental-enrichment-events)
- [Differences vs. the Daily Audit Log batch export](#differences-vs-the-daily-audit-log-batch-export)
- [Delivery](#delivery)
- [Known limitations](#known-limitations)

## What you get

- **One OCSF event per API request** against the Contentful Management API
  (CMA) for your organization.
- **Near real-time latency.** Events typically reach your delivery
  destination within minutes of the originating request, subject to your
  destination's own ingest latency.
- **Stable OCSF contract** for the fields covered in
  [Field reference](#field-reference).

## Event shape

Each delivered object is a gzip-compressed NDJSON file
(`*.jsonl.gz`) with one JSON OCSF event per line. Files are organized by
organization and event date (UTC); the exact key layout your destination
sees is described in your EO delivery onboarding guide.

The feed may contain more than one OCSF event class (see
[Experimental: enrichment events](#experimental-enrichment-events)).
Consumers should dispatch on `class_uid` / `class_name` to identify each
event, not on file path or `metadata.product`.

| Field | Value |
|---|---|
| Format | NDJSON, gzipped |
| Encoding | UTF-8 |
| OCSF version | `1.3.0` (`metadata.version`) |
| Initial event class | `API Activity` (`class_uid = 6003`) |
| Timestamps | Epoch milliseconds (`Timestamp_t`), UTC |

> [!NOTE]
> **Why OCSF 1.3.0 and not a newer version?** We are deliberately staying on
> OCSF 1.3.0 (matching the version used by the Daily Audit Log batch
> export) for the fields covered by this contract. There are no breaking
> differences in the `API Activity` class between 1.3.0 and the latest
> OCSF (1.8.0) that would impact this feed, and pinning to a single
> version means a customer can combine NRAL events with batch-export
> events in the same SIEM index without schema reconciliation. This is
> useful in disaster-recovery scenarios: if the NRAL stream is temporarily
> unavailable, the corresponding day from the batch export can be ingested
> alongside without breaking dashboards or detection rules.

## Sample event

A `PUT` to an entry endpoint by an authenticated user:

```json
{
  "activity_id": 3,
  "activity_name": "Update",
  "class_uid": 6003,
  "class_name": "API Activity",
  "category_uid": 6,
  "type_uid": 600303,
  "severity_id": 1,
  "status_id": 1,
  "time": 1779969600123,
  "duration": 142,
  "actor": {
    "user": {
      "uid": "5xUser1234abcd",
      "type": "User",
      "type_id": 1
    }
  },
  "metadata": {
    "version": "1.3.0",
    "uid": "9f1e7a4c-5b3d-4d2e-a8f1-2c6e9a1b4d3f",
    "correlation_uid": "req-7c4e2a1f-8b3d-4a9e-bc12-3456789abcde",
    "tenant_uid": "0XYZ123orgabc",
    "product": {
      "name": "Content Management API",
      "vendor_name": "Contentful"
    }
  },
  "api": {
    "operation": "/spaces/:space_id/environments/:env/entries/:entry_id"
  },
  "resources": [
    { "uid": "abc123space", "type": "space" },
    { "uid": "master", "type": "environment" },
    { "uid": "entry42xyz", "type": "entity" }
  ],
  "http_request": {
    "uid": "req-7c4e2a1f-8b3d-4a9e-bc12-3456789abcde",
    "http_method": "PUT",
    "referrer": "https://app.contentful.com/spaces/abc123space/entries",
    "user_agent": "contentful.js/10.4.2 (Node.js/v20.10.0)",
    "url": {
      "hostname": "api.contentful.com",
      "path": "/spaces/abc123space/environments/master/entries/entry42xyz",
      "query_string": ""
    },
    "http_headers": [
      { "name": "x-contentful-user-agent", "value": "app contentful.js/10.4.2; platform Node.js/v20.10.0;" },
      { "name": "origin", "value": "https://app.contentful.com" }
    ]
  },
  "http_response": {
    "code": 200,
    "length": 2048,
    "latency": 142,
    "http_headers": [
      { "name": "x-cache", "value": "MISS" }
    ]
  }
}
```

## Field reference

The Contentful-specific interpretation of each OCSF field below. Fields not
listed are reserved by OCSF and not currently populated.

| OCSF field | Meaning in this feed |
|---|---|
| `class_uid` | Constant `6003` (API Activity). |
| `class_name` | Constant `"API Activity"`. |
| `category_uid` | Constant `6` (Application Activity). |
| `activity_id` | HTTP-method derived: `GET`/`HEAD`→2 (Read), `POST`→1 (Create), `PUT`/`PATCH`→3 (Update), `DELETE`→4 (Delete), other verbs→99 (Other), missing→0 (Unknown). |
| `activity_name` | OCSF-spec name corresponding to `activity_id`. |
| `type_uid` | `class_uid * 100 + activity_id`. |
| `severity_id` | Constant `1` (Informational). See [General conventions](#general-conventions). |
| `status_id` | `1` (Success) when HTTP status is in `[200, 400)`; `2` (Failure) for any other parsed status; `0` (Unknown) if missing. |
| `time` | Epoch milliseconds of the request. `0` if upstream timestamp was missing. |
| `duration` | Server-side request duration, milliseconds. Omitted when unknown. |
| `actor.user` | Present for authenticated human-user requests. `uid` is the Contentful user ID; `type_id=1`, `type="User"`. **Identifier only** in the early release (no email or full name; see [Known limitations](#known-limitations)). |
| `actor.app_uid` | Present when the actor is a Contentful App installation (rather than a user). Identifier only. |
| `actor.invoked_by` | Present when the request was made on behalf of an app via the `X-Contentful-Delegated-Actor-Id` header. Always an `app:...` identifier. |
| `metadata.version` | Constant `"1.3.0"` (OCSF schema version). |
| `metadata.uid` | Random UUID generated per event (the OCSF event-instance identifier). Unique to this delivery. |
| `metadata.correlation_uid` | The upstream `request_id`. Use to join this event with other logs (application logs, traces) for the same request. |
| `metadata.tenant_uid` | Your Contentful organization ID. |
| `metadata.product` | Constant `{name: "Content Management API", vendor_name: "Contentful"}`. |
| `api.operation` | Route template of the API endpoint, e.g. `/spaces/:space_id/entries/:entry_id`. Treat as an opaque string for grouping. |
| `resources[]` | Up to three entries identifying objects acted on: `{uid, type}` where `type` is one of `space`, `environment`, `entity`. Omitted entirely for non-space-scoped calls (organization- or user-level endpoints). |
| `http_request.uid` | Same value as `metadata.correlation_uid`. |
| `http_request.http_method` | Raw HTTP method. |
| `http_request.url` | `{hostname, path, query_string}`. `path` is the resolved URL path (with concrete IDs), not the route template; the template lives in `api.operation`. |
| `http_request.referrer` | HTTP `Referer` header (note OCSF spelling). |
| `http_request.user_agent` | HTTP `User-Agent` header. |
| `http_request.http_headers[]` | A small set of captured request headers (currently `x-contentful-user-agent` and `origin`). List is omitted entirely when empty. |
| `http_response.code` | HTTP status code. Omitted if unknown. |
| `http_response.length` | Response body size in bytes. Omitted if unknown. |
| `http_response.latency` | Same value as top-level `duration`, milliseconds. |
| `http_response.http_headers[]` | Currently exposes `x-cache` (edge cache status, e.g. `HIT`, `MISS`). Omitted when empty. |

## General conventions

These rules are intentional and will not change without notice during the
early-release period:

- **Missing values are omitted, not zero-filled.** A serialized `0` always
  means "the upstream value really was zero," never "we did not capture it."
  Optional fields and nested objects (`duration`, `http_response.length`,
  `http_response.latency`, `actor`, `resources`, …) are dropped from the JSON
  when absent.
- **`severity_id` is always `1` (Informational).** Audit logs are an event
  stream, not a security signal. Build your SIEM detections on
  `status_id`, `activity_id`, `api.operation`, and resource patterns.
  Severity intentionally stays neutral so transport failures (e.g. HTTP 5xx)
  do not get conflated with security incidents.
- **Anonymous / unauthenticated requests omit the `actor` object entirely.**
  Absence means "no identity captured"; there is no canonical
  anonymous-actor encoding.
- **`metadata.uid` and `metadata.correlation_uid` are distinct.** `uid` is a
  random per-event UUID (the OCSF event-instance identifier). To join an
  audit event with other logs for the same request, use `correlation_uid`
  (or equivalently `http_request.uid`).
- **`time` is epoch milliseconds.** ISO 8601 strings are never emitted;
  consumers should format on read.
- **Resource type vocabulary is coarse.** `resources[].type` is one of
  `space`, `environment`, `entity`. The Contentful Management API exposes
  many specific nouns (entries, assets, content types, tags, releases,
  roles, webhooks, API keys, …); these are all surfaced as `entity` to
  keep the OCSF contract stable as the CMA evolves. If you need the
  specific noun, parse it from `http_request.url.path`.
- **`activity_id` is derived from the HTTP method, not from the lifecycle
  action.** Domain operations such as publish, unpublish, archive,
  unarchive, schedule, and workflow transitions are most commonly expressed
  as `PUT` and therefore appear as `activity_id = 3` (Update). Some, notably
  `unpublish`, can also be expressed as `DELETE`, so the same logical
  action can appear with different `activity_id` values depending on the
  client. **For SIEM rules that need to detect specific lifecycle actions,
  match on `api.operation`, not on `activity_id` alone.**
- **`resources[]` may be omitted.** Organization-level (`/organizations/...`)
  and user-level (`/users/me`) endpoints have no space or environment
  context. `metadata.tenant_uid` still carries your organization ID for
  these events.
- **Unknown enum values use OCSF-defined sentinels.** `activity_id=0`
  ("Unknown") and `99` ("Other") follow the OCSF spec; we never invent
  custom enum integers.

## Experimental: enrichment events

> [!WARNING]
> **Experimental, subject to change during early release.** Most of the
> implementation is in place, but the event shape, field names, and the set
> of correlations described below may change before this becomes a stable
> part of the contract. We may add, rename, or restructure fields without
> notice while the feature is in this phase. Build optional handling on top
> of it; do not yet make production SIEM rules depend on its exact shape.

In addition to the `API Activity` events described above, you may begin to
see a second class of OCSF events on the same feed:

- **Event class:** `Web Resources Activity` (`class_uid = 6001`,
  `class_name = "Web Resources Activity"`).
- **Purpose:** carry **extra context** that the originating CMA request
  could not surface inline. Two such contexts are wired today:
  - **AI Actions:** additional metadata about AI-action invocations
    (e.g. provider, action data) that is not visible from the HTTP layer
    alone.
  - **Bulk Actions:** the full set of entity IDs touched by a bulk
    operation. This complements the [bulk-operations limitation](#known-limitations)
    on `API Activity` events, which today capture at most one entity per
    bulk request.
- **Correlation to the originating request:** the enrichment event carries
  the same `metadata.correlation_uid` as the `API Activity` event it
  augments. Consumers can join the two on `correlation_uid` to attach the
  enrichment data to the originating call.
- **Distinguishing it on read:** dispatch on `class_uid` or `class_name`.
  `6003` / `"API Activity"` is the main audit event; `6001` /
  `"Web Resources Activity"` is the experimental enrichment event. Both
  classes share `metadata.tenant_uid` (your organization ID) and the same
  partition layout on disk.

> [!NOTE]
> **You can filter or drop enrichment events at parse time** by matching on
> `class_name == "Web Resources Activity"` (or equivalently
> `class_uid == 6001`). If you only want the main audit-event contract for
> now, add a filter in your parser to skip these and revisit when the
> contract stabilises.

Because this path is still evolving, expect:

- Field names under `web_resources[].data` (e.g. `type`, `type_version`,
  `provider`, `action`, `entities`, `created_time`) to potentially be
  renamed or restructured.
- Emitting of events to be interrupted and resumed at arbitrary points in time
- Enrichments might show up within the API Activity event at some phase (as detailed below)

> [!NOTE]
> **Possible future change:** we are evaluating folding the enrichment
> payload directly into the originating `API Activity` event (likely under
> OCSF's `enrichments` field) instead of emitting a separate
> `Web Resources Activity` event correlated by `correlation_uid`. If we
> make that change, the two-event model documented here would be replaced
> by a single, self-contained `API Activity` event. Based on the outcomes
> of the current prototype, this will be evaluated for adoption in the
> final general availability version.

## Differences vs. the Daily Audit Log batch export

If you have already integrated with Contentful's existing **Daily Audit Log
batch export** (audit logs delivered as a once-per-day file), the NRAL feed
intentionally differs in several important ways. The list below covers the
breaking shape changes. Re-validate your SIEM ingest rules against it.

> [!NOTE]
> **Spec conformance:** the legacy Daily batch export deviated from the
> OCSF 1.3.0 spec in a few places (notably encoding `class_uid`,
> `category_uid`, and `type_uid` as strings, emitting `time` as an
> ISO-8601 string instead of epoch milliseconds, and adding non-standard
> `actor.id` / `actor.type` fields that are not defined on the OCSF Actor
> object). NRAL aims to be fully OCSF-conformant: numeric UIDs, epoch-ms
> timestamps, and only OCSF-defined Actor attributes (`user`, `app_uid`,
> `invoked_by`). If your existing parsers tolerate or rely on the
> non-standard shapes, expect to tighten them up when adopting NRAL.

### File format

| | Daily batch export | NRAL (this feed) |
|---|---|---|
| Delivery cadence | Once per day | Continuous, near real-time |
| File contents | A **single JSON object per file** (the entire day on one line) | **NDJSON**, one JSON event per line |
| Compression | Uncompressed | gzip |

Switching from the batch export to NRAL therefore requires reading events
line-by-line and decompressing gzip, not parsing a single top-level JSON
document.

### OCSF event class

| | Daily batch export | NRAL |
|---|---|---|
| OCSF class | `Web Resources Activity` (`class_uid = "6001"`) | `API Activity` (`class_uid = 6003`) |
| `category_uid` | `"6"` (string) | `6` (number) |
| `class_uid` / `type_uid` | strings (e.g. `"6001"`, `"600101"`) | numbers (e.g. `6003`, `600303`) |
| `metadata.product` | not set | `{name: "Content Management API", vendor_name: "Contentful"}` |
| `metadata.tenant_uid` | not set | your Contentful organization ID |
| `metadata.correlation_uid` | not set | the originating `request_id` (also `http_request.uid`) |

The move from `Web Resources Activity` to `API Activity` reflects what the
event actually represents (a Contentful Management API request) and lets us
reserve `Web Resources Activity` for [enrichment events](#experimental-enrichment-events)
that describe touched resources separately (like in case of bulk actions).

### `time`

| | Daily batch export | NRAL |
|---|---|---|
| Type | ISO-8601 string (e.g. `"2025-02-26 16:52:57.054"`) | **Epoch milliseconds** (e.g. `1779969600123`) |

Per OCSF `Timestamp_t`. Consumers must format on read.

### `severity_id`

| | Daily batch export | NRAL |
|---|---|---|
| Value | `0` (Unknown) | `1` (Informational), constant |

Audit logs are an event stream, not a security signal. See
[General conventions](#general-conventions); escalate on `status_id`,
`activity_id`, `api.operation`, or resource patterns instead of severity.

### `status_id`

| | Daily batch export | NRAL |
|---|---|---|
| Field | not emitted | derived from HTTP status: `1` (Success) for `[200, 400)`, `2` (Failure) otherwise, `0` (Unknown) if missing |

### `actor`

| | Daily batch export | NRAL |
|---|---|---|
| Deprecated `actor.id` / `actor.type` | present (kept for backwards compatibility) | **not emitted** (these fields are deprecated in OCSF; use `actor.user.uid` / `actor.user.type`) |
| `actor.user` for human users | `{type: "User", type_id: 2, uid, email_addr, full_name}` (identifier **and** full profile) | `{type: "User", type_id: 1, uid}` (identifier only; full profile not yet expanded) |
| Apps | `actor.user = {type: "App", type_id: 3, uid}` | `actor.app_uid = "<id>"` (no `actor.user`) |
| Delegated actor (app on behalf of user) | not represented | `actor.invoked_by = "app:<id>"` |
| Anonymous / unauthenticated | actor object with `type: "Unknown"` | `actor` object **omitted entirely** |

NRAL today identifies actors but does **not** expand them. The
`API Activity` event carries the actor identifier (`actor.user.uid` for
users, `actor.app_uid` for apps) and, when applicable, the delegated
actor (`actor.invoked_by`); it does not yet inline the user's email or
name as the batch export does. Expanded actor details are planned for
a future release before general availability (see
[Known limitations](#known-limitations)). In the meantime, if your
downstream needs the user's email or display name, resolve it from
`actor.user.uid` via the Contentful Users API.

### `enrichments` and `web_resources`

| | Daily batch export | NRAL |
|---|---|---|
| `enrichments[]` on the main event | yes (synthetic entries derived from `http_request.url.path`, plus AI/Bulk action payloads) | **not populated on the `API Activity` event today** |
| `web_resources[]` on the main event | yes, `[{type: "<NounName>", uid: "<id>"}]` (e.g. `BulkAction`, `Entry`) | **not populated**; see `resources[]` instead |
| Acted-on resources on the main event | `web_resources[]` with the specific CMA noun (e.g. `BulkAction`, `Entry`, `Tag`) | `resources[]` with coarse types only: `space`, `environment`, `entity` |
| AI Actions / Bulk Actions extra data | inline in the same event's `enrichments[]` | emitted as a **separate experimental `Web Resources Activity` event** (`class_uid = 6001`), joinable on `metadata.correlation_uid`. See [Experimental: enrichment events](#experimental-enrichment-events). |

The coarsening to `space`/`environment`/`entity` is intentional. The CMA
exposes dozens of resource nouns that change over time, and tying the OCSF
contract to that surface forced schema churn on every CMA addition. If you
need the specific noun, parse it from `http_request.url.path`.

### `activity_id` semantics

| | Daily batch export | NRAL |
|---|---|---|
| Derivation | HTTP-method derived (same enum) | HTTP-method derived (same enum) |
| `Other` enum value | `99` | `99` |
| `Unknown` value | `0` (and `severity_id` was also `0`) | `0` |

The mapping itself is unchanged; what's new is the **explicit guidance**:
many CMA domain operations (publish, unpublish, archive, schedule, workflow
transitions) surface as `PUT` and therefore appear as `activity_id = 3`
(Update). For SIEM rules that need to detect specific lifecycle actions,
match on `api.operation`, **not** on `activity_id`.

### `http_request` / `http_response`

| | Daily batch export | NRAL |
|---|---|---|
| `http_request.url` | `{path}` only | `{hostname, path, query_string}` |
| `http_request.args` | populated with raw query string | **removed**; query is in `url.query_string` |
| `http_request.referrer`, `http_request.http_method` | present | present (unchanged) |
| `http_request.user_agent` | not present | populated |
| `http_request.http_headers[]` | not present | `x-contentful-user-agent`, `origin` when present |
| `http_request.uid` | not present | equal to `metadata.correlation_uid` |
| `http_response.code` | present | present |
| `http_response.length` | not present | populated (response body bytes) |
| `http_response.latency` | not present | populated, ms (same value as top-level `duration`) |
| `http_response.http_headers[]` | not present | `x-cache` (edge cache status) when present |

### `metadata`

| | Daily batch export | NRAL |
|---|---|---|
| `metadata.uid` | the upstream `request_id` | a **fresh random UUID per event** (OCSF event-instance identifier) |
| `metadata.correlation_uid` | not set | the upstream `request_id`; use this for cross-log correlation |
| `metadata.tenant_uid` | not set | your Contentful organization ID |
| `metadata.product` | not set | `{name: "Content Management API", vendor_name: "Contentful"}` |
| `metadata.version` | `"1.3.0"` | `"1.3.0"` |

If you previously joined audit events to other logs on `metadata.uid` (the
request ID), switch the join key to **`metadata.correlation_uid`** in NRAL.

### Missing-value handling

| | Daily batch export | NRAL |
|---|---|---|
| Empty fields | often emitted as `""`, `0`, or empty arrays | **omitted entirely** |

A serialized `0` or `""` in NRAL always means "the upstream value really
was zero or empty," never "we didn't capture it."

## Delivery

The NRAL logs we create are pushed through the [Enterprise Observability](https://www.contentful.com/developers/docs/concepts/enterprise-observability/) 
log delivery feature. The configuration, destination types, credentials model, and 
authentication flow are identical to other EO log options;

What you do need is an **NRAL-specific delivery configuration** alongside
your existing Observability ones. Audit logs are treated as their own log type, so
each destination you want them sent to has to be opted in explicitly
(this is what lets you route audit logs to a different bucket / index /
account from your access logs if you want to, and lets you turn audit
delivery on or off independently). Ask your Contentful contact to set up
the NRAL configuration for the destinations you want.


## Known limitations

- **Bulk operations capture at most one entity on the `API Activity` event.**
  CMA endpoints such as `bulk_actions` and batch publish operate on many
  entries in a single HTTP request, but the `API Activity` event currently
  surfaces only a single `entity_id` (or none) in `resources[]`. The full
  set of touched entities is being delivered via a separate
  [enrichment event](#experimental-enrichment-events) (experimental);
  correlate on `metadata.correlation_uid` to attach the bulk-action
  details to the originating request.
- **CMA-only at launch.** The early-release feed covers the Contentful
  Management API only.
- **Actor identifier only, no profile details yet.** In the current early
  release, `actor.user` carries the user ID only; email address and full
  name are **not** included (the legacy Daily Audit Log batch export does
  expose these; see [Differences vs. the Daily Audit Log batch export](#differences-vs-the-daily-audit-log-batch-export)).
  We aim to provide full actor information in a future release before
  general availability. If you need email or display name today, look the
  user up via the Contentful Users API by `actor.user.uid`.
- **Some `api.operation` values may be empty.** When upstream routing does
  not set a route template, the entire `api` object is dropped from the
  event. Affected endpoints are being closed out during the early-release
  period.

