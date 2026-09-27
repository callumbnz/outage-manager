# Boris/Nautobot API: deriving Impact

Resolves wayfinder ticket [#3](https://github.com/callumbnz/outage-manager/issues/3) on map [#1](https://github.com/callumbnz/outage-manager/issues/1).

**Question.** How do we walk from Network Elements (devices, interfaces, circuits) to Services and Customers through the Nautobot REST/GraphQL API? Which models, relationships and custom fields are standard, and what do we still need to confirm in Boris, Devoli's fork?

**Status.** Researched from primary sources only: Nautobot source and docs, and the official circuit-maintenance app. We have no Boris credentials yet (ticket #2), so every statement below describes **upstream Nautobot**. Anything specific to Boris is listed under [Open questions for Boris](#open-questions-for-boris).

## Sources and versions

- Nautobot core, `main` at commit [`f9cdca3`](https://github.com/nautobot/nautobot/tree/f9cdca3d4f36a0219cd57860b6511472d0870b13) (2026-09-25, version 3.2.6a0). The latest releases are **v3.2.5** (3.x line) and **v2.4.42** (last 2.x LTS). Where 2.x behaves differently, this doc says so.
- Nautobot docs: <https://docs.nautobot.com/projects/core/en/stable/>. These are the same Markdown files as `nautobot/docs/` in the repo.
- nautobot-app-circuit-maintenance, `develop` at commit [`fc50860`](https://github.com/nautobot/nautobot-app-circuit-maintenance/tree/fc508607f07c77e148dd3a94374cf66e73ac219c) (version 3.1.2a0, requires `nautobot >=3.1.0,<4.0.0`). Docs: <https://docs.nautobot.com/projects/circuit-maintenance/en/latest/>.

Short link prefixes used below:
- `NB` = `https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13`
- `CM` = `https://github.com/nautobot/nautobot-app-circuit-maintenance/blob/fc508607f07c77e148dd3a94374cf66e73ac219c`

> **Correction to the ticket's premise.** The ticket says "current 2.x", but Nautobot is now on 3.x. The version Boris is forked from affects API defaults, notably `exclude_m2m` (see [API mechanics](#rest-vs-graphql)). Confirming that version is question #1 for Boris.

---

## 1. Core models that matter for Impact

| Model | Why it matters | Key fields and links | Source |
|---|---|---|---|
| `dcim.Device` | A Network Element | `tenant` FK, `location`, `role`, `status`, reverse `interfaces` | [NB/nautobot/dcim/models/devices.py#L512](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/dcim/models/devices.py#L512) |
| `dcim.Interface` | A Network Element | Is a `CableTermination` and a `PathEndpoint`. Has `parent_interface`, `bridge`, `lag`, `untagged_vlan`, `tagged_vlans` and `ip_addresses`. **No `tenant` field** | [NB/nautobot/dcim/models/device_components.py#L1197](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/dcim/models/device_components.py#L1197) |
| `dcim.Cable` / `dcim.CablePath` | Physical connectivity. `CablePath` is a precomputed origin→destination path | `CablePath.origin`, `.destination`, `.path`, `.is_active` | [NB/nautobot/dcim/models/cables.py#L1420](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/dcim/models/cables.py#L1420) |
| `circuits.Circuit` | A Network Element | `provider`, `circuit_type`, **`tenant` FK**, `circuit_termination_a`/`_z` | [NB/nautobot/circuits/models.py#L123-L160](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/circuits/models.py#L123-L160) |
| `circuits.CircuitTermination` | The joint between a circuit and a device. It is a `PathEndpoint` and a `CableTermination` | `circuit`, `term_side` (A/Z), and exactly one of `location` / `provider_network` / `cloud_network` | [NB/nautobot/circuits/models.py#L192-L250](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/circuits/models.py#L192-L250) |
| `circuits.Provider` | Upstream carrier. It is the key for the circuit-maintenance app | Reverse `circuits` | [NB/nautobot/circuits/models.py#L34](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/circuits/models.py#L34) |
| `tenancy.Tenant` / `TenantGroup` | **Upstream's model for a customer** | `Tenant.tenant_group` (a tree) | [NB/nautobot/tenancy/models.py#L18-L60](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/tenancy/models.py#L18-L60), [docs: Tenants](https://docs.nautobot.com/projects/core/en/stable/user-guide/core-data-model/tenancy/tenant/) |
| `ipam.VLAN`, `Prefix`, `IPAddress`, `VRF` | Logical services riding on interfaces. Each has a `tenant` FK | `tenant` | [NB/nautobot/ipam/models.py](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/ipam/models.py) (VRF L220, Prefix L643, IPAddress L1684, VLAN L2354) |
| `ipam.Service` | **Not a customer service.** It is an L4 port service (for example HTTP or SSH) on a Device or VM | `device`/`virtual_machine`, `protocol`, `ports` | [NB/nautobot/ipam/models.py#L2498-L2505](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/ipam/models.py#L2498-L2505) |
| `dcim.Location` | Site/POP. Used to scope Impact, and by the circuit-maintenance overlap job | `tenant` FK, tree | [NB/nautobot/dcim/models/locations.py](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/dcim/models/locations.py) |
| `extras.Relationship` / `RelationshipAssociation` | User-defined links between any two models (1:1, 1:many, many:many, symmetric) | Exposed as `?include=relationships` in REST and as `rel_<key>` in GraphQL | [docs: Relationships](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/relationship/) |
| `extras.CustomField` | Arbitrary per-model fields stored as JSON on the object | `custom_fields` in REST, `cf_<key>` in GraphQL | [docs: Custom Fields](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/customfield/) |
| `extras.DynamicGroup` | Saved filter or set of objects, with cached membership | Members queryable both ways | [docs: Dynamic Groups](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/dynamicgroup/) |
| `extras.Tag` | Free labelling. Could mark "customer-facing" elements | `tags` (always included in REST) | [docs: Tags](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/tag/) |
| `extras.Contact` / `Team` / `ContactAssociation` | Contacts attachable to most models. A possible home for the Zendesk org or escalation contacts | | [NB/nautobot/extras/models/contacts.py](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/extras/models/contacts.py) |
| `vpn.*`, `cloud.*`, `load_balancers.*`, `wireless.*` | Newer 2.x/3.x models. `vpn`, `wireless` and `load_balancers` carry `tenant` FKs. Relevant if Devoli models L2/L3 VPN services this way | | [NB/nautobot/vpn/models.py](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/vpn/models.py) |

## 2. How "Service" and "Customer" are modelled in upstream Nautobot

- **Customer means Tenant.** The upstream docs say: "Typically, tenants are used to represent individual customers or internal departments… Tenant assignment is used to signify the *ownership* of an object… each object may only be owned by a single tenant… if the firewall serves multiple customers… tenant assignment would not be appropriate." ([docs: Tenants](https://docs.nautobot.com/projects/core/en/stable/user-guide/core-data-model/tenancy/tenant/))
  - The consequence is that **shared infrastructure, such as a core router, normally has no tenant.** A Maintenance on a shared device therefore has to find Customers downstream of it, through connected interfaces, circuits, VLANs or prefixes. The device's own `tenant` field won't tell us.
- **Nautobot core has no customer-service model.** `ipam.Service` is an L4 port/protocol record ([source](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/ipam/models.py#L2498-L2505)). It is not a product such as "Fibre 1G for Customer X". Operators usually represent a service with one of these:
  1. a **`Circuit`** with a `tenant`. This is the most natural fit for access and backhaul services an ISP sells;
  2. a **tenant-owned VLAN, Prefix, VRF or VPN**;
  3. a **custom model in an internal App**. A fork like Boris may well have one;
  4. **Relationships** or **Custom Fields**, for example a `service_id` custom field on Interface or Circuit, or a Relationship Interface ↔ Circuit.
- **Zendesk organisation.** Upstream has nothing built in. The likely homes are a Tenant custom field (for example `zendesk_org_id`), a Contact/Team, or a mapping kept in Outage Manager.

In short, the chain Network Element → Service → Customer → Zendesk org is **not standard in Nautobot**. Devoli's conventions decide it, so the Boris questions below matter more than anything else here.

## 3. Walking the chain: traversal primitives

### 3.1 Physical path (cable tracing)

- `Interface`, `CircuitTermination`, and console and power ports are **`PathEndpoint`s**. Nautobot precomputes `CablePath` rows between endpoints, stepping through front and rear pass-through ports ([source](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/dcim/models/device_components.py#L717-L760)).
- REST serializers for path endpoints expose `cable_peer`/`cable_peer_type`, which is the object at the other end of the directly attached cable. They also expose `connected_endpoint`/`connected_endpoint_type`/`connected_endpoint_reachable`, which is the far endpoint of the whole path ([source](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/dcim/api/serializers.py#L124-L175)).
- **`GET /api/dcim/interfaces/{id}/trace/`** and **`GET /api/circuits/circuit-terminations/{id}/trace/`** return the full path as a list of `[near_end, cable, far_end]` triples ([`PathEndpointMixin.trace`](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/dcim/api/views.py#L100-L133); [circuits view](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/circuits/api/views.py#L47)). The code comments that this endpoint's OpenAPI schema is wrong.
- **`GET /api/dcim/front-ports/{id}/paths/`** and **`/rear-ports/{id}/paths/`** return every `CablePath` that passes through a patch panel port ([source](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/dcim/api/views.py#L136-L146)). These are useful for Maintenance on patch panels and fibre.
- A path counts as reachable only if **every cable has status `Connected`** ([docs: Cables](https://docs.nautobot.com/projects/core/en/stable/user-guide/core-data-model/dcim/cable/)).
- In GraphQL, interfaces and terminations expose `cable_peer_<type>` and `connected_<type>`, for example `connected_interface` and `connected_circuit_termination` ([source](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/dcim/graphql/types.py#L176-L190)).

**Limit:** tracing is physical and one hop per path. It stops at the far endpoint. It doesn't follow logical adjacency: LAG members to LAG, subinterfaces to parent, VLAN membership, routing or VPNs. Nautobot has no "everything downstream of device X" endpoint, so a multi-hop "what is behind this aggregation switch" walk has to be computed by our code.

### 3.2 Logical links on an Interface

`parent_interface`, `bridge`, `lag`, `untagged_vlan`, `tagged_vlans` and `ip_addresses` (M2M through `IPAddressToInterface`) are all standard fields ([source](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/dcim/models/device_components.py#L1049-L1240); [docs: Interface](https://docs.nautobot.com/projects/core/en/stable/user-guide/core-data-model/dcim/interface/)). VLANs, IPs and prefixes each carry a `tenant`.

### 3.3 A candidate Impact algorithm, to confirm against Boris data

Given the Network Elements selected on a Maintenance or Incident:

1. **Device** → all its `interfaces` (include subinterfaces and LAG members) → go to 2. Also include `device.tenant` if it is set.
2. **Interface** → `connected_endpoint`:
   - a far `Interface` on a tenant-owned Device (CPE) → Customer = that device's tenant;
   - a `CircuitTermination` → go to 3;
   - also collect `untagged_vlan`/`tagged_vlans`/`ip_addresses→prefix` tenants, plus any Relationship or custom field that Boris uses for a service.
3. **Circuit** → `circuit.tenant` → Customer. Also the far termination's `connected_endpoint` for onward hops, if needed.
4. **Customer (Tenant)** → Zendesk org, via a custom field, Relationship or Contact, whichever Boris uses.
5. Deduplicate. Optionally roll up by `TenantGroup`.

Where to stop recursing (only direct neighbours, or the full downstream tree for aggregation devices) is a Devoli topology question. See the open questions.

## 4. REST vs GraphQL

### Common

- **Auth:** `Authorization: Token <token>`, the same for REST and GraphQL ([docs: REST auth](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/rest-api/authentication/); [docs: GraphQL](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/graphql/)). Tokens can be marked **read-only** (`write_enabled=False`) and can **expire** ([source](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/users/models.py#L262-L264)). We should ask for a read-only token on a dedicated service user.
- **Permissions:** both APIs enforce object-level `view` permissions. In GraphQL, missing permissions give `null` or a silently shortened list rather than an error ([docs: GraphQL permissions](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/graphql/)). **A service account with incomplete permissions would under-report Impact without failing**, so we must check its permissions.

### REST

- **Versioning:** send `Accept: application/json; version=X.Y`, or `?api_version=X.Y`. If you don't, the server uses its latest version, which can break on upgrade. The docs say clients "*should always* request the exact Nautobot REST API version". Responses carry an `API-Version` header ([docs: Versioning](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/rest-api/overview/#versioning)).
- **Pagination:** `limit`/`offset` with a `next` URL. The default is `PAGINATE_COUNT` = 50 and the maximum is `MAX_PAGE_SIZE` = 1000. The server can allow `limit=0` for everything, but that isn't advised ([docs](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/rest-api/overview/#pagination)).
- **`?depth=0..10`** controls nested serialization. The default of 0 returns `{id, object_type, url}` stubs. Nested objects never include relationships or computed fields ([docs](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/rest-api/overview/#depth-query-parameter)).
- **`exclude_m2m`:** **in 3.x, many-to-many fields (other than tags, content_types and object_types) are excluded by default.** Pass `exclude_m2m=False` to get them, for example `tagged_vlans` and `ip_addresses`. In 2.4 they are included by default ([3.x docs](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/rest-api/overview/); [v3.0 release notes](https://docs.nautobot.com/projects/core/en/stable/release-notes/version-3.0/)).
- **Relationships** are only returned with `?include=relationships` ([docs](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/relationship/)).
- **Filtering:** list endpoints accept FilterSet filters, for example `?device=<id>` and `?tenant=<id>` ([docs: filtering](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/rest-api/filtering/)).
- **REST only:** `trace/` and `paths/`.

### GraphQL

- Read-only, `POST /api/graphql/` with `{"query": ..., "variables": ...}` ([docs](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/graphql/)).
- One request can fetch Device → interfaces → `connected_interface`/`connected_circuit_termination` → circuit → tenant → `cf_zendesk_org`. **This avoids the N+1 round trips REST would need for the same walk.**
- List fields accept `limit` and `offset` arguments ([source](https://github.com/nautobot/nautobot/blob/f9cdca3d4f36a0219cd57860b6511472d0870b13/nautobot/core/graphql/generators.py#L377-L473)). There is no cursor pagination.
- Custom fields appear as `cf_<key>` (or all together in `_custom_field_data`), and relationships as `rel_<key>`. **Both only appear after the web service restarts** following creation ([docs](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/graphql/)).
- **Saved GraphQL Queries** can be stored in Nautobot and run with `POST /api/extras/graphql-queries/{id}/run/` ([docs](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/graphql/)).
- GraphQL isn't versioned the way REST is, so schema drift after a Boris upgrade shows up as query errors.

**Verdict:** use **GraphQL for bulk and walk queries**: syncing inventory, and walking device → interfaces → connected endpoints → circuit → tenant. Use **REST for `trace/` and `paths/`**, and whenever we need pinned-version semantics.

## 5. nautobot-app-circuit-maintenance: use or replicate?

What it is ([README](https://github.com/nautobot/nautobot-app-circuit-maintenance/blob/fc508607f07c77e148dd3a94374cf66e73ac219c/README.md); [docs](https://docs.nautobot.com/projects/circuit-maintenance/en/latest/)):

- A Nautobot App, installed **inside Nautobot**, that takes in **provider email notifications** (IMAP or Gmail API). It parses them with the [`circuit-maintenance-parser`](https://github.com/networktocode/circuit-maintenance-parser) library, which follows the draft NANOG maintenance-notification BCOP, and creates records ([use cases](https://github.com/nautobot/nautobot-app-circuit-maintenance/blob/fc508607f07c77e148dd3a94374cf66e73ac219c/docs/user/app_use_cases.md)).
- Models ([source](https://github.com/nautobot/nautobot-app-circuit-maintenance/blob/fc508607f07c77e148dd3a94374cf66e73ac219c/nautobot_circuit_maintenance/models.py#L40-L280)):
  - `CircuitMaintenance`: name, start/end, status, ack;
  - `CircuitImpact`: maintenance ↔ circuit plus impact level;
  - `Note`, `NotificationSource`, `RawNotification`, `ParsedNotification`.
- Enums ([choices.py](https://github.com/nautobot/nautobot-app-circuit-maintenance/blob/fc508607f07c77e148dd3a94374cf66e73ac219c/nautobot_circuit_maintenance/choices.py)):
  - Status: `TENTATIVE, CONFIRMED, CANCELLED, IN-PROCESS, COMPLETED, RE-SCHEDULED, UNKNOWN`;
  - Impact: `NO-IMPACT, REDUCED-REDUNDANCY, DEGRADED, OUTAGE`.
- REST at `/api/plugins/circuit-maintenance/` with `maintenance`, `circuitimpact`, `note` and the other notification endpoints ([api/urls.py](https://github.com/nautobot/nautobot-app-circuit-maintenance/blob/fc508607f07c77e148dd3a94374cf66e73ac219c/nautobot_circuit_maintenance/api/urls.py)). A Job flags overlapping maintenances at a location, which is a redundancy check.
- Version support: app 2.4.x supports Nautobot 2.4.20–2.x, and app 3.x supports Nautobot 3.x ([compatibility matrix](https://github.com/nautobot/nautobot-app-circuit-maintenance/blob/fc508607f07c77e148dd3a94374cf66e73ac219c/docs/admin/compatibility_matrix.md)).

Assessment:

- **Scope is narrow.** It covers only *inbound, provider-originated maintenance on Circuits*. It doesn't handle Devoli's own planned work on devices or interfaces, approvals, Incidents, PIRs, or notifications to Customers. Impact stops at the Circuit; it never reaches Customers.
- **Don't replicate its models.** Outage Manager's Maintenance is broader, and the source of truth for Maintenance should be Outage Manager, not Boris.
- **Reuse its vocabulary.** Adopt its impact levels (`NO-IMPACT / REDUCED-REDUNDANCY / DEGRADED / OUTAGE`), since they match CONTEXT.md's "Outage is an impact level". Its status set is also a useful reference.
- **Optional integration.** If Boris already runs this app, Outage Manager could *read* `CircuitMaintenance`/`CircuitImpact` through its API and offer them as candidate Maintenances. Alternatively, we could use `circuit-maintenance-parser` directly as a library to parse provider emails ourselves. Either choice belongs in a separate ticket.

## 6. Keeping a local cache in sync

- **Webhooks:** per content type, on `created`/`updated`/`deleted`. The default payload has `event`, `model`, `username`, `timestamp`, `data`, and `snapshots` (`prechange`/`postchange`/`differences`). An optional secret gives an HMAC-SHA512 `X-Hook-Signature` header ([docs: Webhooks](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/webhook/)).
- **Job Hooks:** run a `JobHookReceiver` Job *inside Nautobot* when an object changes. They only run if the user who made the change has `run` permission on the Job ([docs: Job Hooks](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/jobs/jobhook/)). These are less useful to us because they need code deployed into Boris.
- **Event notifications (2.4+):** topics `nautobot.create|update|delete.<app>.<model>` published to Redis Pub/Sub or syslog brokers, configured in `nautobot_config.py`. The docs say webhooks and job hooks will likely be reimplemented on top of this system ([docs: Events](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/events/)). This needs Boris ops to register a broker.
- **Change log:** `/api/extras/object-changes/` (ObjectChange, [docs](https://docs.nautobot.com/projects/core/en/stable/user-guide/platform-functionality/change-logging/)) can be polled by time as a fallback or for reconciliation.

---

## Recommended approach

1. **Keep a local, read-only mirror of the Impact graph in Outage Manager's Postgres:**
   - Devices, Interfaces with their connected endpoints and logical links, Circuits and Terminations, Tenants, VLANs/Prefixes if Boris uses them for services, and the Tenant → Zendesk mapping;
   - compute Impact locally with the algorithm in §3.3;
   - reasons: Impact must be computed quickly and repeatedly while an Author edits, must still work if Boris is slow or down during an Incident, and must be **snapshotted** onto the Maintenance or Incident so the record shows what was believed at the time.
2. **Sync strategy:**
   - a scheduled full sync by Celery beat using GraphQL with `limit`/`offset` paging. Periodic reconciliation also covers missed webhooks and permission gaps;
   - Nautobot **webhooks** (HMAC-signed) into an Outage Manager endpoint for near-real-time deltas;
   - use `/trace/` REST calls only for on-demand drill-down in the UI.
3. **Client hygiene:**
   - a dedicated read-only, non-expiring or rotated token on a service user with `view` on every model involved;
   - pin the `Accept: ...; version=X.Y` header;
   - pass `exclude_m2m=False` where M2M fields are needed;
   - alert when a sync's row counts drop sharply, because GraphQL silently omits objects the user can't view.
4. **Don't install or replicate nautobot-app-circuit-maintenance.** Adopt its impact-level vocabulary. Treat ingesting provider notices as a possible separate feature.
5. **Treat the Service and Customer mapping as configuration, not code.** Boris's conventions for Service and the Zendesk org are unknown. Build the walker around a small set of rules configured in the main app (not Django admin): which fields, custom fields or Relationships count as "service → customer" edges.

## Open questions for Boris

These need a Boris admin, or the credentials from ticket #2.

**Platform**
1. Which upstream Nautobot version is Boris forked from, and how far does it diverge? This determines REST defaults (`exclude_m2m`) and whether event brokers exist.
2. Which Apps are installed? Is there a custom Service or Customer app? Is nautobot-app-circuit-maintenance installed, and in use?
3. What is the API base URL, and can our servers reach it from the production network? What are `MAX_PAGE_SIZE` and `PAGINATE_COUNT`? Is GraphQL enabled?

**Customer and Service model**

4. Are Customers modelled as **Tenants**? Are TenantGroups used, for example for resellers or wholesale versus retail?
5. How is a **Service** represented: a Circuit, a VLAN, a Prefix, a VRF or VPN, a custom model, a Relationship, or a custom field such as a service ID? Is there one convention or several?
6. Where is the **Zendesk organisation** recorded: a Tenant custom field, a Contact/Team, or not at all? What is its key, and is it the Zendesk org ID or its name?
7. Which **custom fields, Relationships and Tags** exist on Device, Interface, Circuit and Tenant? We need a listing of `/api/extras/custom-fields/` and `/api/extras/relationships/`.

**Topology fidelity**

8. How completely are **cables and circuit terminations** recorded? Are CPE devices in Boris, and are cable statuses kept `Connected`?
9. Are last-mile and wholesale circuits (for example Chorus/LFC UFB) modelled as Circuits with a tenant? What does the Provider/CircuitType taxonomy look like?
10. For shared devices such as aggregation switches and BNGs, how should "downstream" be found: physical cables only, VLAN membership, or a device-hierarchy Relationship? How deep should Impact recurse?
11. Are LAGs, subinterfaces and VLANs populated consistently enough to walk?

**Sync and access**

12. Can Boris send **webhooks** to Outage Manager (network path, a secret)? Or would ops rather register a Redis **event broker** we can subscribe to?
13. Can we get a **read-only token** on a dedicated service user with `view` on every model involved, and what is the rotation policy?
14. How often does Boris data change, and roughly how many devices, interfaces, circuits and tenants are there? This sizes the sync.
