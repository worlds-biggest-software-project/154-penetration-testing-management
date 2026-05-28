# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Penetration Testing Management · Created: 2026-05-19

## Philosophy

This model combines a conventional relational schema for operational CRUD (engagements, users, reports) with a property graph layer for relationship-heavy analysis. The graph layer models attack paths, asset-to-vulnerability-to-technique relationships, and cross-engagement finding correlations as nodes and edges — enabling queries like "show all attack paths from the DMZ to the database server" or "find all findings that share an ATT&CK tactic chain with a known APT group."

Penetration testing is fundamentally about relationships: an attacker exploits vulnerability A on asset B, pivots via technique C to asset D, escalates privileges using vulnerability E, and exfiltrates data from asset F. This attack chain is a graph. MITRE ATT&CK itself is a graph (techniques relate to tactics, groups relate to techniques, mitigations relate to techniques). The finding-to-asset-to-engagement-to-client hierarchy is a tree. Cross-engagement finding deduplication is a similarity graph.

Traditional relational models flatten these relationships into junction tables and lose the traversal semantics. A graph-relational hybrid uses PostgreSQL's `ltree` extension for hierarchical paths, a generic `graph_node`/`graph_edge` pattern for flexible relationship modelling, and optionally delegates complex graph queries to a dedicated graph database (Neo4j) via synchronisation. This is the approach used by threat intelligence platforms like ThreatConnect (which maps ATT&CK relationships as a graph) and security orchestration platforms that model kill chains.

**Best for:** Platforms emphasising attack path analysis, cross-engagement correlation, AI-powered finding deduplication, and ATT&CK-aligned threat modelling.

**Trade-offs:**
- (+) Natural representation of attack chains and kill chain analysis
- (+) ATT&CK technique-tactic-group relationships queryable as graph traversals
- (+) Cross-engagement finding correlation via similarity edges
- (+) Asset relationship mapping (network topology, trust relationships)
- (+) Enables AI/ML graph algorithms (community detection, centrality analysis)
- (-) Two data models to maintain — relational CRUD and graph layer
- (-) Graph query language (Cypher or recursive CTEs) steeper learning curve
- (-) Graph synchronisation adds operational complexity
- (-) Graph databases (Neo4j) add infrastructure requirements if used
- (-) Overkill for simple pentest management without attack path analysis

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| MITRE ATT&CK (STIX 2.1) | ATT&CK knowledge base modelled as native graph — techniques, tactics, groups, software as nodes; relationships as edges |
| STIX 2.1 / TAXII | Finding and threat intelligence data uses STIX Relationship Objects pattern (source → relationship_type → target) |
| CVSS 4.0 (FIRST.org) | Scoring stored on finding nodes; supplemental metrics (automatable, value density) become queryable node properties |
| CWE | Weakness hierarchy modelled as graph with parent-child edges matching CWE's ChildOf/ParentOf relationships |
| OCSF 1.5 | Security event nodes align with OCSF event class structure |
| Cyber Kill Chain (Lockheed Martin) | Attack path edges labelled with kill chain phases for strategic analysis |
| PTES | Engagement lifecycle modelled in relational layer; attack path graph captures exploitation and post-exploitation phases |
| ISO/IEC 27001:2022 | Control-to-finding mapping modelled as graph edges, enabling impact analysis across control frameworks |

---

## Relational Layer (Operational CRUD)

### Core Entities

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
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
    role            VARCHAR(30) NOT NULL DEFAULT 'tester',
    auth_provider   VARCHAR(50) DEFAULT 'local',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Engagements and Reports

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

CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES clients(id),
    name            VARCHAR(255) NOT NULL,
    asset_type      VARCHAR(50) NOT NULL,
    hostname        VARCHAR(255),
    ip_address      INET,
    url             TEXT,
    operating_system VARCHAR(100),
    business_criticality VARCHAR(20) DEFAULT 'medium',
    environment     VARCHAR(50),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TYPE severity_level AS ENUM ('critical', 'high', 'medium', 'low', 'informational');

CREATE TABLE findings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_id   UUID NOT NULL REFERENCES engagements(id),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT NOT NULL,
    severity        severity_level NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    cwe_id          INTEGER,
    cve_id          VARCHAR(30),
    cvss_vector     TEXT,
    cvss_base_score DECIMAL(3,1),
    source          VARCHAR(50) DEFAULT 'manual',
    proof_of_concept TEXT,
    remediation_recommendation TEXT,
    discovered_by   UUID REFERENCES users(id),
    discovered_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    engagement_id   UUID NOT NULL REFERENCES engagements(id),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    title           VARCHAR(255) NOT NULL,
    report_type     VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    output_format   VARCHAR(20) NOT NULL,
    file_path       TEXT,
    executive_summary TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE evidence (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    finding_id      UUID NOT NULL REFERENCES findings(id) ON DELETE CASCADE,
    evidence_type   VARCHAR(50) NOT NULL,
    title           VARCHAR(255),
    content         TEXT,
    file_path       TEXT,
    mime_type       VARCHAR(100),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE remediation_tickets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    finding_id      UUID NOT NULL REFERENCES findings(id),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    external_system VARCHAR(50),
    external_ticket_id VARCHAR(255),
    external_ticket_url TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    assigned_to     VARCHAR(255),
    due_date        DATE,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Standard indexes
CREATE INDEX idx_engagements_client ON engagements(client_id);
CREATE INDEX idx_engagements_status ON engagements(status);
CREATE INDEX idx_assets_client ON assets(client_id);
CREATE INDEX idx_findings_engagement ON findings(engagement_id);
CREATE INDEX idx_findings_severity ON findings(severity);
CREATE INDEX idx_findings_cwe ON findings(cwe_id);
CREATE INDEX idx_evidence_finding ON evidence(finding_id);
CREATE INDEX idx_remediation_finding ON remediation_tickets(finding_id);
```

---

## Graph Layer

The graph layer uses a generic node/edge pattern within PostgreSQL. Each domain entity (finding, asset, technique, tactic) becomes a graph node. Relationships (exploits, affects, uses_technique, pivots_to, similar_to) become edges with typed labels and properties.

### Graph Schema

```sql
-- Enable ltree for hierarchical path queries
CREATE EXTENSION IF NOT EXISTS ltree;

CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
    -- Node types: 'finding', 'asset', 'attack_technique', 'attack_tactic',
    --             'cwe_weakness', 'engagement', 'threat_actor', 'malware',
    --             'network_segment', 'credential', 'data_store'
    entity_id       UUID,                  -- links back to relational table (finding.id, asset.id, etc.)
    label           VARCHAR(500) NOT NULL,  -- human-readable label
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties vary by node_type:
    --
    -- finding node:
    -- {"severity": "critical", "cvss_score": 9.3, "cwe_id": 89, "status": "confirmed"}
    --
    -- asset node:
    -- {"asset_type": "host", "ip": "10.0.1.50", "os": "Windows Server 2022", "criticality": "high"}
    --
    -- attack_technique node:
    -- {"technique_id": "T1110", "name": "Brute Force", "platforms": ["windows", "linux"]}
    --
    -- network_segment node:
    -- {"cidr": "10.0.1.0/24", "name": "DMZ", "zone": "external"}
    --
    -- credential node:
    -- {"type": "password_hash", "algorithm": "NTLM", "privilege_level": "domain_admin"}
    hierarchy_path  LTREE,                 -- for hierarchical traversal (e.g., 'tactic.technique.subtechnique')
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
    -- Edge types:
    --   'exploits'         — finding → asset (this vulnerability was exploited on this asset)
    --   'affects'          — finding → asset (this vulnerability affects this asset)
    --   'uses_technique'   — finding → attack_technique (ATT&CK mapping)
    --   'pivots_to'        — asset → asset (lateral movement path)
    --   'escalates_via'    — finding → finding (privilege escalation chain)
    --   'similar_to'       — finding → finding (cross-engagement dedup similarity)
    --   'belongs_to'       — attack_technique → attack_tactic (ATT&CK hierarchy)
    --   'child_of'         — cwe → cwe (CWE weakness hierarchy)
    --   'connected_to'     — asset → asset (network connectivity)
    --   'hosts'            — asset → asset (VM on hypervisor, container on host)
    --   'authenticates_to' — credential → asset (credential grants access to asset)
    --   'leads_to'         — finding → finding (attack chain step)
    --   'mitigated_by'     — finding → remediation action
    --   'tested_in'        — finding → engagement
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties vary by edge_type:
    --
    -- 'pivots_to':
    -- {"method": "pass_the_hash", "protocol": "smb", "step_order": 2, "kill_chain_phase": "lateral_movement"}
    --
    -- 'similar_to':
    -- {"similarity_score": 0.92, "algorithm": "tfidf_cosine", "computed_at": "2026-05-15"}
    --
    -- 'uses_technique':
    -- {"confidence": "high", "mapped_by": "user-uuid"}
    --
    -- 'exploits':
    -- {"timestamp": "2026-05-10T14:30:00Z", "success": true, "impact": "remote_code_execution"}
    weight          DECIMAL(5,4) DEFAULT 1.0,  -- for graph algorithms (similarity, importance)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Core graph indexes
CREATE INDEX idx_graph_nodes_org ON graph_nodes(organisation_id);
CREATE INDEX idx_graph_nodes_type ON graph_nodes(node_type);
CREATE INDEX idx_graph_nodes_entity ON graph_nodes(entity_id);
CREATE INDEX idx_graph_nodes_properties ON graph_nodes USING GIN (properties);
CREATE INDEX idx_graph_nodes_hierarchy ON graph_nodes USING GIST (hierarchy_path);

CREATE INDEX idx_graph_edges_org ON graph_edges(organisation_id);
CREATE INDEX idx_graph_edges_source ON graph_edges(source_node_id);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_node_id);
CREATE INDEX idx_graph_edges_type ON graph_edges(edge_type);
CREATE INDEX idx_graph_edges_source_type ON graph_edges(source_node_id, edge_type);
CREATE INDEX idx_graph_edges_target_type ON graph_edges(target_node_id, edge_type);
CREATE INDEX idx_graph_edges_properties ON graph_edges USING GIN (properties);
```

---

## Graph Query Examples

### Attack Path Traversal

```sql
-- Find all attack paths from a specific asset (e.g., DMZ web server) to a target (e.g., database server)
-- Using recursive CTE for multi-hop traversal
WITH RECURSIVE attack_paths AS (
    -- Start from the source asset
    SELECT
        e.id AS edge_id,
        e.source_node_id,
        e.target_node_id,
        e.edge_type,
        e.properties,
        ARRAY[e.source_node_id, e.target_node_id] AS path,
        1 AS depth
    FROM graph_edges e
    JOIN graph_nodes src ON src.id = e.source_node_id
    WHERE src.entity_id = 'dmz-webserver-asset-uuid'
      AND e.edge_type IN ('pivots_to', 'exploits', 'escalates_via')

    UNION ALL

    -- Traverse further
    SELECT
        e.id,
        e.source_node_id,
        e.target_node_id,
        e.edge_type,
        e.properties,
        ap.path || e.target_node_id,
        ap.depth + 1
    FROM graph_edges e
    JOIN attack_paths ap ON ap.target_node_id = e.source_node_id
    WHERE e.edge_type IN ('pivots_to', 'exploits', 'escalates_via')
      AND NOT (e.target_node_id = ANY(ap.path))  -- prevent cycles
      AND ap.depth < 10  -- max path length
)
SELECT
    ap.path,
    ap.depth,
    ap.edge_type,
    ap.properties,
    target.label AS target_label,
    target.properties AS target_properties
FROM attack_paths ap
JOIN graph_nodes target ON target.id = ap.target_node_id
WHERE target.entity_id = 'database-server-asset-uuid'
ORDER BY ap.depth ASC;
```

### Cross-Engagement Finding Similarity

```sql
-- Find findings similar to a given finding across all engagements
SELECT
    target_node.entity_id AS similar_finding_id,
    target_node.label AS similar_finding_title,
    target_node.properties->>'severity' AS severity,
    e.properties->>'similarity_score' AS similarity_score,
    e.properties->>'algorithm' AS algorithm
FROM graph_edges e
JOIN graph_nodes source_node ON source_node.id = e.source_node_id
JOIN graph_nodes target_node ON target_node.id = e.target_node_id
WHERE source_node.entity_id = 'current-finding-uuid'
  AND e.edge_type = 'similar_to'
  AND (e.properties->>'similarity_score')::decimal > 0.8
ORDER BY (e.properties->>'similarity_score')::decimal DESC;
```

### ATT&CK Coverage Analysis

```sql
-- For a given engagement, show which ATT&CK tactics were covered and which techniques were used
SELECT
    tactic_node.label AS tactic,
    technique_node.label AS technique,
    technique_node.properties->>'technique_id' AS technique_id,
    finding_node.label AS finding_title,
    finding_node.properties->>'severity' AS severity
FROM graph_edges finding_technique
JOIN graph_nodes finding_node ON finding_node.id = finding_technique.source_node_id
JOIN graph_nodes technique_node ON technique_node.id = finding_technique.target_node_id
JOIN graph_edges technique_tactic ON technique_tactic.source_node_id = technique_node.id
JOIN graph_nodes tactic_node ON tactic_node.id = technique_tactic.target_node_id
WHERE finding_node.node_type = 'finding'
  AND finding_technique.edge_type = 'uses_technique'
  AND technique_tactic.edge_type = 'belongs_to'
  AND finding_node.properties->>'engagement_id' = 'engagement-uuid'
ORDER BY tactic_node.label, technique_node.label;
```

### Asset Blast Radius Analysis

```sql
-- From a compromised asset, what other assets are reachable within N hops?
WITH RECURSIVE reachable AS (
    SELECT
        n.id AS node_id,
        n.entity_id,
        n.label,
        n.properties,
        0 AS hops,
        ARRAY[n.id] AS path
    FROM graph_nodes n
    WHERE n.entity_id = 'compromised-asset-uuid'

    UNION ALL

    SELECT
        target.id,
        target.entity_id,
        target.label,
        target.properties,
        r.hops + 1,
        r.path || target.id
    FROM reachable r
    JOIN graph_edges e ON e.source_node_id = r.node_id
    JOIN graph_nodes target ON target.id = e.target_node_id
    WHERE e.edge_type IN ('pivots_to', 'connected_to', 'hosts')
      AND NOT (target.id = ANY(r.path))
      AND r.hops < 5
)
SELECT entity_id, label, properties, hops
FROM reachable
WHERE hops > 0
ORDER BY hops, (properties->>'criticality') DESC;
```

### Network Topology via ltree

```sql
-- Model network hierarchy: org.site.zone.subnet.host
-- e.g., 'acme.hq.dmz.web_tier.webserver01'

-- Find all assets in the DMZ zone
SELECT id, label, properties
FROM graph_nodes
WHERE hierarchy_path <@ 'acme.hq.dmz'
  AND node_type = 'asset';

-- Find all assets at the same level as a given asset
SELECT id, label, properties
FROM graph_nodes
WHERE hierarchy_path ~ 'acme.hq.*.web_tier.*'
  AND node_type = 'asset';
```

---

## Graph Synchronisation (Optional Neo4j Layer)

For very large deployments or advanced graph analytics, a Neo4j synchronisation layer can be added:

```sql
-- Sync tracking table
CREATE TABLE graph_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sync_type       VARCHAR(20) NOT NULL,  -- 'full', 'incremental'
    status          VARCHAR(20) NOT NULL,  -- 'started', 'completed', 'failed'
    nodes_synced    INTEGER DEFAULT 0,
    edges_synced    INTEGER DEFAULT 0,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    error_message   TEXT
);

-- Change tracking for incremental sync
CREATE TABLE graph_change_queue (
    id              BIGSERIAL PRIMARY KEY,
    operation       VARCHAR(10) NOT NULL,  -- 'INSERT', 'UPDATE', 'DELETE'
    table_name      VARCHAR(50) NOT NULL,  -- 'graph_nodes' or 'graph_edges'
    record_id       UUID NOT NULL,
    change_data     JSONB,
    processed       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_change_unprocessed ON graph_change_queue(processed) WHERE processed = false;
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
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Clients | 2 | Standard relational |
| Users | 1 | Standard relational |
| Engagements | 1 | Operational lifecycle management |
| Assets | 1 | Asset inventory |
| Findings | 1 | Core finding data |
| Evidence | 1 | Finding evidence and attachments |
| Remediation | 1 | Ticket tracking |
| Reports | 1 | Generated reports |
| Graph Nodes | 1 | All domain entities as graph nodes |
| Graph Edges | 1 | All relationships between nodes |
| Graph Sync (Optional) | 2 | Neo4j sync log and change queue |
| Audit Log | 1 | Change tracking |
| **Total** | **14** | Plus 2 optional for Neo4j sync |

---

## Key Design Decisions

1. **Dual-layer architecture** — relational tables handle standard CRUD operations (create engagement, add finding, generate report) while the graph layer handles relationship-heavy analysis (attack paths, similarity, ATT&CK coverage). Application code writes to both layers transactionally.

2. **Generic graph_node/graph_edge pattern** — rather than purpose-built tables for every relationship type, a generic node/edge pattern with typed labels and JSONB properties provides unlimited flexibility. New node types or edge types require zero schema changes.

3. **PostgreSQL as the graph database** — for most pentest management deployments (hundreds of engagements, thousands of findings), PostgreSQL's recursive CTEs and ltree extension are sufficient for graph queries. Neo4j is optional for very large deployments.

4. **STIX 2.1 relationship pattern** — graph edges follow the STIX Relationship Object pattern (source → relationship_type → target) used by MITRE ATT&CK. This means ATT&CK data can be loaded directly into the graph layer without transformation.

5. **Similarity edges for AI deduplication** — the `similar_to` edge type with a `similarity_score` property enables ML-based finding deduplication. An async job computes TF-IDF or embedding cosine similarity between findings and creates edges for pairs above a threshold.

6. **Weighted edges for graph algorithms** — the `weight` column on edges enables PageRank-style importance analysis (which assets are most central?), shortest-path analysis (fastest attack path?), and community detection (which findings cluster together?).

7. **ltree for hierarchical data** — PostgreSQL's ltree extension models network topology hierarchies (org.site.zone.subnet.host) and ATT&CK hierarchies (tactic.technique.subtechnique) with efficient ancestor/descendant queries.

8. **Optional Neo4j synchronisation** — for deployments that outgrow PostgreSQL's graph capabilities, a change queue pattern enables incremental sync to Neo4j for Cypher-based querying without disrupting the primary data layer.

9. **Graph nodes reference relational entities** — the `entity_id` column on graph nodes links back to the relational table's primary key. This means standard CRUD operations use the relational layer (fast, indexed, well-understood) while graph queries traverse the graph layer.

10. **Attack path as first-class concept** — unlike the other models where attack chains must be inferred from finding descriptions, this model makes attack paths explicit as sequences of edges. This enables visualisation (network attack maps), quantitative analysis (path length, blast radius), and AI-powered attack chain inference.
