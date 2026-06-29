# AppOmni API to ServiceNow VR Integration Notes

## Purpose

This document describes how ServiceNow Vulnerability Response (VR) can consume AppOmni API data to:

1. identify the monitored service and its external ID for CMDB CI lookup and assignment;
2. retrieve remediation information for posture findings;
3. retrieve the risk level / risk score for each posture finding;
4. calculate the total number of policies per monitored service; and
5. calculate the number of open posture findings per monitored service.

Primary AppOmni API documentation: <https://api.appomni.com/>  
Additional connector reference used to validate supported AppOmni finding filters: <https://docs.brinqa.com/docs/connectors/appomni>

---

## Base API pattern

Replace `[tenant]` with the customer AppOmni tenant/subdomain.

```http
https://[tenant].appomni.com/api/v1/
```

Authentication should use an AppOmni API token with read access to monitored services, posture findings, and policies.

```http
Authorization: Bearer <APPOMNI_API_TOKEN>
Content-Type: application/json
```

Where pagination is returned, the integration should follow `next` links until all records are processed. Do not assume the first page contains the full dataset.

---

## 1. Getting the External ID of a monitored service

### Objective

ServiceNow VR needs a stable value to look up the correct CMDB CI and route/assign the vulnerability item. In AppOmni, this should come from the monitored service record.

### Endpoint

```http
GET /api/v1/core/monitoredservice/
```

Example:

```bash
curl -s \
  -H "Authorization: Bearer $APPOMNI_API_TOKEN" \
  "https://[tenant].appomni.com/api/v1/core/monitoredservice/"
```

### Fields to extract

For each monitored service, capture:

```text
id
name
service_type
external_id
service_id
org_id / organization identifier, if present
```

### Recommended CMDB lookup logic

Use `external_id` as the primary CMDB CI lookup value.

If `external_id` is empty or not populated, use `service_id` as a fallback, but only if ServiceNow CMDB has been designed to store that SaaS-side identifier.

Recommended mapping:

```json
{
  "appomni_monitored_service_id": "<monitored_service.id>",
  "appomni_monitored_service_name": "<monitored_service.name>",
  "appomni_service_type": "<monitored_service.service_type>",
  "cmdb_ci_lookup_value": "<monitored_service.external_id>",
  "cmdb_ci_lookup_fallback": "<monitored_service.service_id>"
}
```

### ServiceNow VR usage

ServiceNow should use the value as a CI correlation key, for example:

```text
cmdb_ci.correlation_id = AppOmni monitored_service.external_id
```

or, if UCB uses application/service identifiers:

```text
cmdb_ci.u_external_id = AppOmni monitored_service.external_id
```

The exact ServiceNow field should be confirmed with the CMDB owner.

---

## 2. Getting remediation information from posture findings

### Objective

ServiceNow VR needs remediation guidance so the application/service owner understands what action is required.

### Finding endpoint

```http
GET /api/v1/findings/finding/
```

Example:

```bash
curl -s \
  -H "Authorization: Bearer $APPOMNI_API_TOKEN" \
  "https://[tenant].appomni.com/api/v1/findings/finding/?status=open"
```

### Policy / definition endpoint

Remediation guidance is normally held at the policy or posture finding definition level, not uniquely on every individual finding instance.

Use the policy endpoint to enrich each finding:

```http
GET /api/v1/core/policy/
GET /api/v1/core/policy/{policy_id}/
```

Example:

```bash
curl -s \
  -H "Authorization: Bearer $APPOMNI_API_TOKEN" \
  "https://[tenant].appomni.com/api/v1/core/policy/{policy_id}/"
```

### Recommended remediation enrichment flow

```text
1. Pull open posture findings from /findings/finding/?status=open.
2. For each finding, identify the linked policy or finding definition reference.
3. Pull the related policy/definition from /core/policy/{policy_id}/, or cache all policies from /core/policy/.
4. Extract the remediation, recommendation, description, rationale, or guidance fields exposed by the policy object.
5. Send that content to ServiceNow VR as the remediation text.
```

### ServiceNow VR mapping

```json
{
  "short_description": "<finding title / policy name>",
  "description": "<finding description>",
  "remediation": "<policy remediation / recommendation guidance>",
  "appomni_policy_id": "<policy.id>",
  "appomni_finding_id": "<finding.id>"
}
```

### Implementation note

If the finding response already contains remediation guidance directly, ServiceNow can use that directly. If not, enrich the finding using the policy/definition endpoint.

---

## 3. Getting the risk level of each posture finding

### Endpoint

```http
GET /api/v1/findings/finding/
```

The AppOmni posture finding API supports risk filtering, including `riskScoreGte` and `riskScoreLte`, and monitored service filtering using `monitoredServiceIn`.

Example: all open findings:

```bash
curl -s \
  -H "Authorization: Bearer $APPOMNI_API_TOKEN" \
  "https://[tenant].appomni.com/api/v1/findings/finding/?status=open"
```

Example: high-risk open findings only, depending on agreed threshold:

```bash
curl -s \
  -H "Authorization: Bearer $APPOMNI_API_TOKEN" \
  "https://[tenant].appomni.com/api/v1/findings/finding/?status=open&riskScoreGte=7"
```

### Fields to extract

Capture the AppOmni risk field from the finding object, for example:

```text
risk_score
risk_level / severity, if present
status
source_type
monitored_service reference
```

### ServiceNow VR mapping

```json
{
  "risk_score": "<finding.risk_score>",
  "severity": "<finding.risk_level or derived severity>",
  "state": "<finding.status>",
  "source": "AppOmni",
  "source_type": "<finding.source_type>"
}
```

### Suggested severity derivation

If AppOmni provides a numeric risk score only, derive the ServiceNow VR severity using an agreed internal model.

Example model:

```text
0-2   = Low / Informational
3-5   = Medium
6-8   = High
9-10  = Critical
```

This thresholding must be validated against UCB’s ServiceNow VR severity and risk acceptance model before go-live.

---

## 4. Getting the total number of policies per monitored service

### Objective

For each monitored service, ServiceNow or reporting dashboards need to know how many AppOmni posture policies apply to that service.

There are two practical API patterns. Use the first one if the tenant API response exposes policy posture directly on the monitored service detail response. Use the second one if policy posture must be calculated by filtering/enriching policies.

---

### Option A — Use monitored service detail / policy posture, if exposed

The AppOmni API documentation references monitored service detail metadata and policy posture for a single monitored service instance.

Likely pattern:

```http
GET /api/v1/core/monitoredservice/{id}/
```

or a service-type/org-specific detail endpoint, depending on the exact AppOmni tenant API schema.

Example:

```bash
curl -s \
  -H "Authorization: Bearer $APPOMNI_API_TOKEN" \
  "https://[tenant].appomni.com/api/v1/core/monitoredservice/{monitored_service_id}/"
```

Then count the policy posture entries returned for that monitored service.

Example derived value:

```json
{
  "appomni_monitored_service_id": "12345",
  "total_policy_count": 86
}
```

---

### Option B — Calculate from the policy list

If the monitored service detail response does not include policy posture, calculate the total from the policy endpoint.

```http
GET /api/v1/core/policy/
```

Recommended flow:

```text
1. Pull all monitored services from /core/monitoredservice/.
2. Pull all policies from /core/policy/.
3. For each policy, identify which monitored service(s), service type(s), or service instance(s) it applies to.
4. Increment total_policy_count for each matching monitored service.
```

Pseudo-code:

```python
policy_count_by_service = {}

for service in monitored_services:
    policy_count_by_service[service["id"]] = 0

for policy in policies:
    applicable_services = resolve_applicable_services(policy, monitored_services)
    for service_id in applicable_services:
        policy_count_by_service[service_id] += 1
```

### Output to ServiceNow / reporting

```json
{
  "appomni_monitored_service_id": "<monitored_service.id>",
  "appomni_monitored_service_name": "<monitored_service.name>",
  "total_policy_count": "<count of policies applicable to this service>"
}
```

### Important implementation note

The exact policy applicability field must be confirmed from the tenant response. It may be represented by monitored service ID, service type, policy scope, policy group, or another policy definition relationship. Do not hard-code assumptions until one real policy response is inspected.

---

## 5. Getting the number of open posture findings per monitored service

### Objective

For each monitored service, ServiceNow or reporting dashboards need the count of currently open AppOmni posture findings.

### Endpoint

```http
GET /api/v1/findings/finding/
```

The finding endpoint supports filtering by status and monitored service.

Use:

```http
status=open
monitoredServiceIn=<monitored_service_id>
```

Example:

```bash
curl -s \
  -H "Authorization: Bearer $APPOMNI_API_TOKEN" \
  "https://[tenant].appomni.com/api/v1/findings/finding/?status=open&monitoredServiceIn=12345"
```

### Counting approach

If the API response includes a top-level `count` field, use that value.

Example:

```json
{
  "count": 14,
  "next": null,
  "previous": null,
  "results": []
}
```

If no `count` field is returned, count the total number of returned finding records across all pages.

Pseudo-code:

```python
open_finding_count_by_service = {}

for service in monitored_services:
    service_id = service["id"]
    findings = get_all_pages(
        f"/findings/finding/?status=open&monitoredServiceIn={service_id}"
    )
    open_finding_count_by_service[service_id] = len(findings)
```

### Output to ServiceNow / reporting

```json
{
  "appomni_monitored_service_id": "<monitored_service.id>",
  "appomni_monitored_service_name": "<monitored_service.name>",
  "open_posture_finding_count": "<count of open findings>"
}
```

---

## Combined monitored service summary object

For each monitored service, produce a normalized object like this:

```json
{
  "source": "AppOmni",
  "appomni_monitored_service_id": "12345",
  "appomni_monitored_service_name": "Salesforce Production",
  "service_type": "salesforce",
  "external_id": "SN-CI-00012345",
  "service_id": "00Dxxxxxxxxxxxx",
  "cmdb_ci_lookup_value": "SN-CI-00012345",
  "total_policy_count": 86,
  "open_posture_finding_count": 14
}
```

---

## Combined posture finding object for ServiceNow VR

For each posture finding, produce a normalized object like this:

```json
{
  "source": "AppOmni",
  "appomni_finding_id": "98765",
  "appomni_policy_id": "45678",
  "appomni_monitored_service_id": "12345",
  "cmdb_ci_lookup_value": "SN-CI-00012345",
  "finding_status": "open",
  "risk_score": 8,
  "severity": "High",
  "finding_title": "Example AppOmni posture finding title",
  "finding_description": "Description of the issue detected by AppOmni.",
  "remediation": "Remediation guidance from the AppOmni policy or finding definition.",
  "service_type": "salesforce",
  "monitored_service_name": "Salesforce Production"
}
```

---

## Recommended end-to-end integration flow

```text
1. Pull all monitored services.
   GET /api/v1/core/monitoredservice/

2. For each monitored service, capture:
   - id
   - name
   - service_type
   - external_id
   - service_id

3. For each monitored service, calculate policy count.
   Preferred: use monitored service detail policy posture if available.
   Fallback: calculate from /api/v1/core/policy/.

4. For each monitored service, calculate open posture finding count.
   GET /api/v1/findings/finding/?status=open&monitoredServiceIn=<service_id>

5. Pull open posture findings.
   GET /api/v1/findings/finding/?status=open

6. For each finding:
   - identify monitored service
   - enrich with monitored_service.external_id for CI lookup
   - extract risk score / risk level
   - enrich with policy remediation guidance

7. Create or update ServiceNow VR records.
   Correlation keys should include:
   - appomni_finding_id
   - appomni_policy_id
   - appomni_monitored_service_id
   - cmdb_ci lookup value

8. Close or update records when AppOmni finding status changes from open to closed.
```

---

## Minimum ServiceNow field mapping

| ServiceNow VR field / concept | AppOmni source |
|---|---|
| Source | Static value: `AppOmni` |
| External vulnerability ID | `finding.id` |
| Vulnerability definition ID | `policy.id` or finding definition ID |
| CI lookup value | `monitored_service.external_id` |
| CI lookup fallback | `monitored_service.service_id` |
| SaaS application name | `monitored_service.name` |
| SaaS application type | `monitored_service.service_type` |
| Status | `finding.status` |
| Risk score | `finding.risk_score` |
| Severity | `finding.risk_level` or derived from `risk_score` |
| Remediation | Policy / definition remediation guidance |
| Total policies on service | Count from monitored service policy posture or policy calculation |
| Open posture findings on service | Count from `/findings/finding/?status=open&monitoredServiceIn=<id>` |

---

## Key validation questions before build

1. Is `external_id` populated for every monitored service in AppOmni?
2. Which ServiceNow CMDB field stores the matching CI lookup value?
3. Does the monitored service detail endpoint expose policy posture directly in the tenant?
4. Which field in the policy response contains the preferred remediation text?
5. Should ServiceNow severity use AppOmni’s native severity/risk level or UCB-derived severity thresholds?
6. What status mapping should be used when an AppOmni finding is closed?
7. Should ServiceNow VR create one item per AppOmni finding, or one grouped item per policy per CI?

---

## Recommended build approach

Start with a read-only proof of concept that exports one CSV/JSON file with the following columns:

```text
monitored_service_id
monitored_service_name
service_type
external_id
service_id
cmdb_ci_lookup_value
total_policy_count
open_posture_finding_count
finding_id
policy_id
finding_status
risk_score
severity
remediation
```

Once the values are validated with AppOmni, CMDB, and ServiceNow VR owners, convert the export logic into the production ServiceNow integration.
