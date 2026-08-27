# API Gateway Analytics — Elasticsearch Index Lifecycle Management Design

**Role:** API Gateway / Middleware Specialist
**Stack:** webMethods API Gateway 10.x · Elasticsearch (Index Templates, ILM) · Kibana

This repo documents an Index Lifecycle Management (ILM) setup I built for the analytics data emitted by webMethods API Gateway — every API transaction the gateway handles gets logged as a document, and this defines how those documents are indexed, mapped, and eventually rolled off.

---

## 1. The data

Each API call the gateway processes produces one transactional event document — full schema in [`mappings/gateway-analytics-mapping.json`](mappings/gateway-analytics-mapping.json). It captures the whole lifecycle of a request:

- **Identity**: `apiId`/`apiName`, `applicationId`/`applicationName`, `planId`/`planName`, `packageId`/`packageName`
- **Timing**: `gatewayTime`, `providerTime`, `totalTime`, `creationDate`
- **Payloads**: `nativeReqPayload` / `nativeResPayload`, native headers, response codes
- **Downstream calls**: `externalCalls` — a `nested` array recording every call the gateway made *out* to backend services on behalf of this request (URL, start/end time, duration, response code)

That last one is what makes the schema genuinely useful for troubleshooting: a single document can show you the inbound request, the gateway's own processing time, and every outbound call it triggered, all correlated by `correlationID`.

## 2. Index template design

Full file: [`templates/gateway-analytics-index-template.json`](templates/gateway-analytics-index-template.json)

A few decisions worth calling out:

- **`mapping.total_fields.limit: 50000`** — raised well above the ES default (1000) because this schema is wide and includes nested objects (`externalCalls`) plus multiple `object`-type fields for headers. Default limits would reject documents outright once enough distinct field combinations showed up.
- **Two ingest pipelines wired in**: `default_pipeline: gateway_analytics_xRequestId` runs on ingest (tagging/correlating the request), and `final_pipeline: parse-respayload` runs last, after all other pipeline processing, to parse the response payload field.
- **A custom analyzer, `not_analyzed_ignorecase`** — this is the one I'm most likely to get asked about in an interview. Several `text` fields (`apiName`, `nativeHttpMethod`, `request`, etc.) needed case-insensitive *exact*-value matching and aggregation, not tokenized full-text search. Elasticsearch's `keyword` type would normally handle that, but `keyword` has a hard 32,766-byte term limit — and payload/header-adjacent fields in this schema can exceed that, which throws a hard indexing exception rather than truncating. The fix: a custom analyzer using the `keyword` tokenizer (treats the whole value as one token, like `keyword` type does) + `lowercase` filter (case-insensitivity) + a custom `truncate_max` filter capping tokens at 32,766 characters (avoids the Lucene term-length exception instead of failing on it). `fielddata: true` is then enabled on top so these fields remain aggregatable/sortable despite being mapped as `text`.

## 3. ILM policy

Full file: [`policies/apigateway-ilm-policy.json`](policies/apigateway-ilm-policy.json) (reconstructed from the Kibana editor — screenshots below).

| Phase | Trigger | Action |
|---|---|---|
| **Hot** | always active | Rollover when primary shard hits **30gb** *or* index reaches **1 day** old |
| **Cold** | *not enabled* | — |
| **Delete** | index is **10 days** old | Delete (no snapshot wait — no snapshot lifecycle policy exists on this cluster yet) |

![Hot phase configuration](<img width="872" height="404" alt="02-ilm-hot-phase" src="https://github.com/user-attachments/assets/ec341d67-630b-44bb-9a16-345159428112" />)
![Delete phase configuration](<img width="944" height="399" alt="03-ilm-delete-phase" src="https://github.com/user-attachments/assets/04d2a3ea-5a00-4c18-a946-26dd0e741d6b" />)

**Why no warm/cold tier:** with only a 10-day total retention, the cost/complexity of relocating data to a separate tier before deleting it isn't worth it here — everything just lives on hot storage for its full 10-day life and then gets deleted directly. Cold/warm tiering earns its keep when retention stretches into weeks/months and query patterns clearly shift from "recent, hot" to "occasional, historical" — that's not this workload.

## 4. What the live indices actually look like

![Index Management view](<img width="942" height="411" alt="01-index-management-overview" src="https://github.com/user-attachments/assets/5a9c9b6d-0bb7-40e5-bad1-de3a900ecb53" />)

Real index sizes from Index Management, one per day under the `gateway_default_analytics_transactionalevents-*` rollover alias:

| Index (date) | Docs | Storage |
|---|---|---|
| 2026.08.16 | 82,832 | 4.18gb |
| 2026.08.17 | 1,503,937 | 5.44gb |
| 2026.08.18 | 681,091 | 8.55gb |
| 2026.08.19 | 1,322,617 | 6.59gb |
| 2026.08.20 | 361,598 | 5.32gb |
| 2026.08.21 | 137,668 | 4.25gb |
| 2026.08.22 | 172,312 | 4.93gb |
| 2026.08.23 | 301,904 | 6.4gb |

Two things worth pointing out about this data (and worth being ready to talk through if asked):

1. **Doc counts swing 20x day to day (82K → 1.5M) but storage stays in a narrow 4–8.5GB band.** That tells you the **1-day `max_age`** is the rollover trigger actually firing here, not the 30gb size ceiling — traffic never gets close to 30gb/day, so the size threshold is just a safety ceiling for a future traffic spike, not what's driving rollover day-to-day.
2. **Every index shows `yellow` health.** With `number_of_replicas: 1` set in the template, a persistently yellow cluster across every index usually means the replica shard isn't being assigned — most commonly because there's only one data node available to hold it. That's a real open item, not something this design fixes on its own: either the cluster needs a second data node, or `number_of_replicas` should be dropped to `0` if single-node is the intended topology for this environment.

## 5. What I'd do next

- Resolve the replica allocation issue causing persistent yellow health (add a node, or set `number_of_replicas: 0` if single-node is by design)
- Set up a snapshot lifecycle policy so the delete phase can optionally wait for a snapshot before removing data, giving a recovery window past the 10-day hot retention
- Revisit the 10-day retention against actual audit/troubleshooting SLA requirements — if longer retention is ever needed, that's when a cold tier becomes worth the added complexity

---

## Repo structure

```
.
├── README.md
├── templates/
│   └── gateway-analytics-index-template.json
├── mappings/
│   └── gateway-analytics-mapping.json
├── policies/
│   └── apigateway-ilm-policy.json
└── screenshots/
    ├── 01-index-management-overview.png
    ├── 02-ilm-hot-phase.png
    └── 03-ilm-delete-phase.png
```
