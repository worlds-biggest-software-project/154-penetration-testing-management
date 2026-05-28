# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Penetration Testing Management · Created: 2026-05-19

## Philosophy

This model uses a relational backbone for core entities (engagements, findings, users, clients) with PostgreSQL JSONB columns for variable, domain-specific, and scanner-specific data. The key insight is that penetration testing data is inherently heterogeneous: a web application finding has different attributes than a network infrastructure finding; a Burp Suite import has different fields than a Nessus import; OWASP methodology test cases differ from PTES checklists; and each client may require jurisdiction-specific compliance fields.

Rather than creating dozens of narrow tables or using an EAV (Entity-Attribute-Value) anti-pattern, this model stores stable, universally-queried fields as typed columns and pushes variable fields into JSONB. PostgreSQL's JSONB supports GIN indexes, containment queries (`@>`), path queries (`->>`, `#>>`), and partial indexing — making it possible to query deeply into the variable data without sacrificing performance.

This is the approach PlexTrac uses internally (their Finding object is "a nested JSON object" stored in the database with stable fields extracted for querying). It is also the dominant pattern in modern SaaS applications that need to support multi-tenant customisation without per-tenant schema changes. The hybrid approach delivers 80% of the normalised model's query power with 50% of the table count, and can ship an MVP significantly faster.

**Best for:** Rapid MVP development, multi-scanner integration, teams that value flexibility and iteration speed, platforms that need to support diverse engagement types and methodologies.

**Trade-offs:**
- (+) Dramatically fewer tables — variable data lives in JSONB rather than junction tables
- (+) New scanner formats, methodology fields, or compliance requirements need zero schema migrations
- (+) JSONB GIN indexes support efficient containment and path queries
- (+) Natural fit for scanner import — store raw scanner JSON alongside normalised fields
- (+) Faster development iteration — add fields without ALTER TABLE
- (-) No foreign key constraints inside JSONB — referential integrity for embedded data is application-enforced
- (-) JSONB storage is larger than normalised columns for repeated values
- (-) Complex JSONB queries can be harder to optimise and debug than simple JOINs
- (-) Schema validation for JSONB must be enforced at the application layer (JSON Schema or Zod)
- (-) Reporting queries that span JSONB and relational fields are more verbose

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CVSS 4.0 (FIRST.org) | Stored as a JSONB object within findings matching the official CVSS 4.0 JSON Schema structure from first.org |
| MITRE ATT&CK (STIX 2.1) | ATT&CK mappings stored as JSONB arrays of technique objects within findings; reference data in relational table |
| CWE | CWE ID as a relational column; CWE metadata available in reference table |
| OWASP WSTG / API Top 10 | Methodology checklists stored as JSONB arrays on engagements, supporting multiple frameworks without separate tables |
| PTES | Engagement phases as relational enum; phase-specific metadata in JSONB |
| CVE Record Format 5.1 | CVE references stored as JSONB matching CVE JSON 5.1 structure |
| OCSF 1.5 | Audit events stored with JSONB payloads aligned to OCSF event classes |
| Scanner-specific formats | Raw scanner output preserved in JSONB `source_data` field on findings |

---

## Organisation, Client & User Management

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "industry": "finance",
    --   "default_methodology": "ptes",
    --   "compliance_frameworks": ["pci_dss_v4", "soc2"],
    --   "branding": {"logo_url": "...", "primary_color": "#1a73e8"},
    --   "integrations": {
    --     "jira": {"base_url": "https://company.atlassian.net", "project_key": "SEC"},
    --     "slack": {"webhook_url": "https://hooks.slack.com/..."}
    --   }
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE clients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {
    --   "industry": "healthcare",
    --   "contacts": [
    --     {"name": "Jane Doe", "email": "jane@client.com", "role": "CISO"},
    --     {"name": "John Smith", "email": "john@client.com", "role": "IT Director"}
    --   ],
    --   "compliance_requirements": ["hipaa", "soc2"],
    --   "data_classification": "confidential",
    --   "jurisdiction": "US-CA"
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    role            VARCHAR(30) NOT NULL DEFAULT 'tester',  -- 'admin', 'tester', 'analyst', 'client_viewer'
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- permissions example:
    -- ["findings:write", "reports:generate", "clients:read", "engagements:manage"]
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "certifications": ["OSCP", "CREST_CRT"],
    --   "specialisations": ["web_application", "api", "cloud"],
    --   "notification_preferences": {
    --     "finding_created": ["email", "slack"],
    --     "report_ready": ["email"]
    --   }
    -- }
    auth_provider   VARCHAR(50) DEFAULT 'local',
    client_access   UUID[] DEFAULT '{}',   -- client IDs this user can access
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_clients_org ON clients(organisation_id);
CREATE INDEX idx_users_org ON users(organisation_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
```

## Assets

```sql
CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES clients(id),
    name            VARCHAR(255) NOT NULL,
    asset_type      VARCHAR(50) NOT NULL,  -- 'host', 'web_application', 'api', 'network', 'wireless', 'mobile_app', 'cloud_service'
    hostname        VARCHAR(255),
    ip_address      INET,
    url             TEXT,
    business_criticality VARCHAR(20) DEFAULT 'medium',
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties example for host:
    -- {
    --   "operating_system": "Ubuntu 22.04",
    --   "environment": "production",
    --   "owner": "Platform Team",
    --   "ports": [
    --     {"port": 443, "protocol": "tcp", "service": "nginx/1.24", "state": "open"},
    --     {"port": 22, "protocol": "tcp", "service": "openssh/8.9", "state": "open"}
    --   ],
    --   "tags": ["dmz", "web-tier"],
    --   "cloud": {"provider": "aws", "region": "us-east-1", "instance_id": "i-0abc123"}
    -- }
    --
    -- properties example for web_application:
    -- {
    --   "technology_stack": ["react", "node.js", "postgresql"],
    --   "authentication": "oauth2",
    --   "has_api": true,
    --   "api_spec_url": "https://app.example.com/openapi.json",
    --   "waf": "cloudflare"
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assets_client ON assets(client_id);
CREATE INDEX idx_assets_type ON assets(asset_type);
CREATE INDEX idx_assets_ip ON assets(ip_address);
CREATE INDEX idx_assets_properties ON assets USING GIN (properties);
```

## Engagements

```sql
CREATE TYPE engagement_status AS ENUM (
    'draft', 'scoping', 'approved', 'in_progress', 'on_hold',
    'testing_complete', 'reporting', 'delivered', 'closed'
);

CREATE TABLE engagements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES clients(id),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    title           VARCHAR(255) NOT NULL,
    engagement_type VARCHAR(50) NOT NULL,
    status          engagement_status NOT NULL DEFAULT 'draft',
    methodology     VARCHAR(50) DEFAULT 'ptes',

    -- Core scheduling (relational for easy querying)
    start_date      DATE,
    end_date        DATE,
    report_due_date DATE,
    lead_tester_id  UUID REFERENCES users(id),

    -- Scope and rules of engagement in JSONB for flexibility
    scope           JSONB NOT NULL DEFAULT '{}',
    -- scope example:
    -- {
    --   "description": "External penetration test of production web applications",
    --   "in_scope_assets": ["asset-uuid-1", "asset-uuid-2"],
    --   "out_of_scope": ["10.0.0.0/8", "staging.example.com"],
    --   "rules_of_engagement": "No DoS testing. Testing window: Mon-Fri 9am-5pm EST.",
    --   "ip_whitelist": ["203.0.113.50", "203.0.113.51"],
    --   "emergency_contacts": [
    --     {"name": "SOC Lead", "phone": "+1-555-0100", "email": "soc@client.com"}
    --   ],
    --   "nda_signed": true,
    --   "nda_document_id": "doc-uuid"
    -- }

    -- Methodology checklist in JSONB — supports any framework
    test_cases      JSONB NOT NULL DEFAULT '[]',
    -- test_cases example (OWASP WSTG):
    -- [
    --   {"id": "WSTG-CONF-01", "name": "Test Network Infrastructure Configuration", "status": "complete", "tester": "user-uuid", "notes": "..."},
    --   {"id": "WSTG-ATHZ-01", "name": "Test Directory Traversal", "status": "in_progress", "tester": "user-uuid"},
    --   {"id": "WSTG-INPV-05", "name": "Testing for SQL Injection", "status": "not_started"}
    -- ]

    -- Team assignment
    team            JSONB NOT NULL DEFAULT '[]',
    -- team example:
    -- [
    --   {"user_id": "uuid", "role": "lead", "assigned_at": "2026-05-01"},
    --   {"user_id": "uuid", "role": "tester", "assigned_at": "2026-05-01"}
    -- ]

    -- Compliance context
    compliance      JSONB NOT NULL DEFAULT '{}',
    -- compliance example:
    -- {
    --   "frameworks": ["pci_dss_v4"],
    --   "controls_assessed": [
    --     {"id": "11.3.1", "status": "pass", "notes": "Annual external pentest completed"},
    --     {"id": "11.3.2", "status": "fail", "notes": "Internal pentest found critical SQL injection"}
    --   ]
    -- }

    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_engagements_client ON engagements(client_id);
CREATE INDEX idx_engagements_org ON engagements(organisation_id);
CREATE INDEX idx_engagements_status ON engagements(status);
CREATE INDEX idx_engagements_dates ON engagements(start_date, end_date);
CREATE INDEX idx_engagements_scope ON engagements USING GIN (scope);
CREATE INDEX idx_engagements_compliance ON engagements USING GIN (compliance);
```

## Findings

```sql
CREATE TYPE severity_level AS ENUM ('critical', 'high', 'medium', 'low', 'informational');

CREATE TABLE findings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_id   UUID NOT NULL REFERENCES engagements(id),
    organisation_id UUID NOT NULL REFERENCES organisations(id),

    -- Stable, universally-queried fields as typed columns
    title           VARCHAR(500) NOT NULL,
    severity        severity_level NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    cwe_id          INTEGER,
    cve_id          VARCHAR(30),
    source          VARCHAR(50) DEFAULT 'manual',
    discovered_by   UUID REFERENCES users(id),
    discovered_at   TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Rich finding data in JSONB
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "description": "The login form at /auth/login is vulnerable to SQL injection...",
    --   "impact": "An attacker could extract the entire user database...",
    --   "likelihood": "high",
    --   "attack_vector": "The username parameter accepts unescaped SQL...",
    --   "proof_of_concept": "POST /auth/login HTTP/1.1\nHost: app.example.com\n...",
    --   "steps_to_reproduce": [
    --     "Navigate to /auth/login",
    --     "Enter ' OR 1=1-- in the username field",
    --     "Observe the SQL error in the response"
    --   ],
    --   "remediation": {
    --     "recommendation": "Use parameterised queries for all database interactions",
    --     "effort": "low",
    --     "references": ["https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html"]
    --   },
    --   "affected_component": "/auth/login"
    -- }

    -- CVSS scoring as JSONB matching FIRST.org CVSS 4.0 JSON Schema
    cvss            JSONB,
    -- cvss example:
    -- {
    --   "version": "4.0",
    --   "vectorString": "CVSS:4.0/AV:N/AC:L/AT:N/PR:N/UI:N/VC:H/VI:H/VA:H/SC:N/SI:N/SA:N",
    --   "baseScore": 9.3,
    --   "baseSeverity": "CRITICAL",
    --   "attackVector": "NETWORK",
    --   "attackComplexity": "LOW",
    --   "attackRequirements": "NONE",
    --   "privilegesRequired": "NONE",
    --   "userInteraction": "NONE",
    --   "vulnConfidentialityImpact": "HIGH",
    --   "vulnIntegrityImpact": "HIGH",
    --   "vulnAvailabilityImpact": "HIGH",
    --   "subConfidentialityImpact": "NONE",
    --   "subIntegrityImpact": "NONE",
    --   "subAvailabilityImpact": "NONE",
    --   "exploitMaturity": "ATTACKED",
    --   "automatable": "YES"
    -- }

    -- ATT&CK mapping as JSONB array
    attack_mappings JSONB NOT NULL DEFAULT '[]',
    -- attack_mappings example:
    -- [
    --   {"technique_id": "T1190", "technique_name": "Exploit Public-Facing Application", "tactic": "Initial Access", "confidence": "high"},
    --   {"technique_id": "T1078", "technique_name": "Valid Accounts", "tactic": "Persistence", "confidence": "medium"}
    -- ]

    -- Affected assets with instance-specific evidence
    affected_assets JSONB NOT NULL DEFAULT '[]',
    -- affected_assets example:
    -- [
    --   {
    --     "asset_id": "asset-uuid",
    --     "asset_name": "app.example.com",
    --     "url_path": "/auth/login",
    --     "parameter": "username",
    --     "evidence": "SQL error returned: You have an error in your SQL syntax..."
    --   },
    --   {
    --     "asset_id": "asset-uuid-2",
    --     "asset_name": "api.example.com",
    --     "url_path": "/api/v1/users",
    --     "parameter": "search",
    --     "evidence": "Blind SQL injection confirmed via time-based technique"
    --   }
    -- ]

    -- Raw scanner data preserved for reference
    source_data     JSONB,
    -- Stores the original scanner output (Burp XML parsed to JSON, Nessus plugin output, etc.)
    -- Enables re-processing if normalisation logic improves

    -- Library reference
    library_finding_id UUID,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Relational indexes for common queries
CREATE INDEX idx_findings_engagement ON findings(engagement_id);
CREATE INDEX idx_findings_org ON findings(organisation_id);
CREATE INDEX idx_findings_severity ON findings(severity);
CREATE INDEX idx_findings_status ON findings(status);
CREATE INDEX idx_findings_cwe ON findings(cwe_id);
CREATE INDEX idx_findings_cve ON findings(cve_id);
CREATE INDEX idx_findings_source ON findings(source);
CREATE INDEX idx_findings_discovered ON findings(discovered_at);

-- JSONB indexes for querying into variable fields
CREATE INDEX idx_findings_cvss_score ON findings ((cvss->>'baseScore'));
CREATE INDEX idx_findings_attack ON findings USING GIN (attack_mappings);
CREATE INDEX idx_findings_affected ON findings USING GIN (affected_assets);
CREATE INDEX idx_findings_details ON findings USING GIN (details);
```

## Evidence (Separate Table for Binary/Large Data)

```sql
CREATE TABLE evidence (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    finding_id      UUID NOT NULL REFERENCES findings(id) ON DELETE CASCADE,
    evidence_type   VARCHAR(50) NOT NULL,
    title           VARCHAR(255),
    content         TEXT,                  -- text content (request/response, code snippets)
    file_path       TEXT,                  -- path in object storage for files/screenshots
    mime_type       VARCHAR(100),
    file_size       BIGINT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {"tool": "burp", "request_method": "POST", "response_code": 500, "timestamp": "2026-05-15T14:30:00Z"}
    sort_order      INTEGER NOT NULL DEFAULT 0,
    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_evidence_finding ON evidence(finding_id);
```

## Findings Library

```sql
CREATE TABLE findings_library (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    title           VARCHAR(500) NOT NULL,
    default_severity severity_level NOT NULL,
    cwe_id          INTEGER,
    details         JSONB NOT NULL DEFAULT '{}',
    -- Same structure as findings.details: description, remediation, references
    attack_mappings JSONB NOT NULL DEFAULT '[]',
    tags            TEXT[],
    usage_count     INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_library_org ON findings_library(organisation_id);
CREATE INDEX idx_library_severity ON findings_library(default_severity);
CREATE INDEX idx_library_tags ON findings_library USING GIN (tags);
```

## Remediation Tracking

```sql
CREATE TABLE remediation_tickets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    finding_id      UUID NOT NULL REFERENCES findings(id),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    external_ticket JSONB,
    -- external_ticket example:
    -- {
    --   "system": "jira",
    --   "ticket_id": "SEC-1234",
    --   "url": "https://company.atlassian.net/browse/SEC-1234",
    --   "assignee": "dev@company.com",
    --   "priority": "high",
    --   "last_synced_at": "2026-05-15T10:00:00Z"
    -- }
    due_date        DATE,
    retests         JSONB NOT NULL DEFAULT '[]',
    -- retests example:
    -- [
    --   {
    --     "id": "retest-uuid",
    --     "tested_by": "user-uuid",
    --     "tested_at": "2026-05-20T14:00:00Z",
    --     "result": "not_fixed",
    --     "evidence": "Vulnerability still present...",
    --     "notes": "Developer applied fix to wrong endpoint"
    --   },
    --   {
    --     "id": "retest-uuid-2",
    --     "tested_by": "user-uuid",
    --     "tested_at": "2026-05-25T10:00:00Z",
    --     "result": "fixed",
    --     "evidence": "Parameterised queries now used. Injection no longer possible."
    --   }
    -- ]
    resolution_notes TEXT,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_remediation_finding ON remediation_tickets(finding_id);
CREATE INDEX idx_remediation_org ON remediation_tickets(organisation_id);
CREATE INDEX idx_remediation_status ON remediation_tickets(status);
CREATE INDEX idx_remediation_due ON remediation_tickets(due_date);
```

## Reports

```sql
CREATE TABLE report_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    framework       VARCHAR(50),
    template_format VARCHAR(20) NOT NULL,
    template_body   TEXT NOT NULL,
    template_config JSONB NOT NULL DEFAULT '{}',
    -- template_config example:
    -- {
    --   "sections": ["executive_summary", "scope", "methodology", "findings", "appendix"],
    --   "finding_sort": "severity_desc",
    --   "include_cvss_breakdown": true,
    --   "include_attack_mapping": true,
    --   "cover_page": {"show_logo": true, "classification": "CONFIDENTIAL"}
    -- }
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_id   UUID NOT NULL REFERENCES engagements(id),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    template_id     UUID REFERENCES report_templates(id),
    title           VARCHAR(255) NOT NULL,
    report_type     VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    output_format   VARCHAR(20) NOT NULL,
    file_path       TEXT,
    content         JSONB NOT NULL DEFAULT '{}',
    -- content example:
    -- {
    --   "executive_summary": "During the engagement period...",
    --   "scope_notes": "The following systems were tested...",
    --   "finding_ids": ["uuid-1", "uuid-2", "uuid-3"],
    --   "finding_overrides": {
    --     "uuid-1": {"custom_narrative": "Report-specific description..."}
    --   },
    --   "statistics": {
    --     "total_findings": 15,
    --     "by_severity": {"critical": 2, "high": 5, "medium": 6, "low": 2}
    --   }
    -- }
    generated_by    UUID REFERENCES users(id),
    approved_by     UUID REFERENCES users(id),
    delivered_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_reports_engagement ON reports(engagement_id);
CREATE INDEX idx_reports_org ON reports(organisation_id);
CREATE INDEX idx_reports_status ON reports(status);
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID,
    changes         JSONB NOT NULL DEFAULT '{}',
    -- changes example:
    -- {
    --   "severity": {"old": "high", "new": "critical"},
    --   "status": {"old": "draft", "new": "confirmed"}
    -- }
    context         JSONB NOT NULL DEFAULT '{}',
    -- context example:
    -- {"ip_address": "192.168.1.100", "user_agent": "...", "source": "web_ui"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

## Reference Data

```sql
CREATE TABLE ref_attack_techniques (
    id              VARCHAR(15) PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    tactic_names    TEXT[],
    is_subtechnique BOOLEAN NOT NULL DEFAULT false,
    parent_id       VARCHAR(15),
    external_url    TEXT,
    last_synced_at  TIMESTAMPTZ
);

CREATE TABLE ref_cwe_weaknesses (
    cwe_id          INTEGER PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(100),
    external_url    TEXT,
    last_synced_at  TIMESTAMPTZ
);
```

---

## JSONB Query Examples

```sql
-- Find all findings with CVSS base score > 7.0
SELECT id, title, severity, cvss->>'baseScore' AS base_score
FROM findings
WHERE (cvss->>'baseScore')::decimal > 7.0
ORDER BY (cvss->>'baseScore')::decimal DESC;

-- Find all findings mapped to a specific ATT&CK technique
SELECT id, title, severity
FROM findings
WHERE attack_mappings @> '[{"technique_id": "T1190"}]';

-- Find all findings affecting a specific asset
SELECT id, title, severity
FROM findings
WHERE affected_assets @> '[{"asset_id": "target-asset-uuid"}]';

-- Find assets with a specific open port
SELECT id, name, hostname
FROM assets
WHERE properties @> '{"ports": [{"port": 443, "state": "open"}]}';

-- Get engagement test case completion stats
SELECT
    e.id,
    e.title,
    jsonb_array_length(e.test_cases) AS total_cases,
    (SELECT count(*) FROM jsonb_array_elements(e.test_cases) tc WHERE tc->>'status' = 'complete') AS completed,
    (SELECT count(*) FROM jsonb_array_elements(e.test_cases) tc WHERE tc->>'status' = 'in_progress') AS in_progress
FROM engagements e
WHERE e.id = 'engagement-uuid';

-- Find all findings with automatable exploit (CVSS 4.0 supplemental metric)
SELECT id, title, severity, cvss->>'baseScore' AS score
FROM findings
WHERE cvss->>'automatable' = 'YES';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Clients | 2 | Settings and contacts in JSONB |
| Users | 1 | Permissions and profile in JSONB |
| Assets | 1 | Port/service/cloud data in JSONB properties |
| Engagements | 1 | Scope, team, test cases, compliance all in JSONB |
| Findings | 1 | CVSS, ATT&CK, affected assets, details all in JSONB |
| Evidence | 1 | Separate table for binary/large evidence files |
| Findings Library | 1 | Reusable templates with JSONB details |
| Remediation | 1 | External tickets and retests in JSONB |
| Reports | 2 | Templates and generated reports |
| Audit Log | 1 | Change tracking with JSONB diffs |
| Reference Data | 2 | ATT&CK techniques and CWE weaknesses |
| **Total** | **14** | Less than half the table count of the normalised model |

---

## Key Design Decisions

1. **JSONB for variable data, typed columns for filterable data** — the rule is: if you filter, sort, or GROUP BY a field frequently, it gets a typed column. Everything else goes in JSONB. Severity, status, CWE ID, and dates are columns; descriptions, proof-of-concept, remediation steps, and scanner-specific data are JSONB.

2. **CVSS stored as FIRST.org-compatible JSON** — the CVSS JSONB field matches the official CVSS 4.0 JSON Schema structure published by FIRST.org. This means scanner imports that already produce CVSS JSON can be stored as-is, and exports produce standards-compliant output without transformation.

3. **ATT&CK mappings denormalised into findings** — rather than a junction table, technique mappings are stored as a JSONB array on each finding. This eliminates a JOIN for the most common read pattern (display a finding with its ATT&CK tags) at the cost of needing GIN index queries for "find all findings with technique X."

4. **Scanner raw data preserved** — the `source_data` JSONB field on findings stores the original scanner output. This means if the normalisation logic improves, findings can be re-processed from the raw data without re-running scans.

5. **Test case checklists as JSONB on engagements** — rather than separate methodology, test_case, and engagement_test_case tables, the checklist lives directly on the engagement. This supports any methodology (OWASP, PTES, custom) without schema changes and makes the engagement a self-contained document.

6. **Retests embedded in remediation tickets** — rather than a separate retest_results table, retests are a JSONB array on the ticket. This keeps the remediation lifecycle in one place and makes it easy to render a complete remediation timeline in the UI.

7. **GIN indexes on all JSONB columns used in queries** — PostgreSQL GIN indexes support the `@>` containment operator, enabling efficient queries like "find findings affecting asset X" or "find findings with ATT&CK technique T1190" without full table scans.

8. **Organisation settings as JSONB** — integration configuration (Jira URLs, Slack webhooks, branding) varies widely per organisation. JSONB avoids a sprawling configuration table and makes it easy to add new integration types.

9. **14 tables instead of 36** — the hybrid approach consolidates what would be junction tables, configuration tables, and phase-specific tables into JSONB fields on their parent entities, dramatically reducing schema complexity.

10. **Application-layer JSON Schema validation** — the trade-off for JSONB flexibility is that the database cannot enforce the structure of JSONB fields. The application layer must validate JSONB payloads against JSON Schema or Zod definitions before writing. This is standard practice in modern TypeScript/Node.js applications.
