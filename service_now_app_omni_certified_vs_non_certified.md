# ServiceNow AppOmni Integration: Certified vs Non-Certified

## Overview
This document outlines the differences between Certified and Non-Certified AppOmni integrations for ServiceNow, focusing on architecture, security, and operational impact.

---

## Comparison Table

| Area | Certified Version | Non-Certified Version | Practical Impact |
|------|------------------|----------------------|------------------|
| Installation Method | ServiceNow Store app | Manual update set (XML import) | Certified is simpler; Non-certified requires engineering effort |
| ServiceNow Validation | Certified by ServiceNow | Not certified | Certified aligns with platform governance |
| Security Model | Restricted access | Broader access | Non-certified increases visibility but expands attack surface |
| Threat Detection | Partial | Full | Key differentiator |
| System Events (sysevent) | Not supported | Supported | Critical for real-time detection |
| Role Audit | Partial | Full | Better IAM visibility in non-certified |
| Data Exfiltration Detection | Limited | Full | Non-certified supports deeper monitoring |
| API/Table Access | Limited scope | Expanded scope | Non-certified enables deeper insights |
| Performance Impact | Lower | Higher | Non-certified may require indexing |
| Security Risk | Lower | Medium | Trade-off between visibility and exposure |
| Setup Complexity | Low | Medium/High | Manual steps required for non-certified |
| Upgrade Process | Store-managed | Manual | Certified easier to maintain |
| AgentGuard Support | Supported | Supported | No difference |
| Governance Acceptance | High | Requires justification | Important for regulated environments |

---

## Core Differences

### Certified Integration
- ServiceNow approved and governed
- Easier deployment and maintenance
- Limited access to system data
- Reduced threat detection capability

### Non-Certified Integration
- Not ServiceNow certified
- Manual deployment and configuration
- Expanded access to logs and system data
- Full threat detection capability

---

## Technical Capability Comparison

| Capability | Certified | Non-Certified |
|------------|----------|--------------|
| Posture Management | Yes | Yes |
| Access / RBAC Analysis | Yes | Yes |
| Threat Detection | Partial | Full |
| Event Monitoring | No | Yes |
| Audit Monitoring | Partial | Yes |
| SOC Integration Value | Limited | High |

---

## Risk vs Benefit

| Factor | Certified | Non-Certified |
|--------|----------|--------------|
| Security Risk | Low | Medium |
| Detection Capability | Low–Medium | High |
| SOC Value | Low | High |
| Compliance Acceptance | High | Requires justification |

---

## Decision Guidance

### Use Certified When:
- Strong governance or regulatory constraints exist
- Minimal risk tolerance for expanded access
- Focus is posture management only

### Use Non-Certified When:
- Threat detection and SOC integration are required
- Full visibility into events and audit logs is needed
- Detection of insider threats or misuse is a priority

---

## Recommendation

For enterprise SSPM programmes:

- Use **Non-Certified** for full security visibility and detection capability
- Use **Certified** where governance constraints prevent expanded access

---

## Summary

**Certified = compliant but limited visibility**  
**Non-Certified = full visibility with higher risk and effort**

