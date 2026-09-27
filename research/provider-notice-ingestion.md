# Provider Notice ingestion options

Ticket: [#21](https://github.com/callumbnz/outage-manager/issues/21) · Map: [#1](https://github.com/callumbnz/outage-manager/issues/1) · Researched 2026-09-28

**Question.** How can upstream carrier maintenance/outage notifications (**Provider Notices**) be ingested? What does
`circuit-maintenance-parser` support, how is it licensed and maintained, how do we handle formats it does not support,
and how do parsed notices match Boris Circuits by provider circuit ID (`cid`)?

**Short answer.**
- Poll a dedicated shared mailbox (Microsoft Graph if Devoli uses M365, otherwise IMAP over TLS).
- Store every raw `.eml` unchanged.
- Parse with `circuit-maintenance-parser` (Apache-2.0, actively maintained), plus our own in-repo parsers for NZ carriers,
  which the library does not cover.
- Match `(Provider, cid)` case-insensitively against the local Boris mirror, then reuse the existing Impact computation.
- De-duplicate and version each notice on `(provider, maintenance_id)` and on `stamp`/`sequence`.
- Anything that cannot be parsed goes to a triage queue with a manual entry form. LLM extraction is at most an optional,
  off-by-default, human-confirmed fallback.

---

## 1. Ingestion channels

| Channel | How it works | Pros | Cons |
|---|---|---|---|
| **Dedicated mailbox, polled** (recommended) | Carriers send to e.g. `provider-notices@devoli…`. A Celery beat task fetches new messages, stores the raw MIME, then parses. | Carriers already send email, so nothing changes on their side. Polling is outbound-only (no inbound firewall hole). Replayable. This is the model the Nautobot app uses (IMAP, Gmail API, EWS). | Polling latency (1–5 min is fine for maintenance). Needs mailbox credentials and modern auth. |
| &nbsp;&nbsp;↳ **Microsoft Graph** (if M365) | App registration with application permission `Mail.Read` (or `Mail.ReadWrite` to move or flag messages). Fetch the raw MIME with `GET /users/{id}/messages/{id}/$value`. `delta` queries fetch only new or changed messages. Access can be scoped to the one mailbox with an application access policy or RBAC for Applications. | No passwords (client credentials, certificate preferred). Full MIME, so attachments and `.ics` files survive. Exchange Online has retired Basic auth for IMAP, so OAuth is required anyway. | Azure AD admin consent needed. Must restrict the app to one mailbox or it can read all of them. |
| &nbsp;&nbsp;↳ **IMAP (TLS, port 993)** | stdlib `imaplib` or `IMAPClient` (BSD, 3.x). Search `UNSEEN`/`SINCE`, `FETCH RFC822`, then move processed mail to a folder. IDLE (RFC 2177) can push with lower latency. | Works with any provider (Google Workspace, on-prem, other hosts). Simple. | On M365 it needs XOAUTH2 (Basic auth is retired). On Google it needs an app password or OAuth. |
| &nbsp;&nbsp;↳ Gmail API | Service account with domain-wide delegation, or OAuth. Scope `gmail.readonly` (add `gmail.modify` for tagging). This is what the Nautobot app does. | Native to Google Workspace. | Only relevant if Devoli is on Google. |
| **SMTP forwarding into the app** | An MX/transport rule forwards to an SMTP listener in our stack (e.g. `aiosmtpd`) or to an inbound-mail SaaS webhook. | Near real-time. | Adds a public-facing SMTP service to run and harden: spam, SPF/DKIM breakage on forwarding, and a new ingress on RHEL. A mail SaaS sends carrier email to a third party. Not worth it for maintenance-grade latency. |
| **Carrier portals / APIs / RSS** | Per carrier. | Structured data where it exists. | Rare and inconsistent. Vocus NZ's own page says business customers get planned-maintenance notices **by email to the nominated outage contact** ([vocus.co.nz/network-status](https://www.vocus.co.nz/network-status)). Chorus, Spark Wholesale and Southern Cross don't expose a public maintenance feed that we could find from their public sites (their portals need a login). Megaport has an API ([docs.megaport.com/api](https://docs.megaport.com/api/)), but its maintenance notices are email, and the library parses them. Treat these as later, per-carrier adapters. |
| **Manual entry** | NOC form: Provider, cid(s), window, impact, status, reference, and an attachment or pasted email. | Always works. It is the fallback for phone or portal-only notices and for unparseable email. | Human effort and error. |

**Recommendation:** one dedicated shared mailbox, with a `MailSource` adapter interface and Graph and IMAP implementations,
chosen by config. Carriers keep emailing a Devoli address. A transport rule can BCC or forward from today's NOC inbox
during the transition.

Sources: [Graph: get MIME content](https://learn.microsoft.com/en-us/graph/outlook-get-mime-message) ·
[Graph: message delta](https://learn.microsoft.com/en-us/graph/api/message-delta) ·
[Graph change notifications](https://learn.microsoft.com/en-us/graph/change-notifications-overview) ·
[Limit app access to specific mailboxes](https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access) ·
[RBAC for Applications in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac) ·
[IMAP OAuth for Exchange Online](https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/how-to-authenticate-an-imap-pop-smtp-application-by-using-oauth) ·
[Basic auth deprecation in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/deprecation-of-basic-authentication-exchange-online) ·
[imaplib](https://docs.python.org/3/library/imaplib.html) · [IMAPClient](https://imapclient.readthedocs.io/en/master/) ([PyPI](https://pypi.org/project/IMAPClient/)) ·
[RFC 2177 IMAP IDLE](https://datatracker.ietf.org/doc/html/rfc2177) ·
Nautobot app sources: [`sources.py`](https://github.com/nautobot/nautobot-app-circuit-maintenance/blob/develop/nautobot_circuit_maintenance/handle_notifications/sources.py) (IMAP via `imaplib.IMAP4_SSL`, `ews` scheme via `exchangelib`, Gmail API with `gmail.readonly` scope).

## 2. `circuit-maintenance-parser`

### Health, licence, runtime
- **Licence:** Apache-2.0 ([LICENSE](https://github.com/networktocode/circuit-maintenance-parser/blob/develop/LICENSE), PyPI metadata). It bundles the Simplemaps Basic World Cities DB (CC BY 4.0) for city→timezone lookups ([README "License notes"](https://github.com/networktocode/circuit-maintenance-parser#license-notes)). This is compatible with our use.
- **Maintenance:** latest release **v2.12.0 (2026-06-16)**. Commits and merged community PRs are still landing in Sept 2026 (e.g. Telxius cancelled-status fix, SummitIG UTC fix). Maintained by Network to Code, not archived. [Releases](https://github.com/networktocode/circuit-maintenance-parser/releases) · [PyPI](https://pypi.org/project/circuit-maintenance-parser/)
- **Python:** `>=3.10,<3.15` (3.10–3.14 classifiers). Dependencies: pydantic (v1 or v2), icalendar 5, bs4/lxml, geopy, timezonefinder, numpy, python-dateutil. Extras: `openai`, `xlsx` ([pyproject.toml](https://github.com/networktocode/circuit-maintenance-parser/blob/develop/pyproject.toml)). All are pure-Python or have manylinux x86_64 wheels, so RHEL is fine.
- **Beware:** `Geolocator.get_location_from_api` falls back to the public **Nominatim** geocoding API when a city is not in the bundled DB ([utils.py](https://github.com/networktocode/circuit-maintenance-parser/blob/develop/circuit_maintenance_parser/utils.py)). Only a few parsers use it, but it is an outbound call to know about. Allow or deny it in egress policy.

### Output model (`Maintenance`, [output.py](https://github.com/networktocode/circuit-maintenance-parser/blob/develop/circuit_maintenance_parser/output.py))
`provider`, `account`, `maintenance_id`, `circuits: [CircuitImpact(circuit_id, impact)]`, `start`/`end`/`stamp` (int epoch, **UTC**),
`organizer`, `status`, `summary`, `sequence` (default `1`, or `-1` if an iCal omits it), `uid` (default `"0"`), and
`metadata` (`provider`, `processor`, `parsers`, **`generated_by_llm`**).
- `Impact`: `NO-IMPACT`, `REDUCED-REDUNDANCY`, `DEGRADED`, `OUTAGE`
- `Status`: `TENTATIVE`, `CONFIRMED`, `CANCELLED`, `IN-PROCESS`, `COMPLETED`, `RE-SCHEDULED`, `NO-CHANGE`

On CANCELLED or COMPLETED notices some providers **omit the circuit list** ([README](https://github.com/networktocode/circuit-maintenance-parser#readme)).

### Supported providers (v2.12 / `develop`, from [README](https://github.com/networktocode/circuit-maintenance-parser#supported-providers) and `SUPPORTED_PROVIDERS` in [`__init__.py`](https://github.com/networktocode/circuit-maintenance-parser/blob/develop/circuit_maintenance_parser/__init__.py))
- **BCOP iCalendar:** Arelion (Telia), EuNetworks, EXA (GTT), NTT, PacketFabric, PCCW, **Telstra**
- **Custom parsers:** Apple, AT&T, AWS, **Aqua Comms**, BSO, Cirion, Cogent, Colt, Crown Castle, Equinix, EXA (GTT), FLAG, Global Cloud Xchange, Google, **Hawaiki**, HGC, KPN, **Lumen**, **Megaport**, Momentum, Netflix (AS2906), PCCW, RETN, Seaborn, Sparkle, SummitIG, Tata, **Telstra** (two HTML formats plus iCal), Telxius, Turkcell, Verizon, Vodafone, Windstream, **Zayo**
- **`GenericProvider`:** any sender that emits BCOP-compliant iCalendar parses with no custom code.

**NZ/AU relevance**
- **Covered:** Telstra (sender `gpen@team.telstra.com`, Telstra International/global), Megaport (AU-founded, NZ PoPs), Hawaiki (NZ–AU–US cable, sender `support@bw-digital.com`), Aqua Comms, plus global transit/IX providers Devoli may buy (Lumen, Zayo, Cogent, NTT, Equinix, Arelion, PCCW).
- **Not covered:** **Chorus, Spark Wholesale, Vocus NZ (Macquarie), One NZ, Southern Cross Cables, Enable/Tuatahi/Northpower (LFCs), FX Networks, Kordia, Hurricane Electric.** The repo has no issues or PRs mentioning Chorus, Spark, Vocus, Southern Cross or Macquarie (GitHub issue search, 2026-09-28).
- The `Vodafone` class matches Vodafone Group's `networkchangemanagement@vodafone.com` format. It is unlikely to fit One NZ (ex-Vodafone NZ) without samples.
- Telstra coverage is Telstra International. A Telstra AU domestic format would need samples to confirm.

So **the carriers that probably matter most to a NZ ISP (Chorus, the LFCs, Spark Wholesale, Vocus, Southern Cross) need our own parsers.**

### How the library processes a notice
`init_provider("<type>")` returns a `Provider`, which holds ordered `Processors` (`SimpleProcessor`/`CombinedProcessor`), which in turn run `Parsers` (`ICal`, `Html`, `Text`, `Csv`, `Xlsx`, `EmailDateParser`, `EmailSubjectParser`, `LLM`).
`NotificationData.init_from_email_bytes(raw)` splits a MIME message into typed data parts.
`provider.get_maintenances(data)` applies `_include_filter`/`_exclude_filter` (regex per data type, e.g. subject). It tries each processor in turn, and uses the LLM processor last, only if one is enabled.
`get_provider_class_from_sender(email)` maps a sender to a provider by `_default_organizer`. That is a single address per provider, so we should keep our own sender→provider mapping.
([provider.py](https://github.com/networktocode/circuit-maintenance-parser/blob/develop/circuit_maintenance_parser/provider.py), [parser.py](https://github.com/networktocode/circuit-maintenance-parser/blob/develop/circuit_maintenance_parser/parser.py))

### Adding custom parsers (documented extension)
[docs/dev/extending.md](https://github.com/networktocode/circuit-maintenance-parser/blob/develop/docs/dev/extending.md):
1. Subclass a base parser, e.g. `Html.parse_html(soup)` / `Text.parse_text(text)` / `EmailSubjectParser`, returning dicts of `Maintenance` fields.
2. Subclass `GenericProvider`, setting `_processors = [CombinedProcessor(data_parsers=[EmailDateParser, MyParser])]`, `_default_organizer`, and optional include/exclude filters.
3. Add unit and end-to-end tests with sample emails.
4. Add the provider to `SUPPORTED_PROVIDERS`.

`get_provider_class()` only looks in `SUPPORTED_PROVIDERS`, so we should **not** fork the library. Instead, keep NZ provider
classes in our own package (e.g. `provider_notices/parsers/chorus.py`) that subclass `GenericProvider` and use the library's
base parsers, and keep our own registry keyed by Provider. We call `ChorusProvider().get_maintenances(data)` directly. Where
a format is generic enough, upstream it (the project asks contributors to open an issue first). Real sample emails from
Devoli's mailbox become our test fixtures.

## 3. Unsupported formats

1. **Triage queue, always.** An email that no parser handles, or that a parser handles only partly, becomes a `ProviderNotice` in state `NEEDS_TRIAGE`, with the raw email attached. Nothing is silently dropped. (The Nautobot app tags such messages `UNKNOWN_PROVIDER`/`PARSING_FAILED`/`UNKNOWN_CIDS`; we should do the same.)
2. **Manual entry form.** Pre-fill what we can: the sender→Provider mapping, dates found in the subject, and cid candidates found by regex against the known cids for that Provider in Boris. This last heuristic is cheap and strong: if the text contains a string equal to a known Boris cid for that Provider, suggest it.
3. **LLM-assisted extraction (optional, later, off by default).**
   - The library already ships an `OpenAIParser`. It is enabled by `PARSER_OPENAI_API_KEY` (plus `PARSER_OPENAI_MODEL`, default `gpt-3.5-turbo`), runs last, and sets `metadata.generated_by_llm=True` ([README "LLM-powered Parsers"](https://github.com/networktocode/circuit-maintenance-parser#llm-powered-parsers)).
   - **Risks:** it can invent circuit IDs or dates, or get timezones wrong, so its output must never auto-apply. It should pre-fill the manual form for a NOC engineer to confirm.
   - **Privacy:** carrier emails contain circuit IDs, addresses and contacts. Sending them to an external API needs Devoli sign-off and a data-processing review.
   - **Self-hosted vs API:** a self-hosted model keeps data on-prem, but it needs GPU or CPU capacity on RHEL and the output quality is lower. An API model is better and cheaper to run, but the data leaves Devoli.
   - Our own triage heuristics (known-cid matching) may make an LLM unnecessary. Decide once there are real samples.

## 4. Matching to Boris Circuits and Impact

- **Provider resolution:** use the email sender (and optionally the `X-MAINTNOTE-PROVIDER` domain) to find the **Boris Provider**, via a Provider-notice config maintained in *our app* (no Django admin): sender addresses/domains → Boris Provider slug → parser class. This mirrors the Nautobot app's "Emails for Circuit Maintenance" and "Provider Parser" custom fields ([getting started](https://github.com/nautobot/nautobot-app-circuit-maintenance/blob/develop/docs/user/app_getting_started.md)), but lives locally because we are not installing the app into Boris.
- **Circuit resolution:** use a case-insensitive `cid` match scoped to that Provider, against the **local Boris mirror** from #3 (GraphQL sync + webhooks). This is exactly the Nautobot app's rule: `Circuit.objects.filter(cid__iexact=…, provider=provider)` ([handler.py](https://github.com/nautobot/nautobot-app-circuit-maintenance/blob/develop/nautobot_circuit_maintenance/handle_notifications/handler.py)). Add a normalisation step that strips spaces, dashes and prefixes, because carriers often format IDs differently from how they were entered in Boris. Keep per-Provider normalisation rules and an optional **alias table**, since one carrier circuit can carry several reference IDs.
- **Impact:** every matched Circuit is a **Network Element**. Feed those into the existing Impact computation (Circuit → terminations/connected endpoints → Services → Customers → Status.io components) and snapshot the result on the Provider Notice. Keep the carrier's per-circuit impact level (`NO-IMPACT`…`OUTAGE`) alongside, because it drives whether the notice matters (a `NO-IMPACT` or `REDUCED-REDUNDANCY` notice on a protected path may need no customer notification).
- **Unknown cids:** keep the notice, record each unknown cid as an `UnmatchedCircuitRef` (value, provider), flag the notice `UNKNOWN_CIDS` and show it in triage. An engineer can map it to a Boris Circuit, which creates an alias so the next notice auto-matches, or mark it "not ours / decommissioned". Unknown cids are also a useful **Boris data-quality signal** (a missing or mistyped cid).
- **Linking to Devoli work:** a Provider Notice is not a Maintenance. From an impacting notice the NOC can create or link a Devoli Maintenance (and later a Status.io scheduled maintenance). Automatic creation should wait for policy.

## 5. De-duplication, updates, reschedules, cancellations

- **Raw level:** skip any message whose `Message-ID` has already been ingested (and fall back to a hash of the raw bytes). This is idempotent across re-polls. The Nautobot app de-duplicates on (subject, provider, email Date) ([handler.py](https://github.com/nautobot/nautobot-app-circuit-maintenance/blob/develop/nautobot_circuit_maintenance/handle_notifications/handler.py)).
- **Notice identity:** key on `(provider, maintenance_id)`, the same as the Nautobot app's `"{provider}-{maintenance_id}"`. For BCOP iCal, `UID` identifies the thread and `SEQUENCE` increments on every update. The consumer compares UID plus SEQUENCE and applies the update only if SEQUENCE is greater ([MAINTNOTE BCOP §3.7.1–3.7.3](https://github.com/jda/maintnote-std/blob/master/standard.md)). Non-iCal parsers default `uid="0"` and `sequence=1`, so ordering must fall back to **`stamp`** (email Date), as the Nautobot app does. It skips older notices as `OUT_OF_SEQUENCE`.
- **Versioning:** store every parsed notification as an immutable `ProviderNoticeRevision` linked to its raw email. The current `ProviderNotice` reflects the latest revision in order (sequence, then stamp). Keep history so reschedules are visible.
- **Reschedule:** a new window with the same ID means updating start/end and **re-computing Impact**. If a linked Devoli Maintenance or Status.io notice exists, flag it for review rather than silently moving it.
- **Circuit list changes:** diff by cid: add new, update impact, remove missing (the same pattern as the Nautobot app's `update_circuit_maintenance`). **Exception:** CANCELLED/COMPLETED notices may omit circuits, so do **not** treat an empty list as "remove all" for those statuses.
- **Cancellation:** status `CANCELLED` (BCOP §3.7.3). Mark the notice cancelled, keep the Impact snapshot, and alert the owner of any linked Maintenance.
- **Status `NO-CHANGE`:** the library emits this when an iCal omits status. Keep the previous status.
- **Several maintenances per email:** `get_maintenances()` returns a list, so handle 1..n.

## 6. Timezones

- The library always outputs **UTC epoch** values. iCal `DTSTART/DTEND` are UTC (`…Z`) in the BCOP examples.
- **Trap:** `Parser.dt2ts()` treats **naive datetimes as UTC** ([parser.py](https://github.com/networktocode/circuit-maintenance-parser/blob/develop/circuit_maintenance_parser/parser.py)). The `convert_timezone` helper only knows `ET/CT/MT/PT`, plus anything `pytz` accepts ([utils.py](https://github.com/networktocode/circuit-maintenance-parser/blob/develop/circuit_maintenance_parser/utils.py)). `NZST`/`NZDT` are **not** pytz zone names.
- **Consequence:** NZ carriers that write "22:00–04:00 NZDT", or give local time with no zone, will be **13 h off** unless our parsers localise explicitly. Every NZ parser must attach `zoneinfo.ZoneInfo("Pacific/Auckland")` (DST-aware) and map `NZST`/`NZDT` → `Pacific/Auckland`. `AEST`/`AEDT` map to `Australia/Sydney` (confirm per carrier).
- Store UTC `timestamptz` in Postgres and display in `Pacific/Auckland` (Django `USE_TZ=True`, `TIME_ZONE="Pacific/Auckland"`).
- Windows that span a DST change (early April and late September) are the classic edge case. Add test fixtures for them.
- When there is no explicit zone and the sender is a NZ carrier, assume `Pacific/Auckland` and **flag the assumption** on the notice.

## 7. Raw email for audit

- Store the **raw RFC 822 bytes** as immutable `.eml` (Graph `$value` or IMAP `RFC822`) on the same backed-up evidence storage as PIR evidence (#8: FileSystemStorage volume, or S3 via django-storages). Record sha256, `Message-ID`, From, To, Subject, Date, received-at, mailbox id, parser used, parse result/errors, and `generated_by_llm`.
- Render bodies to the NOC **sandboxed**: HTML sanitised with no remote images and no scripts. Don't render inline in the main origin, because carrier HTML is untrusted input.
- After a successful import, move the message to a `Processed`/`Failed` mailbox folder. Don't delete it: the mailbox is a second audit copy and makes replays possible.
- Retention period is an open question for Devoli (likely ≥ 2 years, to match PIR/SLA reporting).

## 8. Recommended approach

**MVP**
1. `MailSource` adapter with **Graph** (if M365) or **IMAP+OAuth/TLS**. Celery beat polls every 2–5 min. A `Processed`/`Failed` folder workflow.
2. `RawProviderEmail` (immutable `.eml`, dedupe on Message-ID/hash) → parse → `ProviderNotice` + `ProviderNoticeRevision` + `ProviderNoticeCircuit` (matched Circuit or unmatched cid, carrier impact).
3. Pin `circuit-maintenance-parser` for the carriers it supports and for `GenericProvider` (BCOP iCal). Keep in-app config for sender → Boris Provider → parser.
4. **Custom parsers for the top 2–3 NZ carriers by volume**, built from real samples and each tested with DST fixtures.
5. Match on Provider + normalised cid against the Boris mirror, then Impact snapshot. Handle unknown cids in triage, with aliases.
6. Triage queue plus manual entry form (including paste or upload of a `.eml`). Show reschedules and cancellations as a revision history.
7. Link or create a Devoli Maintenance from a notice manually. No automatic Status.io posts from Provider Notices.

**After MVP**
- More NZ parsers as volume justifies. Upstream the generic ones.
- Overlap detection: Provider Notice windows against Devoli Maintenance on the same Network Elements (already in "Not yet specified").
- Optional LLM pre-fill (off by default, human-confirmed, privacy-reviewed).
- Carrier API or portal adapters where a carrier offers one.
- Integration-health monitoring (mailbox auth failure, last successful poll, parse-failure rate).
- Graph change notifications instead of polling, if the latency ever matters.

## 9. Open questions for Devoli

1. Where do carrier notices arrive today (NOC shared mailbox, distribution list, individual inboxes, the Zendesk tool)? Can carriers be pointed at, or BCC'd to, a dedicated address?
2. Mailbox platform: **M365 / Exchange Online**, Google Workspace, or other? Who can grant an app registration (Graph `Mail.Read` scoped to one mailbox)?
3. **Top carriers by notice volume**, and which are NZ-domestic (Chorus, LFCs, Spark Wholesale, Vocus, One NZ, FX, Kordia) versus international (Southern Cross, Hawaiki, Telstra, Megaport, transit/IX)?
4. **Sample emails:** 3–5 per top carrier, covering new, update/reschedule, cancel and complete, and outage (unplanned) notices if carriers send them. These become parser fixtures.
5. Is the carrier's circuit ID in Boris `cid` for all Providers, entered consistently? Do some carriers quote a different reference (order ID, service ID, account)?
6. Should unplanned **outage** notices from carriers create a Draft Incident, or stay as Provider Notices for the NOC to link?
7. Is sending carrier emails to an external LLM API acceptable at all? Is a self-hosted model an option?
8. Retention period for raw emails.
9. Any carriers that notify only through a portal or by phone (these would be manual-entry only)?

## Sources (primary)

- circuit-maintenance-parser: [repo](https://github.com/networktocode/circuit-maintenance-parser) · [README](https://github.com/networktocode/circuit-maintenance-parser#readme) · [output.py](https://github.com/networktocode/circuit-maintenance-parser/blob/develop/circuit_maintenance_parser/output.py) · [provider.py](https://github.com/networktocode/circuit-maintenance-parser/blob/develop/circuit_maintenance_parser/provider.py) · [parser.py](https://github.com/networktocode/circuit-maintenance-parser/blob/develop/circuit_maintenance_parser/parser.py) · [utils.py](https://github.com/networktocode/circuit-maintenance-parser/blob/develop/circuit_maintenance_parser/utils.py) · [extending.md](https://github.com/networktocode/circuit-maintenance-parser/blob/develop/docs/dev/extending.md) · [pyproject.toml](https://github.com/networktocode/circuit-maintenance-parser/blob/develop/pyproject.toml) · [releases](https://github.com/networktocode/circuit-maintenance-parser/releases) · [PyPI](https://pypi.org/project/circuit-maintenance-parser/)
- nautobot-app-circuit-maintenance: [repo](https://github.com/nautobot/nautobot-app-circuit-maintenance) · [docs](https://docs.nautobot.com/projects/circuit-maintenance/en/latest/) · [getting started](https://github.com/nautobot/nautobot-app-circuit-maintenance/blob/develop/docs/user/app_getting_started.md) · [sources.py](https://github.com/nautobot/nautobot-app-circuit-maintenance/blob/develop/nautobot_circuit_maintenance/handle_notifications/sources.py) · [handler.py](https://github.com/nautobot/nautobot-app-circuit-maintenance/blob/develop/nautobot_circuit_maintenance/handle_notifications/handler.py)
- MAINTNOTE / NANOG BCOP: [standard.md](https://github.com/jda/maintnote-std/blob/master/standard.md) (draft, last updated 2018) · [RFC 5545 iCalendar](https://datatracker.ietf.org/doc/html/rfc5545)
- Mail: [Graph MIME](https://learn.microsoft.com/en-us/graph/outlook-get-mime-message) · [Graph delta](https://learn.microsoft.com/en-us/graph/api/message-delta) · [Graph change notifications](https://learn.microsoft.com/en-us/graph/change-notifications-overview) · [Limit mailbox access](https://learn.microsoft.com/en-us/graph/auth-limit-mailbox-access) · [RBAC for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac) · [IMAP OAuth (EXO)](https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/how-to-authenticate-an-imap-pop-smtp-application-by-using-oauth) · [Basic auth deprecation](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/deprecation-of-basic-authentication-exchange-online) · [imaplib](https://docs.python.org/3/library/imaplib.html) · [IMAPClient](https://imapclient.readthedocs.io/en/master/) · [RFC 2177](https://datatracker.ietf.org/doc/html/rfc2177)
- Carriers: [Vocus NZ network status](https://www.vocus.co.nz/network-status) · [Megaport API](https://docs.megaport.com/api/) · [Chorus](https://www.chorus.co.nz) · [Spark Wholesale](https://www.sparkwholesale.co.nz) · [Southern Cross Cables](https://www.southerncrosscables.com). None of the last three publishes a public maintenance feed or API that we could find; their portals need a login.
