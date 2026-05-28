# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Penetration Testing Management · Created: 2026-05-19

## Philosophy

This model treats every state change in the system as an immutable event appended to a central event store. The current state of any entity (engagement, finding, remediation ticket) is derived by replaying its event stream. Separate read-optimised materialised views serve the application's query needs (CQRS — Command Query Responsibility Segregation).

Penetration testing management is inherently audit-sensitive. Regulated industries (finance, healthcare, government) require proof of what was found, when, by whom, and how the finding changed over time. Traditional audit logs are bolted on as an afterthought; in this model, the audit trail IS the database. Every finding severity change, every status transition, every CVSS rescore, every report approval is a first-class event with a timestamp, actor, and payload.

Event sourcing is used in production by financial trading platforms, healthcare record systems, and compliance-heavy SaaS products. PostgreSQL serves as the event store using append-only tables with JSONB payloads, while materialised views or projection tables provide the fast reads needed for dashboards and reports. This approach is ideal when "what happened and when" is as important as "what is true now" — which is precisely the case for penetration testing where the timeline of discovery, disclosure, and remediation matters for compliance.

**Best for:** Compliance-driven environments requiring complete audit trails, temporal queries ("what was the finding status on date X?"), and AI analytics over change patterns.

**Trade-offs:**
- (+) Complete, immutable audit trail by design — not an afterthought
- (+) Temporal queries trivially answered by replaying events to a point in time
- (+) AI/ML can analyse event streams for patterns (average time-to-remediate, tester productivity)
- (+) Event replay enables "what-if" scenarios and undo operations
- (+) Natural fit for real-time event streaming (WebSocket, SSE) to dashboards
- (-) Higher write amplification — every change writes an event AND updates projections
- (-) Projection tables must be maintained and can drift if event handlers have bugs
- (-) More complex application code — developers must think in events, not CRUD
- (-) Initial queries require projection tables; cannot just "SELECT * FROM findings"
- (-) Snapshots needed to avoid replaying thousands of events for old engagements

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OCSF 1.5 (Vulnerability Finding) | Event payloads for finding-related events align with OCSF Vulnerability Finding event class structure |
| CVSS 4.0 (FIRST.org) | CVSS score changes are captured as `FindingScoredCvss` events with full before/after vector strings |
| MITRE ATT&CK | ATT&CK mapping changes are discrete events enabling timeline of technique attribution |
| PTES | Engagement phase transitions modelled as explicit events (e.g., `EngagementMovedToExploitation`) |
| NIST SP 800-115 | Assessment planning and documentation requirements satisfied by immutable engagement event streams |
| PCI DSS v4.0 Req 11.3 | Remediation timelines provably reconstructed from event sequences |
| ISO/IEC 27001:2022 A.8.8 | Vulnerability management process fully traceable through event replay |
| CVE Record Format 5.1 | CVE reference additions/changes captured as events with full CVE JSON payload |

---

## Event Store (Core)

```sql
-- The single source of truth. All state changes are appended here.
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,             -- aggregate root ID (e.g., finding ID, engagement ID)
    stream_type     VARCHAR(50) NOT NULL,      -- 'engagement', 'finding', 'remediation', 'report', 'user'
    event_type      VARCHAR(100) NOT NULL,     -- e.g., 'FindingCreated', 'FindingSeverityChanged'
    event_version   INTEGER NOT NULL,          -- monotonically increasing per stream
    payload         JSONB NOT NULL,            -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- actor, IP, correlation IDs
    organisation_id UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Ensure ordering within a stream
    UNIQUE (stream_id, event_version)
);

-- Append-only enforcement via trigger
-- (application also enforces, but this is defence in depth)
CREATE OR REPLACE FUNCTION prevent_event_mutation() RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'Events are immutable. UPDATE and DELETE are not permitted.';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_event_update BEFORE UPDATE ON events
    FOR EACH ROW EXECUTE FUNCTION prevent_event_mutation();
CREATE TRIGGER no_event_delete BEFORE DELETE ON events
    FOR EACH ROW EXECUTE FUNCTION prevent_event_mutation();

-- Primary query patterns
CREATE INDEX idx_events_stream ON events(stream_id, event_version);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_stream_type ON events(stream_type);
CREATE INDEX idx_events_org ON events(organisation_id);
CREATE INDEX idx_events_created ON events(created_at);
CREATE INDEX idx_events_org_type_created ON events(organisation_id, stream_type, created_at);
```

### Event Payload Examples

```sql
-- FindingCreated event payload:
-- {
--   "title": "SQL Injection in Login Form",
--   "description": "The login form is vulnerable to...",
--   "severity": "critical",
--   "engagement_id": "550e8400-e29b-41d4-a716-446655440000",
--   "cwe_id": 89,
--   "source": "manual",
--   "discovered_by": "user-uuid-here",
--   "affected_assets": [
--     {"asset_id": "asset-uuid", "url_path": "/login", "parameter": "username"}
--   ]
-- }

-- FindingSeverityChanged event payload:
-- {
--   "previous_severity": "high",
--   "new_severity": "critical",
--   "reason": "Exploit confirmed with admin-level access"
-- }

-- FindingScoredCvss event payload:
-- {
--   "cvss_version": "4.0",
--   "vector_string": "CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N",
--   "base_score": 9.3,
--   "base_severity": "critical",
--   "previous_vector_string": null,
--   "previous_base_score": null
-- }

-- EngagementStatusChanged event payload:
-- {
--   "previous_status": "in_progress",
--   "new_status": "testing_complete",
--   "reason": "All test cases completed"
-- }

-- RemediationVerified event payload:
-- {
--   "finding_id": "finding-uuid",
--   "result": "fixed",
--   "evidence": "Retested SQL injection — parameterised queries now in place",
--   "tested_by": "user-uuid"
-- }
```

## Event Metadata Structure

```sql
-- metadata JSONB always contains:
-- {
--   "actor_id": "user-uuid",
--   "actor_email": "tester@example.com",
--   "ip_address": "192.168.1.100",
--   "user_agent": "Mozilla/5.0...",
--   "correlation_id": "request-uuid",  -- ties related events from one API call
--   "causation_id": "event-uuid",      -- the event that caused this event (for sagas)
--   "source": "web_ui"                 -- 'web_ui', 'api', 'scanner_import', 'system'
-- }
```

## Snapshots (Performance Optimisation)

```sql
CREATE TABLE snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    event_version   INTEGER NOT NULL,       -- version at which snapshot was taken
    state           JSONB NOT NULL,         -- full materialised state at this version
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, event_version)
);

-- To rebuild state: load latest snapshot, then replay events after snapshot version
-- Example: SELECT state FROM snapshots WHERE stream_id = $1 ORDER BY event_version DESC LIMIT 1;
-- Then: SELECT * FROM events WHERE stream_id = $1 AND event_version > $snapshot_version ORDER BY event_version;

CREATE INDEX idx_snapshots_stream ON snapshots(stream_id, event_version DESC);
```

---

## Read Model Projections (CQRS Query Side)

These tables are derived from events and can be fully rebuilt by replaying the event store.

### Engagement Projection

```sql
CREATE TABLE v_engagements (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    client_id       UUID NOT NULL,
    client_name     VARCHAR(255),
    title           VARCHAR(255) NOT NULL,
    engagement_type VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    methodology     VARCHAR(50),
    start_date      DATE,
    end_date        DATE,
    report_due_date DATE,
    lead_tester_id  UUID,
    lead_tester_name VARCHAR(255),
    finding_count   INTEGER NOT NULL DEFAULT 0,
    critical_count  INTEGER NOT NULL DEFAULT 0,
    high_count      INTEGER NOT NULL DEFAULT 0,
    medium_count    INTEGER NOT NULL DEFAULT 0,
    low_count       INTEGER NOT NULL DEFAULT 0,
    info_count      INTEGER NOT NULL DEFAULT 0,
    last_event_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_engagements_org ON v_engagements(organisation_id);
CREATE INDEX idx_v_engagements_client ON v_engagements(client_id);
CREATE INDEX idx_v_engagements_status ON v_engagements(status);
```

### Finding Projection

```sql
CREATE TABLE v_findings (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    engagement_id   UUID NOT NULL,
    client_id       UUID NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    severity        VARCHAR(20) NOT NULL,
    status          VARCHAR(30) NOT NULL,
    cwe_id          INTEGER,
    cwe_name        VARCHAR(255),
    cve_id          VARCHAR(30),
    cvss_version    VARCHAR(10),
    cvss_base_score DECIMAL(3,1),
    cvss_vector     TEXT,
    source          VARCHAR(50),
    discovered_by_id UUID,
    discovered_by_name VARCHAR(255),
    attack_techniques TEXT[],              -- denormalised ATT&CK technique IDs
    affected_asset_count INTEGER NOT NULL DEFAULT 0,
    evidence_count  INTEGER NOT NULL DEFAULT 0,
    remediation_status VARCHAR(30),
    last_event_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_findings_org ON v_findings(organisation_id);
CREATE INDEX idx_v_findings_engagement ON v_findings(engagement_id);
CREATE INDEX idx_v_findings_severity ON v_findings(severity);
CREATE INDEX idx_v_findings_status ON v_findings(status);
CREATE INDEX idx_v_findings_cwe ON v_findings(cwe_id);
CREATE INDEX idx_v_findings_cvss ON v_findings(cvss_base_score);
```

### Remediation Projection

```sql
CREATE TABLE v_remediation (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    finding_id      UUID NOT NULL,
    finding_title   VARCHAR(500),
    finding_severity VARCHAR(20),
    engagement_id   UUID NOT NULL,
    client_id       UUID NOT NULL,
    external_system VARCHAR(50),
    external_ticket_id VARCHAR(255),
    external_ticket_url TEXT,
    status          VARCHAR(30) NOT NULL,
    assigned_to     VARCHAR(255),
    due_date        DATE,
    retest_count    INTEGER NOT NULL DEFAULT 0,
    last_retest_result VARCHAR(20),
    last_retest_at  TIMESTAMPTZ,
    days_open       INTEGER,               -- computed on projection update
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_remediation_org ON v_remediation(organisation_id);
CREATE INDEX idx_v_remediation_status ON v_remediation(status);
CREATE INDEX idx_v_remediation_finding ON v_remediation(finding_id);
CREATE INDEX idx_v_remediation_due ON v_remediation(due_date);
```

### Client Dashboard Projection

```sql
CREATE TABLE v_client_dashboard (
    client_id       UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    client_name     VARCHAR(255) NOT NULL,
    total_engagements INTEGER NOT NULL DEFAULT 0,
    active_engagements INTEGER NOT NULL DEFAULT 0,
    total_findings  INTEGER NOT NULL DEFAULT 0,
    open_findings   INTEGER NOT NULL DEFAULT 0,
    critical_open   INTEGER NOT NULL DEFAULT 0,
    high_open       INTEGER NOT NULL DEFAULT 0,
    remediated_findings INTEGER NOT NULL DEFAULT 0,
    avg_remediation_days DECIMAL(6,1),
    last_engagement_date DATE,
    risk_trend      VARCHAR(20),           -- 'improving', 'stable', 'worsening'
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_v_client_dashboard_org ON v_client_dashboard(organisation_id);
```

### Analytics Projection (Time-Series for AI)

```sql
CREATE TABLE v_finding_timeline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    client_id       UUID NOT NULL,
    engagement_id   UUID NOT NULL,
    finding_id      UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    severity        VARCHAR(20),
    status          VARCHAR(30),
    event_timestamp TIMESTAMPTZ NOT NULL,
    tester_id       UUID,
    days_since_discovery INTEGER
);

-- Partition by month for efficient time-range queries
CREATE INDEX idx_v_timeline_org_time ON v_finding_timeline(organisation_id, event_timestamp);
CREATE INDEX idx_v_timeline_client ON v_finding_timeline(client_id, event_timestamp);
CREATE INDEX idx_v_timeline_engagement ON v_finding_timeline(engagement_id, event_timestamp);
```

---

## Reference Data (Shared Across Read/Write)

```sql
-- These are traditional tables, not event-sourced, as they represent
-- external reference data that doesn't change through user actions.

CREATE TABLE ref_organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES ref_organisations(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    auth_provider   VARCHAR(50) DEFAULT 'local',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_attack_techniques (
    id              VARCHAR(15) PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    tactic_ids      TEXT[],
    is_subtechnique BOOLEAN NOT NULL DEFAULT false,
    external_url    TEXT,
    last_synced_at  TIMESTAMPTZ
);

CREATE TABLE ref_cwe_weaknesses (
    cwe_id          INTEGER PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(100),
    last_synced_at  TIMESTAMPTZ
);

CREATE TABLE ref_clients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES ref_organisations(id),
    name            VARCHAR(255) NOT NULL,
    industry        VARCHAR(100),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES ref_clients(id),
    name            VARCHAR(255) NOT NULL,
    asset_type      VARCHAR(50) NOT NULL,
    hostname        VARCHAR(255),
    ip_address      INET,
    url             TEXT,
    business_criticality VARCHAR(20) DEFAULT 'medium',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Event Processing Infrastructure

```sql
-- Tracks which projections have processed which events
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for failed event processing
CREATE TABLE projection_failures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projection_name VARCHAR(100) NOT NULL,
    event_id        UUID NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    last_retry_at   TIMESTAMPTZ,
    resolved        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_projection_failures_unresolved ON projection_failures(projection_name) WHERE resolved = false;
```

---

## Temporal Query Examples

```sql
-- "What was the state of finding X on 2026-03-15?"
-- Replay events up to that date:
SELECT payload
FROM events
WHERE stream_id = 'finding-uuid'
  AND created_at <= '2026-03-15T23:59:59Z'
ORDER BY event_version ASC;

-- "How long did it take to remediate critical findings for client Y in Q1 2026?"
SELECT
    f.finding_id,
    f.severity,
    MIN(CASE WHEN f.event_type = 'FindingCreated' THEN f.event_timestamp END) AS discovered_at,
    MIN(CASE WHEN f.event_type = 'RemediationVerified' THEN f.event_timestamp END) AS remediated_at,
    EXTRACT(EPOCH FROM (
        MIN(CASE WHEN f.event_type = 'RemediationVerified' THEN f.event_timestamp END) -
        MIN(CASE WHEN f.event_type = 'FindingCreated' THEN f.event_timestamp END)
    )) / 86400 AS days_to_remediate
FROM v_finding_timeline f
WHERE f.client_id = 'client-uuid'
  AND f.severity = 'critical'
  AND f.event_timestamp BETWEEN '2026-01-01' AND '2026-03-31'
GROUP BY f.finding_id, f.severity;

-- "Show all severity changes for this finding over time"
SELECT
    event_version,
    payload->>'previous_severity' AS from_severity,
    payload->>'new_severity' AS to_severity,
    payload->>'reason' AS reason,
    metadata->>'actor_email' AS changed_by,
    created_at
FROM events
WHERE stream_id = 'finding-uuid'
  AND event_type = 'FindingSeverityChanged'
ORDER BY event_version ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Single append-only events table (source of truth) |
| Snapshots | 1 | Performance optimisation for long event streams |
| Engagement Projections | 1 | Denormalised read model |
| Finding Projections | 1 | Denormalised with CVSS and ATT&CK data |
| Remediation Projections | 1 | Denormalised with finding context |
| Dashboard Projections | 1 | Client-level aggregations |
| Analytics Projections | 1 | Time-series for AI/ML analytics |
| Reference Data | 5 | Organisations, users, clients, assets, ATT&CK, CWE |
| Event Infrastructure | 2 | Checkpoints and dead letter queue |
| **Total** | **15** | Plus projection tables are rebuildable from events |

---

## Key Design Decisions

1. **Single `events` table with JSONB payload** — rather than one table per event type, all events share a single table. This simplifies the event store, enables global ordering, and makes event replay straightforward. The `stream_type` and `event_type` columns enable efficient filtering.

2. **Immutability enforced at the database level** — triggers prevent UPDATE and DELETE on the events table. This is defence-in-depth beyond application-level enforcement and satisfies auditors who need proof that the trail cannot be tampered with.

3. **Snapshots for performance** — long-lived aggregates (engagements with hundreds of events) use periodic snapshots so that rebuilding current state doesn't require replaying from event #1 every time.

4. **Projection tables are disposable** — every `v_*` table can be dropped and rebuilt by replaying the event store. This means schema changes to read models are low-risk — just update the projection handler and rebuild.

5. **Reference data is NOT event-sourced** — external reference data (ATT&CK techniques, CWE weaknesses, organisation config) lives in traditional tables. Event sourcing is applied to domain aggregates where the change history matters, not to static reference data.

6. **Correlation and causation IDs in metadata** — enables tracing which API request or which upstream event caused a given event, critical for debugging and for compliance officers tracing chains of action.

7. **Analytics projection as a denormalised timeline** — the `v_finding_timeline` table is specifically designed for AI/ML consumption: one row per significant event, with enough context to compute metrics like mean-time-to-remediate without joining back to the event store.

8. **Projection checkpoints prevent double-processing** — each projection tracks the last event it processed, enabling crash recovery and exactly-once processing semantics.

9. **Event types as domain language** — event names like `FindingSeverityChanged`, `EngagementMovedToReporting`, `RemediationVerified` read as a natural audit narrative, making compliance reviews easier than parsing generic CRUD logs.

10. **Temporal queries without bi-temporal tables** — by replaying events to a point in time, the system can answer "what was true on date X?" without maintaining separate valid-time and transaction-time columns on every table.
