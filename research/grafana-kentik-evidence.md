# Grafana and Kentik APIs for evidence

- **Ticket:** [#8 Grafana and Kentik APIs for evidence](https://github.com/callumbnz/outage-manager/issues/8). Map: [#1](https://github.com/callumbnz/outage-manager/issues/1)
- **Related:** [#7 NMS research](https://github.com/callumbnz/outage-manager/blob/research/nms-librenms-observium/research/nms-librenms-observium.md), which covers the `NmsAdapter` and snapshotting graphs when a PIR is created
- **Researched:** 2026-09-28, against grafana.com/docs (the `latest` docs, which cover Grafana 12/13 and note the `/api` → `/apis` transition), kb.kentik.com, and the GitHub repos `grafana/grafana-image-renderer`, `kentik/api-schema-public` and `minio/minio`

## Question

How do we fetch rendered panel images or snapshots from Grafana, and traffic trend data or charts from Kentik, so we can embed them in PIRs and show them as Incident context? What are the auth models?

## TL;DR / recommendation

- **Grafana.** Use a **Viewer service-account token** (`Authorization: Bearer …`). Celery calls `GET /render/d-solo/{dashUid}/{slug}?panelId=…&from=…&to=…&var-…=…&width=…&height=…&tz=UTC` and stores the returned PNG. This needs Devoli's Grafana to have the **Grafana Image Renderer** service (`grafana/grafana-image-renderer`, which ships linux/amd64 images). It is a separate container wired to Grafana through `[rendering] server_url` and `renderer_token`. Grafana recommends at least 16 GiB of RAM and 4 cores for it. Snapshots are not suitable: the API needs the full dashboard JSON *including the data*, it is designed for the UI, and the snapshot stays inside Grafana. For live Incident context, use deep links (`/d/{uid}/{slug}?from=&to=&var-x=`). Optionally store raw numbers too via `POST /api/ds/query`.
- **Kentik.** Auth uses the headers `X-CH-Auth-Email` and `X-CH-Auth-API-Token` for a **dedicated Member-level user**. Use the **V5 Query API** on the US (`api.kentik.com`) or EU (`api.kentik.eu`) cluster:
  - `POST /api/v5/query/topXchart` returns `{"dataUri": base64}` as PNG, SVG, JPG or PDF.
  - `POST /api/next/v5/query/topXdata` returns time-series JSON.
  - `POST /api/v5/query/url` returns a Data Explorer short URL for live context.

  One JSON query body serves all three. The best way to write it is to build the view in Data Explorer and copy it with **Show API Call**. Filter with `i_device_name`, `i_input|output_interface_description` (the interface name), `i_input|output_snmp_alias` (the interface description), or a custom dimension or saved filter for Customer. Rate limits for the Query API: 4 concurrent, 30/min soft, 100/min hard, 1500/hour.
- **Risk:** Kentik labels V5 as *deprecated*, but the V5 Query API is still documented, and the public V6 gRPC schema has **no flow-query package**. Build against V5, behind the adapter, and ask Kentik about a replacement roadmap.
- **Abstraction.** LibreNMS, Grafana and Kentik all fit one `EvidenceSource` protocol: `render(spec, start, end) -> EvidenceArtifact`, `data(spec, start, end)` (optional), and `live_url(spec, start, end)`. It uses capability flags, as in #7. Only LibreNMS also implements the NMS-specific methods (alerts, event log, maintenance). **Evidence Specs** are saved templates, configured in the main app, that say which panel or query to capture for a given Network Element type.
- **Storage.** Store evidence as immutable files through Django `STORAGES`. Use `FileSystemStorage` on a dedicated, backed-up volume by default, and switch to an S3-compatible backend (django-storages `endpoint_url`) if Devoli has an existing object store. **Do not adopt MinIO community edition:** the `minio/minio` repo is now archived and marked "no longer maintained".
- **Capture timing.** Snapshot when the PIR is created, as #7 decided. Kentik's *Full* dataseries is the default only for queries under 24h and has shorter retention than *Fast*, which is downsampled and snaps to the hour.

---

## 1. Grafana

### 1.1 Auth: Viewer service account

- Service accounts are Grafana users meant for automation. Each can have multiple tokens, and tokens **inherit the account's permissions**. Unlike API keys, they aren't tied to a human user. Tokens have **no expiry by default**. Admins can cap this with `token_expiration_day_limit`, and Grafana recommends short expiries. [Service accounts](https://grafana.com/docs/grafana/latest/administration/service-accounts/)
- An Org Admin creates the account and assigns it an organisation role: **Viewer**, Editor or Admin. Since 10.2 there is also a `None` role that starts with zero permissions and gets RBAC grants. [Service accounts](https://grafana.com/docs/grafana/latest/administration/service-accounts/)
- Header: `Authorization: Bearer <SERVICE_ACCOUNT_TOKEN>`. The API docs use this in every example. [Search API](https://grafana.com/docs/grafana/latest/developers/http_api/folder_dashboard_search/)
- Rendered images require "authorized viewer permission" on the dashboard. [Share dashboards and panels](https://grafana.com/docs/grafana/latest/dashboards/share-dashboards-panels/)
- **Proposal:** a service account `outage-manager` with the **Viewer** role. If Devoli uses folder permissions, grant View on only the relevant folders. Querying data sources through `/api/ds/query` also needs query permission on the data source, which Viewers normally have for data sources they can see (to confirm on Devoli's RBAC setup).

### 1.2 Rendering a panel to PNG for a time range, with variables

- Endpoint pattern, from the documented example: [Share dashboards and panels, "Query string parameters for server-side rendered images"](https://grafana.com/docs/grafana/latest/dashboards/share-dashboards-panels/)

  ```
  GET {grafana}/render/d-solo/{dashboardUid}/{slug}?orgId=1
      &panelId={panelId}
      &from=2026-09-01T10:00:00Z&to=2026-09-01T14:00:00Z   # ISO or epoch ms
      &var-device=akl-core-01&var-interface=xe-0/0/1       # template variables
      &width=1000&height=500&scale=1&tz=UTC&timeout=60
  Authorization: Bearer <token>
  → 200 image/png
  ```

  | Param | Meaning (from the docs) |
  |---|---|
  | `width` / `height` | Pixels. The default and **minimum** are 1000 and 500. Self-hosted instances can change the renderer's minimums |
  | `scale` | Device scale factor (DPI). The default is 1 |
  | `tz` | Timezone, for example `UTC` or `UTC%2BHH%3AMM` |
  | `timeout` | Seconds. The default is 30. Raise it for slow panel queries |
  | `from` / `to` | Time range, as epoch ms or ISO (the docs' example uses ISO) |
  | `var-<name>` | Sets a dashboard variable. Repeat the parameter for multiple values (`var-x=a&var-x=b`). Ad-hoc filters use `var-filter=key|=|value` ([Dashboard URL variables](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/create-dashboard-url-variables/)) |

- **`panelId` format:** the docs' example uses `panelId=panel-13`, the newer dashboard-schema form. Older dashboards use a numeric `panelId=13`. The adapter should read the panel's ID from the dashboard JSON (§1.5) and pass it through unchanged. **Verify both forms on Devoli's version.**
- Grafana enforces a server-side limit on concurrent renders: `[rendering] concurrent_render_request_limit`, default 30. [Configure Grafana, [rendering]](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/#rendering)
- Always send **absolute** `from`/`to` values, never `now-…`, so the evidence can be reproduced.

### 1.3 Renderer deployment requirement

- The image renderer is now a **standalone service**. The old Grafana *plugin* "is deprecated and no longer receives updates", and "we only support the latest version of the service". [Set up image rendering](https://grafana.com/docs/grafana/latest/setup-grafana/image-rendering/)
- **Image:** `grafana/grafana-image-renderer:<pinned version>`. The latest release is `v5.12.4` (2026-09-21), per [GitHub releases](https://github.com/grafana/grafana-image-renderer/releases). Grafana says to pin a version in production. Images are published for **linux/amd64** and linux/arm64, so it runs on x86_64 RHEL under docker/podman compose. The current image is a Go binary on Debian with Chromium bundled. [Dockerfile](https://github.com/grafana/grafana-image-renderer/blob/main/Dockerfile)
- **Resources:** Grafana says to allocate "at least 16 GiB of memory and at least 4 CPU cores", because load is spiky. If you set a memory limit, set `GOMEMLIMIT` below it: about 1 GiB of `GOMEMLIMIT` per 8 GiB of container limit, leaving headroom for Chromium. [Set up image rendering](https://grafana.com/docs/grafana/latest/setup-grafana/image-rendering/)
- **Wiring:** this is set up on *Grafana's* side, not ours.
  - Grafana settings: `GF_RENDERING_SERVER_URL=http://renderer:8081/render`, `GF_RENDERING_CALLBACK_URL=http://grafana:3000/` and `GF_RENDERING_RENDERER_TOKEN=<secret>`.
  - The renderer's `AUTH_TOKEN` (`--server.auth-token`) must match that token. **Change the default token `-`.**
  - The renderer exposes Prometheus `/metrics`.
- **Where it runs:** the renderer has to sit next to *Devoli's Grafana*, which calls it over HTTP, and Grafana must be reachable from the renderer at the callback URL. Outage Manager never talks to the renderer directly. It calls Grafana's `/render/...`. So whether this goes in *our* compose file depends on who runs Grafana:
  - **Grafana Cloud:** rendering is provided, and the minimum sizes can't be changed.
  - **Self-hosted:** Devoli adds the renderer container to Grafana's deployment. Adding it to our compose file only works if Grafana can reach it.
- **Fallback if Devoli won't run the renderer:** store the data (§1.6) and draw charts ourselves, for example server-side with matplotlib, or client-side when viewing, with a PNG export for PDF/Status.io. This is more work and doesn't look like their dashboards.

### 1.4 Snapshots vs rendered images for permanence

- `POST /api/snapshots`: "you have to provide the full dashboard payload **including the snapshot data**. This endpoint is designed for the Grafana UI." A snapshot lives in Grafana (or in the external `snapshots.raintank.io` service), expires when its `expires` setting says (default never), and can be deleted by anyone holding its key. [Snapshot API](https://grafana.com/docs/grafana/latest/developers/http_api/snapshot/)
- Snapshots strip queries and keep only the visible series. Anyone with the link can view them. [Share dashboards and panels](https://grafana.com/docs/grafana/latest/dashboards/share-dashboards-panels/)
- **Externally shared (public) dashboards:** "Variables and queries including variables are not supported". They also need Admin or `dashboards.public:write` to create. [Externally shared dashboards](https://grafana.com/docs/grafana/latest/dashboards/share-dashboards-panels/shared-dashboards/). Not suitable.
- **Decision:** use **rendered PNG plus optional raw data, stored in Outage Manager**. That is the only option we fully control for permanence, and it survives changes to Grafana's retention or dashboards. Snapshots and public dashboards are out.

### 1.5 Finding panels by dashboard UID and panel ID

- Search: `GET /api/search?type=dash-db&query=…&tag=…&folderUIDs=…&dashboardUIDs=…&limit=&page=` returns `uid`, `title`, `url` (`/d/{uid}/{slug}`), `folderUid` and `tags`. Results are limited to what the token can see. [Search API](https://grafana.com/docs/grafana/latest/developers/http_api/folder_dashboard_search/)
- Dashboard JSON: the legacy `GET /api/dashboards/uid/:uid` ([v11.6 docs](https://grafana.com/docs/grafana/v11.6/developers/http_api/dashboard/)) or the new `GET /apis/dashboard.grafana.app/v1/namespaces/:namespace/dashboards/:uid` ([latest](https://grafana.com/docs/grafana/latest/developers/http_api/dashboard/)).
  - The JSON has `panels[]` (`id`, `title`, `type`, `datasource`, `targets`) and `templating.list[]` (variable `name`, `options` and `current`).
  - From Grafana 13, `/api` is deprecated in favour of `/apis` but "remain[s] fully accessible". Support both, depending on version.
- **UX:** the Evidence Spec editor in the main app lets an admin pick a dashboard (search), then a panel (from `panels[]`), then map template variables to Network Element attributes. For example, `var-device ← NE.hostname` and `var-interface ← NE.if_name`. Store `dashboard_uid`, `panel_id` and the variable mapping. Store titles only for display.

### 1.6 Deep links for live Incident context

- `{grafana}/d/{uid}/{slug}?orgId=1&from={ms}&to={ms}&var-device=…&var-interface=…`. `from`/`to` take epoch ms or relative `now-6h`, and there are also `time` and `time.window`. [Manage dashboard links](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/manage-dashboard-links/), [URL variables](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/create-dashboard-url-variables/)
- For an ongoing Incident, use `from=<start−1h>&to=now`. After resolution, use absolute times.
- To open a single panel full-screen, the Grafana UI uses `viewPanel=<id>`. This was **not found in the docs pages reviewed**, so treat it as best effort and verify on Devoli's version.
- Viewers need a Grafana login. The link is only a convenience for engineers.

### 1.7 Raw numbers via `/api/ds/query`

- `POST /api/ds/query` takes `{"from":"<ms|now-…>","to":"…","queries":[{"refId":"A","datasource":{"uid":"…"},"maxDataPoints":…,"intervalMs":…, …datasource-specific fields}]}` and returns data frames. It only works for data sources with a backend implementation, which the built-in ones have. The query fields depend on the data source: Grafana suggests copying them from browser dev tools. [Data source API](https://grafana.com/docs/grafana/latest/developers/http_api/data_source/)
- **Practical approach:** take `panel.targets[]` and `panel.datasource` from the dashboard JSON (§1.5), substitute `$var` values ourselves, and post them. Variable interpolation is **our** job here, and it's fragile for chained or query-based variables. **Recommendation:** use this optionally, to store numbers such as peak, min and p95 during the Incident next to the image. It doesn't replace the image.

---

## 2. Kentik

### 2.1 Auth, clusters and API versions

- Every request carries the headers `X-CH-Auth-Email: <user email>` and `X-CH-Auth-API-Token: <token>`. The token appears on the user's profile page and **belongs to that user**, so there are no scoped or service tokens. [Kentik APIs](https://kb.kentik.com/docs/apis-overview)
- Base URLs:
  - US: `https://api.kentik.com/api/v5` and `https://grpc.api.kentik.com` (V6)
  - EU: `https://api.kentik.eu/api/v5` and `https://grpc.api.kentik.eu`

  [Kentik APIs](https://kb.kentik.com/docs/apis-overview)
- **Member-level** users can only call the GET methods of the Admin APIs. [Admin APIs](https://kb.kentik.com/docs/admin-apis). **Proposal:** a dedicated Member user such as `outage-manager@devoli…`, with its token stored encrypted in the main-app configuration. That limits the impact if the token leaks.
- **Versions:** V5 (REST) is labelled "Deprecated", and the V5 API *tester* was discontinued in Jan 2025. V6 is gRPC/REST-gateway, and Kentik says its Alpha and Beta APIs are "not supported or recommended for production". [Kentik APIs](https://kb.kentik.com/docs/apis-overview)
  - The V5 **Query** API is still fully documented and live. Only its SQL method was removed, on 2025-05-01. [Query API](https://kb.kentik.com/docs/query-api)
  - I found **no V6 flow-query service** among the packages in [`kentik/api-schema-public/proto/kentik`](https://github.com/kentik/api-schema-public/tree/master/proto/kentik): device, interface, synthetics, and so on.
  - So V5 Query is the only option for traffic charts today. Ask Kentik for their roadmap (open question).

### 2.2 Querying traffic for a device, interface or Customer over a time range

A single query JSON body is shared by `topXchart`, `topXdata` and `url`. [Query API JSON](https://kb.kentik.com/docs/query-api)

```json
{
  "queries": [{
    "bucket": "Left +Y Axis", "bucketIndex": 0, "isOverlay": false,
    "query": {
      "metric": "bytes",
      "dimension": ["Traffic"],
      "viz_type": "line",
      "topx": 8, "depth": 25,
      "lookback_seconds": 0,
      "starting_time": "2026-09-01 10:00:00",
      "ending_time":   "2026-09-01 14:00:00",
      "time_format": "UTC",
      "device_name": "akl-core-01",
      "all_selected": false,
      "fastData": "Auto",
      "filters_obj": { "connector": "All", "filterGroups": [{
        "connector": "All", "not": false,
        "filters": [{ "filterField": "i_output_interface_description", "operator": "=", "filterValue": "xe-0/0/1" }]
      }]},
      "query_title": "INC-123 akl-core-01 xe-0/0/1 egress"
    }
  }],
  "imageType": "png"
}
```

- **Time range:** set `lookback_seconds: 0` and give `starting_time`/`ending_time` as `'YYYY-MM-DD HH:mm:00'`, rounded to the minute. Otherwise `lookback_seconds` overrides them. `time_format` (UTC/Local) is documented as used only by the URL method, so **confirm that absolute times are UTC for chart and data calls** (open question). [Query object](https://kb.kentik.com/docs/query-api)
- **Device:** `device_name` takes a comma-delimited list of names. You can instead use `device_labels`, which takes label IDs only, not names. `all_selected: true` ignores `device_name`.
- **Interface:** use filters, not dimensions. [General dimensions](https://kb.kentik.com/docs/general-dimensions)

  | Portal name | KDE `filterField` |
  |---|---|
  | Device Name | `i_device_name` |
  | Site | `i_device_site_name` |
  | Interface Name (e.g. `GigabitEthernet0/1`) | `i_input_interface_description` / `i_output_interface_description` |
  | Interface Description (ifAlias) | `i_input_snmp_alias` / `i_output_snmp_alias` |
  | Interface ID (ifIndex) | `input_port` / `output_port` |

  Note that the *KDE* names are counter-intuitive: `interface_description` holds the **name**, and `snmp_alias` holds the **description**.
- **Customer:** Kentik has no built-in "customer". Options, best first:
  1. A **custom dimension** (for example `c_customer`), populated by populators. Admins can manage these through the portal or API. [Custom dimensions](https://kb.kentik.com/docs/custom-dimensions)
  2. **Saved filters** per Customer, referenced by `saved_filters: [{filter_id}]`.
  3. `ILIKE` on `i_*_snmp_alias`, if Devoli's interface descriptions follow a naming convention that includes the customer or circuit ID.

  Which of these applies depends on how Devoli has set up Kentik (open question).
- **Direction:** `metric` can be `bytes`, `in_bytes`, `out_bytes`, `packets` and so on. Filter on both input and output interface, or use two buckets for in and out.
- **Resolution:** without `fastData`/`i_fast_dataset`, queries under 24h use the **Full** dataseries, and 24h or longer uses **Fast** (downsampled, snapped to the hour). Full has shorter ("standard") retention than Fast ("extended"). [Resolution overview](https://kb.kentik.com/docs/resolution-overview). Incident windows are usually under 24h, so **capture when the PIR is created**, while Full data still exists.
- **Writing queries:** in Data Explorer, go to Options → **Show API Call** → Chart/Data → JSON, then copy it. [Data Explorer, Show API Call](https://kb.kentik.com/docs/data-explorer). The Evidence Spec editor should accept a pasted query JSON with placeholders (`{{device_name}}`, `{{if_name}}`, `{{start}}`, `{{end}}`) rather than trying to model every Kentik field.

### 2.3 Chart image vs data

| | Chart | Data |
|---|---|---|
| Endpoint | `POST https://api.kentik.{com,eu}/api/v5/query/topXchart` | `POST https://api.kentik.{com,eu}/api/next/v5/query/topXdata` (note the `/api/next/` path) |
| Output | JSON `{"dataUri": "<base64>"}`, with `imageType` of `svg` (the default, "recommended"), `png`, `jpg` or `pdf` | `{"results":[{"bucket","data":[{key, avg_/p95th_/max_bits_per_sec, timeSeries:{both_bits_per_sec:{flow:[[ts_ms, value, step_s],…]}}}]}]}` |
| Use | PIR image | Numbers (peak, p95, dip) and our own charts |

Sources: [Query Chart method, Query Data method](https://kb.kentik.com/docs/query-api). The docs note that the Chart and Data results can differ slightly, because Data results are processed for accuracy.

**Recommendation:** request `imageType: "png"` for PIR embedding (safe to show inline, and portable to PDF, email and Status.io). Also call `topXdata` for the headline numbers. Storing SVG is possible, but it needs sanitising before being served inline (it's active content), so PNG is simpler.

### 2.4 Deep links into Data Explorer

- `POST /api/v5/query/url` takes the same body without `imageType` and returns a quoted short URL such as `https://portal.kentik.com/portal/#Charts/shortUrl/<hash>`. [Query URL method](https://kb.kentik.com/docs/query-api)
- Generate one per Evidence Spec when the Incident is opened (use a lookback, or absolute times once resolved), and store it. Viewers need a Kentik login. Kentik also has "internal share" links from the portal Share dialog, which are UI-only. [Portal sharing](https://kb.kentik.com/docs/portal-sharing-and-export)
- Kentik "public shares" exist but expose data without authentication. **Don't use them.**

### 2.5 Rate limits (per customer)

| | Non-query APIs | **Query API** |
|---|---|---|
| Max concurrent | 1 | **4** |
| Soft limit per minute (adds a delay) | 20 (1.5 s delay) | **30** (1 s delay) |
| Hard limit per minute (HTTP 429) | 60 | **100** |
| Hourly limit (HTTP 429) | 3750 | **1500** |

These are rolling windows, and the Query and Admin APIs are counted separately. [Kentik APIs, API Rate Limiting](https://kb.kentik.com/docs/apis-overview). The limits are **per Kentik customer**, so they are shared with Devoli's other integrations. Use a dedicated Celery queue with a concurrency of 2 and backoff on 429. A PIR with, say, 5 Network Elements × 2 specs × (chart + data) = 20 calls is well inside the limits.

### 2.6 Matching Kentik names to Boris / LibreNMS

- A Kentik **device** has a user-defined `device_name`, `sending_ips` (flow exporter IPs) and `device_snmp_ip`, from `GET /api/v5/devices`. [Device API](https://kb.kentik.com/docs/network-assets-apis)
- A Kentik **interface** has `snmp_id` (ifIndex), `interface_description` (the interface *name*, from SNMP or set manually), `snmp_alias` (the *description*), `interface_ip` and `snmp_speed`, from `GET /api/v5/device/{device_id}/interfaces`. [Interface List](https://kb.kentik.com/docs/network-assets-apis). Values can be manually overridden in Kentik. [Interface classification](https://kb.kentik.com/docs/interface-classification)
- LibreNMS (#7) exposes `hostname`, `sysName` and IPs for devices, and `ifName`, `ifAlias` and `ifIndex` for ports.
- **Matching strategy:**
  1. **Device:** match on management/SNMP IP (`device_snmp_ip` against the LibreNMS or Boris IP) first, then on a normalised name (`device_name` against `sysName`/hostname, lower-cased, domain stripped).
  2. **Interface:** within the matched device, use exact `interface_description == ifName`, falling back to `snmp_id == ifIndex`. ifIndex can change after a reboot on some platforms, so prefer the name.
  3. **Persist** the resolved Kentik `device_id` and interface name against the Boris Network Element, with an "unmatched" report in the main app for manual fixing. Run it as a periodic Celery sync using the Admin GET methods, which count against the separate non-query limit.

  Whether names line up exactly depends on Devoli's naming (open question). It's feasible, because both sides read the same SNMP ifName/ifAlias from the same routers.

### 2.7 Synthetics (optional)

Kentik Synthetics has its own V6 APIs for test results and configuration ([Synthetics Monitoring APIs](https://kb.kentik.com/docs/synthetics-monitoring-apis)). This could later add "latency/loss during the Incident" evidence. It's out of scope for the MVP, and we'd need to confirm whether Devoli uses Synthetics.

---

## 3. Proposed common `EvidenceSource` interface

The #7 `NmsAdapter` mixes two concerns: **NMS state** (devices, alerts, event log, maintenance) and **evidence** (graphs and links). Split out the evidence part, so that LibreNMS, Grafana and Kentik all implement it.

```python
class EvidenceCapability(StrEnum):
    IMAGE = "image"        # can return a rendered chart
    DATA = "data"          # can return time-series numbers
    LIVE_URL = "live_url"  # can build a deep link

@dataclass(frozen=True)
class EvidenceTarget:              # resolved from a Boris Network Element
    network_element_id: int
    device_name: str | None
    if_name: str | None
    ip: str | None
    customer_ref: str | None       # e.g. Kentik custom-dimension value / saved filter
    external_ids: dict[str, str]   # {"librenms_device_id": "12", "kentik_device_id": "345"}

@dataclass(frozen=True)
class EvidenceArtifact:
    content: bytes
    content_type: str              # image/png (preferred), image/svg+xml
    width: int | None
    height: int | None
    source_url: str                # request URL (secrets removed)
    request_params: dict           # exact params / query JSON used
    fetched_at: datetime
    range_start: datetime
    range_end: datetime

@dataclass(frozen=True)
class EvidenceSeries:
    series: list[dict]             # [{"name": ..., "points": [(ts, value), ...], "unit": "bps"}]
    summary: dict                  # {"max": ..., "min": ..., "p95": ...}
    raw: dict                      # verbatim response for audit

class EvidenceSource(Protocol):
    kind: str                                      # "librenms" | "grafana" | "kentik"
    capabilities: frozenset[EvidenceCapability]
    def validate_spec(self, spec: "EvidenceSpec") -> list[str]: ...
    def render(self, spec, target: EvidenceTarget, start, end) -> EvidenceArtifact: ...
    def data(self, spec, target: EvidenceTarget, start, end) -> EvidenceSeries: ...      # optional
    def live_url(self, spec, target: EvidenceTarget, start, end | None) -> str: ...
```

**`EvidenceSpec`** is a model configured in the main app, not Django admin. Fields:

- `source` (FK to a connection: base URL, auth, cluster)
- `name`
- `applies_to` (Network Element type: device, interface or circuit, plus optional role or tag filters)
- `params`, source-specific JSON:
  - Grafana: `{dashboard_uid, slug, panel_id, vars: {"device": "{{device_name}}"}, width, height}`
  - Kentik: `{query_template: {...}, image_type: "png"}`
  - LibreNMS: `{graph: "port_bits"}`
- `pad_before` / `pad_after` (default 1h)
- `enabled`

| Method | LibreNMS | Grafana | Kentik | Gaps |
|---|---|---|---|---|
| `render` | `/devices/:id/ports/:if/:graph?graph_type=png` | `/render/d-solo/...` | `topXchart` (base64 in JSON) | Grafana needs the renderer. The minimum size is 1000×500 unless Devoli changes it |
| `data` | *no series API in #7 scope* (RRD) → unsupported | `/api/ds/query`, where we interpolate panel targets ourselves (fragile) | `topXdata` | Aggregations differ per source. Normalise to bps |
| `live_url` | `{base}/device/device={id}/` (+ port tab) | `/d/{uid}/{slug}?from&to&var-…` | `query/url`, which costs 1 Query API call | Kentik links count against rate limits. Generate once and cache |
| Target resolution | `librenms_device_id` | Template vars from NE attributes | `kentik_device_id` + interface name | Each source needs its own ID mapping, kept in `external_ids` |
| Auth | Bearer (read-only user) | Bearer (Viewer service account) | Email + token headers (Member user) | Kentik tokens belong to a person-like user. Rotate them manually |

**Orchestration**, which is the same for all sources: when a PIR is created, find the Incident's Network Elements, then the matching `EvidenceSpec`s, and fan out Celery tasks, one per (spec, target), using the window `[start − pad, end + pad]`. Each task stores an `EvidenceItem`, and failures are recorded against the item so the Author can retry. Add a manual "refresh evidence" action. The Incident page shows `live_url` links (use a).

---

## 4. Storing evidence

- **Formats:**
  - PNG is the canonical format for all sources: Grafana renders only PNG, LibreNMS can return PNG, and Kentik can return PNG.
  - Optionally store Kentik's SVG as well. Sanitise it or serve it as an attachment only.
  - Store the raw data JSON (`EvidenceSeries.raw`) next to each image.
- **Sizes (estimates, not measured):** a 1000×500 PNG panel at scale 1 is usually tens to a few hundred KB, depending on complexity. A typical PIR with about 10 images is only a few MB. At, say, 100 PIRs a year, total growth is in the hundreds of MB to low GB. Storage size isn't a constraint.
- **Model:** `EvidenceItem`, holding:
  - `pir` FK, `spec` FK and `network_element` FK
  - a `FileField`
  - `sha256`, `content_type`, `width` and `height`
  - `source_url` and `request_params`
  - `fetched_at`, `range_start` and `range_end`
  - `data_json` and `status`/`error`

  Items are **immutable** once the PIR is submitted for review (a refresh creates a new version).
- **Object storage on docker-compose / RHEL:**
  - **Default:** Django `STORAGES["default"] = FileSystemStorage` on a **dedicated named volume or bind mount** (for example `/srv/outage-manager/media`, mounted into web and worker containers, with an SELinux `:Z` label on RHEL). Include it in the existing host backup alongside the Postgres dumps. Serve it through Django with a permission check (`X-Accel-Redirect`/`X-Sendfile` via the reverse proxy), not as a public `/media`. This is the simplest option, with zero extra containers, and is enough for this volume.
  - **If Devoli has S3-compatible storage** (AWS S3, a NetApp/Ceph/StorageGRID endpoint, and so on), switch through `django-storages` `S3Storage` with `endpoint_url`. It's a configuration-only change, provided code only uses `default_storage`.
  - **MinIO:** the `minio/minio` GitHub repo is **archived**, and its README says "THIS REPOSITORY IS NO LONGER MAINTAINED", pointing to the commercial "AIStor" editions. [minio/minio](https://github.com/minio/minio). Don't add it to a production stack. If a self-hosted S3 endpoint becomes necessary later, evaluate a maintained alternative. I haven't researched those here.

---

## 5. Recommended approach

1. **MVP (PIR evidence, use c):**
   - Build the `EvidenceSource` protocol and the `EvidenceSpec` / `EvidenceItem` models, with Celery snapshots when the PIR is created.
   - Implement `LibreNmsEvidenceSource` (from #7) and `GrafanaEvidenceSource.render` first, if Devoli confirms the renderer is available.
   - Then `KentikEvidenceSource.render` + `.data`, via the V5 Query API.
   - Store everything with `FileSystemStorage` on a backed-up volume.
2. **Incident live context (use a):** implement `live_url` for all three sources. Grafana and LibreNMS links are free. Kentik `query/url` is generated once per spec and target and cached.
3. **Later:**
   - Grafana `data` via `/api/ds/query`, to add summary numbers.
   - Kentik custom-dimension support for Customers.
   - Maintenance before/after health checks (use b), which reuse the same specs with two windows.
   - Kentik Synthetics.
4. **Config UI:** in the main app, a Connections page (Grafana URL and token, Kentik cluster, email and token, all encrypted) and an Evidence Specs page with pickers (Grafana search → panel → variable mapping, or pasting Kentik query JSON), each with a "test render" against a sample Network Element and time range.

## 6. Open questions for Devoli

1. **Grafana:** which version, and where does it run: self-hosted (RHEL/docker?), Grafana Cloud, or Azure/AWS managed? Is it reachable from the Outage Manager host?
2. **Is the Grafana Image Renderer installed**, and is it the service or the deprecated plugin? If not, can Devoli run `grafana/grafana-image-renderer` (16 GiB and 4 cores recommended) next to Grafana?
3. **Which dashboards and panels matter** for Incidents and PIRs (core/backbone, interface utilisation, BNG/sessions, customer circuits)? What template variables do they use (device, interface), and do those values match Boris or LibreNMS names?
4. Can we have a **Viewer service account** (and folder permissions if needed)? Does Devoli set a token expiry policy (`token_expiration_day_limit`)?
5. **Kentik:** which **cluster (US `api.kentik.com` or EU `api.kentik.eu`)**, and which plan? How long is Full vs Fast dataseries **retention**?
6. Can we have a dedicated **Kentik Member user** for API access? Are other integrations already using the shared Query API limits (1500/hour)?
7. How are **Customers** represented in Kentik: custom dimension, saved filters, interface description convention, or not at all?
8. Do Kentik `device_name`s and interface names match LibreNMS `sysName`/`ifName` and Boris? Are any overridden manually in Kentik?
9. Has Kentik said anything about a **V6 replacement for the V5 Query API**, or a deprecation date?
10. Is there an existing **S3-compatible object store** at Devoli, or is a backed-up host volume acceptable? What are the backup and retention requirements for PIR evidence?
11. Does Devoli use **Kentik Synthetics**, and would latency/loss evidence be wanted?

## Sources

- Grafana: [Set up image rendering](https://grafana.com/docs/grafana/latest/setup-grafana/image-rendering/), [Configure Grafana [rendering]](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/#rendering), [Share dashboards and panels](https://grafana.com/docs/grafana/latest/dashboards/share-dashboards-panels/), [Externally shared dashboards](https://grafana.com/docs/grafana/latest/dashboards/share-dashboards-panels/shared-dashboards/), [Snapshot API](https://grafana.com/docs/grafana/latest/developers/http_api/snapshot/), [Search API](https://grafana.com/docs/grafana/latest/developers/http_api/folder_dashboard_search/), [Dashboard API (latest)](https://grafana.com/docs/grafana/latest/developers/http_api/dashboard/), [Dashboard API (v11.6)](https://grafana.com/docs/grafana/v11.6/developers/http_api/dashboard/), [Data source API](https://grafana.com/docs/grafana/latest/developers/http_api/data_source/), [Service accounts](https://grafana.com/docs/grafana/latest/administration/service-accounts/), [Dashboard links](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/manage-dashboard-links/), [Dashboard URL variables](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/create-dashboard-url-variables/), [grafana-image-renderer releases](https://github.com/grafana/grafana-image-renderer/releases), [Dockerfile](https://github.com/grafana/grafana-image-renderer/blob/main/Dockerfile)
- Kentik: [Kentik APIs (auth, clusters, rate limits, V5/V6)](https://kb.kentik.com/docs/apis-overview), [V5 Query API](https://kb.kentik.com/docs/query-api), [Resolution overview](https://kb.kentik.com/docs/resolution-overview), [Data Explorer](https://kb.kentik.com/docs/data-explorer), [General dimensions](https://kb.kentik.com/docs/general-dimensions), [Custom dimensions](https://kb.kentik.com/docs/custom-dimensions), [Network assets APIs (devices, interfaces)](https://kb.kentik.com/docs/network-assets-apis), [Admin APIs](https://kb.kentik.com/docs/admin-apis), [Interface APIs (V6 alpha)](https://kb.kentik.com/docs/interface-apis), [Interface classification](https://kb.kentik.com/docs/interface-classification), [Portal sharing and export](https://kb.kentik.com/docs/portal-sharing-and-export), [Synthetics Monitoring APIs](https://kb.kentik.com/docs/synthetics-monitoring-apis), [api-schema-public](https://github.com/kentik/api-schema-public)
- Storage: [minio/minio (archived)](https://github.com/minio/minio), [Django STORAGES](https://docs.djangoproject.com/en/5.2/ref/settings/#storages), [django-storages S3](https://django-storages.readthedocs.io/en/latest/backends/amazon-S3.html)
