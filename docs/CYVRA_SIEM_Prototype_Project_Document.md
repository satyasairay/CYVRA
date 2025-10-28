# SIEM Prototype Project Document

## 1. Project Overview
**Goal:**  
Design and implement a custom Security Information and Event Management (SIEM) system from first principles, owned fully under personal IP.  
The prototype will evolve iteratively, from minimal log ingestion to advanced correlation, enrichment, and visualization.

**Objectives:**
- Ingest and normalize security events from diverse sources.
- Detect and correlate security incidents.
- Provide alerting, storage, and audit capabilities.
- Maintain full ownership of architecture, logic, and source code.

---

## 2. Vision & Scope
**Vision:**  
A modular, self-owned SIEM platform built incrementally, demonstrating capability for log ingestion, rule-based correlation, and actionable alerts without dependency on third-party licensing.

**Initial Scope (MVP):**
- Ingest logs from multiple formats (JSON, syslog, CSV, etc.).
- Normalize and categorize events into a common schema.
- Implement basic rule-based alerting and search.
- Provide minimal reporting capability.

**Future Scope:**
- Event correlation across multiple entities.
- Threat intelligence enrichment.
- Machine learning–based anomaly detection.
- Multi-tenant architecture and retention management.

---

## 3. System Design Principles
- **Ownership:** All source code, schemas, and rules authored internally.
- **Extensibility:** Modular components for easy replacement or scaling.
- **Transparency:** Clear event flow and auditable logic.
- **Data Integrity:** Maintain chain of custody for ingested events.
- **Security by Design:** Encryption, authentication, and audit at every stage.

---

## 4. Core Components
| Component | Purpose | MVP Deliverable |
|------------|----------|-----------------|
| **Collector** | Receives events from sources | Basic HTTP/Syslog listener |
| **Parser** | Normalizes fields | Configurable parsing templates |
| **Storage Layer** | Retains normalized + raw data | Indexed time-series store |
| **Rule Engine** | Evaluates event patterns | YAML/JSON rule definitions |
| **Alerting Module** | Notifies on matched rules | Email/webhook-style alert output |
| **User Interface** | Query & visualize alerts | Minimal search + timeline view |
| **Config Database** | Stores rules & metadata | Lightweight key/value store |

---

## 5. Data Model
**Event Schema (MVP fields):**
```
@timestamp  
event.category  
event.action  
src.ip  
dest.ip  
user.name  
host.name  
message  
labels (key/value)
```

**Extended Schema (later):**
```
process.name  
file.path  
dns.query  
http.method  
cloud.account.id  
geo.country  
```

---

## 6. Rule Definition Format
Rules are declarative configurations with conditions, thresholds, and time windows.

```yaml
id: auth-bruteforce
name: Multiple Failed Logins
query: event.category:auth AND event.action:login_failed
group_by: src.ip
condition: count() > 10
window: 5m
severity: medium
tags: [authentication, brute-force]
description: Detects excessive failed logins from same IP within 5 minutes.
```

---

## 7. Alert Lifecycle
1. **New:** Rule triggered and alert generated.  
2. **Open:** Analyst reviewing or triaging.  
3. **Resolved:** False positive or mitigated.  
4. **Closed:** Confirmed and archived.

Each alert includes:
- Rule ID
- Timestamp
- Source(s)
- Evidence (event samples)
- Status
- Owner/assignee
- Notes/audit log

---

## 8. Roadmap
| Phase | Goals | Deliverables |
|-------|--------|--------------|
| **Phase 0** | Define schema & architecture | Design doc, schema, field mapping |
| **Phase 1** | Build ingestion & normalization | Parser config, data validation |
| **Phase 2** | Implement rule engine | Rule DSL + scheduler |
| **Phase 3** | Develop alerting & search | Alert model + simple queries |
| **Phase 4** | Add enrichment | GeoIP, threat feeds |
| **Phase 5** | Visualization & UX | Timeline, dashboards |
| **Phase 6** | Hardening & scaling | Access control, retention policy |

---

## 9. IP & Licensing Notes
- All original code authored and owned by the project creator.  
- Use only dependencies with permissive licenses (MIT, BSD, Apache-2.0).  
- Maintain a `NOTICE` file for any third-party code attribution.  
- Internal documentation, schemas, and rule sets are proprietary unless explicitly open-sourced.  
- Keep dated commits and design logs for proof of authorship.

---

## 10. Testing & Validation
- Generate synthetic logs for known scenarios.  
- Simulate brute force, port scans, DNS tunneling, privilege escalation.  
- Validate rule triggers and false-positive rates.  
- Regression testing for every rule set update.

---

## 11. Security & Compliance
- Enforce encryption in transit and at rest.  
- Apply authentication to all APIs.  
- Maintain audit logs for configuration and rule changes.  
- Align event taxonomy with open standards (ECS, MITRE ATT&CK, etc.) for compatibility.

---

## 12. Deliverables & Ownership
| Artifact | Owner | Status |
|-----------|--------|--------|
| Event Schema | Internal | Draft |
| Parser Config | Internal | Planned |
| Rule Engine Spec | Internal | Draft |
| Alert Lifecycle Document | Internal | Complete |
| IP Ownership Record | Creator | Maintained |

---

## 13. Versioning & Change Control
- Use semantic versioning (e.g., v0.1.0 → MVP).  
- Maintain changelog.md for every iteration.  
- Tag significant architectural updates.

---

## 14. Long-Term Goals
- Unified data pipeline supporting >1M EPS.  
- Adaptive correlation with behavior baselines.  
- Automated case management integration.  
- Multi-environment deployment support.

---

**Author:** _[Satyasai Ray]_  
**Created:** 2025-10-28  
**License:** Proprietary / All Rights Reserved
