# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Penetration Testing Management · Created: 2026-05-19

## Philosophy

This model follows a fully normalized relational design where every distinct domain concept occupies its own table with foreign key relationships enforcing referential integrity. The schema is structured around the PTES engagement lifecycle (pre-engagement, intelligence gathering, vulnerability analysis, exploitation, post-exploitation, reporting) with dedicated tables for each phase artefact.

Normalized relational models are the backbone of enterprise security platforms. PlexTrac structures its data around clients, reports, findings, and assets as primary containers. Dradis uses a three-tier model of Issues, Evidence, and Nodes. This suggestion takes those patterns and applies rigorous normalization with standards-aligned reference data (CVSS 4.0, MITRE ATT&CK, CWE, OWASP).

This approach maximises data integrity and supports complex cross-entity queries — for example, "show me all findings tagged with ATT&CK technique T1110 across all engagements for client X in the last 12 months, grouped by remediation status." It is the most natural fit for teams with SQL expertise and compliance-heavy environments where every relationship must be auditable.

**Best for:** Compliance-driven environments with complex cross-engagement reporting and teams comfortable with relational modelling.

**Trade-offs:**
- (+) Maximum referential integrity — no orphaned records, no inconsistent state
- (+) Standard SQL queries for any cross-cutting analysis
- (+) Straightforward ORM mapping for application development
- (+) Well-understood migration and backup strategies
- (-) High table count increases schema complexity and migration overhead
- (-) Adding jurisdiction-specific or scanner-specific fields requires schema changes
- (-) Many-to-many junction tables add query complexity for common operations
- (-) Performance can degrade on deep joins across 6+ tables without careful indexing

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CVSS 4.0 (FIRST.org) | Dedicated `finding_cvss_scores` table stores all four metric groups (Base, Threat, Environmental, Supplemental) with vector string and computed scores |
| MITRE ATT&CK (STIX 2.1) | `attack_techniques` and `attack_tactics` reference tables mirror STIX object structure; junction table `finding_attack_mappings` links findings to techniques |
| CWE (MITRE) | `cwe_weaknesses` reference table stores weakness IDs and descriptions; findings link via `cwe_id` foreign key |
| OWASP Testing Guide v4.2 | `methodology_test_cases` table stores test case IDs and categories from OWASP WSTG |
| PTES | Engagement phases modelled as an enum; engagement scoping fields align with PTES pre-engagement phase |
| CVE Record Format 5.1 | `cve_references` table stores CVE IDs in standard format with links to NVD entries |
| OCSF 1.5 | Audit event structure aligns with OCSF Vulnerability Finding event class fields |
| ISO/IEC 27001:2022 | Compliance report tables reference specific Annex A controls (A.8.8, A.5.36) |
| PCI DSS v4.0 Req 11.3 | Engagement metadata supports PCI DSS scope definition and annual testing requirements |

---

## Organisation & Client Management

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    industry        VARCHAR(100),          -- e.g. 'finance', 'healthcare', 'government'
    billing_email   VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE clients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    industry        VARCHAR(100),
    primary_contact_name  VARCHAR(255),
    primary_contact_email VARCHAR(255),
    notes           TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);

CREATE INDEX idx_clients_organisation ON clients(organisation_id);
```

## User & Access Control

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),          -- null if SSO-only
    auth_provider   VARCHAR(50) DEFAULT 'local',  -- 'local', 'okta', 'entra_id'
    auth_provider_id VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(100) NOT NULL,  -- 'admin', 'tester', 'analyst', 'client_viewer'
    description     TEXT,
    permissions     TEXT[] NOT NULL DEFAULT '{}',
    is_system_role  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES users(id),
    PRIMARY KEY (user_id, role_id)
);

CREATE TABLE user_client_access (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    client_id       UUID NOT NULL REFERENCES clients(id) ON DELETE CASCADE,
    access_level    VARCHAR(20) NOT NULL DEFAULT 'read',  -- 'read', 'write', 'admin'
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, client_id)
);

CREATE INDEX idx_users_organisation ON users(organisation_id);
CREATE INDEX idx_users_email ON users(email);
```

## Asset Management

```sql
CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES clients(id),
    name            VARCHAR(255) NOT NULL,
    asset_type      VARCHAR(50) NOT NULL,  -- 'host', 'web_application', 'api', 'network', 'wireless', 'mobile_app', 'cloud_service'
    hostname        VARCHAR(255),
    ip_address      INET,
    url             TEXT,
    operating_system VARCHAR(100),
    business_criticality VARCHAR(20) DEFAULT 'medium',  -- 'critical', 'high', 'medium', 'low'
    environment     VARCHAR(50),           -- 'production', 'staging', 'development'
    owner           VARCHAR(255),
    notes           TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE asset_ports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    port_number     INTEGER NOT NULL,
    protocol        VARCHAR(10) NOT NULL DEFAULT 'tcp',  -- 'tcp', 'udp'
    service_name    VARCHAR(100),
    service_version VARCHAR(100),
    state           VARCHAR(20) NOT NULL DEFAULT 'open',
    banner          TEXT,
    discovered_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assets_client ON assets(client_id);
CREATE INDEX idx_assets_type ON assets(asset_type);
CREATE INDEX idx_assets_ip ON assets(ip_address);
CREATE INDEX idx_asset_ports_asset ON asset_ports(asset_id);
```

## Engagement Management

```sql
CREATE TYPE engagement_status AS ENUM (
    'draft', 'scoping', 'approved', 'in_progress', 'on_hold',
    'testing_complete', 'reporting', 'delivered', 'closed'
);

CREATE TYPE engagement_type AS ENUM (
    'external_network', 'internal_network', 'web_application', 'api',
    'mobile_application', 'wireless', 'social_engineering', 'physical',
    'red_team', 'purple_team', 'cloud', 'iot'
);

CREATE TABLE engagements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES clients(id),
    title           VARCHAR(255) NOT NULL,
    engagement_type engagement_type NOT NULL,
    status          engagement_status NOT NULL DEFAULT 'draft',
    methodology     VARCHAR(50) DEFAULT 'ptes',  -- 'ptes', 'owasp_wstg', 'nist_800_115', 'osstmm'
    scope_description TEXT,
    rules_of_engagement TEXT,
    start_date      DATE,
    end_date        DATE,
    report_due_date DATE,
    lead_tester_id  UUID REFERENCES users(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE engagement_assets (
    engagement_id   UUID NOT NULL REFERENCES engagements(id) ON DELETE CASCADE,
    asset_id        UUID NOT NULL REFERENCES assets(id),
    in_scope        BOOLEAN NOT NULL DEFAULT true,
    scope_notes     TEXT,
    PRIMARY KEY (engagement_id, asset_id)
);

CREATE TABLE engagement_testers (
    engagement_id   UUID NOT NULL REFERENCES engagements(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'tester',  -- 'lead', 'tester', 'reviewer', 'observer'
    assigned_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (engagement_id, user_id)
);

CREATE INDEX idx_engagements_client ON engagements(client_id);
CREATE INDEX idx_engagements_status ON engagements(status);
CREATE INDEX idx_engagements_dates ON engagements(start_date, end_date);
```

## Findings Management

```sql
CREATE TYPE severity_level AS ENUM ('critical', 'high', 'medium', 'low', 'informational');

CREATE TYPE finding_status AS ENUM (
    'draft', 'confirmed', 'reported', 'accepted',
    'remediation_planned', 'remediated', 'verified_fixed',
    'risk_accepted', 'false_positive'
);

CREATE TABLE findings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_id   UUID NOT NULL REFERENCES engagements(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    severity        severity_level NOT NULL,
    status          finding_status NOT NULL DEFAULT 'draft',
    cwe_id          INTEGER,               -- CWE weakness ID, e.g. 89 for SQL Injection
    cve_id          VARCHAR(30),           -- e.g. 'CVE-2024-12345'
    affected_component TEXT,
    attack_vector   TEXT,
    impact          TEXT,
    likelihood      TEXT,
    remediation_recommendation TEXT,
    remediation_effort VARCHAR(20),        -- 'trivial', 'low', 'medium', 'high', 'complex'
    proof_of_concept TEXT,
    steps_to_reproduce TEXT,
    references      TEXT[],
    source          VARCHAR(50) DEFAULT 'manual',  -- 'manual', 'burp', 'nessus', 'qualys', 'nuclei'
    source_reference_id VARCHAR(255),      -- original ID from scanner
    is_from_library BOOLEAN NOT NULL DEFAULT false,
    library_finding_id UUID,               -- reference to findings_library if applicable
    discovered_by   UUID REFERENCES users(id),
    discovered_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE finding_affected_assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    finding_id      UUID NOT NULL REFERENCES findings(id) ON DELETE CASCADE,
    asset_id        UUID NOT NULL REFERENCES assets(id),
    port_id         UUID REFERENCES asset_ports(id),
    url_path        TEXT,                  -- specific URL path for web findings
    parameter       VARCHAR(255),          -- affected parameter name
    evidence        TEXT,                  -- instance-specific evidence
    notes           TEXT,
    UNIQUE (finding_id, asset_id, url_path, parameter)
);

CREATE TABLE finding_evidence (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    finding_id      UUID NOT NULL REFERENCES findings(id) ON DELETE CASCADE,
    evidence_type   VARCHAR(50) NOT NULL,  -- 'screenshot', 'request_response', 'log', 'code_snippet', 'file'
    title           VARCHAR(255),
    content         TEXT,                  -- text content or base64 for images
    file_path       TEXT,                  -- path in object storage
    mime_type       VARCHAR(100),
    file_size       BIGINT,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_findings_engagement ON findings(engagement_id);
CREATE INDEX idx_findings_severity ON findings(severity);
CREATE INDEX idx_findings_status ON findings(status);
CREATE INDEX idx_findings_cwe ON findings(cwe_id);
CREATE INDEX idx_findings_cve ON findings(cve_id);
CREATE INDEX idx_finding_affected_assets_finding ON finding_affected_assets(finding_id);
CREATE INDEX idx_finding_affected_assets_asset ON finding_affected_assets(asset_id);
CREATE INDEX idx_finding_evidence_finding ON finding_evidence(finding_id);
```

## CVSS Scoring (v4.0 Aligned)

```sql
CREATE TABLE finding_cvss_scores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    finding_id      UUID NOT NULL REFERENCES findings(id) ON DELETE CASCADE,
    cvss_version    VARCHAR(10) NOT NULL DEFAULT '4.0',  -- '3.1', '4.0'

    -- Base Metrics (CVSS 4.0)
    attack_vector           VARCHAR(20),  -- 'network', 'adjacent', 'local', 'physical'
    attack_complexity       VARCHAR(10),  -- 'low', 'high'
    attack_requirements     VARCHAR(10),  -- 'none', 'present' (new in v4.0)
    privileges_required     VARCHAR(10),  -- 'none', 'low', 'high'
    user_interaction        VARCHAR(10),  -- 'none', 'passive', 'active' (expanded in v4.0)
    vuln_confidentiality_impact  VARCHAR(10),  -- 'none', 'low', 'high'
    vuln_integrity_impact        VARCHAR(10),
    vuln_availability_impact     VARCHAR(10),
    sub_confidentiality_impact   VARCHAR(10),  -- subsequent system impact (new in v4.0)
    sub_integrity_impact         VARCHAR(10),
    sub_availability_impact      VARCHAR(10),

    -- Threat Metrics
    exploit_maturity        VARCHAR(20),  -- 'not_defined', 'attacked', 'poc', 'unreported'

    -- Environmental Metrics
    modified_attack_vector  VARCHAR(20),
    modified_attack_complexity VARCHAR(10),
    confidentiality_requirement VARCHAR(10),  -- 'not_defined', 'low', 'medium', 'high'
    integrity_requirement   VARCHAR(10),
    availability_requirement VARCHAR(10),

    -- Supplemental Metrics (new in v4.0)
    safety                  VARCHAR(20),  -- 'not_defined', 'negligible', 'present'
    automatable             VARCHAR(10),  -- 'not_defined', 'no', 'yes'
    recovery                VARCHAR(20),  -- 'not_defined', 'automatic', 'user', 'irrecoverable'
    value_density           VARCHAR(20),  -- 'not_defined', 'diffuse', 'concentrated'
    vulnerability_response_effort VARCHAR(20),
    provider_urgency        VARCHAR(20),

    -- Computed Scores
    vector_string           TEXT NOT NULL,  -- e.g. 'CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N'
    base_score              DECIMAL(3,1) NOT NULL,
    base_severity           VARCHAR(10) NOT NULL,  -- 'none', 'low', 'medium', 'high', 'critical'
    threat_score            DECIMAL(3,1),
    environmental_score     DECIMAL(3,1),

    scored_by               UUID REFERENCES users(id),
    scored_at               TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cvss_finding ON finding_cvss_scores(finding_id);
CREATE INDEX idx_cvss_base_score ON finding_cvss_scores(base_score);
```

## MITRE ATT&CK Mapping

```sql
CREATE TABLE attack_tactics (
    id              VARCHAR(10) PRIMARY KEY,  -- e.g. 'TA0001'
    name            VARCHAR(100) NOT NULL,     -- e.g. 'Initial Access'
    description     TEXT,
    external_url    TEXT,
    stix_id         VARCHAR(100),              -- STIX 2.1 identifier
    last_synced_at  TIMESTAMPTZ
);

CREATE TABLE attack_techniques (
    id              VARCHAR(15) PRIMARY KEY,   -- e.g. 'T1110' or 'T1110.001'
    parent_id       VARCHAR(15) REFERENCES attack_techniques(id),  -- for sub-techniques
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    is_subtechnique BOOLEAN NOT NULL DEFAULT false,
    platforms       TEXT[],                    -- ['windows', 'linux', 'macos', 'cloud']
    external_url    TEXT,
    stix_id         VARCHAR(100),
    last_synced_at  TIMESTAMPTZ
);

CREATE TABLE attack_technique_tactics (
    technique_id    VARCHAR(15) NOT NULL REFERENCES attack_techniques(id),
    tactic_id       VARCHAR(10) NOT NULL REFERENCES attack_tactics(id),
    PRIMARY KEY (technique_id, tactic_id)
);

CREATE TABLE finding_attack_mappings (
    finding_id      UUID NOT NULL REFERENCES findings(id) ON DELETE CASCADE,
    technique_id    VARCHAR(15) NOT NULL REFERENCES attack_techniques(id),
    confidence      VARCHAR(10) DEFAULT 'high',  -- 'high', 'medium', 'low'
    notes           TEXT,
    mapped_by       UUID REFERENCES users(id),
    mapped_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (finding_id, technique_id)
);

CREATE INDEX idx_attack_techniques_parent ON attack_techniques(parent_id);
CREATE INDEX idx_finding_attack_finding ON finding_attack_mappings(finding_id);
CREATE INDEX idx_finding_attack_technique ON finding_attack_mappings(technique_id);
```

## CWE Reference Data

```sql
CREATE TABLE cwe_weaknesses (
    cwe_id          INTEGER PRIMARY KEY,       -- e.g. 89
    name            VARCHAR(255) NOT NULL,      -- e.g. 'Improper Neutralization of Special Elements used in an SQL Command'
    description     TEXT,
    extended_description TEXT,
    category        VARCHAR(100),               -- e.g. 'Injection'
    likelihood_of_exploit VARCHAR(20),
    external_url    TEXT,
    last_synced_at  TIMESTAMPTZ
);

CREATE TABLE owasp_categories (
    id              VARCHAR(20) PRIMARY KEY,    -- e.g. 'A01:2021'
    name            VARCHAR(255) NOT NULL,      -- e.g. 'Broken Access Control'
    description     TEXT,
    guide           VARCHAR(50) NOT NULL,       -- 'top10_2021', 'wstg_v42', 'api_top10_2023'
    external_url    TEXT
);

CREATE TABLE cwe_owasp_mappings (
    cwe_id          INTEGER NOT NULL REFERENCES cwe_weaknesses(cwe_id),
    owasp_id        VARCHAR(20) NOT NULL REFERENCES owasp_categories(id),
    PRIMARY KEY (cwe_id, owasp_id)
);
```

## Findings Library (Reusable Templates)

```sql
CREATE TABLE findings_library (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    default_severity severity_level NOT NULL,
    cwe_id          INTEGER REFERENCES cwe_weaknesses(cwe_id),
    remediation_recommendation TEXT,
    remediation_effort VARCHAR(20),
    references      TEXT[],
    tags            TEXT[],
    usage_count     INTEGER NOT NULL DEFAULT 0,
    created_by      UUID REFERENCES users(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_findings_library_org ON findings_library(organisation_id);
CREATE INDEX idx_findings_library_cwe ON findings_library(cwe_id);
CREATE INDEX idx_findings_library_severity ON findings_library(default_severity);
```

## Remediation Tracking

```sql
CREATE TYPE remediation_status AS ENUM (
    'open', 'assigned', 'in_progress', 'pending_retest',
    'retesting', 'verified_fixed', 'verified_not_fixed',
    'risk_accepted', 'deferred'
);

CREATE TABLE remediation_tickets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    finding_id      UUID NOT NULL REFERENCES findings(id),
    external_system VARCHAR(50),           -- 'jira', 'servicenow', 'github', 'manual'
    external_ticket_id VARCHAR(255),       -- e.g. 'SEC-1234'
    external_ticket_url TEXT,
    status          remediation_status NOT NULL DEFAULT 'open',
    assigned_to     VARCHAR(255),          -- may be external user not in our system
    priority        VARCHAR(20),
    due_date        DATE,
    resolution_notes TEXT,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE retest_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    finding_id      UUID NOT NULL REFERENCES findings(id),
    ticket_id       UUID REFERENCES remediation_tickets(id),
    tested_by       UUID NOT NULL REFERENCES users(id),
    result          VARCHAR(20) NOT NULL,  -- 'fixed', 'partially_fixed', 'not_fixed', 'regressed'
    evidence        TEXT,
    notes           TEXT,
    tested_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_remediation_finding ON remediation_tickets(finding_id);
CREATE INDEX idx_remediation_status ON remediation_tickets(status);
CREATE INDEX idx_remediation_external ON remediation_tickets(external_system, external_ticket_id);
CREATE INDEX idx_retest_finding ON retest_results(finding_id);
```

## Reporting

```sql
CREATE TABLE report_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    framework       VARCHAR(50),           -- 'owasp', 'ptes', 'nist', 'custom'
    template_format VARCHAR(20) NOT NULL,  -- 'html', 'docx', 'pdf'
    template_body   TEXT NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_id   UUID NOT NULL REFERENCES engagements(id),
    template_id     UUID REFERENCES report_templates(id),
    title           VARCHAR(255) NOT NULL,
    report_type     VARCHAR(50) NOT NULL,  -- 'full', 'executive_summary', 'compliance', 'retest'
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',  -- 'draft', 'review', 'approved', 'delivered'
    output_format   VARCHAR(20) NOT NULL,
    file_path       TEXT,                  -- generated report file location
    executive_summary TEXT,
    generated_by    UUID REFERENCES users(id),
    approved_by     UUID REFERENCES users(id),
    delivered_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE report_findings (
    report_id       UUID NOT NULL REFERENCES reports(id) ON DELETE CASCADE,
    finding_id      UUID NOT NULL REFERENCES findings(id),
    include_in_report BOOLEAN NOT NULL DEFAULT true,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    custom_narrative TEXT,                 -- report-specific override of description
    PRIMARY KEY (report_id, finding_id)
);

CREATE INDEX idx_reports_engagement ON reports(engagement_id);
```

## Methodology & Test Cases

```sql
CREATE TABLE methodology_test_cases (
    id              VARCHAR(30) PRIMARY KEY,  -- e.g. 'WSTG-ATHZ-01'
    methodology     VARCHAR(50) NOT NULL,      -- 'owasp_wstg_v42', 'ptes', 'owasp_api_top10'
    category        VARCHAR(100) NOT NULL,     -- e.g. 'Authorization Testing'
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    external_url    TEXT
);

CREATE TABLE engagement_test_cases (
    engagement_id   UUID NOT NULL REFERENCES engagements(id) ON DELETE CASCADE,
    test_case_id    VARCHAR(30) NOT NULL REFERENCES methodology_test_cases(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'not_started',  -- 'not_started', 'in_progress', 'complete', 'not_applicable'
    tester_id       UUID REFERENCES users(id),
    notes           TEXT,
    completed_at    TIMESTAMPTZ,
    PRIMARY KEY (engagement_id, test_case_id)
);
```

## Compliance Reporting

```sql
CREATE TABLE compliance_frameworks (
    id              VARCHAR(50) PRIMARY KEY,  -- e.g. 'pci_dss_v4'
    name            VARCHAR(255) NOT NULL,
    version         VARCHAR(20),
    description     TEXT
);

CREATE TABLE compliance_controls (
    id              VARCHAR(50) PRIMARY KEY,  -- e.g. 'pci_dss_v4_11.3'
    framework_id    VARCHAR(50) NOT NULL REFERENCES compliance_frameworks(id),
    control_number  VARCHAR(50) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    description     TEXT,
    testing_requirement TEXT
);

CREATE TABLE engagement_compliance_mappings (
    engagement_id   UUID NOT NULL REFERENCES engagements(id) ON DELETE CASCADE,
    control_id      VARCHAR(50) NOT NULL REFERENCES compliance_controls(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'not_assessed',
    assessor_notes  TEXT,
    assessed_at     TIMESTAMPTZ,
    PRIMARY KEY (engagement_id, control_id)
);
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID REFERENCES users(id),
    action          VARCHAR(50) NOT NULL,     -- 'create', 'update', 'delete', 'login', 'export', 'report_generate'
    entity_type     VARCHAR(50) NOT NULL,     -- 'finding', 'engagement', 'report', 'user', etc.
    entity_id       UUID,
    old_values      JSONB,                    -- previous state for updates
    new_values      JSONB,                    -- new state for updates/creates
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_action ON audit_log(action);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

## Notifications & Integrations

```sql
CREATE TABLE notification_preferences (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    event_type      VARCHAR(50) NOT NULL,     -- 'finding_created', 'report_ready', 'remediation_due'
    channel         VARCHAR(20) NOT NULL,     -- 'email', 'slack', 'in_app'
    is_enabled      BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (user_id, event_type, channel)
);

CREATE TABLE integration_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    integration_type VARCHAR(50) NOT NULL,    -- 'jira', 'servicenow', 'slack', 'email'
    config_name     VARCHAR(255) NOT NULL,
    base_url        TEXT,
    auth_token_encrypted BYTEA,
    additional_config JSONB,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_integration_org ON integration_configs(organisation_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Clients | 2 | Multi-tenant foundation |
| Users & Access Control | 4 | RBAC with client-level access |
| Assets | 2 | Host, port, and service inventory |
| Engagements | 3 | Lifecycle, scope, team assignment |
| Findings | 3 | Core findings, affected assets, evidence |
| CVSS Scoring | 1 | Full CVSS 4.0 metric storage |
| ATT&CK Mapping | 4 | Tactics, techniques, and finding links |
| CWE / OWASP Reference | 3 | Weakness and category reference data |
| Findings Library | 1 | Reusable finding templates |
| Remediation | 2 | Ticket tracking and retest results |
| Reporting | 3 | Templates, reports, finding selection |
| Methodology | 2 | Test case checklists |
| Compliance | 3 | Framework, control, and assessment mapping |
| Audit Log | 1 | Immutable action log |
| Notifications & Integrations | 2 | User preferences and external system config |
| **Total** | **36** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — enables distributed ID generation for self-hosted deployments and future federation between instances.

2. **Separate `finding_affected_assets` junction table** — mirrors Dradis's Issue/Evidence/Node pattern where one vulnerability type can affect multiple assets, each with instance-specific evidence.

3. **Full CVSS 4.0 column decomposition** — stores every metric as a discrete column rather than just the vector string, enabling SQL queries like "show all findings where automatable = 'yes' AND base_score > 7.0".

4. **ATT&CK reference tables synced from STIX data** — keeps technique and tactic data in normalised relational tables rather than embedding STIX JSON, allowing standard SQL joins for reporting.

5. **Findings library as a separate table** — supports the Dradis-style reusable issue library pattern; `is_from_library` and `library_finding_id` on findings enable tracking of library usage without coupling.

6. **Audit log with old/new JSONB values** — the one concession to JSONB in an otherwise fully normalised schema, capturing before/after state for any entity change without requiring per-entity audit tables.

7. **Engagement status as PostgreSQL enum** — enforces valid state transitions at the database level; the application layer validates transition rules.

8. **External remediation ticket references** — rather than reimplementing Jira/ServiceNow, stores external ticket IDs and URLs for bidirectional linking.

9. **Methodology test case tracking** — engagement_test_cases provides a checklist-style progress tracker aligned to OWASP WSTG or PTES test cases.

10. **Multi-tenant via `organisation_id` foreign keys** — row-level security can be applied using PostgreSQL RLS policies on the organisation_id column for data isolation.
