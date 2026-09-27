# Zendesk API for Support and Customer Notifications

Resolves wayfinder ticket [#5](https://github.com/callumbnz/outage-manager/issues/5) on map [#1](https://github.com/callumbnz/outage-manager/issues/1).

**Question.** Which Zendesk APIs create internal tickets, notify organisations and end users proactively, and link or update related tickets? How are organisations modelled, and how could they map to Boris Customers? What are the auth and rate limits?

**Status.** This is researched from primary Zendesk sources only (developer.zendesk.com and support.zendesk.com), fetched 2026-09-27. We have **not** made any live calls against Devoli's Zendesk. Anything that depends on Devoli's plan, configuration or data is listed under [Open questions](#11-open-questions-for-devolis-zendesk-admins).

## TL;DR

- **Everything we need is in the Ticketing (Support) API.** It covers creating tickets (`POST /api/v2/tickets`, `create_many`), updating them (`PUT /api/v2/tickets/{id}`, `update_many`), linking through ticket `type` (`problem`/`incident` + `problem_id`), organisations (`external_id`, custom org fields, memberships) and webhooks for callbacks.
- **Auth must be OAuth.** Zendesk is **removing API tokens**. From 27 Oct 2026 no new tokens can be created, and on 30 Apr 2027 every remaining token stops working. Use a confidential OAuth client with the **`client_credentials`** grant, created by a dedicated Outage Manager service user. The existing plugin probably uses an API token, which is a cutover risk.
- **Rate limits depend on plan:** 200 (Team) / 400 (Growth, Professional) / 700 (Enterprise) / 2500 (Enterprise Plus or the High Volume add-on) requests per minute. Ticket updates also have their own limits: **30 updates per ticket per user per 10 min** and **100 per minute per account**. A 429 comes with a `Retry-After` header. Background jobs are capped at **30 in flight**.
- **Model:** make the **Support Notification** an internal `problem` ticket. Make each **Customer Notification** a separate `incident` ticket in that Customer's org, with the requester set to the org's notification contact and `problem_id` pointing at the Support Notification. Agents then see every Customer ticket from one place. Solving the problem solves all linked incidents, **and copies the solve comment to them** (a leak risk: see §5).
- **Map Customer to org with `organization.external_id` = Boris Tenant ID.** It is unique, case-insensitive, and searchable by exact match through `GET /api/v2/organizations/search?external_id=`. Cache the resolved org ID in a local mapping table.
- **There is no native mass email.** Zendesk says bulk campaigns need Marketplace apps, and its own Proactive Tickets app is **discontinued on 21 Oct 2026**. So proactive notices mean creating agent-submitted tickets **on behalf of** an end user with a public comment. The standard trigger "Notify requester of new proactive ticket" then emails them.
- **We found no documented idempotency key.** Ticket `external_id` is **not unique**. Instead, stamp a deterministic `external_id` and tag on each ticket, and check `GET /api/v2/tickets?external_id=` before creating (with a local DB record as the lock).
- **Sandboxes:** Support Enterprise includes 1 and Suite Enterprise/Enterprise Plus include 2. They are a paid add-on from Suite Growth or Support Professional up.

## Sources

| Short name | Source |
|---|---|
| `RL` | <https://developer.zendesk.com/api-reference/introduction/rate-limits/> |
| `RL-BP` | <https://developer.zendesk.com/documentation/api-basics/best-practices/best-practices-for-avoiding-rate-limiting/> |
| `AUTH` | <https://developer.zendesk.com/api-reference/introduction/security-and-auth/> |
| `GRANT` | <https://developer.zendesk.com/api-reference/ticketing/oauth/grant_type_tokens/> |
| `OAUTH-TOK` | <https://developer.zendesk.com/api-reference/ticketing/oauth/oauth_tokens/> |
| `HC-OAUTH` | <https://support.zendesk.com/hc/en-us/articles/4408845965210> (Using OAuth authentication with your application) |
| `HC-OAUTH-MGR` | <https://support.zendesk.com/hc/en-us/articles/8889508417946> (Managing OAuth token access to the API) |
| `HC-TOKEN-EOL` | <https://support.zendesk.com/hc/en-us/articles/10840968198042> (Announcing the removal of API tokens) |
| `HC-AUTH` | <https://support.zendesk.com/hc/en-us/articles/4408831452954> (How can I authenticate API requests?) |
| `TICKETS` | <https://developer.zendesk.com/api-reference/ticketing/tickets/tickets/> |
| `TICKETS-GUIDE` | <https://developer.zendesk.com/documentation/ticketing/managing-tickets/creating-and-updating-tickets/> |
| `COMMENTS` | <https://developer.zendesk.com/api-reference/ticketing/tickets/ticket_comments/> |
| `JOBS` | <https://developer.zendesk.com/api-reference/ticketing/ticket-management/job_statuses/> |
| `FIELDS` / `FORMS` | <https://developer.zendesk.com/api-reference/ticketing/tickets/ticket_fields/>, <https://developer.zendesk.com/api-reference/ticketing/tickets/ticket_forms/> |
| `HC-FORMS` | <https://support.zendesk.com/hc/en-us/articles/4408846520858> (Creating multiple ticket forms) |
| `ORGS` | <https://developer.zendesk.com/api-reference/ticketing/organizations/organizations/> |
| `ORG-FIELDS` | <https://developer.zendesk.com/api-reference/ticketing/organizations/organization_fields/> |
| `MEMBERSHIPS` | <https://developer.zendesk.com/api-reference/ticketing/organizations/organization_memberships/> |
| `HC-MULTIORG` | <https://support.zendesk.com/hc/en-us/articles/4408838140314> (Enabling multiple organizations for users) |
| `USERS` | <https://developer.zendesk.com/api-reference/ticketing/users/users/> |
| `SEARCH` | <https://developer.zendesk.com/api-reference/ticketing/ticket-management/search/> |
| `HC-PROBINC` | <https://support.zendesk.com/hc/en-us/articles/4408835103898> (Working with problem and incident tickets) |
| `HC-LINK` | <https://support.zendesk.com/hc/en-us/articles/4408885828762> (Is there a way to link tickets in Support?) |
| `SIDE` | <https://developer.zendesk.com/api-reference/ticketing/side_conversation/side_conversation/> |
| `HC-PROACTIVE` | <https://support.zendesk.com/hc/en-us/articles/4408828120218> (Using the Proactive Tickets app) |
| `HC-MASS` | <https://support.zendesk.com/hc/en-us/articles/4408887422746> (Can I send out mass email campaigns?) |
| `HC-STDTRIG` | <https://support.zendesk.com/hc/en-us/articles/4408828984346> (About the standard ticket triggers) |
| `WEBHOOKS` | <https://developer.zendesk.com/api-reference/webhooks/webhooks-api/webhooks/> |
| `WH-VERIFY` | <https://developer.zendesk.com/documentation/webhooks/verifying/> |
| `WH-EVENTS` | <https://developer.zendesk.com/api-reference/webhooks/event-types/webhook-event-types/> |
| `HC-WEBHOOK` | <https://support.zendesk.com/hc/en-us/articles/4408839108378> (Creating webhooks to interact with third-party systems) |
| `HC-SANDBOX` | <https://support.zendesk.com/hc/en-us/articles/6150628316058> (About Zendesk sandbox environments) |
| `DEV-TRIAL` | <https://developer.zendesk.com/documentation/developer-tools/getting-started/getting-a-trial-or-sponsored-account-for-development/> |

---

## 1. Authentication

**Options** (`AUTH`, `HC-AUTH`)

| Method | Header | Status |
|---|---|---|
| OAuth access token | `Authorization: Bearer {token}` | **Recommended.** Tokens are tied to a single Zendesk instance |
| API token | `Authorization: Basic base64({email}/token:{api_token})` | **Deprecated.** "Can be used to impersonate anyone in the account, including admins" |
| Email + password | Basic | Not suitable for an integration |

**Timeline for removing API tokens** (`HC-TOKEN-EOL`)

- **28 Jul 2026:** tokens unused for 30 days are deactivated, then deleted after a further 60 days. New accounts cannot use tokens at all.
- **27 Oct 2026:** no new API tokens can be created, through either the UI or the API.
- **30 Apr 2027:** all remaining tokens are permanently deactivated.
- This affects the Ticketing, Help Center and Voice APIs.

**OAuth for a server-side integration** (`GRANT`, `HC-OAUTH`, `HC-OAUTH-MGR`)

- An admin creates an OAuth client under Admin Center → Apps and integrations → APIs → OAuth clients. Choose client kind **Confidential**, which allows a client secret.
- **`client_credentials` grant:** `POST https://{subdomain}.zendesk.com/oauth/tokens` with `grant_type=client_credentials`, `client_id`, `client_secret`, and optionally `scope` and `expires_in`. It needs no user interaction and returns **no refresh token**; you simply request a new token. "The token's user will be the one associated with the client."
- **If the admin who created the client loses access, `client_credentials` tokens stop working** (`HC-OAUTH`). So create the client **as a dedicated Outage Manager service user** (an admin or a suitably privileged agent), not as a person's account.
- **Token lifetimes** for clients created on or after 30 Apr 2026: access tokens default to 30 min (minimum 5 min, maximum 48 h), and refresh tokens default to 30 days (minimum 7, maximum 90 days). For `client_credentials`, `expires_in` must be between 300 and 172,800 s (`GRANT`).
- **Scopes** can restrict a token to read-only or to specific resources. Our token needs to read and write tickets, and to read organisations and users.
- Tickets created by the service user show "created by Outage Manager". To attribute a ticket to the Author, set `submitter_id` or `comment.author_id` to the Author's agent ID. This needs a mapping from Outage Manager users to Zendesk agent IDs, which is optional.

**Recommendation.** Build on OAuth `client_credentials` from day one. Keep the client secret in our secrets store, cache the access token in Redis until shortly before expiry, and re-mint it on a 401.

## 2. Rate limits and how to handle 429s

**Account-wide limits per minute** (`RL`), for Support and Help Center combined (Help Center has its own separate bucket)

| Suite plan | Team | Growth | Professional | Enterprise | Enterprise Plus |
|---|---|---|---|---|---|
| req/min | 200 | 400 | 400 | 700 | 2500 |

Support-only plans: Team 200, Professional 400, Enterprise 700. The **High Volume API add-on** raises the limit to 2500. It is available on Suite Growth+ or Support Professional+ with at least 10 agent seats.

**Endpoint limits that matter to us** (`RL`, `TICKETS`)

| Endpoint | Limit |
|---|---|
| `PUT /api/v2/tickets/{id}` (and any PUT that updates tickets) | **30 updates per 10 min per user per ticket**; **100/min per account** (300 with High Volume). Sandbox and trial accounts get 20/min |
| `PUT /api/v2/organizations/{id}` | 5/min per organisation |
| `PUT /api/v2/users/{id}`, `POST /users/create_or_update` | 5/min per user |
| Search (`/api/v2/search`) | Has its own limit (`Zendesk-RateLimit-search-index` header). Max 1,000 results per query, 100 per page |
| Side conversations create/reply | 300 per 10 min |
| Background jobs (`create_many`, `update_many`, etc.) | **At most 30 queued or running** per account, otherwise a `TooManyJobs` error. Header: `zendesk-ratelimit-inflight-jobs: total=30; remaining=29; resets=60` |

**Headers** (`RL`, `RL-BP`)

- `X-Rate-Limit` / `ratelimit-limit`, `X-Rate-Limit-Remaining` / `ratelimit-remaining`, and `ratelimit-reset` (in seconds).
- Endpoint-specific headers such as `Zendesk-RateLimit-tickets-update: total=…; remaining=…; resets=…`.
- On a 429 the response carries **`Retry-After: <seconds>`**. Zendesk says to "stop making additional API requests until enough time has passed to retry". Check the account-wide header first; on a 429, check the endpoint header. Use exponential backoff.
- Zendesk may also throttle the whole account during unusual spikes ("Account limit").

**What this means for us.** Our volume is tiny compared with these limits: one Support Notification plus up to N Customer tickets per record, and a few updates each. The limit that bites is the **per-ticket update limit** (30 per 10 min) during a noisy Incident. Our Celery tasks should:

- serialise writes per ticket
- honour `Retry-After`
- coalesce rapid Incident updates into one comment

## 3. Creating tickets

**Endpoints** (`TICKETS`, `TICKETS-GUIDE`)

- `POST /api/v2/tickets`: `comment` is the only required property. Use `comment.html_body` for HTML. Returns 201 with the ticket and an audit.
- `POST /api/v2/tickets?async=true`: returns 202 with the ticket ID and a `job_status`. The ticket may not be fetchable until the job completes.
- `POST /api/v2/tickets/create_many`: **up to 100 tickets per call**. It runs as a background job, so poll `GET /api/v2/job_statuses/{id}` (states: `queued`, `working`, `failed`, `completed`). The results give `{id, index}` for each ticket, so results map back to request order (`JOBS`). Zendesk warns: "Every ticket created with this endpoint may be affected by your business rules, which can include sending email notifications to your end users."

**Key ticket properties** (`TICKETS`)

| Property | Use for us |
|---|---|
| `subject`, `comment.body` / `html_body` | Notice content |
| `comment.public` | `true` = public reply (emailed to requester/CCs by triggers); `false` = internal note. "The initial value set on ticket creation persists for any additional comment unless you change it" (`COMMENTS`) |
| `requester_id` / `requester` | Who the ticket is about. `requester` accepts an email, an ID or `{name, email}`, and may auto-create the user depending on account settings |
| `submitter_id` | Who created it. Defaults to the authenticated user and "always becomes the author of the first comment". An agent submitter means the ticket was created "on behalf of" the requester |
| `organization_id` | Must be an organisation the requester is a member of (`MEMBERSHIPS`) |
| `type` | `problem`, `incident`, `question` or `task` |
| `problem_id` | For `incident` tickets, the linked problem ticket |
| `status` | `new`, `open`, `pending`, `hold`, `solved` or `closed` (or `custom_status_id` if custom statuses are enabled) |
| `priority` | `urgent`, `high`, `normal` or `low` |
| `group_id`, `assignee_id` | Route the Support Notification to the NOC or support group |
| `tags` | e.g. `outage_manager`, `om_maintenance`, `om_record_<id>`. Also used in triggers and views |
| `custom_fields` | `[{id, value}]`, e.g. record type, window start/end, Outage Manager URL, impact level (`FIELDS`) |
| `ticket_form_id` | Requires multiple ticket forms, which means Suite Growth+ or Support Enterprise (`HC-FORMS`). A dedicated "Outage Manager" form is optional |
| `external_id` | "An id you can use to link Zendesk Support tickets to local records". **Not unique** (see §9) |
| `email_ccs` / `collaborators` / `additional_collaborators` | Extra recipients. **At most 48 email CCs per ticket.** Beyond that the create call returns a 400, and on update the list is silently truncated (`TICKETS-GUIDE`) |
| `metadata` | Custom audit metadata, e.g. the Outage Manager record ID and version |

**Requester and submitter for our two ticket kinds**

- **Support Notification:** requester = the Outage Manager service user (or a dedicated "NOC" user). Use an internal note, or a public comment with no end-user requester. Set `group_id` to support. This ticket should **never** email a Customer.
- **Customer Notification:** requester = the Customer's primary notification contact (an end user in that org), `submitter_id` = the service agent, first comment `public: true`. The standard trigger **"Notify requester of new proactive ticket"** fires when an agent creates a ticket with a public comment, and emails the requester (`HC-STDTRIG`). Put the other contacts in `email_ccs` (max 48).

## 4. Proactive outbound notification options

| Option | Verdict |
|---|---|
| **Agent-created ticket per Customer, on behalf of a contact** (`TICKETS`, `HC-STDTRIG`) | **Recommended.** Native, and emails go out through Devoli's normal support address and triggers. Replies land on the same ticket |
| `create_many` for many Customers at once | Use it when N is large (up to 100 per job, max 30 jobs). For a few Customers, sequential `POST /tickets` is simpler and returns IDs immediately |
| Proactive Tickets app | **Discontinued 21 Oct 2026** (`HC-PROACTIVE`). It was UI-only (200 tickets/min) anyway. Don't use it |
| Native mass email to every user in an org | **Not available.** "There is no native way to send out mass email campaigns within Zendesk Support" (`HC-MASS`). Emulate it by listing org users (`GET /api/v2/organizations/{id}/users`, 100 per page) and adding them as CCs (max 48), or by creating one ticket per contact |
| Side conversations from the Support Notification (`SIDE`) | Possible: one internal ticket, with a side-conversation email per Customer (child tickets are also possible). But it **requires the Collaboration add-on**, is limited to 300 per 10 min, and customer replies land in side conversations rather than normal tickets. Hold it as an alternative only |
| Zendesk messaging | Not suitable for outbound notices. When a problem is solved, "Customers are not notified using the messaging channel" (`HC-PROBINC`). Email is the channel |

**Double-notification with Status.io** (research [#4](https://github.com/callumbnz/outage-manager/issues/4)). Status.io emails its own subscribers, and its API calls carry per-call notify flags. A Customer contact who is also a Status.io subscriber would get two notices. Options:

- (a) Zendesk Customer Notifications only for Customers with specific Impact, while Status.io covers the broad or public audience. Accept some overlap, and make the Zendesk copy clearly "your services affected: X".
- (b) Turn Status.io notify flags off whenever a Zendesk Customer Notification is sent.
- (c) Per-Customer preference.

This is a policy decision, not an API constraint. See the suggested ticket below.

## 5. Linking: problem and incident tickets

**Mechanics** (`TICKETS`, `HC-PROBINC`, `HC-LINK`)

- Create the Support Notification with `type: "problem"`. Create each Customer ticket with `type: "incident"` and `problem_id: <support ticket id>`.
- `GET /api/v2/tickets/{problem_id}/incidents` lists the linked incidents. The `has_incidents` flag sits on the problem.
- The ticket **Type** field must be on the ticket form (`HC-PROBINC`). The feature is available on all Suite and Support plans.
- **Solving the problem ticket solves every linked incident.** "The comment added to the problem ticket when solved is added to all incident tickets that aren't already solved." Empty required fields on the incidents are ignored.
- Reopening a solved problem does **not** reopen its incidents.

**Risk.** The solve comment on the internal Support Notification is copied to every Customer ticket. If that comment is public, it is emailed to Customers, and it may contain internal detail. The help-centre article describes the UI flow. We have **not confirmed** whether a private (internal-note) solve comment propagates, or what visibility it gets. **Test this in a sandbox** before relying on it.

**Recommendation**

- Use problem/incident linking for **agent visibility and bulk solve**. Support sees every Customer ticket under the Support Notification.
- **Update Customer tickets explicitly** from Outage Manager with Customer-safe wording, using `update_many` batch updates (§6). Don't rely on text propagated from the internal ticket.
- When closing: first solve the Customer tickets with a public "resolved/completed" comment, then solve the problem with an internal note.

## 6. Updating and closing when the Maintenance or Incident changes

- **Single ticket:** `PUT /api/v2/tickets/{id}` with `comment` (public or internal), `status`, `custom_fields` or `tags`. It returns an audit.
- **Many tickets** (`TICKETS`): `PUT /api/v2/tickets/update_many`, up to 100 per job.
  - *Bulk* (`?ids=1,2,3` + one ticket object) applies the same change to every ticket. Use it, for example, for "Maintenance rescheduled to …" to all Customers.
  - *Batch* (an array of ticket objects with `id`) applies different changes per ticket.
  - `additional_tags` / `remove_tags` edit tags without overwriting them.
  - The response is a `job_status`, and the results report `{id, action, status, success}` for each ticket (`JOBS`).
- **Collision safety** (`TICKETS-GUIDE`): Zendesk uses optimistic locking and returns a 409 on collisions. For explicit safety, send `safe_update: true` plus `updated_stamp` (the last known `updated_at`). You get a 409 (single) or an `UpdateConflict` job result (bulk), so re-fetch and retry.
- **Closing:** set `status: "solved"`. Zendesk moves tickets to `closed` automatically later, per the account's automations. Closed tickets can't be reopened; a new reply creates a follow-up ticket (`via_followup_source_id`). So for Maintenance that is cancelled and then re-planned, create new tickets rather than trying to reopen closed ones.
- **Lifecycle mapping:**

| Outage Manager event | Zendesk action |
|---|---|
| Maintenance approved or Incident declared | Create the Support Notification (problem). If the Author opts in, create Customer tickets (incidents) |
| Maintenance rescheduled or Incident update | Internal note on the problem; public comment on the incidents via `update_many`, if Customer Notifications were sent |
| Maintenance started/completed, Incident resolved | Public comment and `solved` on the incidents; internal note and `solved` on the problem |
| Maintenance cancelled | Public "cancelled" comment and `solved` on the incidents, then solve the problem |
| PIR approved | Optional internal note on the (possibly closed → follow-up) Support Notification with the PIR link |

## 7. Organisations and mapping to Boris Customers (Tenants)

**Organisation model** (`ORGS`)

| Field | Notes |
|---|---|
| `id` | Zendesk numeric ID |
| `name` | Required and **unique** (whitespace-trimmed) |
| `external_id` | "A unique external id to associate organizations to an external record". **Case-insensitive** uniqueness |
| `domain_names` | Auto-assigns users by email domain |
| `group_id` | Default group for new tickets from the org |
| `organization_fields` | Custom org fields (`ORG-FIELDS`): text, textarea, checkbox, date, integer, decimal, regexp, dropdown, lookup, multiselect |
| `tags`, `notes`, `details`, `shared_tickets`, `shared_comments` | |

**Lookup** (`ORGS`, `SEARCH`)

- `GET /api/v2/organizations/search?external_id={id}` returns an exact, case-insensitive match. You can search by `external_id` **or** `name`, not both.
- `POST /api/v2/organizations/create_or_update` matches on `id` or `external_id` (not name). We will **not** create orgs; that stays with Zendesk admins. But the endpoint shows `external_id` is a first-class key.
- The unified Search API (`type:organization …`) can also query custom org fields. It is limited to 1,000 results and indexing lags by "a few minutes".
- `PUT /organizations/{id}` is limited to 5/min per org, which matters for a one-off backfill of `external_id`.

**Memberships and users** (`MEMBERSHIPS`, `HC-MULTIORG`, `USERS`)

- A user can belong to many organisations (up to 300) only if "multiple organizations" is enabled, which needs Suite Growth+ or Support Professional+. On Team plans a user belongs to exactly one.
- Each membership has `default` and `view_tickets`.
- `GET /api/v2/organizations/{id}/users` lists the members, 100 per page.
- A ticket's `organization_id` must be one of the requester's orgs. For multi-org contacts, set it explicitly so the ticket lands on the right Customer.

**Recommended mapping**

1. **Canonical key:** Zendesk `organization.external_id` = **Boris Tenant ID** (UUID, or slug if preferred; confirm with #3). It is unique, case-insensitive and directly searchable, so we store no Zendesk ID in Boris.
2. **Local cache:** a `CustomerZendeskOrg` mapping table in Outage Manager (Tenant ID → Zendesk org ID, name, last verified). It is refreshed by a periodic Celery sync that pages `GET /api/v2/organizations` and reads `external_id`. It is also managed in **our own UI** (not Django admin), where a user can link unmapped Tenants by searching org name. An "unmapped Customers" report stops Customers being silently skipped when notifications go out.
3. **Fallback,** if Devoli's admins won't set `external_id` or it's already used by another system: use a custom org field `boris_tenant_id`, or rely on the local mapping table alone. Research #3 also floated a Zendesk org ID custom field on the Boris Tenant. That works, but it makes Boris depend on Zendesk IDs, so keep it only as an option.

**Which contacts receive notices?** Zendesk has no built-in "notification contact" concept. Options:

- (a) a **user tag** (e.g. `outage_contact`) or a checkbox **custom user field** on end users, maintained by support in Zendesk, queried via Search `type:user organization:<id> tags:outage_contact` or by filtering the org user list
- (b) all org members, which is risky
- (c) contacts held in Outage Manager

(a) keeps contact data where support already works. Requester = first contact; CC = the rest, up to 48.

## 8. Webhooks and triggers back to us (optional)

- **Webhooks** (`WEBHOOKS`, `HC-WEBHOOK`): a webhook is either *connected to a trigger or automation* (`subscriptions: ["conditional_ticket_events"]`) or *subscribed to Zendesk events* (user, org and similar; `WH-EVENTS`). Ticket activity has to go through a trigger, e.g. "ticket updated AND tags include `om_customer_notification` AND comment is public AND current user is end user" → notify webhook. The request body is templated with placeholders.
- **Security** (`WH-VERIFY`): `X-Zendesk-Webhook-Signature` = `base64(HMAC-SHA256(timestamp + body))` with the webhook's signing secret, plus `X-Zendesk-Webhook-Signature-Timestamp`. Verify both and reject stale timestamps. Basic, bearer or API-key auth on the outbound request is also supported.
- **Delivery:** it is queued, **with no ordering guarantee**. It is retried up to 3 times on certain response codes, and failures don't deactivate the webhook. Trial accounts are limited to 10 webhooks.
- **Uses:** show "Customer replied" on the Incident timeline, alert the Author, or detect bounces. **This is not needed for the MVP**; Outage Manager can just link to the Zendesk tickets.

## 9. Idempotency

- In the Ticketing API pages we consulted, **we found no idempotency-key header** for ticket creation.
- Ticket `external_id` is **not unique**: "External ids don't have to be unique for each ticket… the request may return multiple tickets with the same external id" (`TICKETS`, List Tickets `?external_id=`).
- Async and `create_many` can skip ticket IDs on failure (`TICKETS-GUIDE`).

**Pattern**

1. Before calling Zendesk, write a local `ZendeskTicket` row with a deterministic key, e.g. `om:{maintenance|incident}:{record_id}:support` or `om:…:{record_id}:customer:{tenant_id}`. Give it a unique constraint and status `pending`.
2. Create the ticket with `external_id` = that key, plus tag `om_record_{id}`.
3. On retry, or if the result is unknown (a timeout, or a worker crash after the POST): `GET /api/v2/tickets?external_id={key}`. If it exists, adopt its ID; otherwise create.
4. For `create_many`, map `results[].index` → ticket ID, and reconcile any gaps with step 3.
5. Updates are naturally idempotent for status and fields. For comments, store a hash per update in Outage Manager, and use audit `metadata` to carry the Outage Manager update ID so duplicates can be detected.

## 10. Sandbox and test environments

- **Sandboxes** (`HC-SANDBOX`): Support Enterprise includes **1**, Suite Enterprise and Enterprise Plus include **2**, and Support Professional+ and Suite Growth+ can **buy** them as an add-on. They replicate settings, templates and much production data, including tickets and end users, but don't stay in sync. Update rate limits are lower in sandboxes (20/min account-wide for ticket updates, `TICKETS`). Beware: a sandbox holds **real end-user emails**, so a test that creates a public proactive ticket could email a real customer. Tests must use dedicated test orgs and users, or disable the notify triggers in the sandbox.
- **Without a sandbox:** a 14-day Zendesk trial account (`DEV-TRIAL`). Sponsored developer accounts are only for Marketplace partners.
- Our own automated tests should hit a recorded fake or local stub of these endpoints. Integration tests go against the sandbox.

## 11. Open questions for Devoli's Zendesk admins

1. **Plan tier:** Suite or Support, and Team, Growth, Professional or Enterprise? This decides the rate limit, multiple ticket forms, multi-org users, and whether a sandbox is included. Is High Volume or Collaboration bought?
2. **Current plugin:** how does it authenticate (API token → breaks by 30 Apr 2027; no new tokens after 27 Oct 2026)? Which ticket shape, tags, fields, triggers and views does it use, and what should we replicate during the parallel run?
3. **Org data quality:** does every Boris Tenant have exactly one Zendesk org? Is `organization.external_id` already populated or used by another integration? Are org names consistent with Boris?
4. **Notification contacts:** who at each Customer should receive notices? Is there an existing tag or user field (e.g. "technical contact")? Is "multiple organizations" enabled?
5. **Support routing:** which group and assignee should own the Support Notification? Is there an existing ticket form, fields, priorities or macros for outages?
6. **Triggers:** are "Notify requester of new proactive ticket" and "Notify requester and CCs of comment update" active and unmodified? Is there a branded support address for outage notices?
7. **Problem/incident usage:** is the Type field on the form support uses? Do agents already use problem/incident linking?
8. **Service user:** can we get a dedicated agent or admin seat for Outage Manager, to create the OAuth client and own the tickets?
9. **Status.io overlap:** which Customer contacts also subscribe to Status.io, and what is the preferred policy (see §4)?
10. **Retention and privacy:** anything we must avoid putting in customer-visible comments, such as internal hostnames?

## 12. Recommended approach (summary)

1. **Auth:** OAuth confidential client with `client_credentials`, owned by a dedicated service user. Cache the token and re-mint it on 401 or expiry.
2. **Support Notification:** one `type: problem` ticket per Maintenance or Incident. It goes to the support group, is tagged `outage_manager` + `om_<kind>` + `om_record_<id>`, has custom fields for record type, window and link, and uses `external_id = om:<kind>:<id>:support`.
3. **Customer Notifications** (Author opt-in): one `type: incident` ticket per affected Customer org, linked with `problem_id`. The requester is the primary notification contact, CCs are the other contacts (≤48), the submitter is the service agent, and the first comment is public (triggers email it). Use `external_id = om:<kind>:<id>:customer:<tenant_id>`. Use `create_many` only when there are more than ~20 Customers.
4. **Updates:** internal notes on the problem; Customer-safe public comments on the incidents via `update_many` (bulk). Serialise per ticket, honour `Retry-After`, use `safe_update`.
5. **Close:** solve the incidents with a public closing comment, then solve the problem with an internal note. Don't rely on text propagated from the problem until it's verified in the sandbox.
6. **Mapping:** org `external_id` = Boris Tenant ID, with a local mapping cache synced by Celery and managed in our UI. Report unmapped Customers before sending.
7. **Callbacks:** a trigger → webhook for customer replies, verified by HMAC. This comes after the MVP.
8. **Resilience:** all Zendesk calls run in Celery with a local outbox table (`pending` → `sent`/`failed`), the external_id lookup for idempotency, and a job-status poller for bulk jobs.
