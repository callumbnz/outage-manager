# LibreNMS / Observium APIs for alerts, status and graphs

**Ticket:** [#7](https://github.com/callumbnz/outage-manager/issues/7) · **Map:** [#1](https://github.com/callumbnz/outage-manager/issues/1) · **Researched:** 2026-09-28

**Question:** What do the LibreNMS and Observium APIs expose for alerts, device/port status and graph images, and how would a pluggable NMS adapter abstract both?

**Settled context:** the NMS is a pluggable adapter. LibreNMS comes first, and Observium is added only if Devoli keeps it. The NMS serves three uses:

- (a) live **Incident** context
- (b) before/after health checks for a **Maintenance** window
- (c) **PIR** evidence

The MVP is (c) plus links for (a).

Sources are primary only: the LibreNMS docs, which are generated from `doc/` in the [librenms repo](https://github.com/librenms/librenms), plus the LibreNMS source (`routes/api.php`, `includes/html/api_functions.inc.php`, `LibreNMS/Alert/Transport/Pagerduty.php`), and [docs.observium.org](https://docs.observium.org/). The latest LibreNMS release when this was researched was **26.9.1.1** (2026-09-22). LibreNMS is a monthly rolling release, so Devoli's installed version decides what is available. This matters most for token format and RBAC (see §1).

---

## TL;DR

| Need | LibreNMS (GPL, all features free) | Observium |
|---|---|---|
| REST API exists? | Yes, `/api/v0` | **Subscription Edition only.** Community has **no REST API** |
| Auth | `Authorization: Bearer <token>` (v0 also accepts `X-Auth-Token`) | Bearer token, or HTTP basic auth |
| Read-only credential | A user with the **`global-read`** role, or a custom role with only `viewAny` permissions. The token inherits the user's permissions | The token option **"read only"** allows GET only. Tokens can also be limited to source IPs |
| Active alerts | `GET /alerts?state=1` | `GET /alerts/?status=failed&expand_entities=1` |
| Alert history | `GET /logs/alertlog/:host?from=&to=` | `GET /alert_log/?timestamp_from=&timestamp_to=` |
| Event timeline | `GET /logs/eventlog/:host`, `/logs/syslog/:host` | Alert log only. **No eventlog/syslog endpoint is documented** |
| Device lookup | `GET /devices?type=hostname|sysName|ipv4|display&query=` | `GET /devices/?hostname=|sysname=|ipv4_address=` |
| Port status | `GET /devices/:host/ports/:ifName`, `/ports/search/ifName,ifAlias/...` | `GET /ports/?device_id=&ifDescr=|label=` |
| Graph images | `GET /devices/:host/:graph` and `/devices/:host/ports/:ifName/:graph`, with `from`, `to`, `width`, `height`, `graph_type=png|svg`, `output=base64` | `graph.php?type=&id=|device=&from=&to=&width=&height=` with basic auth. This is **not part of `/api`**, and the docs don't say it is Subscription-only |
| Maintenance suppression | `POST /devices/:host/maintenance` (and devicegroups/locations). **Create only**: there is no list, update or delete. Needs **write** permission | Full CRUD on `/maintenance/` plus associations. Needs **level ≥ 8**, and Scheduled Maintenance itself is Subscription-only |
| PagerDuty transport | Built in. If the rendered template is valid JSON, it becomes `payload.custom_details` | The PagerDuty transport is **Subscription-only** |

**Recommendation:** build a `NmsAdapter` protocol with a LibreNMS implementation first. For PIR evidence, fetch graph PNGs **server-side** in Celery and store them as immutable attachments. Use a read-only LibreNMS account for everything except maintenance. Treat `schedule_maintenance` as an optional capability behind a separate write credential. Change the LibreNMS PagerDuty alert template to emit JSON with `device_id`, `hostname`, `sysName`, `ip` and the faulting `ifName`s, so Draft Incidents can be matched to Boris Network Elements without guessing. Observium support is only feasible if Devoli runs the **Subscription** edition.

---

## 1. Authentication and permission scoping

### LibreNMS
- Tokens are created under **Settings → API Settings → API Access** (`/api-access/`). They have an optional expiry (7/30/90 days, 1 year, custom, or never), and the format is `{id}|{secret}`. The secret is stored as a SHA-256 hash and **shown once**. [API/index](https://docs.librenms.org/API/) ([src](https://github.com/librenms/librenms/blob/master/doc/API/index.md))
  - *Caveat:* the expiry and the hashed `id|secret` format are recent. Older installs have plain tokens with no expiry. Check Devoli's version.
- Header: `Authorization: Bearer <token>`. `X-Auth-Token: <token>` is kept for backwards compatibility on `/api/v0` only. **Use Bearer.**
- **Scoping:** a token acts as its user. LibreNMS uses RBAC. [Extensions/Authorization](https://docs.librenms.org/Extensions/Authorization/)
  - `global-read` is the built-in role that "is assigned all `viewAny` permissions". It is read-only and sees all devices.
  - Custom roles can hold any combination of functional permissions, for example only `device.viewAny`, `port.viewAny`, `alert.viewAny` and `eventlog.viewAny`.
  - Every route is guarded by Laravel policies. For example, `GET alerts` needs `can:viewAny,Alert`, `PUT alerts/{id}` (ack) needs `can:update,Alert`, and `POST devices/{hostname}/maintenance` needs `can:update,Device`. [routes/api.php](https://github.com/librenms/librenms/blob/master/routes/api.php)
- **Proposal:** use two service accounts.
  1. `outage-manager-ro` with the `global-read` role, or a custom role with the minimum `viewAny` permissions. This covers all reads and graphs.
  2. `outage-manager-maint`, optional, with a custom role that has `device.update` (and `device-group.update` if we use group maintenance). It is only needed if we automate §6. `device.update` also allows renaming and editing devices, so treat this credential as sensitive.

### Observium
- **The REST API is a Subscription Edition feature:** "This is a feature which is currently only included in the Subscription Edition of Observium." It must also be enabled with `$config['api']['enable'] = TRUE;`. [Observium API](https://docs.observium.org/api/)
- **Auth:** API tokens (Bearer, recommended), HTTP basic auth against any Observium user (MySQL, LDAP or RADIUS backend), or a web session. `X-API-Token` and `?api_token=` are disabled by default.
- **Token options:**
  - Access is **read/write** or **read only**. Read-only tokens may only use GET, and anything else returns 403.
  - **Allowed IPs** is a CIDR list.
  - Expiry ranges from 30 days to never, and the site can enforce a maximum.
  - A token inherits the user's permission level.
  - The secret is shown once and stored as SHA-256.
  - Failed auth is throttled: 10 failures in 300 s returns 429.
- Writes need these user levels: device update needs level 7 or device write, device delete needs 9, and **maintenance writes need ≥ 8**.
- An OpenAPI spec is served to authenticated clients at `/api/v0/openapi.json`.
- The **Graph API** (`graph.php`) documents **HTTP basic auth only**. [Observium Graph API](https://docs.observium.org/graph_api/)

---

## 2. Alerts

### LibreNMS ([API/Alerts](https://docs.librenms.org/API/Alerts/))
- `GET /api/v0/alerts`
  - `state` filters by state: 0 = ok, 1 = alert, 2 = ack. It accepts a comma-separated list and **defaults to `1`**.
  - `severity` filters by `ok`, `warning` or `critical`.
  - `alert_rule` filters by rule ID.
  - `order` sets the sort, for example `timestamp ASC`.
- `GET /api/v0/alerts/:id` returns one alert.
- **Fields** come from the source query `SELECT D.hostname, A.*, R.severity, R.name, R.proc, R.notes FROM alerts A, devices D, alert_rules R` ([api_functions.inc.php `list_alerts`](https://github.com/librenms/librenms/blob/master/includes/html/api_functions.inc.php)):
  - `id`, `device_id`, `hostname`, `rule_id`, `state`, `alerted`, `open`, `timestamp`, plus the rule's `severity`, `name`, `proc` (procedure URL) and `notes`.
  - Alerts are **per device and rule**. They carry no `port_id`. Port detail lives in the alert's *faults*, which are available in the alertlog `details` (below) and in templates.
- `PUT /api/v0/alerts/:id` acknowledges an alert (body: `note`, `until_clear`). `PUT /alerts/unmute/:id` unmutes one. Both need `alert.update`. **Outage Manager should not ack**, in line with the #6 decision not to drive ack/resolve from Outage Manager.
- Rules: `GET /rules` and `GET /rules/:id` are read-only. The API also supports add, edit and delete.
- Templates: `GET /alert_templates` and `/alert_templates/:id` return the template body, which is useful for checking the PagerDuty template (§7).

### Observium ([API](https://docs.observium.org/api/), Subscription only)
- `GET /api/v0/alerts/` and `/alerts/<id>` accept these filters:
  - `device_id`
  - `status`: `failed`, `ok`, `delayed`, `suppressed` or `all`
  - `entity_type` and `entity_id`: alerts are **per entity** (device, port, sensor, …) and **per alert checker**
  - `alert_test_id`
- `expand_entities=1` adds `hostname`, `entity_name`, `entity_shortname` and `entity_descr`. `cache_entities=1` embeds the full device and entity rows.
- `GET /alert_checks/` lists the checkers.
- The only write is `PUT /alerts/<id>` with `ignore_until_ok` or `ignore_until`, which works like an ack. We won't use it.

---

## 3. Device and port status, and matching to Boris

### LibreNMS ([API/Devices](https://docs.librenms.org/API/Devices/), [API/Ports](https://docs.librenms.org/API/Ports/))
- **Lookup:** `GET /devices?type=<field>&query=<value>`.
  - `type` can be `hostname`, `sysName`, `display`, `ipv4`, `ipv6`, `mac`, `serial`, `location`, `device_id`, or the status filters `up`, `down`, `active`, `ignored` and `disabled`.
  - The hostname, sysName and display searches are LIKE-style "search by" matches. **After fetching, the adapter must pick the exact match.**
- `GET /devices/:hostname` accepts a hostname **or a numeric `device_id`**. Most per-device routes accept either.
- **Device status:**
  - The device record carries `status` (up/down), `status_reason`, `uptime`, `last_polled`, `disabled`, `ignore`, `sysName`, `hardware`, `os`, `version`, `location` and so on.
  - `GET /devices/:host/availability` returns percentages over 1 day, 7 days, 30 days and 1 year.
  - **`GET /devices/:host/outages`** returns `going_down`/`up_again` epoch pairs. This is very useful for the PIR timeline, which is use (c).
  - `GET /devices/:host/maintenance` returns `is_under_maintenance`.
- **Ports:**
  - `GET /devices/:host/ports?columns=port_id,ifName,ifAlias,ifOperStatus,ifAdminStatus,ifSpeed,ifLastChange`
  - `GET /devices/:host/ports/:ifName`, with the ifName URL-encoded, for example `Gi0%2F1%2F0`
  - `GET /ports/:port_id`
  - `GET /ports/search/ifName,ifAlias,ifDescr/<term>`: `ifAlias` usually carries the circuit or customer description, which is a useful Boris cross-check.
  - Port rows include `ifOperStatus`, `ifOperStatus_prev`, `ifAdminStatus`, `ifLastChange`, rates and errors.
- Other useful data: `/devices/:host/transceivers` and `/resources/sensors` (optics dBm and thresholds), `/devices/:host/links` (LLDP/CDP neighbours), `/devicegroups`, and `/routing/*` (BGP sessions: `GET /bgp?hostname=`).

### Observium
- `GET /devices/` supports `hostname`, `sysname`, `device`, `ipv4_address`, `ipv6_address`, `location`, `group`, `status`, `fields=` and more. `GET /devices/<hostname>/` also works.
- `GET /ports/` supports `device_id`, `hostname`, `ifDescr`, `ifAlias`, `label`/`port_label`, `state`, `errors`, `alerted` and `fields=`.
- There is also `/devices/<id>/entities`, `/neighbours/`, `/sensors/` and `/address/`.

### Matching Boris Network Elements (see #3)
Boris (Nautobot) stores the device `name` and the interface `name`. LibreNMS identifies a device by `hostname`, which is the DNS name or IP used for polling, and it also stores the SNMP `sysName`.

- **Pick one canonical join key** and persist the mapping. Store `nms_device_id` on the Boris device, either as a Nautobot custom field or in an Outage Manager mapping table, so we don't fuzzy-match every time.
- The lookup order is `hostname` exact → `sysName` exact → primary IP (`type=ipv4`) → `display`.
- Interfaces match on `ifName`, falling back to `ifDescr` (LibreNMS supports `?ifDescr=true` on port graph routes).
- Naming consistency between Boris and the NMS is an **open question** (see §10).

---

## 4. Graph images (PIR evidence)

### LibreNMS
- **Device:** `GET /api/v0/devices/:host/graphs` lists the available graph names, for example `device_ping_perf`, `device_poller_perf`, `uptime`, `device_bits`, `device_processor` and `device_mempool`. Then `GET /api/v0/devices/:host/:graph` fetches one.
- **Port:** `GET /api/v0/devices/:host/ports/:ifName/:graph`. The graph is one of `port_bits`, `port_upkts`, `port_errors` and so on.
- **Health/wireless:** `GET /devices/:host/graphs/health/:type(/:sensor_id)`, for example optics dBm. Wireless has the same form.
- **Parameters** ([api_functions.inc.php `api_get_graph`](https://github.com/librenms/librenms/blob/master/includes/html/api_functions.inc.php)):
  - `from` and `to` take RRDtool time: epoch seconds or AT-style such as `-1d` or `now`.
  - `width` defaults to 1075 and `height` to 300.
  - `graph_type` is `png` or `svg`.
  - `output=base64` returns JSON `{image, content-type}`. Otherwise the response is raw bytes with the correct `Content-Type`.
  - The code also passes through `legend`, `title`, `graph_title`, `absolute`, `nototal`, `nodetails`, `inverse`, `previous` and `duration`.
- **Server-side fetch: yes.** It is a plain authenticated GET that returns image bytes. A Celery task can call it with the read-only token and store the PNG, with its SHA-256, request URL, time range and fetch time, as an immutable **PIR** attachment.
  - **Snapshot at PIR creation, not on render.** RRD data is consolidated over time (RRAs), so a graph fetched months later shows lower-resolution data, and the NMS may have been pruned or replaced by then.
  - PNG is the safe format for PDF and Status.io embedding. SVG is sharper but may reference fonts.
- *Timezone:* `from`/`to` are epoch values, so always send epoch UTC from Outage Manager.

### Observium
- Graphs come from `GET https://observium/graph.php?type=<entity_graph>&id=<entity_id>|device=<device_id>&from=&to=&width=&height=&legend=no&title=no`, using HTTP basic auth.
  - Graph types include `port_bits`, `device_bits`, `storage_usage` and `multiport_bits` (with `id=a,b,c`).
  - `period=<seconds>` gives a range ending now.
  - The page is PNG only. No format parameter is documented. [Graph API](https://docs.observium.org/graph_api/)
- Server-side fetch is possible with basic auth only, so this needs a dedicated low-privilege Observium user.
- The docs don't say this page is Subscription-only, but in Community we couldn't resolve port or device IDs without the REST API, so we would need a manual mapping.

---

## 5. Eventlog, syslog and alertlog for the timeline

### LibreNMS ([API/Logs](https://docs.librenms.org/API/Logs/))
- The routes are:
  - `GET /api/v0/logs/eventlog/:hostname?`: device events such as interface up/down, reboots and config changes
  - `GET /api/v0/logs/syslog/:hostname?`: only if Devoli sends syslog to LibreNMS
  - `GET /api/v0/logs/alertlog/:hostname?`
  - `GET /api/v0/logs/authlog`
- **Parameters:**
  - `from` and `to` take a datetime string (`YYYY-mm-dd HH:MM:SS`) *or* a numeric event ID, which gives cursor-style paging.
  - `start` is a row offset and `limit` defaults to 50.
  - `sortorder=DESC` reverses the order.
  - The response includes `total`.
- **Timestamp columns:** eventlog uses `datetime`, syslog uses `timestamp` and alertlog uses `time_logged`. The comparisons are against **server-local time**, so the adapter must convert from Outage Manager UTC using the LibreNMS server timezone. That timezone is an open question.
- Each row includes `hostname`, `sysName`, `device_id`, `message`, `type`, `severity` and `reference`. **Alertlog rows include decompressed `details`** (the faults: rule, entities and port fields), which is the richest source of port-level alert history.
- There is **no multi-device filter**: it's one hostname per call, or all devices filtered only by time. For a PIR, loop over the Incident's Network Elements.
- `POST /devices/:host/eventlog` (with `text`, `severity` and `type`) can write a marker such as "Maintenance MNT-123 started". This is optional and needs `device.update`.

### Observium
- `GET /alert_log/` accepts `device_id`, `entity_type`/`entity_id`, `log_type` (`FAIL`, `OK`, `ALERT_NOTIFY`, `RECOVER_NOTIFY`), `timestamp_from`/`timestamp_to` and `expand_entities`. It always paginates, with a maximum of 50,000 rows.
- **No eventlog or syslog endpoint is documented.** This is a gap. The adapter returns alert-log entries only.

---

## 6. Maintenance / scheduled downtime (suppressing alerts during a Maintenance)

### LibreNMS ([Alerting/Scheduled-Maintenances](https://docs.librenms.org/Alerting/Scheduled-Maintenances/))
- **Behaviours** are 1 = Skip alerts (the default: no new alerts and no recoveries), 2 = Mute (alerts are evaluated but no transport is sent) and 3 = Run (cosmetic only). The default is set by `alert.scheduled_maintenance_default_behavior`.
- **The API can create one.** The three routes are:
  - `POST /api/v0/devices/:host/maintenance`
  - `POST /api/v0/devicegroups/:name/maintenance`
  - `POST /api/v0/locations/:location/maintenance`

  The body is `{title, notes, behavior, start: "Y-m-d H:i:00", duration: "H:i"}`, and `start` defaults to now. The source creates a non-recurring `AlertSchedule` and attaches the device, group or location. It needs `can:update` on Device, DeviceGroup or Location.
- **Gaps:**
  - The API has **no endpoint to list, extend, end early or delete** a maintenance (confirmed from [routes/api.php](https://github.com/librenms/librenms/blob/master/routes/api.php)). The only read is `GET /devices/:host/maintenance` → `is_under_maintenance`.
  - The API response doesn't return a schedule ID. It returns only a message string.
  - `start` is **server-local time**.
  - Each call creates **one schedule per device**, so a Maintenance with 20 devices creates 20 schedules. The alternative is one call against a device group, which needs a group to exist.
  - Because the window can't be ended early or extended via API, Outage Manager should create windows at exact start/duration and tell engineers to remove or extend them in the LibreNMS UI.
- **Recommendation:** make this **opt-in per Maintenance** ("Suppress LibreNMS alerts for these devices"). Default the behaviour to **Mute (2)** rather than Skip, so LibreNMS still records state for the before/after health check (use b) and the PIR timeline while PagerDuty isn't paged. Record the API response in the Maintenance audit log. Keep it out of the MVP.

### Observium (Subscription only)
- It offers full CRUD:
  - `GET /maintenance/?active=1|upcoming=1|ended=1`
  - `POST /maintenance/` with `{maint_name, maint_descr, maint_global, maint_start|maint_time_from, maint_end|maint_time_to}`
  - `PUT` and `DELETE /maintenance/<id>/`
  - `POST`/`DELETE /maintenance/<id>/associations/` with `{entity_type: device|group|alert_checker, entity_ids: [...]}`. These calls are idempotent.
- It needs **user level ≥ 8**. Scheduled Maintenance itself is a Subscription feature. [Scheduled Maintenance](https://docs.observium.org/scheduled_maintenance/)
- Observium's model is a better fit (one window with many associations, which can be edited and deleted). LibreNMS is the weaker implementation of this capability.

---

## 7. LibreNMS PagerDuty transport and alert template

Sources: [Transport source](https://github.com/librenms/librenms/blob/master/LibreNMS/Alert/Transport/Pagerduty.php), [Transports/PagerDuty doc](https://github.com/librenms/librenms/blob/master/doc/Alerting/Transports/PagerDuty.md) and [Alerting/Templates](https://docs.librenms.org/Alerting/Templates/).

- **Fixed mappings (Events API v2):**
  - `routing_key` is the integration key.
  - `event_action` is `trigger`, `acknowledge` or `resolve`, following the alert state.
  - `dedup_key` is `alert_id`.
  - `payload.source` is `hostname` and `payload.summary` is the alert title, or `"<rule> on <hostname>"`.
  - `payload.severity`, `payload.class` (the device type) and `payload.group` (the device groups) are also set.
- **`custom_details`:** if the rendered alert template **is valid JSON**, the decoded object is used as `custom_details` as-is. Otherwise the text is stripped of tags and split into a `message` line array. So Devoli can make the payload structured just by assigning a JSON template to the PagerDuty transport.
- **Proposed template** (Blade; the available variables are listed in the Templates doc: `$alert->device_id`, `hostname`, `sysName`, `display`, `ip`, `location`, `rule`, `name`, `severity`, `state`, `timestamp`, `id`/`uid`, `faults[]`, `proc`):

```blade
{!! json_encode([
  'nms'        => 'librenms',
  'nms_url'    => config('app.url'),
  'alert_id'   => $alert->id,
  'rule'       => $alert->name,
  'severity'   => $alert->severity,
  'state'      => $alert->state,
  'timestamp'  => $alert->timestamp,
  'device_id'  => $alert->device_id,
  'hostname'   => $alert->hostname,
  'sysName'    => $alert->sysName,
  'display'    => $alert->display,
  'ip'         => $alert->ip,
  'location'   => $alert->location,
  'ports'      => collect($alert->faults ?? [])->map(fn ($f) => [
                     'port_id' => $f['port_id'] ?? null,
                     'ifName'  => $f['ifName'] ?? null,
                     'ifAlias' => $f['ifAlias'] ?? null,
                  ])->filter(fn ($p) => $p['ifName'])->values(),
  'procedure'  => $alert->proc,
]) !!}
```

  - Test the template in LibreNMS (Alerts → Alert Templates → test) before rolling it out.
  - `faults` is only present when `state != 0`. Recovery events will carry an empty `ports`.
  - `config('app.url')` depends on how Devoli's LibreNMS is configured. It can be hard-coded instead.
- Outage Manager then reads these fields from `GET /incidents/{id}/alerts` → `body.cef_details.details` / `custom_details` (see #6). It uses `device_id` for direct NMS calls and `hostname`/`sysName` plus `ifName` to match Boris.
- Observium's PagerDuty transport is **Subscription-only**, and its payload format is set by PagerDuty's Observium integration. It is not configurable in the same way.

---

## 8. Proposed adapter interface

These are Python signatures for a Django service module. Every adapter method is read-only except `schedule_maintenance`. Each adapter reports its capabilities so the UI can hide unsupported features.

```python
class NmsCapability(StrEnum):
    ALERTS = "alerts"; ALERT_LOG = "alert_log"; EVENT_LOG = "event_log"; SYSLOG = "syslog"
    GRAPHS = "graphs"; PORT_STATUS = "port_status"; OUTAGES = "outages"
    MAINTENANCE_CREATE = "maintenance_create"; MAINTENANCE_MANAGE = "maintenance_manage"

@dataclass(frozen=True)
class NmsDevice:  nms_id: str; hostname: str; sys_name: str | None; ip: str | None
                  status: Literal["up","down","unknown"]; url: str; raw: dict
@dataclass(frozen=True)
class NmsPort:    nms_id: str; device_nms_id: str; if_name: str; if_alias: str | None
                  oper_status: str; admin_status: str; url: str; raw: dict
@dataclass(frozen=True)
class NmsAlert:   nms_id: str; device_nms_id: str; entity: str | None; rule: str; severity: str
                  state: Literal["active","acked","ok"]; started_at: datetime; url: str; raw: dict
@dataclass(frozen=True)
class NmsEvent:   at: datetime; device_nms_id: str; kind: str; message: str; severity: str | None; raw: dict
@dataclass(frozen=True)
class NmsGraph:   content: bytes; content_type: str; source_url: str; start: datetime; end: datetime

class NmsAdapter(Protocol):
    capabilities: frozenset[NmsCapability]
    def find_device(self, *, hostname=None, sys_name=None, ip=None) -> NmsDevice | None: ...
    def get_ports(self, device: NmsDevice, if_names: list[str] | None = None) -> list[NmsPort]: ...
    def active_alerts(self, devices: list[NmsDevice]) -> list[NmsAlert]: ...
    def event_log(self, devices, start: datetime, end: datetime, kinds=("event","alert")) -> list[NmsEvent]: ...
    def outages(self, device: NmsDevice, start, end) -> list[tuple[datetime, datetime | None]]: ...
    def available_graphs(self, target: NmsDevice | NmsPort) -> list[str]: ...
    def graph(self, target: NmsDevice | NmsPort, kind: str, start, end, *, width=1075, height=300) -> NmsGraph: ...
    def schedule_maintenance(self, devices, start, end, *, title, notes) -> str | None: ...  # optional
    def device_url(self, device) -> str: ...   # deep links for Incident view (MVP use (a))
```

| Method | LibreNMS | Observium (Subscription) | Gaps / notes |
|---|---|---|---|
| `find_device` | `GET /devices?type=hostname|sysName|ipv4&query=` then an exact match. Cache `device_id` | `GET /devices/?hostname=|sysname=|ipv4_address=` | Both are LIKE-style. Persist the mapping to the Boris device |
| `get_ports` | `GET /devices/:id/ports?columns=…` or `/devices/:id/ports/:ifName` | `GET /ports/?device_id=&fields=…` | LibreNMS needs URL-encoded `ifName` |
| `active_alerts` | `GET /alerts?state=1,2`, filtered by `device_id` on our side | `GET /alerts/?device_id=&status=failed&expand_entities=1` | The LibreNMS API has no device filter (it returns all active alerts, which is fine at Devoli's scale). LibreNMS alerts are per device and rule, with ports only in faults (alertlog `details`). Observium alerts are per entity |
| `event_log` | `/logs/eventlog/:id`, `/logs/alertlog/:id` and optionally `/logs/syslog/:id` with `from`/`to` (server-local time), paged | `/alert_log/?device_id=&timestamp_from=&timestamp_to=` | **Observium has no eventlog or syslog.** LibreNMS takes one device per call |
| `outages` | `GET /devices/:id/outages`, filtered by time on our side | *Not available.* Derive it from the alert log (device status checker) | |
| `available_graphs` | `GET /devices/:id/graphs`. For ports, use a fixed list (`port_bits`, `port_upkts`, `port_errors`) | Fixed list, as the docs have no discovery endpoint | |
| `graph` | `GET /devices/:id/:kind` or `/devices/:id/ports/:ifName/:kind` with `from`/`to` epoch, `graph_type=png` | `graph.php?type=&id=|device=&from=&to=`, with basic auth | Observium's graph page uses a separate credential and basic auth only |
| `schedule_maintenance` | `POST /devices/:id/maintenance` per device (or per device group), `behavior=2`. Returns no ID | `POST /maintenance/` + `POST /maintenance/{id}/associations`. Returns an ID, and can later be updated or deleted | LibreNMS is **create-only** and needs a write credential. Observium needs level ≥ 8 |
| `device_url` | `{base}/device/device={id}/` | `{base}/device/device={id}/` | Both are UI deep links (URL pattern to be confirmed on Devoli's instance) |

**Implementation notes:**
- Use one `NmsConnection` config model in the main app, not Django admin: base URL, adapter type, token, optional maintenance token, TLS verify/CA bundle, server timezone and timeout. Store secrets encrypted.
- All calls run in Celery with timeouts and retries.
- Graph and event snapshots are taken when a PIR is created, plus a manual "refresh evidence" action. Each snapshot is stored with its source URL and fetch time.
- `raw` payloads are kept for audit.
- Build the Observium adapter only after the edition is confirmed.

---

## 9. Recommended approach

1. **MVP (use c + links for a):** implement `LibreNmsAdapter` with `find_device`, `graph`, `event_log` (eventlog + alertlog), `outages` and `device_url`. On PIR creation, Celery snapshots these for each involved Network Element over `[incident.start − 1 h, incident.end + 1 h]`:
   - device `device_ping_perf` graphs
   - `port_bits` and `port_errors` for involved interfaces
   - eventlog and alertlog entries
   - outages
2. **Devoli action (no code):** assign the JSON alert template (§7) to the LibreNMS PagerDuty transport, so Draft Incidents carry `device_id`, `hostname`, `sysName` and `ports[].ifName`.
3. **Phase 2 (use a):** show live `active_alerts`, device status and port status on the Incident page, polled or on demand, with a short cache.
4. **Phase 3 (use b):**
   - Before/after health snapshots for a Maintenance: port oper status, alerts, and the ping graph at start and end.
   - Opt-in `schedule_maintenance` with Mute behaviour, using the separate write token.
5. **Observium:** implement it only if Devoli keeps Observium **and** it is the Subscription edition. The Community edition has no REST API, so it would be limited to deep links and, at best, `graph.php` images with manually mapped IDs.

---

## 10. Open questions for Devoli

1. **LibreNMS version** and how it's updated (monthly, or pinned)? This affects the token format and expiry, and the RBAC roles (older versions use user levels rather than roles).
2. **Observium:** is it staying? If so, which **edition** (Community, Professional or Enterprise)? Is `$config['api']['enable']` on? What role does it have next to LibreNMS?
3. **Device naming:** is the LibreNMS `hostname` an FQDN, an IP or a short name? Does it equal the Boris device `name`? Is `sysName` consistent? Are interface names (`ifName`) identical to the Boris interface names? Could Boris store the LibreNMS `device_id` as a custom field?
4. **Network reachability:** can the Outage Manager containers reach the LibreNMS and Observium web/API over HTTPS? Is TLS signed by an internal CA? Which **timezone** is the LibreNMS server/PHP set to (log and maintenance `start` values are local time)?
5. **Accounts:** may we create a `global-read` service account and token, with expiry and a rotation policy? Is a separate write-capable account for maintenance acceptable?
6. **PagerDuty:** which LibreNMS alert template does the PagerDuty transport use today? Can we switch it to the JSON template? Are there several transports (per service or team)?
7. **Syslog:** do devices send syslog to LibreNMS? If not, the syslog endpoint is empty.
8. **Maintenance suppression:** do engineers use LibreNMS scheduled maintenance today? Which behaviour (Skip or Mute)? Do they want Outage Manager to create these windows?
9. **Retention:** how long do eventlog, syslog and alertlog, and RRD at full resolution, live in LibreNMS? This decides how late a PIR snapshot can still be accurate.

---

## Sources

- LibreNMS API index/auth: https://docs.librenms.org/API/ · https://github.com/librenms/librenms/blob/master/doc/API/index.md
- LibreNMS Alerts API: https://docs.librenms.org/API/Alerts/
- LibreNMS Devices API (graphs, ports, maintenance, outages): https://docs.librenms.org/API/Devices/
- LibreNMS Ports API: https://docs.librenms.org/API/Ports/
- LibreNMS Logs API: https://docs.librenms.org/API/Logs/
- LibreNMS DeviceGroups API (group maintenance): https://docs.librenms.org/API/DeviceGroups/
- LibreNMS Scheduled Maintenances: https://docs.librenms.org/Alerting/Scheduled-Maintenances/
- LibreNMS Templates: https://docs.librenms.org/Alerting/Templates/
- LibreNMS PagerDuty transport doc: https://github.com/librenms/librenms/blob/master/doc/Alerting/Transports/PagerDuty.md
- LibreNMS PagerDuty transport source: https://github.com/librenms/librenms/blob/master/LibreNMS/Alert/Transport/Pagerduty.php
- LibreNMS Authorization (RBAC, global-read): https://docs.librenms.org/Extensions/Authorization/
- LibreNMS API routes (permissions, no maintenance list/delete): https://github.com/librenms/librenms/blob/master/routes/api.php
- LibreNMS API implementation (`list_alerts`, `list_logs`, `api_get_graph`, `maintenance_device`): https://github.com/librenms/librenms/blob/master/includes/html/api_functions.inc.php
- LibreNMS releases: https://github.com/librenms/librenms/releases
- Observium API (Subscription feature, tokens, endpoints, maintenance): https://docs.observium.org/api/
- Observium Graph API: https://docs.observium.org/graph_api/
- Observium Scheduled Maintenance (Subscription feature): https://docs.observium.org/scheduled_maintenance/
- Observium Alerting Transports (PagerDuty = Subscription): https://docs.observium.org/alerting_transports/
- Observium CE→Subscription migration: https://docs.observium.org/ce-migration/
