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
An impact level meaning full loss of service. It is not a record type.

**Impact**:
The set of Services and Customers affected by a Maintenance or Incident, derived from the Network Elements involved.

**Network Element**:
A device, interface or circuit recorded in Boris.

**Service / Customer**:
A Service is delivered over Network Elements. The Customer is who owns it, and each Customer maps to a Zendesk organisation.

**Post-Incident Report (PIR)**:
A written analysis of an Incident that Reviewers must approve before it is published.
_Avoid_: post-mortem (acceptable as the public-facing name on Status.io), RCA

**Author / Reviewer / Approver**:
The Author writes a Maintenance, Incident or PIR. A Reviewer is chosen by the Author to approve a PIR, and every Reviewer must approve. An Approver signs off a Maintenance, and one is enough.

**Support Notification**:
The internal Zendesk ticket that tells the support team about a Maintenance or Incident.

**Customer Notification**:
An optional proactive notice to affected Customers through Zendesk, which the Author chooses to send.

**Draft Incident**:
An Incident created automatically from a PagerDuty webhook, which an engineer has not yet confirmed.
