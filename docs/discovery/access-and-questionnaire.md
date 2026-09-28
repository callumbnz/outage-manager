# Outage Manager: access and discovery questionnaire

**Purpose:** get Outage Manager the system access it needs, and the facts about each system that we can't find out ourselves. It is the checklist for two tickets:

- [Provision API access to integrated systems](https://github.com/callumbnz/outage-manager/issues/2): the **Access to provision** lists.
- [Discovery questionnaire for integration owners](https://github.com/callumbnz/outage-manager/issues/15): the **Questions** lists, plus the setup for [Status.io behaviour checks on a test page](https://github.com/callumbnz/outage-manager/issues/16).

**From:** Callum (Devoli), **To:** the owner of each system below. **How your answers will be used:** each answer confirms or replaces a working assumption in the [spec](../spec.md) (mostly [§10 "Assumptions to verify"](../spec.md#10-assumptions-to-verify)). It is then used to close the "verify" step of the matching build module.

## Context

Outage Manager is Devoli's new internal tool for planning and approving **Maintenance**, running **Incidents** from detection to **Post-Incident Report (PIR)**, and announcing both to customers on Status.io. It replaces the current Zendesk-plugin tool. To do this it reads the network inventory from Boris, receives alerts from PagerDuty, pulls graphs from LibreNMS, Grafana and Kentik, publishes PIRs to SharePoint, reads carrier maintenance emails, and posts alerts to Slack. It runs in Docker on a RHEL 9 host inside Devoli's network. Most integrations are **read-only**; the only systems it writes to are Status.io, SharePoint, Slack, the Provider Notice mailbox (to move processed mail) and email.

The build doesn't wait for these answers. Every module is built against fakes and a written working assumption. Your answers let us switch each module to the real system and confirm it behaves as assumed. Terms like *Maintenance*, *Impact* and *Provider Notice* are defined in [`CONTEXT.md`](../../CONTEXT.md).

## How to answer

- **Only answer your own section(s).** Each one takes about 15–30 minutes. Access provisioning may take longer on your side.
- **Two ways to reply:** edit this file directly (write under each `> **Answer:**` line and open a PR or commit to `main`), **or** reply as a comment on the relevant GitHub issue ([#2](https://github.com/callumbnz/outage-manager/issues/2) for access, [#15](https://github.com/callumbnz/outage-manager/issues/15) for questions), quoting the question ID (e.g. `B4`).
- "I don't know" and partial answers are useful; say who might know. Every question shows the **working assumption** the build uses until it's answered. If that assumption is correct, a one-word "Confirmed" is enough.
- Tick the status box in the summary table once access is provisioned.

> [!CAUTION]
> **Never write a secret in this file, in a GitHub comment, in Slack or in email.** That covers tokens, API keys, passwords, client secrets, private keys, webhook URLs and webhook signing secrets.
> Put credentials into **HashiCorp Vault** (`https://vault.devoli.co`) and record **only the Vault path** here. The proposed layout is `secret/outage-manager/<env>/<integration>`, where `<env>` is `prod` or `test`; see [HashiCorp Vault](#8-hashicorp-vault). If a secret is ever exposed by mistake, rotate it straight away.

## Summary

| # | System | Owner / team | Access needed | Least-privilege scope | Vault path | Done |
|---|---|---|---|---|---|---|
| 1 | [Boris (Nautobot fork)](#1-boris-nautobot-fork) | | API token for a service user; webhook to Outage Manager | Read-only token (`write_enabled=False`), `view` only | `secret/outage-manager/<env>/boris` | [ ] |
| 2 | [Status.io](#2-statusio) | | API key for a dedicated team member; test status page | Limited role (if enough); production + test page | `secret/outage-manager/<env>/statusio` | [ ] |
| 3 | [PagerDuty](#3-pagerduty) | | Scoped OAuth app; v3 webhook subscription(s) | `incidents.read services.read teams.read users.read` | `secret/outage-manager/<env>/pagerduty` | [ ] |
| 4 | [LibreNMS (and Observium)](#4-librenms-and-observium) | | API token for a service account | `global-read` | `secret/outage-manager/<env>/librenms` | [ ] |
| 5 | [Grafana](#5-grafana) | | Service-account token; image renderer | Viewer role | `secret/outage-manager/<env>/grafana` | [ ] |
| 6 | [Kentik](#6-kentik) | | API token for a dedicated user | Member role | `secret/outage-manager/<env>/kentik` | [ ] |
| 7 | [Microsoft 365](#7-microsoft-365-sharepoint-and-provider-notice-mailbox) | | Entra app registration(s) with certificates | `Sites.Selected` on the PIR site; `Mail.ReadWrite` on one mailbox | `secret/outage-manager/<env>/graph-sharepoint`, `…/graph-mailbox` | [ ] |
| 8 | [HashiCorp Vault](#8-hashicorp-vault) | | AppRole + KV v2 prefix + policy | Read on `secret/outage-manager/<env>/*` only | n/a (bootstrap file on the host) | [ ] |
| 9 | [Slack](#9-slack) | | Incoming webhook (or bot token) for the NOC Channel | One channel; `chat:write` only | `secret/outage-manager/<env>/slack` | [ ] |
| 10 | [DNS / TLS](#10-dns-and-tls) | | Final hostname; DNS API token for DNS-01 | Edit TXT records for `_acme-challenge` only | `secret/outage-manager/<env>/dns` | [ ] |
| 11 | [SMTP relay](#11-smtp-relay) | | Relay host; credentials if required | Send-as one address | `secret/outage-manager/<env>/smtp` | [ ] |
| 12 | [Network / DMZ](#12-network-and-dmz) | | DMZ publish rules; firewall rules | Two public paths only; named egress | n/a | [ ] |
| 13 | [Backups](#13-backups) | | Backup target for DB dump + media | Write to one target path | `secret/outage-manager/<env>/backup` (if needed) | [ ] |
| 14 | [Provider Notice samples](#14-provider-notice-samples) | | Sample carrier emails (`.eml`) | Samples only; no mailbox access | n/a | [ ] |

---

## 1. Boris (Nautobot fork)

Boris is the source of truth for devices, interfaces, circuits and customers. Outage Manager keeps a **read-only local copy** and uses it to work out which Services and Customers a piece of work affects (the **Impact**). Nothing is ever written back. Build module: 9.5 Boris sync and Impact ([spec §5.1](../spec.md#51-boris-nautobot-fork)).

**Access to provision:**

- [ ] A dedicated service user `svc-outage-manager` in Boris, with `view` permission on every model used: devices, interfaces, front/rear ports, cables, circuits, circuit terminations, circuit types, providers, VLANs, tenants, tenant groups, locations, custom fields, relationships, tags and object changes.
- [ ] An API token for that user with **`write_enabled=False`**, stored in Vault at `secret/outage-manager/<env>/boris` (key `token`). Vault path: `________`
- [ ] API base URL (e.g. `https://boris.devoli.co/api/`): `________`
- [ ] GraphQL enabled for that user.
- [ ] A Boris webhook sending create/update/delete events for the models above to `https://<outage-manager-host>/webhooks/boris/` (internal network only), signed with a shared secret stored in Vault (key `webhook_secret`). The URL is supplied once the hostname is fixed.
- [ ] Firewall: HTTPS from the Outage Manager host to Boris, and from Boris to the Outage Manager host (for webhooks).

**Questions:**

### B1. Which upstream Nautobot version is Boris forked from, and how far has it diverged?

_Why it matters: API defaults and GraphQL field names differ between versions, so we pin our client to it._
_Working assumption if unanswered: Nautobot 2.4.x or 3.x, with no changes to the API._

> **Answer:**

### B2. Are Customers recorded as Tenants? Are Tenant Groups used, for example for resellers or wholesale vs retail?

_Why it matters: Impact reports list affected Customers, so we need to know where to find them._
_Working assumption if unanswered: Customer = Tenant; Tenant Groups are informational only._

> **Answer:**

### B3. How is a Service represented in Boris?

For example: a Circuit with a tenant, a VLAN, a custom model or App, a Relationship, or a custom field such as a service ID. Is there one convention, or several?

_Why it matters: this decides which records count as "a Service" when we calculate Impact._
_Working assumption if unanswered: Circuits and VLANs that have a tenant. The rules are configurable in the app._

> **Answer:**

### B4. Which custom fields, Relationships and tags exist on Device, Interface, Circuit and Tenant?

A paste or export of `/api/extras/custom-fields/` and `/api/extras/relationships/` is ideal. Please point out any field that holds a **service type/product** or a **region**.

_Why it matters: service type and region decide which Status.io component and region an affected Service shows under._
_Working assumption if unanswered: we derive the product from the Circuit type and the region from the Location._

> **Answer:**

### B5. How complete are cables, circuit terminations and customer premises equipment (CPE) records? Are cable statuses kept as `Connected`?

_Why it matters: Impact is found by following cables "downstream". Missing cables mean missed customers._
_Working assumption if unanswered: complete enough to follow; physical path plus VLANs, up to 10 hops deep._

> **Answer:**

### B6. Are last-mile and wholesale circuits (for example Chorus/LFC UFB) recorded as Circuits with a tenant? What do the Provider and Circuit Type lists look like?

_Why it matters: carrier maintenance emails quote these circuit IDs, and we match them against Boris._
_Working assumption if unanswered: yes, with the carrier's circuit ID in the Circuit `cid` field._

> **Answer:**

### B7. For shared devices (aggregation switches, BNGs), how should "downstream" be found: cables only, VLAN membership, or a parent/child Relationship?

_Why it matters: a wrong rule over- or under-reports who is affected._
_Working assumption if unanswered: cables plus VLAN membership, no custom hierarchy._

> **Answer:**

### B8. Are LAGs, sub-interfaces and VLAN assignments filled in consistently?

_Why it matters: we follow these links to find affected Services._
_Working assumption if unanswered: yes._

> **Answer:**

### B9. Can Boris send webhooks to Outage Manager over the internal network?

_Why it matters: webhooks keep our copy up to date within seconds, instead of every 5–15 minutes._
_Working assumption if unanswered: yes. If not, we poll the change log every 5 minutes._

> **Answer:**

### B10. Roughly how many devices, interfaces, circuits and tenants are there, and how often does the data change?

_Why it matters: this sizes the sync job and the database._
_Working assumption if unanswered: tens of thousands of interfaces, with changes daily._

> **Answer:**

### B11. What is the token rotation policy, and who rotates it?

_Why it matters: an expired token silently stops the sync, and we alert on that._
_Working assumption if unanswered: no expiry; rotated manually by the Boris admin, who updates Vault._

> **Answer:**

### B12. Which Apps are installed? In particular, is there a custom Service/Customer App, and is `nautobot-app-circuit-maintenance` installed and in use?

_Why it matters: an existing App may already hold Services or carrier maintenance data that we should reuse rather than duplicate._
_Working assumption if unanswered: no relevant Apps._

> **Answer:**

---

## 2. Status.io

Status.io is the only channel Outage Manager uses to tell customers and support staff about Maintenance and Incidents. Outage Manager creates, updates and resolves notices through the Status.io API. Build module: 9.6 Status.io publishing ([spec §5.2](../spec.md#52-statusio)). The test-page checks come from [Status.io behaviour checks on a test page](https://github.com/callumbnz/outage-manager/issues/16).

**Access to provision:**

- [ ] A **Status.io test status page** for non-production use. This is **billed separately**, so it needs budget approval. Test page ID: `________`
- [ ] A dedicated Status.io team member, e.g. `outage-manager@devoli.co` (API keys belong to a person, so it must not be a real staff member's account), with the **Limited** role on the production page and on the test page. If Limited can't use the API, fall back to the lowest role that can.
- [ ] That member's `x-api-id` and `x-api-key`, stored in Vault at `secret/outage-manager/prod/statusio` and `secret/outage-manager/test/statusio` (keys `api_id`, `api_key`). Vault paths: `________`
- [ ] Production status page ID: `________`
- [ ] A few test subscribers on the test page (the tester's own email, subscribed to **one** component only) for the checks below.

**Questions:**

### S1. Which Status.io plan is Devoli on, and what is the subscriber cap?

Does the plan include SMS, ChatOps (Slack/Teams), component-level subscriptions and the audit trail?

_Why it matters: we only turn on notification channels the plan actually has._
_Working assumption if unanswered: email, SMS and webhook are available; component subscriptions and audit trail are included._

> **Answer:**

### S2. Which components and containers exist on the page today? Is that structure fixed, or can it be rearranged?

_Why it matters: we plan to use **components = product types** (Fibre Broadband, Voice, Transit…) and **containers = regions** (Auckland, Wellington…), with no per-customer components._
_Working assumption if unanswered: it can be arranged that way; Devoli creates them in the Status.io dashboard._

> **Answer:**

### S3. Can we have a dedicated team member for the API, and is the Limited role enough?

_Why it matters: a shared staff account breaks when that person leaves, and Admin is more access than we need._
_Working assumption if unanswered: yes, and Limited is enough._

> **Answer:**

### S4. Will Devoli pay for a second (test) status page?

_Why it matters: without one, the checks below and all testing would have to run on the live page, which customers see._
_Working assumption if unanswered: yes._

> **Answer:**

### S5. Is the public Subscribe button enabled today, and should subscribers stay self-service?

_Why it matters: Outage Manager doesn't manage subscribers. Customers and support staff subscribe themselves._
_Working assumption if unanswered: Subscribe is on; subscribers are self-service._

> **Answer:**

### S6. Are social or ChatOps broadcasts (X/Twitter, Slack, Teams) connected? Should they be on by default for new notices?

_Why it matters: this sets the default notification flags on each post._
_Working assumption if unanswered: not connected; email, SMS and webhook only._

> **Answer:**

### S7. Is the manual "Add Postmortem Link" step in the Status.io dashboard acceptable for PIRs?

_Why it matters: the API appears to have no way to set it, so a Reviewer adds the link by hand when prompted by Outage Manager._
_Working assumption if unanswered: yes, a manual step with an in-app reminder. Check V6 below confirms whether any API exists._

> **Answer:**

**Verification (test page checks for [#16](https://github.com/callumbnz/outage-manager/issues/16)):**

Run these on the **test page only**, with a test subscriber signed up to a single component. Record what you saw, not what you expected. An engineer can run them with `curl`; each check names the API calls used.

| # | Check | How | Working assumption if not verified | Result |
|---|---|---|---|---|
| V1 | **Can a resolved incident be updated, and does it reopen?** | `incident/create` → `incident/resolve` → `incident/update` (state 100) on the same ID. Is the update accepted? Does the incident show as open again? Are subscribers notified? | Updating a resolved incident reopens it, so Outage Manager only does this when an Incident is reopened | |
| V2 | **Does resolving reset component status?** | Create an incident at status 400 on one component-container pair, then `incident/resolve`. Is the pair back at 100 Operational? | Yes. If not, Outage Manager also sets the pair back to 100 on resolve | |
| V3 | **Maintenance time zone** | `maintenance/schedule` with a known time (e.g. `01/15/2027` `10:00`). Does the page show 10:00 NZ time or 10:00 UTC? Note the page's configured time zone | Times are read in the page's configured time zone | |
| V4 | **Delete-notification behaviour** | Schedule a maintenance with notifications on, then `maintenance/delete`. Did subscribers get a cancellation notice? Then schedule it again with a "rescheduled" note: did that notify? | `delete` + `schedule` works and the new schedule notifies. If `delete` sends nothing, Outage Manager prompts the Author to cancel in the dashboard | |
| V5 | **Component-subscriber scope** | With a subscriber on component A only, post an incident on component B, then one on A. Repeat with a maintenance. Who was notified each time? | Subscribers only hear about the components they chose, for both incidents and maintenance | |
| V6 | **Does any API set the postmortem link?** | Check the current API docs and ask Status.io support. Try the `incident/update` fields on a resolved incident | No API exists; the manual dashboard step stays | |

---

## 3. PagerDuty

PagerDuty stays the alerting and on-call tool. When PagerDuty raises an incident on a network service, it tells Outage Manager (a webhook), which creates a **Draft Incident** for an engineer to confirm or dismiss. Outage Manager only **reads** from PagerDuty; it never acknowledges, resolves or adds notes. Build module: 9.8 PagerDuty ([spec §5.3](../spec.md#53-pagerduty)).

**Access to provision:**

- [ ] A **Scoped OAuth private app** (client credentials) with scopes `incidents.read services.read teams.read users.read` only. Store the client ID and secret in Vault at `secret/outage-manager/<env>/pagerduty` (keys `client_id`, `client_secret`). Vault path: `________`
  - Fallback if a Scoped OAuth app isn't possible: a **read-only** account REST API key (key `api_key`).
- [ ] **v3 webhook subscription(s)** to `https://<public-host>/webhooks/pagerduty/`, scoped to the network team (or to each network service), with these events: `incident.triggered, acknowledged, unacknowledged, escalated, delegated, reassigned, priority_updated, annotated, status_update_published, responder.added, responder.replied, resolved, reopened, service_updated`. The URL is supplied once the hostname is fixed.
- [ ] Each subscription's **signing secret** (shown once on creation), stored in Vault under the same path with key `webhook_secret_<subscription_id>`.
- [ ] The network service IDs (and team ID): `________`
- [ ] In LibreNMS: the PagerDuty transport uses the **JSON alert template**, so alerts carry `device_id`, `hostname`, `sysName` and `ports[].ifName`. The LibreNMS admin owns this; see [LibreNMS](#4-librenms-and-observium) question L6.

**Questions:**

### P1. Is the PagerDuty account in the US or EU service region?

_Why it matters: it decides the API address and which PagerDuty IP ranges the DMZ must let in._
_Working assumption if unanswered: US._

> **Answer:**

### P2. Which PagerDuty services and teams count as "network"? Is there one Network team we can subscribe to?

_Why it matters: we only want Draft Incidents for network alerts. A team-level subscription is simplest; per-service subscriptions are capped at 10 per scope._
_Working assumption if unanswered: one Network team subscription._

> **Answer:**

### P3. What priority names does Devoli use in PagerDuty (e.g. P1–P5), and how should they map to Outage Manager's P1–P4?

_Why it matters: we pre-fill a Draft Incident's Priority from PagerDuty's._
_Working assumption if unanswered: PagerDuty P1–P4 map one-to-one; P5 maps to P4._

> **Answer:**

### P4. Which plan and add-ons are enabled (AIOps / alert grouping, Incident Workflows, custom fields, Advanced Permissions)?

_Why it matters: if PagerDuty already groups related alerts, we see fewer, larger incidents._
_Working assumption if unanswered: no AIOps grouping; Outage Manager does its own grouping (see P8)._

> **Answer:**

### P5. How do LibreNMS (and Observium, if used) send alerts to PagerDuty: Events API v2, email, or something else?

_Why it matters: we read the hostname and interface from the alert to find the affected equipment._
_Working assumption if unanswered: LibreNMS via Events API v2; Observium not connected._

> **Answer:**

### P6. Does the host name in PagerDuty alerts match the device name in Boris? For example FQDN vs short name, or `sysName` vs hostname.

_Why it matters: a name mismatch means Draft Incidents don't show the affected equipment._
_Working assumption if unanswered: they match after lower-casing and stripping the domain._

> **Answer:**

### P7. Who can create a Scoped OAuth app or an API key (Admin or Account Owner)?

_Why it matters: we need to know who to ask for the access above._
_Working assumption if unanswered: the PagerDuty Account Owner._

> **Answer:**

### P8. Is 30 minutes the right window for grouping repeat alerts on the same service and equipment into one Draft Incident?

_Why it matters: too short gives duplicate Drafts; too long merges unrelated incidents. It's an Admin setting, so this is only the starting value._
_Working assumption if unanswered: 30 minutes._

> **Answer:**

---

## 4. LibreNMS (and Observium)

LibreNMS supplies graphs for PIR evidence, deep links for engineers, and the device IDs that link PagerDuty alerts to Boris equipment. Access is **read-only**. Build modules: 9.8 PagerDuty and 9.10 Evidence ([spec §5.4](../spec.md#54-librenms), [§5.5](../spec.md#55-librenms-grafana-kentik-evidence)). An Observium adapter is deferred, so the Observium questions only decide whether it's needed later.

**Access to provision:**

- [ ] A LibreNMS service account `svc-outage-manager` with the **`global-read`** role (or read-only user level on older versions). No write or admin rights.
- [ ] An API token for that account, stored in Vault at `secret/outage-manager/<env>/librenms` (key `token`). Vault path: `________`
- [ ] LibreNMS base URL (e.g. `https://librenms.devoli.co`): `________`
- [ ] If LibreNMS uses a certificate from an internal CA: the CA certificate (public, not secret; attach it or give its location).
- [ ] The **JSON alert template** assigned to the PagerDuty transport (see L6).
- [ ] Firewall: HTTPS from the Outage Manager host to LibreNMS.

**Questions:**

### L1. Which LibreNMS version runs, and how is it updated (monthly or pinned)?

_Why it matters: older versions use a different permissions model and token format._
_Working assumption if unanswered: a recent release, updated regularly, with the `global-read` role available._

> **Answer:**

### L2. Which time zone is the LibreNMS server set to?

_Why it matters: LibreNMS reports event and alert log times in server-local time, and PIR timelines must be accurate._
_Working assumption if unanswered: `Pacific/Auckland`._

> **Answer:**

### L3. Is the LibreNMS `hostname` an FQDN, an IP or a short name? Does it equal the Boris device name, and do interface names (`ifName`) match Boris?

_Why it matters: this is how we link LibreNMS graphs and PagerDuty alerts to Boris equipment._
_Working assumption if unanswered: they match after lower-casing and stripping the domain._

> **Answer:**

### L4. Could Boris store the LibreNMS `device_id` as a custom field?

_Why it matters: a stored ID makes matching exact instead of name-based._
_Working assumption if unanswered: no; Outage Manager matches by name nightly and lists anything unmatched._

> **Answer:**

### L5. Is the LibreNMS web/API reachable over HTTPS from the Outage Manager host, and is its certificate from an internal CA?

_Why it matters: we need the right network path and the CA to trust._
_Working assumption if unanswered: reachable; internal CA certificate supplied._

> **Answer:**

### L6. Which alert template does the LibreNMS PagerDuty transport use, and can it be switched to the JSON template? Are there several transports?

_Why it matters: the JSON template puts hostname, `sysName` and interface in a form we can read reliably._
_Working assumption if unanswered: yes, one transport, switched to the JSON template._

> **Answer:**

### L7. How long does LibreNMS keep event logs, alert logs and full-resolution graph data?

_Why it matters: evidence is captured when an Incident is resolved; if data ages out first, graphs are coarse._
_Working assumption if unanswered: at least 30 days at full resolution._

> **Answer:**

### L8. Is Observium staying? If so, which edition (Community, Professional or Enterprise), and what does it do that LibreNMS doesn't?

_Why it matters: Community edition has no API, so an Observium adapter would only be built for a paid edition that stays._
_Working assumption if unanswered: Observium is being retired; no adapter._

> **Answer:**

### L9. Do engineers use LibreNMS scheduled maintenance (alert suppression) today?

_Why it matters: automatic suppression during Maintenance is deferred until after the MVP; this sizes that work._
_Working assumption if unanswered: used manually; Outage Manager doesn't create maintenance windows in the MVP._

> **Answer:**

---

## 5. Grafana

Grafana dashboards supply graph images for PIR evidence and deep links for engineers. Access is **read-only**. Build module: 9.10 Evidence ([spec §5.5](../spec.md#55-librenms-grafana-kentik-evidence)).

**Access to provision:**

- [ ] A Grafana **service account** `svc-outage-manager` with the **Viewer** role (plus folder permissions for the dashboards in G3 if folders are restricted).
- [ ] A token for that service account, stored in Vault at `secret/outage-manager/<env>/grafana` (key `token`). Vault path: `________`
- [ ] Grafana base URL: `________`
- [ ] The **Grafana Image Renderer** service running next to Grafana, if Devoli agrees (see G2). Without it we only store links, not images.
- [ ] Firewall: HTTPS from the Outage Manager host to Grafana.

**Questions:**

### G1. Which Grafana version runs, and where (self-hosted, Grafana Cloud, cloud-managed)? Is it reachable from the Outage Manager host?

_Why it matters: it decides how we connect and which render features exist._
_Working assumption if unanswered: self-hosted, recent version, reachable on the internal network._

> **Answer:**

### G2. Is the Grafana Image Renderer installed? If not, can Devoli run it next to Grafana?

_Why it matters: without it, PIRs get Grafana links instead of embedded graph images. The renderer needs a fair amount of memory (about 16 GiB and 4 cores recommended)._
_Working assumption if unanswered: not installed; Grafana gives links only._

> **Answer:**

### G3. Which dashboards and panels matter for Incidents and PIRs, and what variables do they use (device, interface)?

For example: core/backbone, interface utilisation, BNG sessions, customer circuits.

_Why it matters: Admins set up "which graph for which kind of equipment" from this list._
_Working assumption if unanswered: Admins configure them later in the app._

> **Answer:**

### G4. Do the dashboard variable values (device, interface) match the Boris or LibreNMS names?

_Why it matters: we fill the variables from Boris names automatically._
_Working assumption if unanswered: they match LibreNMS names._

> **Answer:**

### G5. Is there a token expiry policy for service accounts?

_Why it matters: an expired token stops evidence capture, so we plan rotation around it._
_Working assumption if unanswered: no expiry; rotated manually._

> **Answer:**

---

## 6. Kentik

Kentik supplies traffic graphs and data for PIR evidence, and deep links for engineers. Access is **read-only**. Build module: 9.10 Evidence ([spec §5.5](../spec.md#55-librenms-grafana-kentik-evidence)).

**Access to provision:**

- [ ] A dedicated Kentik user, e.g. `outage-manager@devoli.co`, with the **Member** role (not Administrator).
- [ ] That user's email and API token, stored in Vault at `secret/outage-manager/<env>/kentik` (keys `email`, `token`). Vault path: `________`
- [ ] Kentik cluster/API host (`api.kentik.com` or `api.kentik.eu`): `________`
- [ ] Firewall: HTTPS egress from the Outage Manager host to the Kentik API.

**Questions:**

### K1. Which Kentik cluster is Devoli on: US or EU?

_Why it matters: it decides the API address and firewall egress._
_Working assumption if unanswered: US (`api.kentik.com`)._

> **Answer:**

### K2. Which plan is Devoli on, and how long is full-resolution data kept?

_Why it matters: evidence captured after full-resolution data expires is less precise._
_Working assumption if unanswered: at least 30 days of full data._

> **Answer:**

### K3. Can we have a dedicated Member user? Do other integrations already use the shared API query limit (1,500 queries per hour)?

_Why it matters: we share that limit with any other tools, so we need to leave headroom._
_Working assumption if unanswered: yes; no other heavy API users._

> **Answer:**

### K4. Do Kentik device and interface names match LibreNMS and Boris? Are any overridden by hand in Kentik?

_Why it matters: we fill Kentik queries from Boris names._
_Working assumption if unanswered: they match._

> **Answer:**

### K5. How, if at all, are Customers represented in Kentik (custom dimension, saved filters, interface description)?

_Why it matters: customer-level traffic filters are a later feature; this sizes it._
_Working assumption if unanswered: not represented; evidence is per device and interface._

> **Answer:**

### K6. Does Devoli use Kentik Synthetics, and would latency/loss evidence be wanted?

_Why it matters: it's a possible later addition to PIR evidence._
_Working assumption if unanswered: not in the MVP._

> **Answer:**

### K7. Has Kentik announced a replacement for, or retirement of, the V5 Query API?

_Why it matters: we build on the V5 API and need warning of any change._
_Working assumption if unanswered: no announcement._

> **Answer:**

---

## 7. Microsoft 365 (SharePoint and Provider Notice mailbox)

Two uses, both through Microsoft Graph with app-only (no user) access and certificate authentication:

1. **SharePoint publishing:** approved PIR PDFs are uploaded to a SharePoint library as the official record, and linked from Status.io. Build module: 9.9 SharePoint publishing ([spec §5.6](../spec.md#56-microsoft-graph-sharepoint-publishing)).
2. **Provider Notice mailbox:** Outage Manager reads carrier maintenance/outage emails from one shared mailbox and moves them into `Processed` / `Failed` folders. Build module: 9.11 Provider Notices ([spec §5.7](../spec.md#57-microsoft-graph-or-imap-provider-notice-mailbox)).

**Access to provision:**

- [ ] **App registration A, "Outage Manager – SharePoint":**
  - [ ] Application permission **`Sites.Selected`** (admin-consented). No tenant-wide `Sites.*` permissions.
  - [ ] A site permission grant for this app on the **PIR site only**, role **`write`** (or `owner` only if the spike shows sharing links or column creation need it). Site URL: `________`
  - [ ] Libraries "PIR – Public" and (optionally) "PIR – Internal" created on that site, with the metadata columns `IncidentNumber`, `Priority`, `PublishedAt`, `PIRRevision`.
- [ ] **App registration B, "Outage Manager – Mailbox":**
  - [ ] Application permission **`Mail.ReadWrite`**, scoped to **one** shared mailbox through RBAC for Applications (or an application access policy), so it can't read any other mailbox.
  - [ ] The shared mailbox itself, e.g. `provider-notices@devoli.co`, with folders `Processed` and `Failed`. Address: `________`
- [ ] For each registration: tenant ID, client ID (not secret; may be written here) and a **certificate** credential. Upload the public certificate to Entra; put the certificate and private key in Vault at `secret/outage-manager/<env>/graph-sharepoint` and `secret/outage-manager/<env>/graph-mailbox` (keys `tenant_id`, `client_id`, `cert_pem`, `key_pem`). **No client secrets.** Vault paths: `________`
- [ ] Firewall: HTTPS egress to `graph.microsoft.com` and `login.microsoftonline.com`.
- [ ] A SharePoint **sandbox** (test site) for the publishing spike, if possible.

**Questions:**

### M1. Does the tenant allow "Anyone" (anonymous) sharing links on SharePoint? If so, is an expiry enforced, and for how many days?

_Why it matters: the public PIR link on Status.io must never expire. If Anyone links can't be permanent, Outage Manager serves the PDF itself at a stable `/pir/INC-….pdf` address._
_Working assumption if unanswered: not allowed; Outage Manager serves the public PDF._

> **Answer:**

### M2. Could the PIR site alone allow non-expiring Anyone links (a per-site override)?

_Why it matters: it would let SharePoint host the public link directly._
_Working assumption if unanswered: no._

> **Answer:**

### M3. Which site and libraries should hold PIRs, who owns them, and which staff groups may see the Internal library?

_Why it matters: this sets where the app is granted access, and who can read internal PIR details._
_Working assumption if unanswered: a new "PIR" site with "PIR – Public" and "PIR – Internal" libraries; NOC and managers can read Internal._

> **Answer:**

### M4. Will site owners pre-create the metadata columns, or should the app be allowed (`owner`) to create them?

_Why it matters: pre-created columns keep the app on the smaller `write` permission._
_Working assumption if unanswered: site owners pre-create them._

> **Answer:**

### M5. Which versioning mode is used, and do PIRs need a retention label, retention policy or eDiscovery hold?

_Why it matters: revised PIRs overwrite the same file, relying on SharePoint version history._
_Working assumption if unanswered: automatic versioning; no special retention._

> **Answer:**

### M6. For app certificates: CA-issued or self-signed, what lifetime, and who rotates them? Is there an app management policy that blocks client secrets?

_Why it matters: an expired certificate stops publishing and mail ingestion; we need a named owner and a date._
_Working assumption if unanswered: self-signed, 12-month lifetime, rotated by the M365 admin with an overlap period._

> **Answer:**

### M7. Do any Conditional Access or workload identity policies (e.g. IP restrictions for service principals) apply to app sign-ins? Which egress IPs should they allow?

_Why it matters: such a policy could block calls from the RHEL host._
_Working assumption if unanswered: none apply._

> **Answer:**

### M8. Should SharePoint publishing and mailbox reading use one app registration or two?

_Why it matters: two registrations keep each one to the smallest permission set._
_Working assumption if unanswered: two separate registrations._

> **Answer:**

### M9. Is the mail platform Microsoft 365 / Exchange Online? Where do carrier notices arrive today (shared mailbox, distribution list, personal inboxes)?

_Why it matters: we need one dedicated mailbox carriers send to (or are forwarded/BCC'd to). If it isn't Exchange Online, we fall back to IMAP._
_Working assumption if unanswered: Exchange Online, with a new dedicated shared mailbox._

> **Answer:**

### M10. Who can grant the mailbox app registration access scoped to that one mailbox?

_Why it matters: we need to know who to ask._
_Working assumption if unanswered: the M365 / Exchange admin._

> **Answer:**

### M11. How long should raw carrier emails be kept?

_Why it matters: Outage Manager stores each original email as evidence._
_Working assumption if unanswered: kept as long as the Provider Notice, with no automatic deletion._

> **Answer:**

---

## 8. HashiCorp Vault

Vault holds **every** secret Outage Manager uses. The only secret on the host is the Vault login file. Build module: 9.1 Foundation ([spec §5.10](../spec.md#510-hashicorp-vault)).

**Access to provision:**

- [ ] A KV v2 prefix for Outage Manager, e.g. `secret/outage-manager/prod/` and `secret/outage-manager/test/`.
- [ ] A policy `outage-manager-<env>` granting **read** on `secret/outage-manager/<env>/*` only (no list or write on anything else).
- [ ] An **AppRole** `outage-manager-<env>` bound to that policy. The `role_id` and `secret_id` are written by the Vault admin (or host admin) straight into `/etc/outage-manager/vault-approle` on the host (root-only, mode 0400). They are never sent by email, chat or GitHub.
- [ ] Write access for the people who put integration credentials in (the system owners in this document), limited to that prefix.
- [ ] Firewall: HTTPS from the Outage Manager host to `vault.devoli.co`.

**Questions:**

### VA1. Is AppRole the right login method for a Docker host, or does Devoli prefer another (e.g. TLS certificate, JWT)?

_Why it matters: it decides how the app logs in to Vault at start-up._
_Working assumption if unanswered: AppRole._

> **Answer:**

### VA2. Is `secret/outage-manager/<env>/<integration>` on KV v2 an acceptable path layout, or does Devoli have a naming convention?

_Why it matters: every path in this document follows it._
_Working assumption if unanswered: accepted as proposed._

> **Answer:**

### VA3. What are the `secret_id` lifetime and the token TTL rules, and who rotates the `secret_id`?

_Why it matters: if the `secret_id` expires, the app can't restart; we alert before that._
_Working assumption if unanswered: `secret_id` doesn't expire; token TTL 1 hour, renewable; rotated by the Vault admin._

> **Answer:**

### VA4. Does `vault.devoli.co` use a certificate from an internal CA?

_Why it matters: the containers must trust that CA._
_Working assumption if unanswered: publicly trusted certificate._

> **Answer:**

---

## 9. Slack

Outage Manager posts internal alerts (new Draft Incidents, overdue sign-offs and PIRs, integration failures) to the **NOC Channel**. Build module: 9.12 Slack NOC Channel alerts and reminders ([spec §5.8](../spec.md#58-slack-noc-channel)).

**Access to provision:**

- [ ] An **incoming webhook** for the NOC Channel (from a Slack app owned by Devoli), stored in Vault at `secret/outage-manager/<env>/slack` (key `webhook_url`). The webhook URL **is a secret**. Vault path: `________`
  - Or, if a Slack app is preferred: a bot token with **`chat:write`** only, invited to the NOC Channel (key `bot_token`).
- [ ] A separate test channel (and its own webhook) for non-production.
- [ ] Firewall: HTTPS egress to `slack.com` / `hooks.slack.com`.

**Questions:**

### SL1. Which Slack workspace and channel is the NOC Channel?

_Why it matters: this is where all operational alerts go._
_Working assumption if unanswered: the existing NOC channel in Devoli's main workspace._

> **Answer:**

### SL2. Incoming webhook or Slack app with a bot token?

_Why it matters: a webhook is simplest; a bot token allows more later (e.g. threads). Both are supported._
_Working assumption if unanswered: incoming webhook to start._

> **Answer:**

### SL3. Which email address should get NOC alerts if Slack is down?

_Why it matters: alerts fall back to email after Slack fails._
_Working assumption if unanswered: the NOC distribution list; address to be supplied._

> **Answer:**

---

## 10. DNS and TLS

Outage Manager is served by Caddy, which gets its certificate from Let's Encrypt using the **DNS-01 challenge**, because the UI is not reachable from the internet. Passkeys are tied to the hostname, so it **must be final before go-live**. Build modules: 9.1 Foundation and 9.2 Auth ([spec §7.3](../spec.md#73-tls-and-hostname)). Decision background: [Deployment exposure, TLS and hostname](https://github.com/callumbnz/outage-manager/issues/19).

**Access to provision:**

- [ ] Internal DNS record for the UI hostname (proposed **`outages.devoli.com`**) pointing to the Outage Manager host.
- [ ] Public DNS for the public routes (the same name via the DMZ, or a separate public name; see D2).
- [ ] A **DNS provider API token** that can only create and delete `_acme-challenge` TXT records for that name (zone-scoped at most), stored in Vault at `secret/outage-manager/<env>/dns` (key `api_token`). Vault path: `________`
- [ ] Firewall: HTTPS egress to Let's Encrypt (`acme-v02.api.letsencrypt.org`) and the DNS provider's API.

**Questions:**

### D1. Is `outages.devoli.com` the final hostname for the UI?

_Why it matters: passkeys are bound to it. Changing it later forces every user to set up their passkeys again._
_Working assumption if unanswered: `outages.devoli.com`._

> **Answer:**

### D2. Should the public routes (PagerDuty webhook, public PIR PDFs) use the same name, or a separate public name?

_Why it matters: public PIR links on Status.io use this name permanently._
_Working assumption if unanswered: the same name, published through the DMZ._

> **Answer:**

### D3. Which DNS provider hosts `devoli.com`?

_Why it matters: Caddy is built with that provider's plugin to answer the DNS-01 challenge._
_Working assumption if unanswered: unknown; needed before go-live._

> **Answer:**

### D4. Can the DNS token be limited to TXT records for the challenge name (or at least one zone)?

_Why it matters: a full-account DNS token is a large risk if leaked._
_Working assumption if unanswered: zone-scoped token._

> **Answer:**

### D5. Where should the container image be stored: GitHub Container Registry or a Devoli registry?

_Why it matters: CI pushes the built image there and the host pulls from it._
_Working assumption if unanswered: GitHub Container Registry._

> **Answer:**

---

## 11. SMTP relay

Outage Manager sends email for invites, password and passkey resets, reminders, approval requests and the Slack fallback. Build modules: 9.2 Auth and 9.12 ([spec §5.9](../spec.md#59-smtp)).

**Access to provision:**

- [ ] Relay host and port (STARTTLS): `________`
- [ ] A sending address, e.g. `outage-manager@devoli.co`, allowed to send through the relay (and SPF/DKIM coverage for it).
- [ ] Credentials, only if the relay requires them, stored in Vault at `secret/outage-manager/<env>/smtp` (keys `username`, `password`). Vault path: `________`
- [ ] Firewall: SMTP from the Outage Manager host to the relay.

**Questions:**

### E1. Which SMTP relay should Outage Manager use, and is it reachable from the Docker host?

_Why it matters: without email, new users can't be invited and nobody can reset a login._
_Working assumption if unanswered: the existing internal relay, reachable on port 587._

> **Answer:**

### E2. Does the relay need authentication, or does it allow the host by IP?

_Why it matters: decides whether we need credentials in Vault._
_Working assumption if unanswered: allowed by IP; no credentials._

> **Answer:**

### E3. Which From address should be used?

_Why it matters: it must pass SPF/DKIM so invites don't land in spam._
_Working assumption if unanswered: `outage-manager@devoli.co`._

> **Answer:**

---

## 12. Network and DMZ

The UI is **internal only** (VPN / office). Only two paths are published through the existing DMZ reverse proxy / WAF. Build modules: 9.1 Foundation, 9.8 PagerDuty and 9.9 SharePoint publishing ([spec §7.2](../spec.md#72-network-exposure)).

**Access to provision:**

- [ ] DMZ reverse proxy / WAF rule publishing **only**:
  - [ ] `POST /webhooks/pagerduty/`, allowed only from **PagerDuty's published webhook IP ranges** (US or EU, see P1).
  - [ ] `GET`/`HEAD /pir/*.pdf` (public PIR PDFs), open to the internet and rate-limited.
  - [ ] Everything else from the DMZ is blocked.
- [ ] The DMZ proxy's source address(es), so Caddy accepts those two paths only from it: `________`
- [ ] Internal access to the UI (`outages.devoli.com`, port 443) from the VPN and office networks.
- [ ] Outbound (egress) from the Outage Manager host:

  | Destination | Port | For |
  |---|---|---|
  | Boris | 443 | Inventory sync |
  | LibreNMS | 443 | Evidence, device mapping |
  | Grafana (+ renderer) | 443 | Evidence |
  | `vault.devoli.co` | 443 | Secrets |
  | SMTP relay | 587 / 25 | Email |
  | `api.status.io` | 443 | Status.io |
  | `api.pagerduty.com`, `identity.pagerduty.com` | 443 | PagerDuty (read) |
  | `graph.microsoft.com`, `login.microsoftonline.com` | 443 | SharePoint, mailbox |
  | `api.kentik.com` (or `api.kentik.eu`) | 443 | Evidence |
  | `slack.com`, `hooks.slack.com` | 443 | NOC Channel |
  | Let's Encrypt ACME + DNS provider API | 443 | TLS certificates |
  | Container registry | 443 | Image pulls |

- [ ] Inbound from Boris to the Outage Manager host (443, `/webhooks/boris/`), internal only.

**Questions:**

### N1. Can the existing DMZ reverse proxy / WAF publish just these two paths to an internal host?

_Why it matters: without the webhook path we fall back to polling PagerDuty every 2 minutes; without `/pir/*.pdf`, public PIR links need SharePoint Anyone links._
_Working assumption if unanswered: yes._

> **Answer:**

### N2. Can the WAF allowlist PagerDuty's webhook IP ranges and keep them up to date from PagerDuty's published list?

_Why it matters: it's a second layer of protection on top of signature checks._
_Working assumption if unanswered: yes, refreshed by the network team._

> **Answer:**

### N3. Does the Outage Manager host need a proxy for internet egress? If so, which one?

_Why it matters: every SaaS integration must be configured to use it._
_Working assumption if unanswered: direct egress, no proxy._

> **Answer:**

### N4. What are the host's public egress IPs?

_Why it matters: Microsoft 365 Conditional Access or other SaaS allowlists may need them (see M7)._
_Working assumption if unanswered: to be supplied by the network team._

> **Answer:**

### N5. Which network segment will the RHEL 9 host sit in, and who provisions it?

_Why it matters: it decides which firewall rules above are needed._
_Working assumption if unanswered: an internal server segment; provisioned by infrastructure._

> **Answer:**

---

## 13. Backups

Outage Manager's database and files (PIR PDFs, evidence, carrier emails) are backed up nightly and **kept for 30 days**. Boris data doesn't need backing up (it can be re-synced), and Vault and SharePoint are backed up by their owners. Build module: 9.1 Foundation ([spec §7.5](../spec.md#75-backups)).

**Access to provision:**

- [ ] A backup target for a nightly Postgres dump plus a copy of `/srv/outage-manager/media`: `________`
- [ ] Credentials for it, if any, stored in Vault at `secret/outage-manager/<env>/backup`. Vault path: `________`
- [ ] A time slot for a restore test before go-live.

**Questions:**

### BK1. Where should nightly backups go (existing backup system, NFS share, S3-compatible store)?

_Why it matters: we set the backup job up for that target._
_Working assumption if unanswered: the existing Devoli backup system picks up files from the host._

> **Answer:**

### BK2. Is 30 days' retention right, and are there any compliance requirements for PIRs or evidence?

_Why it matters: PIRs and evidence may need to be kept longer than routine backups._
_Working assumption if unanswered: 30 days of backups; records stay in the app indefinitely._

> **Answer:**

### BK3. Is there an S3-compatible object store Outage Manager could use for its files instead of a host volume?

_Why it matters: it would simplify backups, but a backed-up host volume works._
_Working assumption if unanswered: no; a host volume is used._

> **Answer:**

### BK4. Who owns the restore test before go-live?

_Why it matters: an untested backup isn't a backup._
_Working assumption if unanswered: the Outage Manager team, with the infrastructure team._

> **Answer:**

---

## 14. Provider Notice samples

Custom parsers for NZ carrier emails are **deferred unless sample emails arrive**. Standard-format notices are handled from day one; the rest go to manual triage. Real samples let us build parsers for the carriers that matter most. Build module: 9.11 Provider Notices ([spec §5.7](../spec.md#57-microsoft-graph-or-imap-provider-notice-mailbox)). Owner: the NOC team.

**Access to provision:**

- [ ] **3–5 real emails per top carrier**, saved as `.eml` files (with headers), covering: new maintenance, update/reschedule, cancellation, completion and (if they send them) unplanned outage notices.
- [ ] Shared through a private location (e.g. a restricted SharePoint folder) and **not committed to GitHub**, because they contain circuit IDs and contact details. Location: `________`

**Questions:**

### PN1. Which carriers send the most notices, and which are NZ-domestic vs international?

For example: Chorus, LFCs (Enable, Tuatahi, Northpower), Spark Wholesale, Vocus, One NZ, FX, Kordia, Southern Cross, Hawaiki, Telstra, Megaport.

_Why it matters: parsers are built in this order._
_Working assumption if unanswered: Chorus, Spark Wholesale and Vocus first._

> **Answer:**

### PN2. Do carriers quote the same circuit ID that's recorded in Boris? Do any quote an order ID, service ID or account number instead?

_Why it matters: notices are matched to Devoli circuits by the ID in the email._
_Working assumption if unanswered: carriers quote the Boris circuit ID; exceptions are mapped by hand once and remembered._

> **Answer:**

### PN3. Can carriers be pointed at (or BCC'd to) the new dedicated mailbox?

_Why it matters: only mail in that mailbox is ingested._
_Working assumption if unanswered: yes, by a forwarding rule from the current address._

> **Answer:**

### PN4. Do any carriers notify only through a portal or by phone?

_Why it matters: those can only be entered by hand in Outage Manager._
_Working assumption if unanswered: a few; they use the manual entry form._

> **Answer:**

### PN5. Would sending carrier emails to an external AI service for parsing ever be acceptable?

_Why it matters: AI parsing is deferred and off by default; this tells us whether it's worth planning._
_Working assumption if unanswered: no._

> **Answer:**

---

## Anything else?

Is there anything about your system that we didn't ask but should know, such as planned upgrades, migrations, change freezes, or other tools that already use the same API?

> **Answer:**

---

## Next steps

1. Callum sends each owner the link to their section.
2. Owners provision access, put credentials in Vault, and record the **Vault path** and answers here or on [#2](https://github.com/callumbnz/outage-manager/issues/2) / [#15](https://github.com/callumbnz/outage-manager/issues/15).
3. Someone with access to the Status.io test page runs checks V1–V6 and records the results on [Status.io behaviour checks on a test page](https://github.com/callumbnz/outage-manager/issues/16).
4. Answers that differ from a working assumption are written into [spec §10](../spec.md#10-assumptions-to-verify), and into the affected module section if the change is material.
5. As each system's access and answers are in, the **verify** step of the matching build ticket is closed: the adapter is switched from its fake to the live system and tested against real data.

   | System | Build module ([spec §9](../spec.md#9-mvp-modules-in-build-order)) |
   |---|---|
   | Vault, DNS/TLS, Network/DMZ, Backups | 9.1 Foundation |
   | SMTP, hostname | 9.2 Auth |
   | Boris | 9.5 Boris sync and Impact |
   | Status.io (+ checks V1–V6) | 9.6 Status.io publishing |
   | PagerDuty, LibreNMS alert template | 9.8 PagerDuty |
   | Microsoft 365: SharePoint | 9.9 SharePoint publishing |
   | LibreNMS, Grafana, Kentik | 9.10 Evidence |
   | Microsoft 365: mailbox, Provider Notice samples | 9.11 Provider Notices |
   | Slack | 9.12 Slack NOC Channel alerts and reminders |

6. When every row in the [summary table](#summary) is ticked, [Provision API access to integrated systems](https://github.com/callumbnz/outage-manager/issues/2) is closed. When every question is answered or its working assumption is accepted, [Discovery questionnaire for integration owners](https://github.com/callumbnz/outage-manager/issues/15) is closed. When V1–V6 are recorded, [Status.io behaviour checks on a test page](https://github.com/callumbnz/outage-manager/issues/16) is closed.
