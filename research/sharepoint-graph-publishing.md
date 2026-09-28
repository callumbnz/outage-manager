# SharePoint publishing via Microsoft Graph

Ticket: [#23](https://github.com/callumbnz/outage-manager/issues/23) · Map: [#1](https://github.com/callumbnz/outage-manager/issues/1) · Researched 2026-09-28

**Question.** How can Outage Manager automatically upload the **Public PDF** to a SharePoint document library through the
Microsoft Graph API? This covers least-privilege app registration, overwriting a file while keeping version history,
view-only anonymous or organisation sharing links (and the tenant policies behind them), whether links stay stable
across overwrites, certificate vs secret auth, throttling, and what Devoli's M365 admins must configure.

**Short answer.**
- **Identity:** use an Entra app registration with the **application** permission `Sites.Selected` (or, tighter,
  `Lists.SelectedOperations.Selected` on just the PIR libraries). An admin grants it the `write` role (or `owner` if
  testing shows it is needed) on the one site with `POST /sites/{id}/permissions`. Authenticate with **MSAL client
  credentials and a certificate**, not a client secret.
- **Upload and overwrite:** the first upload goes by path (`PUT …/{parent-id}:/{name}:/content`). After that, revisions
  **replace content by the stored driveItem id** (`PUT /drives/{drive-id}/items/{item-id}/content`). The item id and its
  permissions stay the same, and the library's versioning keeps every revision.
- **Link stability:** Graph documents that sharing links "always point to the current version of an item unless the item
  is checked out". A link is a permission on the driveItem, so it **survives content overwrites**. It does **not**
  survive delete-and-re-upload, or `conflictBehavior=rename`, because both create a new item.
- **Anonymous links are the weak point.** They only work if both the tenant and the site allow **"Anyone"** sharing.
  Communication and modern sites default to "Only people in your organization". If the tenant requires "Anyone" links
  to expire, **every link we create expires within that maximum**, and shortening the policy later shortens existing
  links too. An `organization` link needs an M365 sign-in, so Status.io readers **cannot** open it.
- **Recommendation:** SharePoint is the **system of record** (storage, version history, metadata, staff access). The
  URL we put on Status.io is a **stable Outage Manager public URL** (for example `https://<public-host>/pir/INC-000123.pdf`).
  That URL either streams the current Public PDF from SharePoint using the app's own token, or 302-redirects to the
  SharePoint anonymous link when the tenant allows one. This decouples Status.io from tenant sharing policy and link
  expiry. #4 found that Status.io postmortem links can only be set by hand, so a URL that never changes matters.

---

## 1. App registration and least-privilege permissions

### Selected scopes

| Scope (application) | Grants access to | Notes |
|---|---|---|
| `Sites.Selected` | one site collection, once granted | Grants at site level don't break inheritance ([Selected overview][sel]). |
| `Lists.SelectedOperations.Selected` | one list or library | Tighter. We could grant only the "PIR – Public" and "PIR – Internal" libraries. Breaks inheritance on the library, which is fine for two libraries. |
| `ListItems.SelectedOperations.Selected` / `Files.SelectedOperations.Selected` | individual items, files or folders | Too fine-grained, because every new PIR file would need its own grant. Not suitable. |

From the Microsoft docs ([Selected overview][sel]):
- Consenting a Selected scope gives **no access by itself**. Three things must all be in place: (1) admin consent in
  Entra, (2) an explicit grant (`POST /sites/{id}/permissions`, `POST /sites/{id}/lists/{id}/permissions`, …) with a role
  of `read`, `write`, `owner` or `fullcontrol`, and (3) a token that carries the scope. Admins can revoke either the
  grant or the consent.
- Creating a **site**-level grant needs `Sites.FullControl.All`. That is an admin's own tool, such as Graph Explorer or a
  one-off script run by a SharePoint Administrator. It is **never** given to Outage Manager. For delegated calls, the
  user must hold SharePoint Administrator or higher ([site-post-permissions][sitepost]). A **list**-level grant can
  also be made by an identity with `Sites.Selected` + `owner` or `fullcontrol` on the site.
- Example grant body ([site-post-permissions][sitepost]):
  ```http
  POST https://graph.microsoft.com/v1.0/sites/{siteId}/permissions
  { "roles": ["write"],
    "grantedToIdentities": [{ "application": { "id": "<client-id>", "displayName": "Devoli Outage Manager" } }] }
  ```

### What each operation's docs list

| Operation | Least-privileged application permission in the docs table | With Selected scopes |
|---|---|---|
| Upload or replace content ([PUT content][put]) | `Files.ReadWrite.All` | Should work through `Sites.Selected` + `write`. The overview says "any scopes presented in the token are honored" ([sel]). **Verify** in a sandbox. |
| `createUploadSession` ([upload session][ups]) | `Sites.ReadWrite.All` | Not needed. Public PDFs are far below the 250 MB simple-upload limit ([put]). |
| `createLink` ([createLink][link]) | `Files.ReadWrite.All` | The table doesn't list Selected scopes. **Verify** that `write` can create an `anonymous` view link. If it can't, grant `owner` on the library only. |
| Update metadata columns ([listItem update][liu]) | `Sites.ReadWrite.All` | Should work with `write`. **Verify**. |
| Create a permission on a driveItem ([driveItem post permissions][dipp]) | `Files.ReadWrite.All` (also lists `Sites.Selected`, `Files/Lists/ListItems.SelectedOperations.Selected`) | Not needed by us. |

The per-operation tables were written for the tenant-wide `.All` scopes, so the **first delivery ticket must include a
sandbox spike** that confirms every call above works with only `Sites.Selected` (or `Lists.SelectedOperations.Selected`)
and the `write` role. Record the minimal role that works.

### Shared app registration with Provider Notice ingestion (#21)
#21 proposed Graph app-only `Mail.Read` on one shared mailbox, restricted with RBAC for Applications. Both features
could live on **one** app registration: one certificate, one consent conversation, and one set of settings in Outage
Manager. Each permission is still resource-scoped, SharePoint by the `Sites.Selected` grant and Exchange by the RBAC
scope. **Two** registrations isolate blast radius and rotation: a leaked mail credential can't write to SharePoint,
and either feature can be revoked on its own. **Suggest two registrations** that share the same code path (a single
`GraphClient` configured per purpose). Let Devoli's admins choose (open question 1).

## 2. Authentication: certificate vs secret

- Microsoft's app-registration guidance: *"use certificate credentials. Don't use password credentials, also known as
  secrets … self-signed certificates are still preferred over passwords"*. It also recommends app management policies
  that limit or block secrets ([app registration best practices][bp]; [certificate credentials][cc]).
- **MSAL for Python** (`msal` 1.39.0, MIT, Python ≥3.9, [PyPI][msal-pypi]) `ConfidentialClientApplication` accepts a
  secret string, or a certificate dict `{"private_key": PEM, "public_certificate": PEM, "passphrase": …}`. Since 1.35.0,
  if the thumbprint is left out and `public_certificate` is supplied, MSAL computes a **SHA-256** thumbprint. The PEM plus
  SHA-1 thumbprint form is marked deprecated in favour of `.pfx` or the SHA-256 path ([msal application.py][msal-src]).
  Use `acquire_token_for_client(scopes=["https://graph.microsoft.com/.default"])`. MSAL caches app tokens in memory.
- **Deployment:** mount the private key PEM as a docker-compose **secret** (a file, not an env var). Store the client ID,
  tenant ID, site and library in Outage Manager's own settings UI (no Django admin), with a "Test connection" button that
  resolves the site and drive and writes then deletes a probe file. Rotate the certificate with overlap: upload the new
  public cert to the app registration, switch the file, then remove the old cert.
- **Library choice:** `msgraph-sdk` (1.63.0, MIT, Python ≥3.10, [PyPI][sdk-pypi], [repo][sdk-repo]) is a large,
  Kiota-generated **async** SDK built on `azure-identity`. We only need about 5 endpoints. **Recommend `msal` +
  `httpx` (or `requests`) in a small in-repo client**. That is easier to run in Celery's sync workers, and we own retry and
  Retry-After handling (§6).

## 3. Upload, overwrite, version history

1. **Resolve once** (at config time, store the ids): `GET /sites/{hostname}:/sites/{path}` → site id.
   `GET /sites/{id}/drives` → the library's drive id. Then the target folder's item id.
2. **First publish:** `PUT /drives/{drive-id}/items/{folder-id}:/INC-000123 PIR.pdf:/content?@microsoft.graph.conflictBehavior=fail`
   ([put]). Use `fail` so we never silently take over someone else's file with the same name. The `conflictBehavior`
   annotation goes in the URL ([driveItem resource][di]). Store the returned **driveItem id**, `webUrl` and `eTag`.
3. **Revision:** `PUT /drives/{drive-id}/items/{item-id}/content`, the documented "replace an existing item" form ([put]).
   Optionally send `If-Match: {eTag}` so that an unexpected manual edit gives a `412` we can surface instead of a blind
   overwrite ([upload session][ups] and [listItem update][liu] both document `if-match`).
4. **Version history:** SharePoint libraries keep versions under the org, site or library version limits. There are two
   modes. "Automatic" is recommended by Microsoft. "Manual" caps major versions (the UI minimum is 100) and optionally
   expires them by age. Retention policies and holds override trimming ([version history limits][vhl]). Each content
   replace adds a version, which we can read via `GET …/items/{id}/versions` ([list versions][lv];
   [driveItemVersion][dv]). **Ask the admins** to confirm that versioning is on for the PIR libraries, and whether a
   retention label or policy should apply to PIRs (open question 6).
5. **Avoid check-out:** don't enable "Require check out" on the library. Links follow the current version *unless the item
   is checked out* ([createLink remarks][link]).

## 4. Sharing links

### createLink
`POST /drives/{drive-id}/items/{item-id}/createLink` with `{"type": "view", "scope": "anonymous" | "organization"}`
([createLink][link]):
- `view` = read-only. `embed` is OneDrive personal only. `password` is OneDrive personal only.
- `anonymous`: *"Anyone with the link has access, without needing to sign in … Anonymous link support may be disabled by
  an administrator."* `organization`: *"Anyone signed into your organization (tenant) can use the link."*
- Returns `201 Created` for a new link, or **`200 OK` if an equivalent link already exists**, so the call is effectively
  idempotent. We can safely call it after every revision and assert the URL hasn't changed.
- The response is a [permission][perm] with `link.webUrl`, `link.scope`, `link.type` and `expirationDateTime`.
  `DateTime.MinValue` (`0001-01-01T00:00:00Z`) means no expiry. [sharingLink][sl] also has `preventsDownload`
  (view in browser only). A Public PDF has no reason to block downloads.
- Remarks: *"Links created using this action don't expire unless a default expiration policy is enforced for the
  organization."* · *"Links are visible in the sharing permissions for the item and can be removed by an owner of the
  item."* · *"Links always point to the current version of a item unless the item is checked out (SharePoint only)."*

### Does the link survive overwriting the file content?
**Yes, as long as it's the same driveItem.** The link is a permission object on the item, and Graph states that links
point to the current version. Replacing content by item id (§3.3) keeps the item, so the link keeps serving the newest
PDF. **What breaks it:** deleting and re-uploading, uploading with `rename`, the file being checked out, a site owner
removing the link in the UI, the org or site turning off "Anyone" sharing (links stop working and come back if it is
turned back on, per [site sharing][sitesh]), and link expiry. Moving or renaming an item keeps its id, but we still
shouldn't do it. The first delivery ticket should confirm this with a sandbox test: create link → replace content twice
→ same `permission.id`, same `webUrl`, new content served anonymously.

### Tenant and site policies that allow or block anonymous links
From [Manage sharing settings][ext] and [Change site sharing][sitesh]:
- The org-level SharePoint sharing slider must be at **"Anyone"**. Each site can only be the **same or more
  restrictive**. The site must also be set to **"Anyone"**.
- **Default site settings:** Communication sites, and modern sites with no group, default to **"Only people in your
  organization"**. Group or Teams sites default to "New and existing guests" or "Existing guests". Only OneDrive and the
  root site default to "Anyone". **So a new PIR site will not allow anonymous links unless an admin changes it.**
- **"Anyone" link advanced settings (org level):** *Link expiration*: "require all Anyone links to expire, and specify
  the maximum number of days allowed. If you change the expiration time, existing links keep their current expiration
  time if the new setting is longer, or **update to the new setting if the new setting is shorter**." *Link
  permissions*: can restrict Anyone links to View, which fits our use.
- The **default link type** (org or site) only affects what the Share dialog preselects. An explicit `scope` in
  `createLink` overrides it ([default link][deflink]; [createLink][link]).

### Do anonymous links obey tenant expiry?
**Yes.** If the tenant enforces an Anyone-link expiry, our links expire within that window whatever we request, and a
later tightening shortens them. A PIR link on a public status page has to last indefinitely, so an expiring link means
the Status.io postmortem link eventually dies. #4 found that link can only be changed by hand in the Status.io
dashboard. Microsoft's pages we reviewed don't document a per-site override of the Anyone-link expiry. If Devoli's admins
say one exists for their tenant, that removes this concern for a dedicated site (open question 3).

### If anonymous links are disabled or expire
| Option | Public can open? | Stable? | Notes |
|---|---|---|---|
| `organization` link | ❌ Needs a Devoli M365 sign-in | ✅ | Good for staff and the Internal PDF, useless for Status.io readers. |
| Dedicated **public PIR site** set to "Anyone", with no expiry | ✅ | ✅ only if the org policy has no expiry | Needs the *org* to allow Anyone, which can be a big security-policy ask. Isolating PIRs in their own site limits the risk. |
| **Outage Manager public PIR URL, streaming from SharePoint** (recommended) | ✅ | ✅ We own the URL | Uses only the app's `Sites.Selected` token (`GET …/items/{id}/content`, 1 RU). Needs **no anonymous sharing at all**. SharePoint stays the record and version history. Cache the bytes locally, invalidating on the stored `eTag`, so a SharePoint outage or throttle doesn't take the page down. Needs a public route or host (#4's open question 9 already asks where public PIRs live). |
| Outage Manager URL that **302-redirects** to the anonymous link | ✅ if Anyone is allowed | ✅ URL is ours. We can re-create an expired link underneath it | Lighter to serve, but still depends on tenant policy. |
| Host the PDF only in Outage Manager, with no SharePoint | ✅ | ✅ | Loses the decided SharePoint record and version history. Last resort. |

## 5. File naming and metadata

- **Stable file name, no revision or date in it**, so overwrites stay the same item and the name matches the link:
  `INC-000123 PIR.pdf` (Public) and `INC-000123 PIR (Internal).pdf` (Internal, in the restricted library or folder).
  Use the Incident number as the key. Keep SharePoint-invalid characters (`" * : < > ? / \ |`) out of names. Keep
  titles out of file names; use metadata instead.
- **Folders:** one flat library, or `YYYY/` sub-folders by incident year. Don't move items after first publish. Use a
  separate library (preferred, so the Internal PDF can be granted separately) or a restricted folder for Internal PDFs.
- **Metadata columns** on the library, set with `PATCH /drives/{d}/items/{id}/listItem/fields`, or with
  `/sites/{s}/lists/{l}/items/{id}/fields` ([listItem update][liu]):
  | Column | Type | Example |
  |---|---|---|
  | `IncidentNumber` | Single line of text (indexed) | `INC-000123` |
  | `IncidentTitle` | Single line of text | `Auckland fibre outage` |
  | `IncidentStart` / `IncidentEnd` | Date and time | UTC |
  | `Priority` | Choice | P1–P4 (CONTEXT.md **Priority**) |
  | `PIRRevision` | Number | `3` |
  | `PublishedAt` | Date and time | first-publish time |
  | `RevisedAt` | Date and time | last revision |
  | `Visibility` | Choice | `Public` / `Internal` |
  | `OutageManagerUrl` | Hyperlink | back-link to the Incident |
  Ask the site owners to **pre-create** these columns, because creating columns needs more than `write`. Outage Manager
  then only writes values. The "Test connection" check should verify that the columns exist.

## 6. Throttling

- Graph returns **429** and SharePoint may also return **503**. Both carry `Retry-After`. Throttled calls still count
  against quota, so honour `Retry-After` exactly. Use exponential backoff only when the header is missing. Continued abuse
  can get the app **blocked** (persistent 503 plus a Message Center notice) ([Graph throttling][thr];
  [SharePoint throttling][spthr]).
- SharePoint limits are in **resource units** per app per tenant: for example 1,250 RU/min and 1.2 M RU/day at the
  smallest tenant tier. Costs: get or download = 1, upload or update = 2, **any permission operation (including
  `createLink`) = 5**. SharePoint doesn't send IETF `RateLimit` headers ([spthr]).
- A publish costs roughly upload 2 + fields 2 + createLink 5 ≈ **10 RU**, so throttling is a non-issue at PIR volume if
  we **don't poll**. A streaming public URL must cache the bytes (see §4) and not hit Graph on every reader request.
- **Decorate traffic:** `User-Agent: NONISV|Devoli|OutageManager/<version>`, because undecorated traffic is deprioritised
  ([spthr]).
- Implement it as a Celery task with `autoretry` that honours `Retry-After`, and is idempotent: re-running a publish
  replaces the same item id, and `createLink` returns the existing link.

## 7. What Devoli's M365 admins must configure

1. **Entra:** create an app registration ("Devoli Outage Manager – SharePoint"). Add the **application** permission
   `Sites.Selected` (or `Lists.SelectedOperations.Selected`) and grant admin consent. Upload our **public certificate**
   (no secret). Optionally apply an app management policy that blocks secrets.
2. **SharePoint:** create or choose a site (for example a communication site "Incident Reviews") with libraries
   **"PIR – Public"** and **"PIR – Internal"** (restricted to staff groups). Keep versioning on. Don't require check-out.
   Pre-create the metadata columns (§5).
3. **Grant:** a SharePoint Administrator runs `POST /sites/{id}/permissions` (or the list-level equivalent) with role
   `write`, or `owner` if the spike shows `createLink` needs it, for the app's client ID.
4. **Sharing (only if we use SharePoint anonymous links):** set org-level sharing to allow **Anyone**, restrict Anyone
   links to **View**, and set the PIR site to **Anyone**. Decide on the Anyone-link expiry policy. With the recommended
   Outage Manager public URL, **none of this is needed**, and the site can stay "Only people in your organization".
5. **Give us:** tenant ID, client ID, site URL, library names, the certificate expiry date or rotation owner, and any
   retention label to apply.

## 8. Recommended approach

1. On PIR Published or Revised, a Celery task renders the Public PDF (and the optional Internal PDF). It uploads the
   first version by path with `conflictBehavior=fail`, or replaces by stored item id. Then it PATCHes the metadata fields
   and stores `drive_id`, `item_id`, `eTag`, `webUrl` and the version id on the PIR.
2. It calls `createLink` (`view`) **only if** the tenant allows Anyone, stores `permission.id`, `webUrl` and
   `expirationDateTime`, and alerts before expiry. For the Internal PDF, use the item `webUrl` or an `organization` link.
3. The **URL given to Status.io is always the Outage Manager public PIR URL**. It streams cached bytes (refreshed by
   `eTag`) or 302-redirects to the anonymous link. SharePoint is the record, with version history, metadata and staff
   browsing.
4. Auth is MSAL client credentials with a certificate mounted as a compose secret. The config lives in the Outage
   Manager settings UI with a connection test. A thin `httpx` Graph client handles `Retry-After`, the decorated
   User-Agent and idempotent retries.
5. The first delivery ticket starts with a **sandbox spike** that verifies: Selected-scope sufficiency per call and the
   minimal role, link id/URL stability across two content replaces, and version creation per replace.

## 9. Open questions for Devoli's M365 admins

1. One app registration shared with Provider Notice mail ingestion (#21), or separate ones? (We suggest separate.)
2. Does the org SharePoint sharing setting allow **Anyone** links? If so, is an **expiry** enforced, and for how many
   days? Is the "View only" restriction on?
3. Would you set a dedicated PIR site to "Anyone"? Does your tenant have any per-site override for Anyone-link expiry?
4. Are you OK with the recommended model, where PIRs are publicly served from an Outage Manager URL and SharePoint holds
   the record without anonymous sharing? Which public hostname? (Ties to #4 Q9.)
5. Which site or library should hold PIRs, who owns it, and which groups may see the Internal library?
6. Versioning mode (Automatic or Manual) and any **retention label or policy** or eDiscovery requirement for PIRs?
7. Certificate: CA-issued or self-signed, what lifetime, and who rotates it? Is there an app management policy blocking
   secrets?
8. Conditional Access or workload identity policies (for example IP restrictions for service principals) that would
   affect calls from our RHEL hosts? What are our egress IPs?
9. Will you pre-create the metadata columns, or should the app be granted `owner` on the library to create them?

## Sources
- [sel]: https://learn.microsoft.com/en-us/graph/permissions-selected-overview — Selected scopes, roles, grant flow
- [sitepost]: https://learn.microsoft.com/en-us/graph/api/site-post-permissions — `POST /sites/{id}/permissions`
- [dipp]: https://learn.microsoft.com/en-us/graph/api/driveitem-post-permissions — driveItem application permissions
- [put]: https://learn.microsoft.com/en-us/graph/api/driveitem-put-content — upload/replace (≤250 MB)
- [ups]: https://learn.microsoft.com/en-us/graph/api/driveitem-createuploadsession — upload sessions, `conflictBehavior`, `if-match`
- [di]: https://learn.microsoft.com/en-us/graph/api/resources/driveitem — `@microsoft.graph.conflictBehavior` (fail/replace/rename)
- [link]: https://learn.microsoft.com/en-us/graph/api/driveitem-createlink — createLink, scopes, remarks on expiry and versions
- [perm]: https://learn.microsoft.com/en-us/graph/api/resources/permission — permission, `expirationDateTime`
- [sl]: https://learn.microsoft.com/en-us/graph/api/resources/sharinglink — sharingLink, `preventsDownload`
- [liu]: https://learn.microsoft.com/en-us/graph/api/listitem-update — update column values
- [lv]: https://learn.microsoft.com/en-us/graph/api/driveitem-list-versions — list versions
- [dv]: https://learn.microsoft.com/en-us/graph/api/resources/driveitemversion — driveItemVersion
- [vhl]: https://learn.microsoft.com/en-us/sharepoint/document-library-version-history-limits — version history limits
- [ext]: https://learn.microsoft.com/en-us/sharepoint/turn-external-sharing-on-or-off — org sharing, Anyone link expiry and permissions
- [sitesh]: https://learn.microsoft.com/en-us/sharepoint/change-external-sharing-site — site sharing, defaults per site type
- [deflink]: https://learn.microsoft.com/en-us/sharepoint/change-default-sharing-link — default link type
- [thr]: https://learn.microsoft.com/en-us/graph/throttling — Graph throttling, Retry-After
- [spthr]: https://learn.microsoft.com/en-us/sharepoint/dev/general-development/how-to-avoid-getting-throttled-or-blocked-in-sharepoint-online — SharePoint resource units, decoration
- [bp]: https://learn.microsoft.com/en-us/entra/identity-platform/security-best-practices-for-app-registration — prefer certificates over secrets
- [cc]: https://learn.microsoft.com/en-us/entra/identity-platform/certificate-credentials — certificate credentials
- [msal-src]: https://github.com/AzureAD/microsoft-authentication-library-for-python/blob/dev/msal/application.py — `client_credential` formats
- [msal-pypi]: https://pypi.org/project/msal/ — 1.39.0, MIT
- [sdk-pypi]: https://pypi.org/project/msgraph-sdk/ — 1.63.0
- [sdk-repo]: https://github.com/microsoftgraph/msgraph-sdk-python — MIT
- Related: [#4 Status.io API research](https://github.com/callumbnz/outage-manager/blob/research/statusio-api/research/statusio-api.md) (postmortem link is external URL, set manually) · [#21 Provider Notice ingestion](https://github.com/callumbnz/outage-manager/blob/research/provider-notice-ingestion/research/provider-notice-ingestion.md) (Graph `Mail.Read`)

[sel]: https://learn.microsoft.com/en-us/graph/permissions-selected-overview
[sitepost]: https://learn.microsoft.com/en-us/graph/api/site-post-permissions
[dipp]: https://learn.microsoft.com/en-us/graph/api/driveitem-post-permissions
[put]: https://learn.microsoft.com/en-us/graph/api/driveitem-put-content
[ups]: https://learn.microsoft.com/en-us/graph/api/driveitem-createuploadsession
[di]: https://learn.microsoft.com/en-us/graph/api/resources/driveitem
[link]: https://learn.microsoft.com/en-us/graph/api/driveitem-createlink
[perm]: https://learn.microsoft.com/en-us/graph/api/resources/permission
[sl]: https://learn.microsoft.com/en-us/graph/api/resources/sharinglink
[liu]: https://learn.microsoft.com/en-us/graph/api/listitem-update
[lv]: https://learn.microsoft.com/en-us/graph/api/driveitem-list-versions
[dv]: https://learn.microsoft.com/en-us/graph/api/resources/driveitemversion
[vhl]: https://learn.microsoft.com/en-us/sharepoint/document-library-version-history-limits
[ext]: https://learn.microsoft.com/en-us/sharepoint/turn-external-sharing-on-or-off
[sitesh]: https://learn.microsoft.com/en-us/sharepoint/change-external-sharing-site
[deflink]: https://learn.microsoft.com/en-us/sharepoint/change-default-sharing-link
[thr]: https://learn.microsoft.com/en-us/graph/throttling
[spthr]: https://learn.microsoft.com/en-us/sharepoint/dev/general-development/how-to-avoid-getting-throttled-or-blocked-in-sharepoint-online
[bp]: https://learn.microsoft.com/en-us/entra/identity-platform/security-best-practices-for-app-registration
[cc]: https://learn.microsoft.com/en-us/entra/identity-platform/certificate-credentials
[msal-src]: https://github.com/AzureAD/microsoft-authentication-library-for-python/blob/dev/msal/application.py
[msal-pypi]: https://pypi.org/project/msal/
[sdk-pypi]: https://pypi.org/project/msgraph-sdk/
[sdk-repo]: https://github.com/microsoftgraph/msgraph-sdk-python
