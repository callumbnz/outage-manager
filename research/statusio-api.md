# Status.io API capabilities

Resolves wayfinder ticket [#4](https://github.com/callumbnz/outage-manager/issues/4) on map [#1](https://github.com/callumbnz/outage-manager/issues/1).

**Question.** What does the Status.io API support for incidents (create/update/resolve), scheduled maintenance, components/containers, subscriber notifications and post-mortem/PIR publication? What are the auth and rate limits?

**Status.** This is researched from primary Status.io sources only (listed below). We have **not** yet made any live API calls against Devoli's account. Anything that depends on Devoli's plan, page structure or live behaviour is listed under [Open questions](#10-open-questions-for-devolis-statusio-account).

## TL;DR

- **The API covers everything we need for live Incidents and scheduled Maintenance.** It is a REST/JSON API at `https://api.status.io/v2`, authenticated with the `x-api-id` and `x-api-key` headers.
- **There is no post-mortem API.** In Status.io a postmortem is only an **external URL** attached to an incident through the dashboard button "Add Postmortem Link". No endpoint or request field sets it: the string "postmortem" appears nowhere in the OpenAPI spec. So a PIR must be **hosted by us** and then linked, either manually in the dashboard or by posting an incident update whose message contains the link.
- **There is no numeric rate limit.** Status.io's rule is: send one request at a time, wait at least 1 s between requests, and poll no more than once a minute.
- **Components and containers are read-only through the API.** We can list them and set a component's status, but we cannot create or rename them.
- **There is no sandbox.** Status.io recommends creating a separate status page for testing, and that page is billed separately.

## Sources

| Short name | Source | Notes |
|---|---|---|
| `OAS` | <https://developer.status.io/openapi.json> (OpenAPI 3.1.0, "Status.io API v2", fetched 2026-09-27, sha256 `6054bf6d1a84…`) | Current authoritative reference, rendered at <https://developer.status.io/> |
| `APIARY` | <https://statusio.docs.apiary.io/> (blueprint `https://jsapi.apiary.io/apis/statusio.apib`) | Legacy docs. Its header says *"The API documentation has moved"* to developer.status.io. The content matches `OAS` |
| `KB-PM` | <https://kb.status.io/incidents/post-mortem/> (modified 2026-01-29) | How postmortems work |
| `KB-INC` | <https://kb.status.io/incidents/incident-overview/> (modified 2026-01-29) | Incident lifecycle in the dashboard |
| `KB-INFRA` | <https://kb.status.io/incidents/add-or-remove-infrastructure/> | Editing affected infrastructure |
| `KB-CODES` | <https://kb.status.io/developers/status-codes/> | Status, state and maintenance-state codes |
| `KB-MLC` | <https://kb.status.io/planned-maintenance/maintenance-lifecycle/> | Maintenance lifecycle and automation |
| `KB-MOD` | <https://kb.status.io/planned-maintenance/maintenance-ondemand/> | Toggling a component into maintenance |
| `KB-CONT` | <https://kb.status.io/design/container-display/> | The component/container model |
| `KB-WH` | <https://kb.status.io/notifications/webhook/> and <https://github.com/statusio/notifications-webhook> | Outbound webhook payloads |
| `KB-TPL` | <https://kb.status.io/notifications/message-templates/> | Email templates |
| `KB-SUBJ` | <https://kb.status.io/notifications/message-subject-variables/> | Subject variables |
| `KB-SUT` | <https://kb.status.io/planned-maintenance/status-update-templates/> | Status update templates |
| `KB-SUBS` | <https://kb.status.io/notifications/subscriber-management/> | Subscriber management |
| `KB-PERM` | <https://kb.status.io/account/user-permissions/> | Team roles |
| `KB-MULTI` | <https://kb.status.io/account/multiple-pages/> | Multiple pages, billing, credential scope |
| `KB-AUDIT` | <https://kb.status.io/security/audit-trail/> | Audit trail by plan |
| `KB-HC` | <https://kb.status.io/account/high-capacity-usage/> | Volume caps |
| `PRICING` | <https://status.io/pricing> (fetched 2026-09-27) | Plan tiers |
| `PY` | <https://github.com/statusio/statusio-python> (`statusio/api.py`, last push 2022-01-28) | Official Python client |

---

## 1. Auth, errors and rate limits

| Topic | Finding | Source |
|---|---|---|
| Base URL | `https://api.status.io/v2` | `OAS` `servers` |
| Auth | Send `x-api-id` and `x-api-key` headers on **every** request. Credentials are in the dashboard's API tab | `OAS` info, "Authentication" |
| Credential scope | **Credentials belong to each team member, not to the page.** One set of credentials can access **every status page that user manages** | `OAS` info ("Each team member has their own unique API credentials"); `KB-MULTI` |
| Roles | Owner, Admin, and Limited ("make status updates and manage incidents and maintenance"). New members are Admin by default | `KB-PERM` |
| Format | JSON with `Content-Type: application/json`. All returned timestamps are UTC | `OAS` info, "Data Format" |
| Errors | Every response contains `status.error` (`"yes"` or `"no"`) and `status.message`. An invalid URL or method returns **403**. Missing or wrong-type parameters return **400**. An auth failure returns `error: "yes"`, `message: "Authentication failed"` | `OAS` info, "Error Handling" |
| Rate limits | **No numeric quota is published.** The rules are: send "only **one API request at a time**… waiting at least **1 second between requests** to avoid triggering rate limits". Polling faster than once a minute "may be rate-limited or blocked". Cache responses server-side, because uncached load "can cause your connection to be throttled". Only send updates when something actually changes | `OAS` info, "Best Practices" |
| Volume | Plans are sized for "typical operational usage". Sustained high event volume may need a High Capacity add-on | `KB-HC` |
| IP ACL / SSL | `/admin/ip_access_control/*` and `/admin/ssl_cert/update` exist. They restrict **status page visibility** only, not API access | `OAS` paths |

## 2. Status pages, components and containers

- **Model.** A **status page** (`statuspage_id`) has **components**, which are the high-level functions such as "Web App" or "API". **Containers** are optional sub-elements, such as locations or regions, and one container can link to many components. Status is held per **component-container pair** ([`KB-CONT`](https://kb.status.io/design/container-display/)).
- **List:** `GET /component/list/{statuspage_id}` returns each component (`_id`, `name`, `position`) with its `containers[]` (`_id`, `name`, `location`) (`OAS`).
- **Set status without an incident:** `POST /component/status/update` takes `statuspage_id`, `component`, `container` (which must be attached to that component), `details` and `current_status`, where `current_status` is one of `100/200/300/400/500/600`. This is the only endpoint that accepts **200 "Planned maintenance"** (`OAS`; [`KB-MOD`](https://kb.status.io/planned-maintenance/maintenance-ondemand/)).
- **Summary:** `GET /status/summary/{statuspage_id}` returns the overall status, active incidents, and active and upcoming maintenance (`OAS`).
- **Not in the API:** creating, renaming or deleting components and containers. There are no such paths in `OAS`. The infrastructure must be modelled in the dashboard, and we then sync the IDs.
- **ID format differs by endpoint:**
  - `infrastructure_affected` on incidents and maintenance is an array of `"<component_id>-<container_id>"` (hyphen).
  - `granular` on subscribers is a comma-separated string of `"<component_id>_<container_id>"` (underscore).
  
  Source: `OAS` schema descriptions and examples.

### Status and state codes

| Code set | Values | Source |
|---|---|---|
| Component/incident **status** (`current_status`) | 100 Operational · 200 Planned maintenance (maintenance only) · 300 Degraded performance · 400 Partial service disruption · 500 Service disruption · 600 Security issue/event | `OAS` `x-enum-descriptions`; `KB-CODES` |
| Incident **state** (`current_state`) | 100 Investigating · 200 Identified · 300 Monitoring · 400 Resolved | `OAS`; `KB-CODES` |
| Maintenance **state** | 100 Pending · 200 Active · 300 Closed | `KB-CODES` |

Custom labels and colours on the page do not change these codes (`OAS`, `/component/status/update`).

## 3. Incidents

| Endpoint | Purpose | Required fields | Optional fields | Returns |
|---|---|---|---|---|
| `POST /incident/create` | Open an incident | `statuspage_id`, `incident_name`, `incident_details`, `current_status` (100/300/400/500/600), `current_state` (100–400), `all_infrastructure_affected` ("1"/"0"), `infrastructure_affected[]`, `social` | `notify_email`, `notify_sms`, `notify_webhook`, `irc`, `msteams`, `slack`, `message_subject` | `result` = new incident ID |
| `POST /incident/update` | Post an update | `statuspage_id`, `incident_id`, `incident_details`, `current_status`, `current_state`, `social` | same notify flags and `message_subject` | |
| `POST /incident/resolve` | Resolve | `statuspage_id`, `incident_id`, `incident_details`, `social` | same notify flags and `message_subject` | `result: true` |
| `POST /incident/delete` | Delete **permanently** | `statuspage_id`, `incident_id` | | |
| `POST /incident/history/create` | Back-fill a past incident | `statuspage_id`, `incident_name`, `incident_details_start`, `incident_details_resolve`, `incident_start_date`/`_time`, `incident_end_date`/`_time` (`MM/dd/yyyy`, `HH:mm`), `current_status` | `all_infrastructure_affected`, `infrastructure_affected[]` | |
| `GET /incident/list/active/{sp}?page=` / `GET /incident/list/resolved/{sp}?page=` | Paginated lists | | | |
| `GET /incidents/{sp}` | IDs of all active and resolved incidents | | | |
| `GET /incident/{sp}/{incident_id}` | Full incident with `messages[]`, `components_affected`, `containers_affected` | | | |
| `GET /incident/message/{sp}/{message_id}` | One update message | | | |
| `GET /incident/list/{sp}` | **Deprecated**, capped at 100 results | | | |

Source for all rows: `OAS` paths and request schemas.

**Notification flags.** Each is a string, `"1"` to send or `"0"` to skip, and each is set **per call**:

| Flag | Channel |
|---|---|
| `notify_email` | Email subscribers |
| `notify_sms` | SMS subscribers |
| `notify_webhook` | Webhook subscribers |
| `social` | Twitter/X. **Required** on create, update and resolve |
| `irc` | IRC |
| `msteams` | Microsoft Teams |
| `slack` | Slack |

`message_subject` sets the email subject and supports `*|VAR|*` variables ([`KB-SUBJ`](https://kb.status.io/notifications/message-subject-variables/)).

**What update cannot do:**
- `update` has **no `infrastructure_affected` and no `incident_name`**. Affected infrastructure and the incident title can only be changed in the dashboard ([`KB-INC`](https://kb.status.io/incidents/incident-overview/), [`KB-INFRA`](https://kb.status.io/incidents/add-or-remove-infrastructure/)).
- The status set by `update` applies to **all** affected pairs. The dashboard can set a different status per pair, but the API cannot (`KB-INC`, "Apply status level to all affected infrastructure").
- When infrastructure is added or removed, "the status level will not be changed automatically" (`KB-INFRA`).

**Resolve** takes no status or state. It is "the same process as an incident update" in the dashboard (`KB-INC`). Whether components go back to 100 automatically needs to be confirmed on a test page (see open questions).

## 4. Scheduled Maintenance

| Endpoint | Purpose | Required fields | Optional fields |
|---|---|---|---|
| `POST /maintenance/schedule` | Schedule | `statuspage_id`, `maintenance_name`, `maintenance_details`, `all_infrastructure_affected`, `infrastructure_affected[]`, `date_planned_start`, `time_planned_start`, `date_planned_end`, `time_planned_end` (`MM/dd/yyyy`, `HH:mm` 24 h), `maintenance_notify_now`, `maintenance_notify_72_hr`, `maintenance_notify_24_hr`, `maintenance_notify_1_hr`, `automation` | `message_subject`, `automation_start_text`, `automation_stop_text`, `automation_notify_{start,stop}_{email,sms,webhook,twitter,irc,msteams,slack}` (default `0`) |
| `POST /maintenance/start` | Move pending to active now | `statuspage_id`, `maintenance_id`, `maintenance_details`, `social` | notify flags and `message_subject` |
| `POST /maintenance/update` | Post an update to an **active** maintenance | same as start | same as start |
| `POST /maintenance/finish` | Close the maintenance and move it to history | same as start | same as start |
| `POST /maintenance/delete` | Delete **permanently** | `statuspage_id`, `maintenance_id` | |
| `POST /maintenance/history/create` | Back-fill | like the incident version | |
| `GET /maintenance/list/{pending,active,closed}/{sp}?page=`, `GET /maintenances/{sp}`, `GET /maintenance/{sp}/{id}`, `GET /maintenance/message/{sp}/{message_id}` | Read | | |

`schedule` returns the new maintenance ID in `result`. Source: `OAS`.

- **Automation.** With `automation: "1"`, Status.io starts and ends the maintenance at the planned times and posts `automation_start_text` and `automation_stop_text`. Each channel is notified only if its flag is set (`OAS`; [`KB-MLC`](https://kb.status.io/planned-maintenance/maintenance-lifecycle/)).
- **Reminders.** The `maintenance_notify_now`, `_72_hr`, `_24_hr` and `_1_hr` flags send subscriber notices at those offsets (`OAS`).
- **Component status.** When a maintenance starts, the affected pairs are set to maintenance status. When it finishes, they are updated again (`KB-MLC`; `OAS` `finish` description).
- **Gaps in the API:**
  - There is **no endpoint to edit or reschedule a pending maintenance**, and no "cancel" separate from delete. The dashboard can reschedule or cancel (`KB-MLC`), but the API can only `delete` and `schedule` again.
  - The **time zone of the input dates is not documented**. The spec only says that returned timestamps are UTC.

## 5. Post-mortems and PIR publication: what is and isn't possible

**Facts from the sources:**
1. In Status.io, "postmortems are linked to incidents **using an external URL**". Status.io recommends publishing the full postmortem on your own blog, docs or knowledge base. It is added in the dashboard: open the incident, click **Add Postmortem Link**, and enter the URL. The link then appears on the incident ([`KB-PM`](https://kb.status.io/incidents/post-mortem/)). The dashboard's incident page lists "Link to a postmortem" as a modification ([`KB-INC`](https://kb.status.io/incidents/incident-overview/)).
2. **No API endpoint or request field sets a postmortem link or body.** A search of the whole `OAS` document for `postmortem`, `post-mortem` and `post_mortem` finds 0 matches. No request schema in `/incident/*` has such a field. The official Python client has no postmortem method either (`PY`).
3. Status.io does not host PIR text. There is no field for rich PIR content. Incident messages (`incident_details`) are the only free text we can write.

**What this means for Outage Manager:**
- ✅ We can publish the PIR **on a public URL we control**, such as a public PIR page served by Outage Manager or an article on Devoli's website or Help Center.
- ✅ Through the API, we can post an **incident message that contains the PIR link** by calling `POST /incident/update` on the resolved incident with `current_state: 400`, the status it resolved with, and a PIR summary plus URL in `incident_details`. Notification flags can be set as needed. **This is unverified.** The spec does not say whether `update` is accepted on a resolved incident, or whether it would reopen it. It must be tested on a test page. If it is rejected, the only other API option is to include the link in the `resolve` message, which only works if the PIR is approved before the incident is resolved. That is unlikely.
- ❌ We **cannot** fill in the native "Add Postmortem Link" field through the API. If Devoli wants the native postmortem link on the page, a person must add it in the dashboard. Outage Manager can prompt for this with a task or notification that includes the URL.
- ❌ We cannot publish PIR content natively into Status.io. The API has no post-mortem object, no attachment support and no long-form page.

## 6. Subscribers and outbound webhooks

**Subscriber endpoints** (`OAS`):

| Endpoint | Notes |
|---|---|
| `GET /subscriber/list/{sp}` | Grouped by method |
| `POST /subscriber/add` | `method`: `email`, `sms` or `webhook`. `address`: SMS numbers need a `+` country code. `silent="1"` skips the welcome message. `granular` restricts the subscriber to chosen component_container pairs |
| `PATCH /subscriber/update` | Same fields plus `subscriber_id` |
| `DELETE /subscriber/remove/{sp}/{subscriber_id}` | |

The public Subscribe button can be switched off, leaving subscribers managed only through the dashboard or API ([`KB-SUBS`](https://kb.status.io/notifications/subscriber-management/)).

**Outbound webhooks.** Webhook *subscribers* receive an HTTP POST on each incident or maintenance update that is sent with `notify_webhook` (or the automation webhook flags) set to `"1"` ([`KB-WH`](https://kb.status.io/notifications/webhook/)).
- **Payload fields:** `id`, `message_id`, `title`, `datetime`, `datetime_start`/`_resolve` (or `_end`), `current_status`, `current_state`, `previous_status`, `previous_state`, `infrastructure_affected[]`, `components[]`, `containers[]`, `details`, `incident_url` or `maintenance_url`, `status_page_url`.
- **Status and state are strings in the payload**, such as `"Degraded Performance"`, not the numeric codes.
- **No signature, secret or HMAC is documented.** If Outage Manager consumes these webhooks, the endpoint needs a secret token in the URL.
- **Use case:** these webhooks are useful for **detecting drift**, meaning changes someone makes directly in the Status.io dashboard.

## 7. Message templates

- **Email templates** are plain text or HTML and are edited in the dashboard. They use `*|VAR|*` variables such as `INCIDENTTITLE`, `CURRENTSTATUS`, `CURRENTSTATE`, `AFFECTEDCOMPONENTS`, `INCIDENTMESSAGE`, `MAINTENANCEURL` and `TIMESTART`. The defaults are at <https://github.com/statusio/templates-email> ([`KB-TPL`](https://kb.status.io/notifications/message-templates/)).
- **Status update templates** are reusable titles and messages, available in the dashboard's Incidents and Maintenance tabs ([`KB-SUT`](https://kb.status.io/planned-maintenance/status-update-templates/)).
- **Neither kind can be reached through the API.** No path in `OAS` covers them. Through the API we control only `incident_details`/`maintenance_details` and `message_subject`, which supports the subject variables ([`KB-SUBJ`](https://kb.status.io/notifications/message-subject-variables/)).
- **Recommendation:** keep the public wording templates in Outage Manager, render the text there, and send the finished text.

## 8. Test and sandbox options

- **There is no sandbox environment.** The "Live Sandbox" on developer.status.io is an API console that makes **live calls that "will make real changes to your status page"**. Status.io "strongly recommend[s] creating a separate status page for testing purposes" (`OAS` info, "API Testing").
- **A test page is a separate subscription.** Each status page has its own billing and plan. One set of API credentials can reach every page the user manages, so the same service user can target either page by `statuspage_id` ([`KB-MULTI`](https://kb.status.io/account/multiple-pages/)).
- **Plans:** Basic $79/mo, Standard $149/mo, Plus $349/mo, and Enterprise from $999/mo. Subscriber caps are 500, 2,000 and 5,000 on Basic, Standard and Plus. Every plan lists the Developer API ([`PRICING`](https://status.io/pricing)). The audit trail is on Plus and Enterprise only ([`KB-AUDIT`](https://kb.status.io/security/audit-trail/)). SMS, ChatOps (IRC/Teams/Slack) and component subscriptions are tiered features on the pricing page. The exact tier for each should be confirmed against Devoli's contract.

## 9. Suggested mapping and integration approach

### 9.1 Impact level to `current_status`

Impact levels come from [research/boris-nautobot-impact.md (#3)](https://github.com/callumbnz/outage-manager/issues/3).

| Our Impact | Status.io `current_status` | Default publish behaviour |
|---|---|---|
| NO-IMPACT | (100 Operational) | **Don't publish** to Status.io |
| REDUCED-REDUNDANCY | (100, or 300 if the Author chooses to publish) | **Don't publish by default**. Status.io has no "at risk" code. The Author may choose to publish at 300 with wording that says so |
| DEGRADED | **300** Degraded performance | Publish |
| OUTAGE, affecting part of a component (some containers or regions, or a subset of Customers) | **400** Partial service disruption | Publish |
| OUTAGE, affecting all of a component | **500** Service disruption | Publish |
| (security-related, manual only) | 600 Security issue | Never auto-mapped. Only set if the Author chooses it |
| Maintenance in progress | 200 | Set by Status.io when the maintenance starts. We never send 200 for incidents |

The API applies **one** status to all affected pairs per incident call. If a single Incident has mixed Impact, the options are:
- (a) send the **highest** Impact (recommended), or
- (b) after create/update, adjust each pair with `/component/status/update`. The interaction between (b) and the incident's own status is unverified.

### 9.2 Lifecycle mapping

| Outage Manager | Status.io call |
|---|---|
| Draft Incident (PagerDuty, not yet confirmed) | Nothing |
| Incident confirmed and published | `incident/create` with state **100 Investigating** |
| Cause identified | `incident/update` with state **200 Identified** |
| Fix applied, watching | `incident/update` with state **300 Monitoring** |
| Incident resolved | `incident/resolve` (state 400) |
| Incident withdrawn (false alarm, already published) | `incident/delete`. This is permanent, so consider `resolve` with an explanation instead |
| Maintenance draft or awaiting Approver | Nothing |
| Maintenance approved | `maintenance/schedule` (state 100 Pending), with reminder flags per the Author's choice. Use `automation` only if we trust the planned times. Otherwise Outage Manager calls start and finish |
| Maintenance started | `maintenance/start` (200 Active), unless automated |
| Maintenance progress note | `maintenance/update` |
| Maintenance completed | `maintenance/finish` (300 Closed) |
| Maintenance rescheduled | No edit API: `maintenance/delete` then `maintenance/schedule` again. This gives a new ID and may re-notify. Alternatively, do it manually in the dashboard |
| Maintenance cancelled | `maintenance/delete`. This sends no cancellation notice, so post a `start`/`finish` message or a Customer Notification first if subscribers were already told |
| Emergency Maintenance | `maintenance/schedule` with start = now and `maintenance_notify_now=1`, then immediately `maintenance/start` |
| PIR approved by all Reviewers | Publish the PIR on our own public URL, then `incident/update` on the resolved incident with the link (to be verified), and/or prompt a person to click "Add Postmortem Link" |

### 9.3 Recommended integration approach

1. **Write a thin in-house client**, a Python `requests` wrapper generated from or checked against `OAS`, rather than depending on `statusio-python`. That library was last pushed in January 2022 and adds nothing we need, such as postmortem support.
2. **Route all writes through one queue.** Every Status.io write goes to a dedicated Celery queue with concurrency 1 and a 1-second rate limit, using an outbox table: a row per intended call, holding the status and returned IDs, with retries and exponential backoff. The response is judged by `status.error`, not only by the HTTP code. This follows `OAS` "Best Practices".
3. **Store Status.io IDs** (incident, maintenance and message IDs) on our Incident and Maintenance records, so later calls update the right object and repeated calls do no harm.
4. **Keep a Status.io infrastructure registry in the app, not the Django admin.** A periodic, cached `GET /component/list` stores the components and containers. An in-app configuration screen maps Boris objects (Services or Locations) to component-container pairs. Components themselves are maintained in the Status.io dashboard.
5. **Put notification choice in the app.** Each publish action records which Status.io channels to notify. These are separate from Zendesk Customer Notifications, and the UI should make the difference clear so Customers aren't notified twice.
6. **Format dates** as UTC `MM/dd/yyyy` and `HH:mm`, once the input time zone is confirmed.
7. **Optionally, detect drift.** Register an Outage Manager webhook subscriber, with a secret token in the URL and `granular` unset, so edits made directly in the Status.io dashboard are seen and recorded.
8. **Host PIRs publicly** at an Outage Manager public route or Devoli site, and link them from Status.io as described in §5.
9. **Use a separate test status page** in non-production environments. `statuspage_id` and the credentials are environment settings.

## 10. Open questions for Devoli's Status.io account

1. **Plan tier.** Which plan is Devoli on, and does it include SMS, ChatOps (Teams/Slack), component subscriptions and the audit trail? What is the subscriber cap?
2. **Existing structure.** Which components and containers exist today, and how should they map to Boris Services or Locations? Is the structure fixed, or can we redesign it?
3. **Service account.** Can we create a dedicated team member for Outage Manager, since API keys belong to each user? Does the **Limited** role have enough API rights, or does it need Admin?
4. **Test page.** Will Devoli pay for a second status page for staging and testing?
5. **Updating a resolved incident.** Does `incident/update` work on a **resolved** incident, for posting the PIR link, without reopening it? This must be tested on the test page.
6. **Resolve and component status.** Does `incident/resolve` return affected components to 100 Operational automatically?
7. **Input time zone.** What time zone does `maintenance/schedule` interpret the dates and times in: UTC, or the page's configured time zone?
8. **Native postmortem link.** Is the manual "Add Postmortem Link" step acceptable, or is a linked incident update enough? Should Devoli ask Status.io support for API support for postmortem links?
9. **Where public PIRs live.** Should PIRs be hosted on an Outage Manager public page, the Devoli website, or the Zendesk Help Center?
10. **Subscribers.** Is the public Subscribe button on today? Should Outage Manager manage Status.io subscribers, for example by syncing Customer contacts, or leave them self-service?
11. **Social and ChatOps.** Are Twitter/X, Slack and Teams broadcasts connected, and should they be enabled by default?
12. **Webhook signing.** Is there any signing of outbound webhooks that isn't documented? Otherwise we rely on a URL token.
