# AppOmni API Integration for ServiceNow Vulnerability Response

## Overview

This document describes how to use the AppOmni API to provide data required for a ServiceNow Vulnerability Response integration.

The integration covers:

- Getting the External ID of a monitored service for CMDB CI lookup.
- Getting remediation information for posture findings.
- Getting the risk level of posture findings.
- Getting the total number of policies per monitored service.
- Getting the number of open posture findings per monitored service.

---

## 1. Get the External ID of a Monitored Service

The External ID should be retrieved from the AppOmni monitored service record.

This value can be used by ServiceNow VR to perform a CMDB CI lookup and assign the vulnerability item to the correct CI.

```text
GET /api/v1/core/monitoredservice/
Authorization: Bearer <token>
```

Relevant fields:

```text
id
name
service_type
external_id
```

ServiceNow mapping:

```text
AppOmni monitored_service.external_id -> ServiceNow CMDB CI lookup value
AppOmni monitored_service.id          -> AppOmni Monitored Service ID
AppOmni monitored_service.name        -> Monitored Service Name
AppOmni monitored_service.service_type -> SaaS Platform Type
```

Integration logic:

```text
1. Call GET /api/v1/core/monitoredservice/.
2. For each monitored service, read the external_id.
3. Store the external_id against the AppOmni monitored service ID.
4. When creating or updating a ServiceNow VR item, use external_id as the CI lookup value.
```

---

## 2. Get Remediation Information from Each Posture Finding

Posture findings are retrieved from the AppOmni findings endpoint.

```text
GET /api/v1/findings/finding/
Authorization: Bearer <token>
```

Each posture finding should contain a reference to the related policy or policy rule.

The remediation guidance should be retrieved from the associated policy record.

```text
GET /api/v1/core/policy/{policy_id}/
Authorization: Bearer <token>
```

Relevant fields to inspect on the policy response:

```text
remediation
recommendation
description
resolution
guidance
```

The exact field name depends on the policy object returned by the tenant API, but the remediation content is normally policy-level guidance rather than unique text generated per individual finding.

ServiceNow mapping:

```text
AppOmni policy remediation/recommendation/guidance -> ServiceNow VR remediation instructions
AppOmni finding ID                                -> External vulnerability/reference ID
AppOmni policy ID                                 -> Vulnerability rule/policy reference
```

Integration logic:

```text
1. Call GET /api/v1/findings/finding/.
2. For each finding, read the associated policy ID.
3. Call GET /api/v1/core/policy/{policy_id}/.
4. Extract the remediation or recommendation text.
5. Populate the ServiceNow VR remediation field.
```

---

## 3. Get the Risk Level of Each Posture Finding

Risk information is returned with each posture finding.

```text
GET /api/v1/findings/finding/
Authorization: Bearer <token>
```

Relevant fields to inspect:

```text
risk_score
risk_level
severity
status
source_type
```

If AppOmni returns a direct severity or risk level, use that value.

If AppOmni only returns a numeric risk score, ServiceNow can derive the severity using an agreed severity model.

Example mapping:

```text
0-2   -> Low
3-5   -> Medium
6-8   -> High
9-10  -> Critical
```

ServiceNow mapping:

```text
AppOmni finding.risk_score -> ServiceNow risk score
AppOmni finding.risk_level -> ServiceNow severity
AppOmni finding.severity   -> ServiceNow severity
```

Integration logic:

```text
1. Call GET /api/v1/findings/finding/.
2. For each finding, read the risk_score, risk_level or severity field.
3. Map the AppOmni risk value into the ServiceNow VR severity model.
4. Store the original AppOmni risk value as supporting metadata.
```

---

## 4. Get the Total Number of Policies per Monitored Service

Policies can be retrieved from the policy endpoint.

```text
GET /api/v1/core/policy/
Authorization: Bearer <token>
```

To calculate the number of policies per monitored service:

```text
1. Retrieve all monitored services.
2. Retrieve all policies.
3. For each policy, identify the monitored service reference.
4. Group policies by monitored service ID.
5. Count the number of policies in each group.
```

Expected output:

```text
Monitored Service ID
Monitored Service Name
External ID
Total Policies
```

Example output structure:

```text
monitored_service_id: 12345
monitored_service_name: Salesforce Production
external_id: CI0001234
total_policies: 87
```

ServiceNow mapping:

```text
AppOmni monitored service policy count -> ServiceNow informational field / integration metric
```

---

## 5. Get the Number of Open Posture Findings per Monitored Service

Open posture findings can be retrieved by filtering findings by status.

```text
GET /api/v1/findings/finding/?status=open
Authorization: Bearer <token>
```

To calculate open findings per monitored service:

```text
1. Call GET /api/v1/findings/finding/?status=open.
2. For each finding, identify the monitored service reference.
3. Group findings by monitored service ID.
4. Count the number of findings in each group.
```

Expected output:

```text
Monitored Service ID
Monitored Service Name
External ID
Open Posture Findings
```

Example output structure:

```text
monitored_service_id: 12345
monitored_service_name: Salesforce Production
external_id: CI0001234
open_posture_findings: 14
```

ServiceNow mapping:

```text
AppOmni open finding count per monitored service -> ServiceNow informational field / dashboard metric
```

---

## End-to-End Integration Flow

```text
1. Retrieve monitored services:
   GET /api/v1/core/monitoredservice/

2. Store monitored service metadata:
   - id
   - name
   - service_type
   - external_id

3. Retrieve posture findings:
   GET /api/v1/findings/finding/

4. For each posture finding, read:
   - finding ID
   - monitored service reference
   - policy reference
   - status
   - risk score / risk level / severity

5. Retrieve policy details:
   GET /api/v1/core/policy/{policy_id}/

6. Extract remediation guidance from the policy.

7. Retrieve all policies:
   GET /api/v1/core/policy/

8. Count total policies per monitored service.

9. Retrieve open findings:
   GET /api/v1/findings/finding/?status=open

10. Count open posture findings per monitored service.

11. Populate ServiceNow VR with:
   - CI lookup value = monitored_service.external_id
   - AppOmni monitored service ID
   - AppOmni finding ID
   - Policy ID
   - Risk level / severity
   - Remediation guidance
   - Total policy count for the monitored service
   - Open posture finding count for the monitored service
```

---

## ServiceNow VR Field Mapping

```text
ServiceNow Field                         AppOmni Source
---------------------------------------------------------------
CI lookup value                          monitored_service.external_id
External vulnerability ID                finding.id
Source                                   AppOmni
Assignment / CI context                  monitored_service.external_id
Severity                                 finding.risk_level / finding.severity / finding.risk_score
Risk score                               finding.risk_score
Remediation instructions                 policy.remediation / policy.recommendation / policy.guidance
Policy reference                         policy.id
Monitored service ID                     monitored_service.id
Monitored service name                   monitored_service.name
SaaS platform                            monitored_service.service_type
Total policies per monitored service     calculated from /core/policy/
Open posture findings per service        calculated from /findings/finding/?status=open
```

---

## Notes for Implementation

```text
- Use pagination if returned by the AppOmni API.
- Cache monitored service records to avoid repeated lookups.
- Cache policy records where possible, as many findings may reference the same policy.
- Use external_id as the only CMDB CI lookup value.
- Store the original AppOmni finding ID to support update and deduplication logic.
- Store the original AppOmni risk value as metadata, even if ServiceNow derives its own severity.
- Recalculate policy counts and open finding counts on each scheduled sync.
```
