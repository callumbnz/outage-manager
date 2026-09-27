# PagerDuty v3 webhooks for Draft Incidents

Ticket: [#6](https://github.com/callumbnz/outage-manager/issues/6) · Map: [#1](https://github.com/callumbnz/outage-manager/issues/1)
Researched: 2026-09-28

**Question.** What do PagerDuty v3 webhooks provide (event types, payload, signature verification)? What data can pre-fill a **Draft Incident**, and how do we de-duplicate?

**Settled context.** PagerDuty stays as the alerting and on-call tool. A PagerDuty webhook creates a **Draft Incident**, and an engineer confirms it. Engineers can also declare Incidents manually. PIRs want an Incident timeline (who was paged or acknowledged, and when) as evidence.

## Sources (primary only)

| Ref | Source |
|---|---|
| [DD-OV] | Webhooks v3 Overview: https://developer.pagerduty.com/docs/webhooks-overview (source: [PagerDuty/developer-docs `docs/webhooks/01-Overview.md`](https://github.com/PagerDuty/developer-docs/blob/main/docs/webhooks/01-Overview.md)) |
| [DD-BEH] | Webhook Behavior: https://developer.pagerduty.com/docs/webhook-behavior (`docs/webhooks/02-Behavior.md`) |
| [DD-SIG] | Verifying Signatures: https://developer.pagerduty.com/docs/verifying-webhook-signatures (`docs/webhooks/04-Signatures.md`) |
| [DD-MTLS] | Mutual TLS: https://developer.pagerduty.com/docs/mutual-tls (`docs/webhooks/03-Mutual-TLS.md`) |
| [DD-IPS] | Webhook IPs: https://developer.pagerduty.com/docs/webhook-ips, JSON lists at `https://developer.pagerduty.com/ip-safelists/webhooks-{us,eu}-service-region-json` |
| [DD-RIPS] | REST API IPs: https://developer.pagerduty.com/docs/rest-api-ips |
| [DD-AUTH] | REST API Authentication: https://developer.pagerduty.com/docs/authentication |
| [DD-RL] | REST API Rate Limits: https://developer.pagerduty.com/docs/rest-api-rate-limits |
| [DD-PRIV] | Private Apps / Scoped OAuth: https://developer.pagerduty.com/docs/private-apps |
| [DD-RID] | Retrieve Incident Details: https://developer.pagerduty.com/docs/retrieve-incident-details |
| [DD-EV2] | Events API v2, Send an Alert Event: https://developer.pagerduty.com/docs/send-alert-event |
| [OAS] | REST API OpenAPI spec (the source of the developer.pagerduty.com API reference): [PagerDuty/api-schema `reference/REST/openapiv3.json`](https://github.com/PagerDuty/api-schema/blob/main/reference/REST/openapiv3.json), commit `60bdc01` |
| [KB-WH] | Support KB, Webhooks: https://support.pagerduty.com/main/docs/webhooks |
| [KB-KEYS] | Support KB, API Access Keys: https://support.pagerduty.com/main/docs/api-access-keys |
| [KB-ALERTS] | Support KB, Alerts: https://support.pagerduty.com/main/docs/alerts |
| [KB-CBAG] | Support KB, Content-Based Alert Grouping: https://support.pagerduty.com/main/docs/content-based-alert-grouping |
| [LNMS] | LibreNMS PagerDuty transport (first-party source code): [librenms/librenms `LibreNMS/Alert/Transport/Pagerduty.php`](https://github.com/librenms/librenms/blob/master/LibreNMS/Alert/Transport/Pagerduty.php) |

## 1. Creating subscriptions and scope

- A v3 **webhook subscription** has three parts: a list of `events`, a `filter` (scope), and a `delivery_method` (an HTTP URL, with optional `custom_headers`) [DD-OV].
- **Scope** is set by `filter.type`, which is one of `service_reference`, `team_reference` or `account_reference`. `filter.id` is required except for account scope. For incident events, you only receive incidents that belong to the filtered service or team [DD-OV][OAS `WebhookSubscription`].
- **UI:** go to *Integrations → Generic Webhooks (v3)*, or open a service and use *Integrations tab → Webhooks*. Admin/Global Admin, Account Owner, Manager base role, or a Manager team role (for its own teams/services) can create them [KB-WH].
- **API:** `POST /webhook_subscriptions` needs Scoped OAuth scope `webhook_subscriptions.write`. The API also offers `GET` with `filter_type`/`filter_id`, `GET/PUT/DELETE /webhook_subscriptions/{id}`, `POST /webhook_subscriptions/{id}/enable`, and `POST /webhook_subscriptions/{id}/ping` [OAS].
- **Secret:** the signing secret is returned once, in the create response (`delivery_method.secret`, "Only provided on the initial create response") or in the UI confirmation modal [OAS][KB-WH]. We must capture it at creation time.
- **Limit:** at most 10 subscriptions per service ID, per team ID, and at account level [KB-WH].
- **Custom headers:** values are redacted on `GET`, but they are sent in clear on delivery [DD-OV]. They can carry a second shared token if we want one.
- The UI lets you **send a test event** (`pagey.ping`) [KB-WH]. The API equivalent is `/ping` [OAS].

## 2. Event types

v3 event types [DD-OV] (the KB list matches [KB-WH]). Each is listed with its `event.data.type`:

| Event type | data.type | Relevance to us |
|---|---|---|
| `incident.triggered` | incident | **Creates the Draft Incident** |
| `incident.acknowledged` / `incident.unacknowledged` | incident | Timeline |
| `incident.escalated` | incident | Timeline (escalated to another user at the same level) |
| `incident.delegated` | incident | Timeline (moved to another escalation policy) |
| `incident.reassigned` | incident | Timeline |
| `incident.priority_updated` | incident | Update the draft's priority hint |
| `incident.resolved` / `incident.reopened` | incident | Timeline, and flags drafts that resolved on their own |
| `incident.annotated` | incident_note | Timeline (note content) |
| `incident.status_update_published` | incident_status_update | Timeline |
| `incident.responder.added` / `.replied` | incident_responder | Timeline |
| `incident.service_updated` | incident | Re-map the service |
| `incident.custom_field_values.updated` | incident_field_values | Optional |
| `incident.conference_bridge.updated` | incident_conference_bridge | Optional (bridge link) |
| `incident.incident_type.changed`, `incident.role.assigned`, `incident.task.*`, `incident.action_invocation.*`, `incident.workflow.*` | various | Ignore for now |
| `service.created/updated/deleted/custom_field_values.updated` | service | Could keep a PagerDuty-service → Outage Manager mapping table in sync |

PagerDuty says it may add event types, and that "Early Access" events can change without notice [DD-OV]. Our handler must ignore unknown types.

## 3. Payload: thin, so the full record needs a REST call

- Each delivery holds exactly **one** `event` object (no batching) [DD-OV][DD-BEH]. Its fields are `id`, `event_type`, `resource_type` (`incident` or `service`), `occurred_at`, `agent` (a user reference, or `null` for automation), `client`, and `data` [DD-OV].
- For incident events, `data` is a compact incident with these fields: `id`, `number`, `html_url`, `self`, `status`, `incident_key`, `created_at`, `title`, `service` (a reference with `id`/`summary`), `assignees[]`, `escalation_policy`, `teams[]`, `priority` (e.g. `summary: "P1"`), `urgency` (`high`/`low`), `conference_bridge`, `resolve_reason`, `incident_type` [DD-OV].
- For non-incident data types (notes, responders, status updates) the incident sits at `data.incident.id` rather than `data.id` [KB-WH].
- **Not in the payload:** the alert body or `custom_details` (e.g. the LibreNMS hostname), alert count, and the log entries. Fetch them from REST:
  - `GET /incidents/{id}?include[]=first_trigger_log_entries` (other includes include `services`, `teams`, `priorities`, `acknowledgers`, `assignees`, `custom_fields`) [OAS].
  - `GET /incidents/{id}/alerts?include[]=first_trigger_log_entries` returns each alert's `alert_key` (dedup key), `severity`, `body.details`, `body.contexts` (links/images), `integration` and `service` [OAS].
  - `GET /incidents/{id}/log_entries?include[]=channels` returns the full timeline. With `channels` included, `trigger_log_entry` items carry the original event data received from the monitoring tool [DD-RID][OAS]. PagerDuty notes that the `channel` schema is not formally published and may change [DD-RID].
- Payload size: ordering and delivery are guaranteed up to 55 KB. Between 55 KB and 256 KB, details may be omitted and delivery is best-effort. Anything over 256 KB is dropped [DD-BEH]. This is another reason to treat the webhook as a signal and fetch the details.

## 4. Signature verification

- Every v3 delivery carries `X-PagerDuty-Signature: v1=<hex>[,v1=<hex>...]` [DD-SIG]. It also carries `User-Agent: PagerDuty-Webhook/V3.0` and `X-Webhook-Subscription: <subscription id>` [KB-WH].
- The signature is an HMAC-SHA256 of the **raw request body**, keyed with the subscription secret and hex (Base16) encoded. Several comma-separated signatures may appear to allow zero-downtime secret rotation, and the request is valid if **any** `v1` signature matches. Use a constant-time compare. Do not let middleware re-serialise the body, and treat it as UTF-8 [DD-SIG].
- In Django, compute over `request.body` (bytes) before any JSON parsing, and look up the secret by the `X-Webhook-Subscription` header so several subscriptions can each have their own secret.
- Optional extra layers [DD-BEH][DD-MTLS]:
  - **mTLS**: PagerDuty presents a client certificate that we verify against the DigiCert root on its certificates page, with depth 2, and we check the subject.
  - **OAuth client credentials**: PagerDuty sends a bearer token (`/webhook_subscriptions/oauth_clients`).
  - **Basic auth** embedded in the URL.
  - **IP safelist.**
- PagerDuty verifies our HTTPS certificate. It must chain to Mozilla's CA list, be presented in order, and not be self-signed [DD-MTLS].

## 5. Delivery semantics

All from [DD-BEH] unless noted:

- **Timeout:** PagerDuty expects a 2xx within **5 s**. Recommended practice is to return `202 Accepted` and process asynchronously. For us that means a Celery task.
- **Retries:** triggered by timeouts, 5xx, 429, connection failures, expired TLS certificates, and DNS errors. Retries continue **for up to 48 h**, then the webhook is dropped. While one webhook is being retried, later webhooks for the same subscription and resource ID are **queued behind it**.
- **Permanent errors (no retry):** any 4xx except 429, and most TLS errors. Signature failures must therefore return 401/403, never 5xx. After a successful write, return 2xx even if the event is ignored.
- **Auto-disable:** after **3 consecutive dropped webhooks** the subscription is disabled for 24 h and its queue is dropped. Events that happen while it is disabled are also dropped. The subscription shows as "Needs Attention" and must be re-enabled in the UI or via `POST /webhook_subscriptions/{id}/enable` (the `delivery_method.temporarily_disabled` flag) [OAS]. **Implication:** we need monitoring of this flag plus a REST reconciliation job to backfill gaps.
- **Ordering:** in order per subscription and incident. It is not guaranteed across incidents, or for oversized payloads.
- **At-least-once:** duplicates are possible. De-duplicate on the **`X-Webhook-Id`** header, which is unique per webhook and repeated on each retry. `event.id` also identifies the outbound event [DD-OV].

## 6. De-duplication model

There are three separate layers:

1. **Delivery dedup:** store `X-Webhook-Id` (and `event.id`) with a unique constraint, so a replayed delivery is a no-op.
2. **PagerDuty incident → Outage Manager Incident:** store the `pagerduty_incident_id` with a unique constraint on the Incident, or on a link table if one Incident can gather several PagerDuty incidents. `incident.triggered` for an ID we already know is a no-op. All later events update the timeline of the linked Incident.
3. **Alert noise upstream:** PagerDuty already collapses repeated events. Events that carry the same `dedup_key` become one alert while it is open, and after the alert resolves a new trigger creates a new alert [DD-EV2][KB-ALERTS]. LibreNMS sets `dedup_key = alert_id`, so a flapping LibreNMS alert stays as one PagerDuty alert while it is open [LNMS]. Alert grouping (Content-Based, Intelligent, or Time-Based) can also fold several alerts into one PagerDuty incident. Content-Based and Intelligent grouping are AIOps features, available as an add-on or in the Reliability Platform plans [KB-CBAG]. For incidents without child alerts, `incident_key` carries the dedup key [OAS `incident_key` parameter].

What this means for **Draft Incidents**:

- Create **one Draft Incident per PagerDuty incident**, not per alert. Let PagerDuty's grouping do the first pass.
- Across incidents, several PagerDuty incidents may describe one real outage (for example many devices on the same service, or repeated triggers on one service). Before creating a new Draft, check whether an **open Draft or Incident** exists for the same PagerDuty service (or the same LibreNMS `source` hostname or Boris Network Element) within a configurable window. If one does, **attach** the new PagerDuty incident to it as a linked alert and do not create another Draft. The engineer can split or merge when confirming.
- Drafts whose PagerDuty incident resolves before anyone confirms them should be flagged ("auto-resolved in PagerDuty"). They can be dismissed in bulk or kept for the record. Flap noise should not become Incidents.

## 7. Fields to pre-fill a Draft Incident

| Draft Incident field | Source |
|---|---|
| Title | `data.title` (the first alert's summary; for LibreNMS this is `title` or "`<rule>` on `<hostname>`") [DD-OV][LNMS] |
| PagerDuty link / number | `data.html_url`, `data.number`, `data.id` |
| Service (PagerDuty) | `data.service.id` / `.summary`, mapped to our Service / Network Element via a configurable mapping table |
| Start time | `data.created_at` (and `event.occurred_at`) |
| Urgency / priority hint | `data.urgency`, `data.priority.summary` (Priority is optional in PagerDuty and may be null) |
| Team / escalation policy | `data.teams[]`, `data.escalation_policy` |
| Device hostname → Network Element | Alert `body.details` (the OpenAPI spec types it only as arbitrary JSON), or the `trigger_log_entry` `channel` data, which holds the original event including `source` [OAS][DD-RID]. The exact field layout must be confirmed against a real LibreNMS alert. LibreNMS sends `payload.source = hostname`, `payload.group = device groups`, `payload.class = alert type`, `payload.severity`, and `custom_details` = the rendered alert template (JSON if the template emits JSON, otherwise `{"message": [...lines]}`) [LNMS][DD-EV2] |
| Severity | Alert `severity` (`critical`/`error`/`warning`/`info`) [OAS][DD-EV2] |
| Links / graphs | Alert `body.contexts` (links/images) [OAS] |
| Conference bridge | `data.conference_bridge` |
| Initial responders | `data.assignees[]`, plus log entries of type `notify_log_entry` and `acknowledge_log_entry` |

**Tip for LibreNMS:** make its PagerDuty alert template emit **JSON** with explicit keys (`hostname`, `sysName`, `ifName`, `device_id`, `location`, rule name). These arrive as structured `custom_details` that we can match against Boris Network Elements deterministically [LNMS].

## 8. REST API auth, rate limits, and the PIR timeline

- **Auth options** [DD-AUTH][KB-KEYS][DD-PRIV]:
  - **Account (general access) API key.** An admin creates it, it is tied to the account rather than a person, and it can be marked **Read-only** (GET only). It is sent as `Authorization: Token token=<key>`. It is the simplest option and enough for read-back.
  - **Scoped OAuth private app with an app token.** It uses the client-credentials grant against `https://identity.pagerduty.com/oauth/token` with `scope=as_account-{region}.{subdomain} incidents.read services.read ...`. Scopes are per resource type (e.g. `incidents.read`, `webhook_subscriptions.write`), which gives least privilege, the app is visible in the PagerDuty admin UI, and tokens expire. PagerDuty itself suggests this for "responding to a webhook or running a scheduled job".
  - **Recommendation:** use a Scoped OAuth private app with `incidents.read services.read teams.read users.read` (add `incidents.write` only if we adopt write-back, and `webhook_subscriptions.write` if the app manages its own subscriptions). A read-only account key is the fallback.
- **Rate limits:** 960 req/min per account API key, per user across a user's keys, or per app per account for app tokens. Responses carry `ratelimit-limit`, `ratelimit-remaining` and `ratelimit-reset` headers. On 429, back off using `ratelimit-reset`, and make requests serially per token [DD-RL]. Our volume is a handful of calls per incident, well within limits.
- **PIR timeline evidence** comes from `GET /incidents/{id}/log_entries` (with `is_overview=false`, `time_zone`, `include[]=channels`). Log entry types include `trigger`, `notify` (who was paged and how), `acknowledge`, `assign`, `escalate`, `delegate`, `annotate`, `snooze`, `unacknowledge`, `urgency_change`, `resolve`, `exhaust_escalation_path`, `repeat_escalation_path`, and `reach_trigger_limit`/`reach_ack_limit` [OAS][DD-RID]. **Recommendation:** store webhook events live as timeline rows, and snapshot the full log entry list when the Incident resolves (or when a PIR is started). Keep the raw JSON so the evidence is immutable even if PagerDuty's data changes.

## 9. Optional write-back to PagerDuty

The REST API supports:

- `POST /incidents/{id}/notes` (`{"note":{"content":...}}`, max 2000 notes per incident, needs a `From` header with the email of a valid user on the account [OAS `from_header`])
- `POST /incidents/{id}/status_updates`
- `PUT /incidents/{id}` (ack/resolve/reassign)
- `PUT /incidents/{id}/custom_fields/values`

These need the `incidents.write` scope [OAS].

**Recommendation:** start with a single note when an engineer confirms or merges a Draft ("Tracked in Outage Manager: <url>"). Do not drive PagerDuty state (ack/resolve) from Outage Manager, because PagerDuty stays the source of truth for paging. Skip status updates for now, since customer comms go through Zendesk and Status.io.

## 10. Networking

- PagerDuty must reach a **publicly accessible** URL, on any port, and HTTPS is strongly preferred [DD-BEH].
- **Webhook source IPs** [DD-IPS], retrieved 2026-09-28. The `/webhook_ips` API endpoint is no longer updated, so use the published lists [KB-WH]:
  - **US:** 44.242.69.192, 52.89.71.166, 54.213.187.133, 35.86.21.47, 52.88.94.18, 44.238.89.29, 54.241.68.46, 54.176.72.216, 54.177.81.67, 13.56.49.27, 34.210.57.30, 34.210.242.134, 52.34.208.156
  - **EU:** 18.192.91.93, 18.158.120.237, 18.194.177.30, 18.158.199.216, 3.126.25.87, 18.197.187.16, 54.76.3.62, 54.170.2.90, 52.213.188.110, 54.195.179.238, 34.250.91.200, 54.76.225.71
  - Machine-readable lists: `https://developer.pagerduty.com/ip-safelists/webhooks-us-service-region-json` (and `-eu-`). These lists can change, so automate the refresh or monitor them.
- **Outbound** from our app to the REST API is TCP 443 to `api.pagerduty.com` (or `api.eu.pagerduty.com`). REST egress IPs are published at [DD-RIPS]: US 44.237.102.140, 35.163.146.4, 52.36.64.228, 34.212.97.30, 44.226.85.110, 52.24.121.31, and EU 3.126.2.17, 3.79.124.225, 18.192.170.210, 3.79.82.135, 3.66.104.172, 3.73.115.116.
- **If Outage Manager is internal-only**, there are two options:
  - (a) Expose **only** `/webhooks/pagerduty/` through a reverse proxy or WAF in the DMZ, safelisted to the IPs above, with HMAC verification plus optional mTLS. This is the preferred option.
  - (b) **Poll** `GET /incidents?since=...&service_ids[]=...&statuses[]=triggered` (and `/log_entries?since=`) every 30–60 s from a Celery beat task. This costs roughly 1–2 req/min, well within 960/min. The trade-off is up to a minute of latency, but it needs no inbound exposure.
  - The poller is worth building anyway as the **reconciliation/backfill job** for webhook gaps (auto-disable, outages), so option (b) is a fallback we get almost for free.

## Recommended approach

1. **Endpoint:** `POST /webhooks/pagerduty/` (a Django view with CSRF exempt). The view:
   - reads the raw `request.body`
   - looks up the subscription secret by `X-Webhook-Subscription`
   - verifies `X-PagerDuty-Signature`, where any `v1` signature may match (constant-time)
   - returns 401 on failure
   - inserts a `PagerDutyWebhookEvent` row keyed by `X-Webhook-Id` (unique), with the raw JSON
   - enqueues a Celery task and returns **202**, all in under 5 s
2. **Subscription:** one v3 subscription per network-relevant PagerDuty **service**, or per **team** if the network team owns a team. Subscribe to `incident.triggered, acknowledged, unacknowledged, escalated, delegated, reassigned, priority_updated, annotated, status_update_published, responder.added, responder.replied, resolved, reopened, service_updated`. Store the secrets encrypted in our DB. They are configured in the Outage Manager settings UI, not Django admin.
3. **Processing task:**
   - On `incident.triggered` for an unknown PagerDuty incident ID, fetch `GET /incidents/{id}?include[]=first_trigger_log_entries` and `/alerts`.
   - Map the PagerDuty service to our Service, and the hostname to a Boris Network Element.
   - Apply correlation: attach to an open Draft or Incident on the same service or element within N minutes, otherwise create a new **Draft Incident**.
   - Every other event type appends a timeline row to the linked Incident. Unknown types are stored and ignored.
4. **Reconciliation:** a Celery beat job polls `/incidents?since=` for the mapped services (catching missed triggers), checks each subscription's `temporarily_disabled` flag and alerts or re-enables, and snapshots `log_entries` when an Incident is resolved.
5. **Auth:** use a Scoped OAuth private app (client-credentials app token, read scopes). Add `incidents.write` only for the optional "linked to Outage Manager" note.
6. **Manual Incidents** can optionally be linked to a PagerDuty incident by URL or number later, and then get the same timeline import.

## Open questions for Devoli's PagerDuty account

- **Plan tier and add-ons:** do we have AIOps (Content-Based or Intelligent Alert Grouping), Incident Workflows, custom fields, or the Advanced Permissions needed for user tokens? This affects how much grouping PagerDuty does before us.
- **Service region:** US or EU? This decides the IP safelist and the API host.
- **Which PagerDuty services and teams are "network"?** Is there a Network team we can scope one subscription to, or do we need per-service subscriptions (with a limit of 10 per scope)?
- **Monitoring sources:** do LibreNMS and Observium feed PagerDuty, and through which integration (Events API v2, email, or another)? What alert templates are used? Is `custom_details` JSON with hostname, ifName and device ID? Can we change the LibreNMS template to emit structured JSON? Is Observium via email, whose `channel` structure is different?
- **Correlation keys:** does the PagerDuty `source`/hostname match Boris Network Element names exactly (e.g. sysName vs FQDN)?
- **Exposure:** can we publish a single webhook path through the DMZ or WAF, or must Outage Manager stay internal (and poll)?
- **Credentials ownership:** who can create a Scoped OAuth private app or an account API key (Admin or Account Owner)?
- **Write-back appetite:** is a PagerDuty note linking to Outage Manager acceptable to the on-call team?
- **Correlation window:** how long should the default window be for folding repeated triggers on a service into one Draft?
