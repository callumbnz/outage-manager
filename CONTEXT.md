# Devoli Outage Manager

## Language

**Maintenance**:
Planned work on the network, scheduled ahead of time, that needs approval before it is announced.
_Avoid_: change, planned outage, PW

**Emergency Maintenance**:
A Maintenance that bypasses normal approval because it must happen urgently.

**Incident**:
Unplanned degradation or loss of service.
_Avoid_: outage (as a record type), event

**Outage**:
The Impact Level OUTAGE, meaning full loss of service. It is not a record type.

**Impact**:
The set of Services and Customers affected by a Maintenance or Incident, derived from the Network Elements involved.

**Impact Level**:
NO-IMPACT, REDUCED-REDUNDANCY, DEGRADED or OUTAGE, assessed for each affected Service.

**Priority**:
The urgency/severity of an Incident, P1 (highest) to P4. It is separate from Impact.
_Avoid_: SEV, severity level

**Incident Lead**:
The one person accountable for running an Incident. The role can be handed over.

**Participant**:
Anyone else working on an Incident.

**Outcome**:
The recorded result of a completed Maintenance: Successful, Partially successful, Rolled back, or Failed.

**Dismissed**:
The final state of a Draft Incident that turned out not to be a real Incident.

**Notice Period**:
The minimum advance notice (5 NZ business days) expected before any Maintenance.

**Network Element**:
A device, interface or circuit recorded in Boris.

**Service / Customer**:
A Service is delivered over Network Elements. The Customer is who owns it.

**Post-Incident Report (PIR)**:
A written analysis of an Incident that its Reviewer must approve before it is published.
_Avoid_: post-mortem (acceptable as the public-facing name on Status.io), RCA

**Author / Reviewer / Approver**:
The Author writes a Maintenance, Incident or PIR. A PIR has exactly one Reviewer, who approves it before publication. An Approver signs off a Maintenance, and one is enough.

**PIR Reviewer**:
The single person who approves a PIR before publication, usually the team manager. It cannot be the Author or the Incident Lead.

**Public PDF**:
The shareable rendering of a PIR, containing only its public sections. It is published to SharePoint and linked from Status.io.

**Internal PDF**:
The full rendering of a PIR, including internal sections.

**Action Item**:
A follow-up task arising from a PIR, with an owner and a due date.

**Waived**:
The final state of a PIR that an Approver decided is not needed, with a recorded reason.

**Draft Incident**:
An Incident created automatically from a PagerDuty webhook, which an engineer has not yet confirmed.

**Status Page Notice**:
A message published to Status.io (an incident, a scheduled maintenance, or a PIR link) that Status.io delivers to subscribers.

**Provider Notice**:
A maintenance or outage notification received from an upstream provider/carrier about their network, which may affect Devoli Circuits. It is not a Maintenance (that is Devoli's own work) until it is linked.
_Avoid_: carrier maintenance, third-party change

**Provider-driven Maintenance**:
A Maintenance created from a Provider Notice. It is exempt from the Notice Period because the timing is set by the provider.

**Needs Triage**:
A Provider Notice that couldn't be parsed or matched automatically and is waiting for an engineer.

**Ignored**:
The final state of a Provider Notice that an engineer judged irrelevant to Devoli.
