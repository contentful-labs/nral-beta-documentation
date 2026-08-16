# Near Real-Time Audit Logs (Early Release)

> [!IMPORTANT]
> This feature is in **early release**. The event contract described below is
> stable for the fields listed, but additional fields may be added over time.
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
- [Enrichments](#enrichments)
- [Differences vs. the Daily Audit Log batch export](#differences-vs-the-daily-audit-log-batch-export)
- [Delivery](#delivery)
- [Known limitations](#known-limitations)

## What you get

- **One OCSF event per API request** against the Contentful Management API
  (CMA) for your organization, including **read requests** (`GET`). The legacy Daily Audit Log batch
  export did not include reads; NRAL does.
- **Near real-time latency.** Events are targeted to reach your delivery
  destination within **5 minutes** of the originating request, subject to
  your destination's own ingest latency.
- **Authorization token included (redacted).** The authorization token
  from the request is captured in a redacted form as the `authorization`
  entry in `http_request.http_headers[]`, so you can correlate activity
  back to a specific token without exposing the token value itself.
- **Stable OCSF contract** for the fields covered in
  [Field reference](#field-reference).
- **At-least-once delivery.** Individual events may be duplicated in the
  feed. See [Delivery](#delivery) for deduplication guidance.

## Event shape

Each delivered object is a gzip-compressed NDJSON file
(`*.jsonl.gz`) with one JSON OCSF event per line. Files are organized by
organization and event date (UTC); the exact key layout your destination
sees is described in your EO delivery onboarding guide.

Every event on the feed is the same OCSF class, `API Activity`
(`class_uid = 6003`). Requests that produce additional context carry it on that
same event rather than on an event of their own (see [Enrichments](#enrichments)).
Dispatch on `class_uid` / `class_name` rather than on file path or
`metadata.product`, so that any class added later is additive for your parser.

| Field | Value |
|---|---|
| Format | NDJSON, gzipped |
| Encoding | UTF-8 |
| OCSF version | `1.3.0` (`metadata.version`) |
| Event class | `API Activity` (`class_uid = 6003`) |
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
      "type_id": 1,
      "email_addr": "jane.doe@example.com",
      "full_name": "Jane Doe"
    }
  },
  "metadata": {
    "version": "1.3.0",
    "uid": "9f1e7a4c-5b3d-4d2e-a8f1-2c6e9a1b4d3f",
    "correlation_uid": "7c4e2a1f-8b3d-4a9e-bc12-3456789abcde",
    "tenant_uid": "0XYZ123orgabc",
    "log_name": "cma-api-audit-log",
    "product": {
      "name": "Content Management API",
      "vendor_name": "Contentful"
    }
  },
  "api": {
    "operation": "/spaces/:spaceId/environments/:environmentId/entries/:entryId"
  },
  "resources": [
    { "uid": "abc123space", "type": "space" },
    { "uid": "master", "type": "environment" },
    { "uid": "entry42xyz", "type": "entity" }
  ],
  "http_request": {
    "uid": "7c4e2a1f-8b3d-4a9e-bc12-3456789abcde",
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
      { "name": "origin", "value": "https://app.contentful.com" },
      { "name": "authorization", "value": "CFPA[REDACTED]x7Qz" }
    ]
  },
  "http_response": {
    "code": 200,
    "length": 2048,
    "latency": 142,
    "http_headers": [
      { "name": "x-cache", "value": "PASS" }
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
| `actor.user` | Present for authenticated human-user requests. `uid` is the Contentful user ID and the field to identify the actor by; `type_id=1`, `type="User"`. `email_addr` and `full_name` carry the user's profile as it stood when the event was emitted, and are omitted rather than zero-filled if unavailable. See [General conventions](#general-conventions). |
| `actor.app_uid` | Present when the actor is a Contentful App installation (rather than a user). Identifier only. |
| `actor.invoked_by` | Present when the request was made on behalf of an app via the `X-Contentful-Delegated-Actor-Id` header. Always an `app:...` identifier. |
| `metadata.version` | Constant `"1.3.0"` (OCSF schema version). |
| `metadata.uid` | Random UUID generated per event (the OCSF event-instance identifier). Unique to this delivery. |
| `metadata.correlation_uid` | The upstream `request_id`. Use to join this event with other logs (application logs, traces) for the same request. |
| `metadata.tenant_uid` | Your Contentful organization ID. |
| `metadata.log_name` | Constant `"cma-api-audit-log"`. Identifies which Contentful log produced the event. |
| `metadata.product` | Constant `{name: "Content Management API", vendor_name: "Contentful"}`. |
| `api.operation` | Route template of the API endpoint, e.g. `/spaces/:spaceId/entries/:entryId`. Placeholder naming can vary in form across the full Contentful API, so treat the value as an opaque string for grouping rather than parsing it. |
| `resources[]` | Up to three entries identifying objects acted on: `{uid, type}` where `type` is one of `space`, `environment`, `entity`. Omitted entirely for non-space-scoped calls (organization- or user-level endpoints). |
| `enrichments[]` | Extra context that the HTTP layer alone cannot express, currently for bulk actions and AI action invocations. Present only on requests that produce it, so most events omit it. See [Enrichments](#enrichments). |
| `http_request.uid` | Same value as `metadata.correlation_uid`. |
| `http_request.http_method` | Raw HTTP method. |
| `http_request.url` | `{hostname, path, query_string}`. `path` is the resolved URL path (with concrete IDs), not the route template; the template lives in `api.operation`. |
| `http_request.referrer` | HTTP `Referer` header (note OCSF spelling). |
| `http_request.user_agent` | HTTP `User-Agent` header. |
| `http_request.http_headers[]` | A small set of captured request headers: `x-contentful-user-agent`, `origin`, and `authorization`. The `authorization` value is redacted, keeping only a short prefix and suffix around a `[REDACTED]` marker, which is enough to recognise the same token across events without exposing it. Individual entries are omitted when the header was absent, and the list is omitted entirely when none were present. |
| `http_response.code` | HTTP status code. Omitted if unknown. |
| `http_response.length` | Response body size in bytes. Omitted if unknown. |
| `http_response.latency` | Same value as top-level `duration`, milliseconds. |
| `http_response.http_headers[]` | Currently exposes `x-cache` (edge cache status). The Content Management API is not cacheable, so in practice this is always `PASS`. Omitted when empty. |

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
  anonymous-actor encoding. NRAL captures every incoming request that
  reaches the Contentful Management API surface, from all clients, not
  only the Contentful web app or official SDKs. The feed can therefore
  include requests to endpoints that are not part of the public CMA
  contract, malformed requests, or requests from misbehaving or malicious
  clients. For such requests an actor often cannot be identified, so the
  `actor` object can be absent. The event itself is still real; do not treat "missing
  actor" as "event is invalid".
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
- **Identify actors by `actor.user.uid`.** It is the stable identifier for a
  user and the right key for correlation, grouping and detection rules.
  `email_addr` and `full_name` are profile attributes attached to the event as
  they stood when it was emitted, and they are there for authenticated user
  requests barring a transient issue. Because they are a point-in-time snapshot,
  a profile that was just changed can take a short while to come through, so
  events for the same user may carry differing values before converging on the
  current profile.

## Enrichments

Some Contentful Management API requests carry information that the HTTP layer
alone cannot express. A bulk action names every entity it addressed in its
request body, and an AI action invocation records which model produced which
output. That context is delivered on the same `API Activity` event as the
request itself, in OCSF's standard
[`enrichments`](https://schema.ocsf.io/1.3.0/objects/enrichment) array. There is
no separate event to correlate and nothing extra to join.

Two kinds are populated today:

- **Bulk Actions** (`type: "BulkActionEnrichment"`): the operation performed and
  the full set of entities it addressed, which `resources[]` does not list
  individually.
- **AI Actions** (`type: "AiActionEnrichment"`): the invocation, the AI action
  and entry involved, and the model that served it.

`enrichments` appears only on requests that produce this data, so most events do
not carry it. Where a request does produce it, expect the array on the event.
Delivery is best-effort like the rest of the feed, so treat a missing
`enrichments` array as inconclusive rather than as evidence that no bulk or AI
action took place.

### Entry shape

Each element of `enrichments[]` describes one enrichment record.

| Field | Meaning |
|---|---|
| `name` | Constant `"resources"`, the OCSF attribute the enrichment data pertains to. |
| `value` | Constant `"N/A"`. OCSF requires the field, but this data supplements `resources` as a whole rather than annotating one existing element of it. |
| `type` | The kind of enrichment: `BulkActionEnrichment` or `AiActionEnrichment`. Dispatch on this to decide how to read `data.payload`. |
| `provider` | `<request_id>/enrichment/<enrichment_id>`, identifying the source record. The request ID is the same value as `metadata.correlation_uid`. |
| `created_time` | Epoch milliseconds, when the enrichment record was created. Shortly after the event's own `time`, so the two are not identical. |
| `data.type_version` | Version of the `data.payload` contract for this `type`. |
| `data.created_time` | The same instant as `created_time`, as epoch milliseconds. |
| `data.payload` | The enrichment content itself: an array whose shape depends on `type`. |

There is one entry per enrichment record, so a single request can produce
several, including entries of different `type`s. Read `data.payload` according to
`type` and `data.type_version`. Both the set of enrichment types and the fields
within `data.payload` may grow over time, so ignore payload fields you do not
recognise rather than rejecting the entry.

### Bulk Actions

`data.payload` is an array of `{action, entities}` objects. `action` is the bulk
operation, such as `publish`, `unpublish`, `validate` or `duplicate`, and
`entities` lists the addressed items as Contentful link objects. A bulk-action
request in full:

```json
{
  "activity_id": 1,
  "activity_name": "Create",
  "class_uid": 6003,
  "class_name": "API Activity",
  "category_uid": 6,
  "type_uid": 600301,
  "severity_id": 1,
  "status_id": 1,
  "time": 1779969612000,
  "duration": 3,
  "actor": {
    "user": {
      "uid": "5xUser1234abcd",
      "type": "User",
      "type_id": 1,
      "email_addr": "jane.doe@example.com",
      "full_name": "Jane Doe"
    }
  },
  "metadata": {
    "version": "1.3.0",
    "uid": "4b7d0e21-9c3f-4a58-8d16-7e2fb0a91c53",
    "correlation_uid": "1f8c3a9d-52b4-4e07-9a6d-c31b8e45f207",
    "tenant_uid": "0XYZ123orgabc",
    "log_name": "cma-api-audit-log",
    "product": {
      "name": "Content Management API",
      "vendor_name": "Contentful"
    }
  },
  "enrichments": [
    {
      "name": "resources",
      "value": "N/A",
      "type": "BulkActionEnrichment",
      "provider": "1f8c3a9d-52b4-4e07-9a6d-c31b8e45f207/enrichment/6d41a0b8-3e57-4c92-b8f1-2a9e07c5d431",
      "created_time": 1779969612480,
      "data": {
        "type_version": "1.0.0",
        "created_time": 1779969612480,
        "payload": [
          {
            "action": "validate",
            "entities": [
              { "sys": { "type": "Link", "linkType": "Entry", "id": "entry42xyz" } },
              { "sys": { "type": "Link", "linkType": "Entry", "id": "entry77abc" } }
            ]
          }
        ]
      }
    }
  ],
  "api": {
    "operation": "/spaces/:spaceId/environments/:environmentId/bulk_actions/validate"
  },
  "resources": [
    { "uid": "abc123space", "type": "space" },
    { "uid": "staging", "type": "environment" },
    { "uid": "bulk42action9x", "type": "entity" }
  ],
  "http_request": {
    "uid": "1f8c3a9d-52b4-4e07-9a6d-c31b8e45f207",
    "http_method": "POST",
    "referrer": "",
    "user_agent": "axios/1.7.2",
    "url": {
      "hostname": "api.contentful.com",
      "path": "/spaces/abc123space/environments/staging/bulk_actions/validate",
      "query_string": ""
    },
    "http_headers": [
      { "name": "x-contentful-user-agent", "value": "sdk contentful-management.js/11.2.0; platform node.js/v20.10.0; os Linux/v5.15;" },
      { "name": "authorization", "value": "CFPA[REDACTED]x7Qz" }
    ]
  },
  "http_response": {
    "code": 201,
    "length": 663,
    "latency": 3,
    "http_headers": [
      { "name": "x-cache", "value": "PASS" }
    ]
  }
}
```

Note that `resources[]` identifies the bulk action itself, not the entries it
addressed. Those are in `enrichments[].data.payload[].entities[]`.

### AI Actions

`data.payload` is an array of invocations, each recording the AI action invoked,
the entry and field affected, and the model that served the request. The
`enrichments` entry, on an event otherwise shaped like the one above:

```json
{
  "name": "resources",
  "value": "N/A",
  "type": "AiActionEnrichment",
  "provider": "8a2e5f14-7b93-4d60-a1c8-5e07b9d3f462/enrichment/c05f9b73-1d24-42a8-9e36-7f18ca40b5de",
  "created_time": 1779970104250,
  "data": {
    "type_version": "1.1",
    "created_time": 1779970104250,
    "payload": [
      {
        "invocationId": "5wQ2mNbT8kRfPzL3vYcH1s",
        "aiActionId": { "sys": { "type": "Link", "linkType": "AiAction", "id": "2pKdR7nMxQwJ4tBvZs9Lqe", "version": 5 } },
        "createdBy": { "sys": { "type": "Link", "linkType": "User", "id": "5xUser1234abcd" } },
        "entryAffected": {
          "entityId": "entry42xyz",
          "entityType": "Entry",
          "fieldId": "productDescription",
          "sourceLocale": "en-US"
        },
        "modelName": "anthropic.claude-4-5-sonnet",
        "modelProvider": "aws_bedrock",
        "modelTemperature": 0.1,
        "outputFormat": "Suggestion"
      }
    ]
  }
}
```

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

### Coverage

| | Daily batch export | NRAL |
|---|---|---|
| Read requests (`GET`) | not included | **included** |
| Authorization token | not included | included (redacted) |

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
event actually represents: a Contentful Management API request. Context that the
batch export carried alongside it, such as bulk-action details, is delivered on
the same event under [`enrichments[]`](#enrichments).

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
| `actor.user` for human users | `uid`, `type: "User"`, `type_id: 2`, plus `email_addr` and `full_name` | `uid`, `type: "User"`, `type_id: 1`, plus `email_addr` and `full_name`. Note the `type_id` difference from the batch export's `2` |
| Apps | on `actor.user`, as `uid`, `type: "App"`, `type_id: 3` | on `actor.app_uid`, holding the app ID. No `actor.user` |
| Delegated actor (app on behalf of user) | not represented | on `actor.invoked_by`, always an `app:<id>` value |
| Anonymous / unauthenticated | an `actor` object with `type: "Unknown"` | `actor` omitted entirely |

NRAL identifies actors by ID (`actor.user.uid` for users, `actor.app_uid`
for apps) and, when applicable, the delegated actor (`actor.invoked_by`).
For human users, `email_addr` and `full_name` are populated as well, matching
the batch export's profile information. Note the `type_id` mismatch: NRAL uses
OCSF's canonical `type_id = 1` for `"User"`; the batch export historically
emits `2`.

### `enrichments` and `web_resources`

| | Daily batch export | NRAL |
|---|---|---|
| `enrichments[]` on the main event | a catch-all: URL-path entities, the delegated actor, the redacted token, **and** AI/Bulk action payloads | **AI/Bulk action payloads only.** The other three are now first-class OCSF fields: URL-path entities in `resources[]`, delegated actor in `actor.invoked_by`, redacted token in `http_request.http_headers[]` |
| `web_resources[]` on the main event | yes, `[{type: "<NounName>", uid: "<id>"}]` (e.g. `BulkAction`, `Entry`) | **not populated**; see `resources[]` instead |
| Acted-on resources on the main event | `web_resources[]` with the specific CMA noun (e.g. `BulkAction`, `Entry`, `Tag`) | `resources[]` with coarse types only: `space`, `environment`, `entity` |
| AI Actions / Bulk Actions extra data | inline in the same event's `enrichments[]` | inline in the same event's `enrichments[]`. See [Enrichments](#enrichments) |

The coarsening to `space`/`environment`/`entity` is intentional. The CMA
exposes dozens of resource nouns that change over time, and tying the OCSF
contract to that surface forced schema churn on every CMA addition. If you
need the specific noun, parse it from `http_request.url.path`.

AI and Bulk action entries stay in `enrichments[]`, and the entry itself is close
to the batch shape. What moved:

| Field | Daily batch export | NRAL |
|---|---|---|
| `name` | `"web_resources"` | `"resources"`: `web_resources` is not an attribute of `API Activity`, and `resources` is its counterpart |
| `value` | `"N/A"` | unchanged |
| `type` | the enrichment kind | unchanged |
| `provider` | `<request_id>/enrichment/<enrichment_id>` | unchanged |
| `created_time` | RFC-3339 string, at entry level | epoch milliseconds at entry level, per OCSF `Timestamp_t`. `data.created_time` is now epoch milliseconds too, the same instant |
| `type_version` | at entry level | `data.type_version`: not an OCSF attribute, so it moves inside `data` |
| the payload array | `data` **is** the array | `data.payload` is the array; `data` is now an object holding it plus its version |

So a batch consumer reading `enrichment.data[0].entities` reads
`enrichment.data.payload[0].entities` in NRAL, one reading
`enrichment.created_time` as a string gets a number, and one reading
`enrichment.data.created_time` as a string also gets a number. Those three are
the only breaking changes in the entry.

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

### Delivery guarantees

NRAL follows the same **at-least-once, best-effort** delivery semantics as
the rest of Enterprise Observability (see the
[EO delivery documentation](https://www.contentful.com/developers/docs/concepts/enterprise-observability/#receiving-duplicate-events)).
Consumers should expect and tolerate duplicates:

- **Deduplicate on `metadata.correlation_uid`.** It is stable per originating
  request; two events carrying the same `correlation_uid` describe the same
  underlying call.
- **`metadata.uid` is not a deduplication key.** It is a fresh UUID per
  emitted event, so a duplicated request produces two distinct `metadata.uid`
  values.

### Destinations

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

- **`resources[]` does not list the entities a bulk operation addressed.**
  CMA endpoints such as `bulk_actions` and batch publish operate on many
  entries in a single HTTP request. `resources[]` identifies the space, the
  environment, and the bulk action itself, not the individual entries. The full
  set is delivered on the same event under `enrichments[]`; see
  [Enrichments](#enrichments).
- **Some `api.operation` values may be empty.** When upstream routing does
  not set a route template, the entire `api` object is dropped from the
  event. Affected endpoints are being closed out during the early-release
  period.

