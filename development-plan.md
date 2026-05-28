# Development Plan: Penetration Testing Management

> Project: 154 - Penetration Testing Management
> Generated: 2026-05-25
> Status: Comprehensive phased development plan

---

## Technology Decisions

### Database: PostgreSQL 16+ with Hybrid Relational + JSONB Model (Data Model Suggestion 3)

**Rationale:** Data Model Suggestion 3 (Hybrid Relational + JSONB) is selected as the primary schema approach, incorporating audit trail patterns from Suggestion 2 and graph-based attack path analysis from Suggestion 4 in later phases.

- **Why not pure normalized (Suggestion 1)?** 36 tables provides excellent referential integrity but slows MVP velocity. Scanner-specific fields, jurisdiction-dependent compliance metadata, and methodology-variant test case checklists all vary per engagement and would require DDL migrations in a normalized model, which is impractical for a multi-tenant platform supporting diverse client environments.
- **Why not event-sourced (Suggestion 2)?** CQRS/ES adds substantial operational complexity (projection maintenance, eventual consistency, snapshot management, replay infrastructure) that is unjustified at the MVP stage. A partitioned `audit_log` table with JSONB change payloads in the hybrid model provides the compliance audit trail required by ISO 27001, PCI DSS, and NIST SP 800-115 without the architectural overhead. Event sourcing patterns can be adopted selectively for the findings lifecycle in a later phase if temporal query demand warrants it.
- **Why Suggestion 3?** 14 tables with JSONB for CVSS scores, ATT&CK mappings, scanner raw data, engagement scope definitions, test case checklists, and remediation retests balances query performance (relational columns for severity, status, CWE ID, dates used in WHERE/JOIN/GROUP BY) with schema flexibility (JSONB for variable attributes). This mirrors PlexTrac's own approach where findings are "nested JSON objects" with stable fields extracted for querying. It enables shipping an MVP significantly faster while preserving the ability to evolve.
- **Graph elements from Suggestion 4 deferred to Phase 10.** The `graph_nodes` / `graph_edges` tables, `ltree` hierarchy paths, and recursive CTE attack path queries will be introduced when attack path visualization and cross-engagement correlation are built. The hybrid relational schema supports all MVP and v1.1 queries efficiently; graph traversal becomes necessary only for multi-hop relationship analysis (lateral movement chains, blast radius calculation, AI-powered finding deduplication).

### Backend: Node.js (TypeScript) with Fastify

**Rationale:**
- TypeScript provides type safety across the stack, critical for a security platform where CVSS score calculations, status transitions, and RBAC policy enforcement must be correct.
- Fastify outperforms Express by 2-3x on JSON serialization benchmarks, important for dashboards aggregating hundreds of findings with CVSS details.
- Native JSON Schema validation in Fastify aligns with the JSONB validation strategy: application-layer validation of JSONB payloads (CVSS objects, ATT&CK mappings, scope definitions) using JSON Schema or Zod before database writes.
- OpenAPI 3.1 spec generation via `@fastify/swagger` satisfies the standards.md requirement for published API documentation, matching the Cobalt and PlexTrac API documentation patterns.
- Prisma ORM for type-safe database access with migration management; raw SQL for complex JSONB aggregation queries and GIN-indexed containment lookups.

### Frontend: Next.js 15 (App Router) with React 19

**Rationale:**
- Server Components reduce client-side JavaScript for findings dashboards and engagement detail views that are primarily read-heavy.
- App Router's layout nesting maps naturally to the client > engagement > findings > evidence navigation hierarchy.
- React Server Actions simplify form mutations (create finding, update severity, log retest result) without dedicated API endpoints for every form.
- Tailwind CSS + shadcn/ui for consistent, accessible UI components without building a design system from scratch.
- Recharts for findings severity distribution charts, remediation timeline visualizations, and compliance posture dashboards. @tanstack/react-table for data-dense findings lists with filtering, sorting, and column customization.

### AI Layer: LangChain.js with Claude API

**Rationale:**
- Claude (Opus / Sonnet models) for natural-language finding narrative generation, executive summary drafting, and scope modelling assistance. Claude's long context window (200K tokens) can process entire engagement findings sets in a single query for report generation.
- LangChain.js for structured tool-calling (the AI agent queries the findings database via defined tools rather than generating raw SQL), retrieval-augmented generation for report narrative drafting from the findings library, and prompt templating for different report frameworks (OWASP, PTES, NIST).
- AI-assisted CVSS scoring: given a finding description and affected component, suggest base metric values with justification.
- Deduplication pipeline: embedding-based similarity computation for cross-engagement finding matching.

### Authentication: NextAuth.js v5 with OIDC/SAML

**Rationale:**
- NextAuth.js v5 supports OAuth 2.0 / OpenID Connect natively and can proxy SAML 2.0 via identity provider federation (Okta, Azure AD / Entra ID), matching the SSO integrations offered by PlexTrac and Cobalt.
- Row-Level Security (RLS) policies on PostgreSQL enforce tenant isolation at the database layer, independent of application bugs. This is essential for a multi-tenant pentest management platform where findings data must be strictly isolated between organisations.
- Client-level access control: users can be scoped to specific clients within an organisation, preventing analysts from viewing findings for clients they are not assigned to.

### Deployment: Docker Compose (self-hosted) + Vercel (managed cloud)

**Rationale:**
- Docker Compose for the self-hosted deployment mode, addressing the air-gapped and data sovereignty deployment gap identified in the features research. Security consultancies and government clients require full control over findings data.
- Vercel for the managed cloud offering: Next.js is natively optimised for Vercel, edge functions handle API routing, and Vercel Postgres provides managed PostgreSQL.
- CI/CD via GitHub Actions with migration checks, type checking, linting, and test execution on every PR.

### Testing: Vitest + Playwright + pgTAP

**Rationale:**
- Vitest for unit and integration tests (CVSS score calculation, findings import parsing, RBAC policy evaluation, scanner normalisation logic).
- Playwright for end-to-end tests (findings dashboard rendering, engagement creation flows, report generation, client portal views).
- pgTAP for database-level tests (RLS policies, JSONB constraint validation, GIN index performance verification).

### Report Generation: Puppeteer + Handlebars

**Rationale:**
- Handlebars templates for HTML report generation, supporting framework-specific layouts (OWASP, PTES, NIST) with customizable sections.
- Puppeteer for PDF rendering from HTML templates, producing client-ready pentest reports with consistent formatting, cover pages, and branding.
- Separation of template engine from rendering engine allows future migration to alternative PDF libraries without changing templates.

---

## Project Structure

```
penetration-testing-management/
  apps/
    web/                        # Next.js 15 frontend
      src/
        app/                    # App Router pages and layouts
          (auth)/               # Login, SSO callback routes
          (dashboard)/          # Authenticated layout
            engagements/        # Engagement list, detail, scoping
            findings/           # Findings list, detail, evidence
            clients/            # Client list and detail
            assets/             # Asset inventory management
            reports/            # Report generation and delivery
            remediation/        # Remediation tracking dashboard
            library/            # Findings library management
            compliance/         # Compliance framework mapping
            analytics/          # Findings analytics and trends
            settings/           # Org config, integrations, RBAC
            ai/                 # AI assistant interface
          portal/               # Client portal (separate auth)
            [client-slug]/      # Client-specific findings view
        components/
          ui/                   # shadcn/ui base components
          engagement/           # Engagement-specific components
          finding/              # Finding detail, CVSS calculator, ATT&CK mapper
          report/               # Report preview and template components
          remediation/          # Remediation timeline and status
          analytics/            # Charts and dashboards
        lib/
          api/                  # API client functions
          hooks/                # React hooks
          utils/                # Formatters, CVSS calculators, validators
    api/                        # Fastify backend API
      src/
        routes/                 # REST API route modules
          engagements/
          findings/
          clients/
          assets/
          reports/
          remediation/
          library/
          compliance/
          analytics/
          integrations/
          ai/
          admin/
          portal/
        services/               # Business logic layer
        models/                 # Prisma-generated types + custom types
        plugins/                # Fastify plugins (auth, RLS, audit)
        jobs/                   # Background job definitions
          scanner-import/       # Scanner output parsing (Burp, Nessus, Qualys)
          report-generation/    # PDF/HTML report rendering
          ai-enrichment/        # AI classification, deduplication
          integration-sync/     # Jira, ServiceNow ticket sync
        ai/                     # AI agent tools and prompts
          tools/                # LangChain tool definitions
          prompts/              # Prompt templates for report generation
        parsers/                # Scanner output parsers
          burp/
          nessus/
          qualys/
          nuclei/
          generic/
        validators/             # JSON Schema / Zod validators for JSONB columns
  packages/
    db/                         # Prisma schema, migrations, seeds
      prisma/
        schema.prisma
        migrations/
        seeds/
          attack-techniques.ts  # MITRE ATT&CK reference data seeder
          cwe-weaknesses.ts     # CWE reference data seeder
          methodology-cases.ts  # OWASP WSTG / PTES test case seeder
    shared/                     # Shared types, constants, utilities
      src/
        types/                  # Domain type definitions
        cvss/                   # CVSS 4.0 calculator library
        constants/              # Severity levels, statuses, enums
    eslint-config/              # Shared ESLint configuration
    tsconfig/                   # Shared TypeScript configuration
  docker/
    docker-compose.yml          # Self-hosted deployment
    Dockerfile.api
    Dockerfile.web
  docs/
    api/                        # Generated OpenAPI docs
    architecture/               # Architecture decision records
  tests/
    e2e/                        # Playwright E2E tests
    db/                         # pgTAP database tests
```

---

## Phase Dependency Graph

```
Phase 1: Foundation & Auth
    |
    v
Phase 2: Engagement & Client CRUD
    |
    +---------------------------+
    |                           |
    v                           v
Phase 3: Findings Core       Phase 4: Asset Management
  & Evidence                    & Scope
    |                           |
    +---------------------------+
    |
    v
Phase 5: CVSS Scoring, ATT&CK Mapping & Findings Library
    |
    v
Phase 6: Report Generation & Client Portal
    |
    +---------------------------+
    |                           |
    v                           v
Phase 7: Remediation         Phase 8: Scanner Integration
  Tracking & Retesting          & Import Pipeline
    |                           |
    +---------------------------+
    |
    v
Phase 9: AI-Assisted Workflows (Report Generation, Classification, Deduplication)
    |
    v
Phase 10: Attack Path Visualization & Cross-Engagement Analytics
    |
    v
Phase 11: Integrations (Jira, ServiceNow, Slack, CI/CD)
    |
    v
Phase 12: Continuous Testing, Compliance Automation & Enterprise Features
```

**Critical path:** Phases 1 -> 2 -> 3+4 -> 5 -> 6 -> 9 -> 10

---

## Phase 1: Foundation & Authentication

**Duration:** 3 weeks
**Depends on:** Nothing (starting point)

### Definition of Done
- PostgreSQL database provisioned with organisation, user, and audit_log tables with RLS policies active
- User can register an organisation, invite users, and sign in via email/password and Google OIDC
- RBAC system enforces organisation-level role scoping (admin, tester, analyst, client_viewer)
- API server returns 401 for unauthenticated requests and 403 for unauthorised requests
- Audit log captures all authentication and authorisation events with JSONB change payloads
- CI pipeline runs type checking, linting, unit tests, and database tests on every PR

### Task 1.1: Database Foundation & Multi-Tenancy

**What:** Create the PostgreSQL database schema for organisations, users, and audit_log tables. Implement Row-Level Security policies for tenant isolation. Set up Prisma ORM with migration management.

**Design:**

```prisma
// packages/db/prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Organisation {
  id            String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  name          String   @db.VarChar(255)
  slug          String   @unique @db.VarChar(255)
  settings      Json     @default("{}")
  // settings JSONB: industry, default_methodology, compliance_frameworks,
  //   branding (logo_url, primary_color), integrations (jira, slack config)
  isActive      Boolean  @default(true) @map("is_active")
  createdAt     DateTime @default(now()) @map("created_at") @db.Timestamptz
  updatedAt     DateTime @updatedAt @map("updated_at") @db.Timestamptz

  users   User[]
  clients Client[]

  @@map("organisations")
}

model User {
  id             String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  organisationId String    @map("organisation_id") @db.Uuid
  email          String    @unique @db.VarChar(255)
  displayName    String    @map("display_name") @db.VarChar(255)
  passwordHash   String?   @map("password_hash") @db.VarChar(255)
  role           String    @default("tester") @db.VarChar(30)
  permissions    Json      @default("[]")
  profile        Json      @default("{}")
  authProvider   String    @default("local") @map("auth_provider") @db.VarChar(50)
  clientAccess   String[]  @default([]) @map("client_access") @db.Uuid
  isActive       Boolean   @default(true) @map("is_active")
  lastLoginAt    DateTime? @map("last_login_at") @db.Timestamptz
  createdAt      DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt      DateTime  @updatedAt @map("updated_at") @db.Timestamptz

  organisation Organisation @relation(fields: [organisationId], references: [id], onDelete: Cascade)

  @@unique([organisationId, email])
  @@index([organisationId])
  @@index([email])
  @@map("users")
}
```

```sql
-- packages/db/migrations/001_rls_policies.sql

ALTER TABLE organisations ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_users ON users
  USING (organisation_id = current_setting('app.current_org_id')::UUID);

CREATE OR REPLACE FUNCTION set_tenant_context(org_id UUID)
RETURNS VOID AS $$
BEGIN
  PERFORM set_config('app.current_org_id', org_id::TEXT, true);
END;
$$ LANGUAGE plpgsql;
```

```typescript
// apps/api/src/plugins/tenant.ts

import { FastifyPluginAsync } from 'fastify';
import fp from 'fastify-plugin';

const tenantPlugin: FastifyPluginAsync = async (fastify) => {
  fastify.addHook('preHandler', async (request) => {
    const orgId = request.user?.organisationId;
    if (orgId) {
      await fastify.prisma.$executeRaw`SELECT set_tenant_context(${orgId}::UUID)`;
    }
  });
};

export default fp(tenantPlugin);
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 1.1.1 | Create organisation with valid data | Integration | Organisation created, slug generated, default settings applied |
| 1.1.2 | Attempt to create organisation with duplicate slug | Integration | 409 Conflict returned |
| 1.1.3 | Query users with RLS policy active for Org A | pgTAP | Only Org A users returned; Org B users invisible |
| 1.1.4 | Query users without tenant context set | pgTAP | Empty result set (RLS blocks all rows) |
| 1.1.5 | Prisma migration applies cleanly to empty database | Integration | All tables created, indexes exist, RLS policies active |
| 1.1.6 | UUID primary keys are valid v4 UUIDs | Unit | gen_random_uuid() produces valid UUIDs |

### Task 1.2: Authentication (NextAuth.js v5 + OIDC)

**What:** Implement authentication with email/password registration, Google OIDC login, and session management using NextAuth.js v5. Generate JWT access tokens for API requests.

**Design:**

```typescript
// apps/web/src/app/api/auth/[...nextauth]/route.ts

import NextAuth from 'next-auth';
import GoogleProvider from 'next-auth/providers/google';
import CredentialsProvider from 'next-auth/providers/credentials';
import { PrismaAdapter } from '@auth/prisma-adapter';
import { prisma } from '@/lib/prisma';
import { verifyPassword } from '@/lib/auth/passwords';

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: PrismaAdapter(prisma),
  session: { strategy: 'jwt', maxAge: 24 * 60 * 60 },
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
    CredentialsProvider({
      name: 'credentials',
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        const user = await prisma.user.findFirst({
          where: { email: credentials.email as string, isActive: true },
          include: { organisation: true },
        });
        if (!user || !user.passwordHash) return null;
        const valid = await verifyPassword(
          credentials.password as string,
          user.passwordHash
        );
        if (!valid) return null;
        return {
          id: user.id,
          email: user.email,
          name: user.displayName,
          organisationId: user.organisationId,
          role: user.role,
        };
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.organisationId = user.organisationId;
        token.role = user.role;
      }
      return token;
    },
    async session({ session, token }) {
      session.user.organisationId = token.organisationId as string;
      session.user.role = token.role as string;
      return session;
    },
  },
});
```

```typescript
// apps/api/src/plugins/auth.ts

import { FastifyPluginAsync, FastifyRequest } from 'fastify';
import fp from 'fastify-plugin';
import jwt from '@fastify/jwt';

interface AuthUser {
  id: string;
  email: string;
  organisationId: string;
  role: 'admin' | 'tester' | 'analyst' | 'client_viewer';
}

declare module 'fastify' {
  interface FastifyRequest {
    user: AuthUser;
  }
}

const authPlugin: FastifyPluginAsync = async (fastify) => {
  await fastify.register(jwt, { secret: process.env.JWT_SECRET! });

  fastify.decorate('authenticate', async (request: FastifyRequest) => {
    await request.jwtVerify();
  });

  fastify.decorate('authorise', (...roles: string[]) => {
    return async (request: FastifyRequest) => {
      if (!roles.includes(request.user.role)) {
        throw fastify.httpErrors.forbidden('Insufficient permissions');
      }
    };
  });
};

export default fp(authPlugin);
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 1.2.1 | Register new user with valid email and password | Integration | User created, password hashed with argon2, 201 returned |
| 1.2.2 | Login with valid credentials | Integration | JWT token returned with organisationId and role claims |
| 1.2.3 | Login with invalid password | Integration | 401 Unauthorized returned |
| 1.2.4 | Login with inactive user account | Integration | 401 Unauthorized returned |
| 1.2.5 | Access protected endpoint without token | E2E | 401 Unauthorized returned |
| 1.2.6 | Access admin endpoint with tester role | E2E | 403 Forbidden returned |
| 1.2.7 | Google OIDC login creates new user with linked organisation | Integration | User created with auth_provider='google' |
| 1.2.8 | JWT token expires after 24 hours | Unit | Token verification fails after TTL |

### Task 1.3: Audit Logging Infrastructure

**What:** Create the audit_log table with JSONB change payloads. Implement a Fastify plugin that automatically logs all state-changing API requests with actor, entity, old/new values, and request metadata.

**Design:**

```sql
-- packages/db/migrations/002_audit_log.sql

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    user_id         UUID,
    action          VARCHAR(50) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID,
    changes         JSONB NOT NULL DEFAULT '{}',
    context         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE audit_log_2026_q1 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE audit_log_2026_q2 PARTITION OF audit_log
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');

CREATE INDEX idx_audit_org ON audit_log(organisation_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

```typescript
// apps/api/src/services/audit.ts

interface AuditEntry {
  organisationId: string;
  userId?: string;
  action: 'create' | 'update' | 'delete' | 'login' | 'export' | 'report_generate';
  entityType: string;
  entityId?: string;
  changes?: Record<string, { old: unknown; new: unknown }>;
  context?: Record<string, unknown>;
}

export class AuditService {
  constructor(private readonly prisma: PrismaClient) {}

  async log(entry: AuditEntry): Promise<void> {
    await this.prisma.auditLog.create({
      data: {
        organisationId: entry.organisationId,
        userId: entry.userId,
        action: entry.action,
        entityType: entry.entityType,
        entityId: entry.entityId,
        changes: entry.changes ?? {},
        context: entry.context ?? {},
      },
    });
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 1.3.1 | Create entity logs audit entry with action='create' | Integration | Audit log entry created with correct entity_type, entity_id, new values |
| 1.3.2 | Update entity logs audit entry with old and new values | Integration | changes JSONB contains {field: {old: ..., new: ...}} |
| 1.3.3 | Login attempt logs audit entry | Integration | action='login' recorded with IP address and user agent |
| 1.3.4 | Audit log entries cannot be updated | pgTAP | UPDATE on audit_log fails or is prevented by policy |
| 1.3.5 | Audit log entries cannot be deleted | pgTAP | DELETE on audit_log fails or is prevented by policy |
| 1.3.6 | Partition routing places records in correct quarterly partition | pgTAP | Q2 2026 record appears in audit_log_2026_q2 partition |

### Task 1.4: CI/CD Pipeline Setup

**What:** Configure GitHub Actions CI/CD pipeline with type checking, linting, unit tests, integration tests, database migration verification, and Docker build validation.

**Design:**

```yaml
# .github/workflows/ci.yml

name: CI
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: '22' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm run lint
      - run: pnpm run typecheck

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: pentest_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
      - uses: actions/setup-node@v4
        with: { node-version: '22' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm run db:migrate
      - run: pnpm run test:unit
      - run: pnpm run test:integration
      - run: pnpm run test:db
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 1.4.1 | CI pipeline runs on pull request | Manual | All jobs execute: lint, typecheck, test |
| 1.4.2 | Failed typecheck blocks merge | Manual | PR cannot be merged with TypeScript errors |
| 1.4.3 | Database migration runs successfully in CI | Integration | All migrations apply to fresh PostgreSQL 16 instance |

---

## Phase 2: Engagement & Client CRUD

**Duration:** 3 weeks
**Depends on:** Phase 1

### Definition of Done
- Client CRUD with metadata JSONB (contacts, industry, compliance requirements)
- Engagement CRUD with status lifecycle (draft -> scoping -> approved -> in_progress -> testing_complete -> reporting -> delivered -> closed)
- Engagement scoping with JSONB scope definition (in-scope assets, rules of engagement, IP whitelist, emergency contacts)
- Client-level access control enforced (users only see clients they are assigned to)
- Dashboard view listing engagements by status with severity count summaries
- All state transitions logged to audit trail

### Task 2.1: Client Management

**What:** Implement client CRUD with JSONB metadata for contacts, industry, compliance requirements, and jurisdiction. Enforce client-level access control so users only see clients they are assigned to.

**Design:**

```typescript
// packages/shared/src/types/client.ts

export interface ClientMetadata {
  industry?: string;
  contacts: Array<{
    name: string;
    email: string;
    role: string;
    phone?: string;
  }>;
  complianceRequirements?: string[];
  dataClassification?: 'public' | 'internal' | 'confidential' | 'restricted';
  jurisdiction?: string;
}

export interface CreateClientInput {
  name: string;
  slug: string;
  metadata: ClientMetadata;
}

export interface ClientResponse {
  id: string;
  organisationId: string;
  name: string;
  slug: string;
  metadata: ClientMetadata;
  isActive: boolean;
  createdAt: string;
  updatedAt: string;
}
```

```typescript
// apps/api/src/routes/clients/index.ts

import { FastifyInstance } from 'fastify';
import { CreateClientInput, ClientResponse } from '@pentest/shared';

export default async function clientRoutes(fastify: FastifyInstance) {
  fastify.get<{ Reply: ClientResponse[] }>('/', {
    preHandler: [fastify.authenticate],
    handler: async (request) => {
      const clients = await fastify.prisma.client.findMany({
        where: {
          organisationId: request.user.organisationId,
          ...(request.user.role === 'client_viewer'
            ? { id: { in: request.user.clientAccess } }
            : {}),
        },
        orderBy: { name: 'asc' },
      });
      return clients;
    },
  });

  fastify.post<{ Body: CreateClientInput; Reply: ClientResponse }>('/', {
    preHandler: [fastify.authenticate, fastify.authorise('admin', 'analyst')],
    handler: async (request) => {
      const client = await fastify.prisma.client.create({
        data: {
          organisationId: request.user.organisationId,
          name: request.body.name,
          slug: request.body.slug,
          metadata: request.body.metadata,
        },
      });
      await fastify.audit.log({
        organisationId: request.user.organisationId,
        userId: request.user.id,
        action: 'create',
        entityType: 'client',
        entityId: client.id,
      });
      return client;
    },
  });
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 2.1.1 | Create client with valid metadata | Integration | Client created, 201 returned, metadata stored as JSONB |
| 2.1.2 | Create client with duplicate slug in same org | Integration | 409 Conflict returned |
| 2.1.3 | List clients as admin | Integration | All organisation clients returned |
| 2.1.4 | List clients as client_viewer with restricted access | Integration | Only assigned clients returned |
| 2.1.5 | Update client metadata contacts | Integration | JSONB metadata updated, audit entry created with old/new values |
| 2.1.6 | Deactivate client | Integration | is_active set to false, client hidden from default list |

### Task 2.2: Engagement Lifecycle Management

**What:** Implement engagement CRUD with JSONB scope definition, status lifecycle transitions with validation rules, team assignment, and methodology selection. Store rules of engagement, IP whitelists, and emergency contacts in the scope JSONB field.

**Design:**

```typescript
// packages/shared/src/types/engagement.ts

export type EngagementStatus =
  | 'draft' | 'scoping' | 'approved' | 'in_progress' | 'on_hold'
  | 'testing_complete' | 'reporting' | 'delivered' | 'closed';

export type EngagementType =
  | 'external_network' | 'internal_network' | 'web_application' | 'api'
  | 'mobile_application' | 'wireless' | 'social_engineering' | 'physical'
  | 'red_team' | 'purple_team' | 'cloud' | 'iot';

export interface EngagementScope {
  description: string;
  inScopeAssets: string[];    // asset UUIDs
  outOfScope: string[];       // CIDR ranges, hostnames
  rulesOfEngagement: string;
  ipWhitelist: string[];
  emergencyContacts: Array<{
    name: string;
    phone: string;
    email: string;
  }>;
  ndaSigned: boolean;
  ndaDocumentId?: string;
}

export interface EngagementTeamMember {
  userId: string;
  role: 'lead' | 'tester' | 'reviewer' | 'observer';
  assignedAt: string;
}

// Valid status transitions
export const ENGAGEMENT_TRANSITIONS: Record<EngagementStatus, EngagementStatus[]> = {
  draft: ['scoping'],
  scoping: ['approved', 'draft'],
  approved: ['in_progress'],
  in_progress: ['on_hold', 'testing_complete'],
  on_hold: ['in_progress', 'closed'],
  testing_complete: ['reporting'],
  reporting: ['delivered'],
  delivered: ['closed'],
  closed: [],
};
```

```typescript
// apps/api/src/services/engagement.ts

export class EngagementService {
  async transitionStatus(
    engagementId: string,
    newStatus: EngagementStatus,
    userId: string
  ): Promise<Engagement> {
    const engagement = await this.prisma.engagement.findUniqueOrThrow({
      where: { id: engagementId },
    });

    const allowed = ENGAGEMENT_TRANSITIONS[engagement.status as EngagementStatus];
    if (!allowed.includes(newStatus)) {
      throw new Error(
        `Invalid transition: ${engagement.status} -> ${newStatus}. ` +
        `Allowed: ${allowed.join(', ')}`
      );
    }

    const updated = await this.prisma.engagement.update({
      where: { id: engagementId },
      data: { status: newStatus },
    });

    await this.audit.log({
      organisationId: engagement.organisationId,
      userId,
      action: 'update',
      entityType: 'engagement',
      entityId: engagementId,
      changes: { status: { old: engagement.status, new: newStatus } },
    });

    return updated;
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 2.2.1 | Create engagement with full scope definition | Integration | Engagement created with scope JSONB populated, status='draft' |
| 2.2.2 | Transition draft -> scoping | Integration | Status updated, audit entry logged |
| 2.2.3 | Transition draft -> in_progress (invalid) | Integration | 400 Bad Request, invalid transition error message |
| 2.2.4 | Transition in_progress -> testing_complete | Integration | Status updated, engagement end_date not required |
| 2.2.5 | Assign testers to engagement team | Integration | Team JSONB updated with user roles |
| 2.2.6 | List engagements filtered by status | Integration | Only engagements matching status filter returned |
| 2.2.7 | List engagements for restricted client_viewer user | Integration | Only engagements for assigned clients visible |
| 2.2.8 | Engagement dashboard returns severity count summary | Integration | Response includes finding_count, critical_count, high_count per engagement |

### Task 2.3: Engagement Dashboard UI

**What:** Build the engagement list dashboard showing all engagements with status badges, severity breakdown chips, date ranges, and lead tester assignment. Implement engagement detail page with scope viewer, team roster, and status transition buttons.

**Design:**

```typescript
// apps/web/src/app/(dashboard)/engagements/page.tsx

import { EngagementListItem } from '@pentest/shared';

interface EngagementCardProps {
  engagement: EngagementListItem;
}

function EngagementCard({ engagement }: EngagementCardProps) {
  return (
    <Card>
      <CardHeader>
        <div className="flex items-center justify-between">
          <CardTitle>{engagement.title}</CardTitle>
          <StatusBadge status={engagement.status} />
        </div>
        <CardDescription>
          {engagement.clientName} · {engagement.engagementType}
        </CardDescription>
      </CardHeader>
      <CardContent>
        <SeverityBreakdown counts={engagement.severityCounts} />
        <div className="mt-2 text-sm text-muted-foreground">
          {engagement.startDate} - {engagement.endDate}
        </div>
      </CardContent>
    </Card>
  );
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 2.3.1 | Engagement list renders with status badges | E2E | All engagement cards display with correct status colours |
| 2.3.2 | Engagement detail page shows scope definition | E2E | In-scope assets, rules of engagement, emergency contacts visible |
| 2.3.3 | Status transition button only shows valid transitions | E2E | "Move to Scoping" visible for draft; "Move to In Progress" hidden |
| 2.3.4 | Empty state displays when no engagements exist | E2E | "Create your first engagement" prompt shown |

---

## Phase 3: Findings Core & Evidence

**Duration:** 3 weeks
**Depends on:** Phase 2

### Definition of Done
- Finding CRUD with relational columns for severity, status, CWE ID, CVE ID and JSONB details for description, impact, PoC, steps to reproduce, remediation
- Evidence attachment system supporting screenshots, request/response captures, code snippets, and files
- Finding status lifecycle (draft -> confirmed -> reported -> accepted -> remediation_planned -> remediated -> verified_fixed)
- Findings list with filtering by severity, status, CWE, engagement
- Finding detail page with evidence gallery and timeline

### Task 3.1: Findings Database & CRUD API

**What:** Create the findings table with the hybrid relational + JSONB approach. Implement CRUD API routes with Zod validation for the details JSONB structure. Support manual finding creation and status transitions.

**Design:**

```typescript
// packages/shared/src/types/finding.ts

import { z } from 'zod';

export const FindingDetailsSchema = z.object({
  description: z.string().min(10),
  impact: z.string().optional(),
  likelihood: z.enum(['high', 'medium', 'low']).optional(),
  attackVector: z.string().optional(),
  proofOfConcept: z.string().optional(),
  stepsToReproduce: z.array(z.string()).optional(),
  remediation: z.object({
    recommendation: z.string(),
    effort: z.enum(['trivial', 'low', 'medium', 'high', 'complex']).optional(),
    references: z.array(z.string().url()).optional(),
  }).optional(),
  affectedComponent: z.string().optional(),
});

export type FindingDetails = z.infer<typeof FindingDetailsSchema>;

export const FindingAffectedAssetSchema = z.object({
  assetId: z.string().uuid(),
  assetName: z.string(),
  urlPath: z.string().optional(),
  parameter: z.string().optional(),
  evidence: z.string().optional(),
});

export type FindingAffectedAsset = z.infer<typeof FindingAffectedAssetSchema>;

export type FindingStatus =
  | 'draft' | 'confirmed' | 'reported' | 'accepted'
  | 'remediation_planned' | 'remediated' | 'verified_fixed'
  | 'risk_accepted' | 'false_positive';

export type SeverityLevel = 'critical' | 'high' | 'medium' | 'low' | 'informational';

export interface CreateFindingInput {
  engagementId: string;
  title: string;
  severity: SeverityLevel;
  cweId?: number;
  cveId?: string;
  source?: string;
  details: FindingDetails;
  affectedAssets?: FindingAffectedAsset[];
}
```

```typescript
// apps/api/src/routes/findings/index.ts

export default async function findingRoutes(fastify: FastifyInstance) {
  fastify.post<{ Body: CreateFindingInput }>('/', {
    preHandler: [fastify.authenticate, fastify.authorise('admin', 'tester', 'analyst')],
    handler: async (request) => {
      const validated = FindingDetailsSchema.parse(request.body.details);
      const finding = await fastify.prisma.finding.create({
        data: {
          engagementId: request.body.engagementId,
          organisationId: request.user.organisationId,
          title: request.body.title,
          severity: request.body.severity,
          status: 'draft',
          cweId: request.body.cweId,
          cveId: request.body.cveId,
          source: request.body.source ?? 'manual',
          discoveredBy: request.user.id,
          details: validated,
          affectedAssets: request.body.affectedAssets ?? [],
          attackMappings: [],
        },
      });
      await fastify.audit.log({
        organisationId: request.user.organisationId,
        userId: request.user.id,
        action: 'create',
        entityType: 'finding',
        entityId: finding.id,
      });
      return finding;
    },
  });
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 3.1.1 | Create finding with full details JSONB | Integration | Finding created with validated JSONB, status='draft' |
| 3.1.2 | Create finding with invalid details (missing description) | Integration | 400 Bad Request, Zod validation error |
| 3.1.3 | List findings filtered by severity=critical | Integration | Only critical findings returned |
| 3.1.4 | List findings filtered by status=confirmed | Integration | Only confirmed findings returned |
| 3.1.5 | List findings filtered by cweId=89 | Integration | Only SQL injection findings returned |
| 3.1.6 | Update finding severity from high to critical | Integration | Severity updated, audit entry with old/new values |
| 3.1.7 | Transition finding status draft -> confirmed | Integration | Status updated, audit logged |
| 3.1.8 | Transition finding status draft -> verified_fixed (invalid) | Integration | 400 Bad Request, invalid transition |
| 3.1.9 | Findings list respects client-level access control | Integration | User only sees findings for assigned clients |

### Task 3.2: Evidence Attachment System

**What:** Create the evidence table and file upload API. Support screenshot uploads (PNG, JPG), HTTP request/response captures (text), code snippets, log files, and generic file attachments. Store binary files in object storage (local filesystem for self-hosted, S3 for cloud).

**Design:**

```typescript
// apps/api/src/routes/findings/evidence.ts

export default async function evidenceRoutes(fastify: FastifyInstance) {
  fastify.post<{ Params: { findingId: string } }>(
    '/:findingId/evidence',
    {
      preHandler: [fastify.authenticate, fastify.authorise('admin', 'tester')],
      handler: async (request) => {
        const parts = request.parts();
        for await (const part of parts) {
          if (part.type === 'file') {
            const filePath = await fastify.storage.upload(part);
            await fastify.prisma.evidence.create({
              data: {
                findingId: request.params.findingId,
                evidenceType: classifyMimeType(part.mimetype),
                title: part.filename,
                filePath,
                mimeType: part.mimetype,
                fileSize: part.file.bytesRead,
                metadata: {},
                uploadedBy: request.user.id,
              },
            });
          }
        }
      },
    }
  );
}

function classifyMimeType(mime: string): string {
  if (mime.startsWith('image/')) return 'screenshot';
  if (mime === 'text/plain') return 'log';
  if (mime === 'application/json') return 'request_response';
  return 'file';
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 3.2.1 | Upload PNG screenshot to finding | Integration | Evidence record created, file stored, file_path populated |
| 3.2.2 | Upload HTTP request/response as text | Integration | Evidence created with evidence_type='request_response' |
| 3.2.3 | Upload file exceeding size limit (50MB) | Integration | 413 Payload Too Large returned |
| 3.2.4 | Download evidence file by ID | Integration | File returned with correct Content-Type header |
| 3.2.5 | Delete evidence removes file from storage | Integration | Evidence record deleted, file removed from storage |
| 3.2.6 | Evidence respects organisation-level RLS | pgTAP | Evidence for Org B finding invisible to Org A user |

### Task 3.3: Finding Detail & Evidence UI

**What:** Build the finding detail page with full details display, severity badge, status transition controls, evidence gallery with lightbox, and inline evidence upload. Build the findings list page with multi-column filtering and sorting.

**Design:**

```typescript
// apps/web/src/components/finding/FindingDetail.tsx

interface FindingDetailProps {
  finding: FindingResponse;
  evidence: EvidenceResponse[];
  onStatusChange: (newStatus: FindingStatus) => Promise<void>;
}

export function FindingDetail({ finding, evidence, onStatusChange }: FindingDetailProps) {
  return (
    <div className="space-y-6">
      <FindingHeader finding={finding} onStatusChange={onStatusChange} />
      <Tabs defaultValue="details">
        <TabsList>
          <TabsTrigger value="details">Details</TabsTrigger>
          <TabsTrigger value="evidence">Evidence ({evidence.length})</TabsTrigger>
          <TabsTrigger value="affected">Affected Assets</TabsTrigger>
          <TabsTrigger value="cvss">CVSS Score</TabsTrigger>
          <TabsTrigger value="attack">ATT&CK Mapping</TabsTrigger>
          <TabsTrigger value="remediation">Remediation</TabsTrigger>
        </TabsList>
        <TabsContent value="details">
          <FindingDetailsPanel details={finding.details} />
        </TabsContent>
        <TabsContent value="evidence">
          <EvidenceGallery evidence={evidence} findingId={finding.id} />
        </TabsContent>
      </Tabs>
    </div>
  );
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 3.3.1 | Finding detail page renders all tabs | E2E | Details, Evidence, Affected Assets, CVSS, ATT&CK, Remediation tabs present |
| 3.3.2 | Evidence gallery displays uploaded screenshots | E2E | Screenshots render in gallery grid with lightbox on click |
| 3.3.3 | Finding list filters by severity and status | E2E | Filter controls narrow displayed findings correctly |
| 3.3.4 | Finding list sorts by severity descending | E2E | Critical findings appear first |

---

## Phase 4: Asset Management & Scope

**Duration:** 2 weeks
**Depends on:** Phase 2

### Definition of Done
- Asset CRUD with JSONB properties for port/service/cloud/technology-stack data
- Assets linked to clients; engagement scoping references assets
- Asset types: host, web_application, API, network, wireless, mobile_app, cloud_service
- Asset inventory filterable by type, client, business criticality
- Engagement scope page shows in-scope and out-of-scope assets

### Task 4.1: Asset Inventory CRUD

**What:** Create the assets table with JSONB properties field. Implement CRUD API with asset type-specific property validation using Zod discriminated unions.

**Design:**

```typescript
// packages/shared/src/types/asset.ts

import { z } from 'zod';

export type AssetType =
  | 'host' | 'web_application' | 'api' | 'network'
  | 'wireless' | 'mobile_app' | 'cloud_service';

export const HostPropertiesSchema = z.object({
  operatingSystem: z.string().optional(),
  environment: z.enum(['production', 'staging', 'development']).optional(),
  owner: z.string().optional(),
  ports: z.array(z.object({
    port: z.number().int().min(1).max(65535),
    protocol: z.enum(['tcp', 'udp']),
    service: z.string().optional(),
    state: z.enum(['open', 'closed', 'filtered']),
  })).optional(),
  tags: z.array(z.string()).optional(),
  cloud: z.object({
    provider: z.enum(['aws', 'azure', 'gcp']),
    region: z.string(),
    instanceId: z.string().optional(),
  }).optional(),
});

export const WebAppPropertiesSchema = z.object({
  technologyStack: z.array(z.string()).optional(),
  authentication: z.string().optional(),
  hasApi: z.boolean().optional(),
  apiSpecUrl: z.string().url().optional(),
  waf: z.string().optional(),
});

export interface CreateAssetInput {
  clientId: string;
  name: string;
  assetType: AssetType;
  hostname?: string;
  ipAddress?: string;
  url?: string;
  businessCriticality: 'critical' | 'high' | 'medium' | 'low';
  properties: Record<string, unknown>;
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 4.1.1 | Create host asset with port/service properties | Integration | Asset created with JSONB properties validated |
| 4.1.2 | Create web_application asset with technology stack | Integration | Asset created with WebApp-specific properties |
| 4.1.3 | Filter assets by type='web_application' | Integration | Only web application assets returned |
| 4.1.4 | Filter assets by business_criticality='critical' | Integration | Only critical assets returned |
| 4.1.5 | Search assets by IP address | Integration | GIN index supports efficient INET lookup |
| 4.1.6 | Search assets with specific open port via JSONB containment | Integration | `properties @> '{"ports": [{"port": 443, "state": "open"}]}'` returns matching assets |

### Task 4.2: Engagement Scope Asset Linking

**What:** Extend the engagement scope JSONB to include validated asset references. Build UI component for selecting in-scope and out-of-scope assets from the client's asset inventory when defining engagement scope.

**Design:**

```typescript
// apps/api/src/services/engagement-scope.ts

export class EngagementScopeService {
  async setScope(
    engagementId: string,
    scope: EngagementScope,
    userId: string
  ): Promise<void> {
    // Validate all referenced asset IDs exist and belong to the engagement's client
    const engagement = await this.prisma.engagement.findUniqueOrThrow({
      where: { id: engagementId },
    });

    if (scope.inScopeAssets.length > 0) {
      const validAssets = await this.prisma.asset.findMany({
        where: {
          id: { in: scope.inScopeAssets },
          clientId: engagement.clientId,
          isActive: true,
        },
        select: { id: true },
      });
      const validIds = new Set(validAssets.map((a) => a.id));
      const invalid = scope.inScopeAssets.filter((id) => !validIds.has(id));
      if (invalid.length > 0) {
        throw new Error(`Invalid asset IDs: ${invalid.join(', ')}`);
      }
    }

    await this.prisma.engagement.update({
      where: { id: engagementId },
      data: { scope },
    });
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 4.2.1 | Set engagement scope with valid asset IDs | Integration | Scope JSONB updated with asset references |
| 4.2.2 | Set scope with asset ID from different client | Integration | 400 Bad Request, invalid asset reference |
| 4.2.3 | Set scope with non-existent asset ID | Integration | 400 Bad Request, invalid asset ID error |
| 4.2.4 | Scope UI renders asset selector with client's inventory | E2E | Asset picker shows all client assets with checkboxes |
| 4.2.5 | Scope page displays in-scope and out-of-scope sections | E2E | Two panels showing included and excluded assets/ranges |

---

## Phase 5: CVSS Scoring, ATT&CK Mapping & Findings Library

**Duration:** 3 weeks
**Depends on:** Phase 3

### Definition of Done
- CVSS 4.0 scoring calculator that computes base, threat, environmental, and supplemental scores from metric inputs
- CVSS score stored as JSONB on findings matching FIRST.org CVSS 4.0 JSON Schema
- Interactive CVSS calculator UI component with metric selector and real-time score preview
- MITRE ATT&CK technique and tactic reference data seeded from STIX 2.1 data
- Finding-to-technique mapping stored in attack_mappings JSONB array
- ATT&CK technique browser/picker UI with tactic grouping
- Reusable findings library with CRUD and usage tracking
- "Apply from library" workflow that populates new findings from library templates

### Task 5.1: CVSS 4.0 Calculator Library

**What:** Implement the CVSS 4.0 scoring algorithm as a shared TypeScript library. The calculator accepts metric values and produces the base score, threat score, environmental score, and vector string conforming to the FIRST.org CVSS 4.0 JSON Schema.

**Design:**

```typescript
// packages/shared/src/cvss/cvss4.ts

export interface CvssV4Metrics {
  // Base Metrics
  attackVector: 'NETWORK' | 'ADJACENT' | 'LOCAL' | 'PHYSICAL';
  attackComplexity: 'LOW' | 'HIGH';
  attackRequirements: 'NONE' | 'PRESENT';
  privilegesRequired: 'NONE' | 'LOW' | 'HIGH';
  userInteraction: 'NONE' | 'PASSIVE' | 'ACTIVE';
  vulnConfidentialityImpact: 'NONE' | 'LOW' | 'HIGH';
  vulnIntegrityImpact: 'NONE' | 'LOW' | 'HIGH';
  vulnAvailabilityImpact: 'NONE' | 'LOW' | 'HIGH';
  subConfidentialityImpact: 'NONE' | 'LOW' | 'HIGH';
  subIntegrityImpact: 'NONE' | 'LOW' | 'HIGH';
  subAvailabilityImpact: 'NONE' | 'LOW' | 'HIGH';

  // Threat Metrics (optional)
  exploitMaturity?: 'NOT_DEFINED' | 'ATTACKED' | 'POC' | 'UNREPORTED';

  // Supplemental Metrics (optional)
  automatable?: 'NOT_DEFINED' | 'NO' | 'YES';
  recovery?: 'NOT_DEFINED' | 'AUTOMATIC' | 'USER' | 'IRRECOVERABLE';
  valueDensity?: 'NOT_DEFINED' | 'DIFFUSE' | 'CONCENTRATED';
  safety?: 'NOT_DEFINED' | 'NEGLIGIBLE' | 'PRESENT';
  providerUrgency?: 'NOT_DEFINED' | 'RED' | 'AMBER' | 'GREEN' | 'CLEAR';
}

export interface CvssV4Result {
  version: '4.0';
  vectorString: string;
  baseScore: number;
  baseSeverity: 'NONE' | 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL';
  threatScore?: number;
  environmentalScore?: number;
}

export function computeCvssV4(metrics: CvssV4Metrics): CvssV4Result {
  const vectorString = buildVectorString(metrics);
  const baseScore = calculateBaseScore(metrics);
  const baseSeverity = scoreToseverity(baseScore);

  return {
    version: '4.0',
    vectorString,
    baseScore,
    baseSeverity,
    ...metrics,
  };
}

function scoreToseverity(score: number): CvssV4Result['baseSeverity'] {
  if (score === 0) return 'NONE';
  if (score < 4.0) return 'LOW';
  if (score < 7.0) return 'MEDIUM';
  if (score < 9.0) return 'HIGH';
  return 'CRITICAL';
}

function buildVectorString(metrics: CvssV4Metrics): string {
  return `CVSS:4.0/AV:${metrics.attackVector[0]}/AC:${metrics.attackComplexity[0]}` +
    `/AT:${metrics.attackRequirements[0]}/PR:${metrics.privilegesRequired[0]}` +
    `/UI:${metrics.userInteraction[0]}` +
    `/VC:${metrics.vulnConfidentialityImpact[0]}` +
    `/VI:${metrics.vulnIntegrityImpact[0]}` +
    `/VA:${metrics.vulnAvailabilityImpact[0]}` +
    `/SC:${metrics.subConfidentialityImpact[0]}` +
    `/SI:${metrics.subIntegrityImpact[0]}` +
    `/SA:${metrics.subAvailabilityImpact[0]}`;
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 5.1.1 | Compute CVSS for network/low/none RCE (all high impacts) | Unit | Base score 9.3, severity CRITICAL |
| 5.1.2 | Compute CVSS for local/high/present info disclosure | Unit | Base score in LOW range |
| 5.1.3 | Compute CVSS with no vulnerability impact (all NONE) | Unit | Base score 0.0, severity NONE |
| 5.1.4 | Vector string format matches FIRST.org specification | Unit | String starts with "CVSS:4.0/" and contains all base metrics |
| 5.1.5 | Parse existing vector string back to metrics | Unit | Round-trip: metrics -> vector -> metrics produces identical values |
| 5.1.6 | Supplemental metrics included in result when provided | Unit | automatable, recovery, safety fields present in result |

### Task 5.2: MITRE ATT&CK Reference Data & Mapping

**What:** Seed the ref_attack_techniques table from the official ATT&CK STIX 2.1 data bundle. Implement the attack_mappings JSONB array on findings with a technique picker UI that groups techniques by tactic.

**Design:**

```typescript
// packages/db/seeds/attack-techniques.ts

import attackData from './data/enterprise-attack.json';

interface StixTechnique {
  type: string;
  id: string;
  name: string;
  external_references: Array<{ source_name: string; external_id: string; url: string }>;
  x_mitre_is_subtechnique: boolean;
  kill_chain_phases?: Array<{ kill_chain_name: string; phase_name: string }>;
}

export async function seedAttackTechniques(prisma: PrismaClient) {
  const techniques = attackData.objects
    .filter((obj: StixTechnique) => obj.type === 'attack-pattern')
    .map((tech: StixTechnique) => {
      const extRef = tech.external_references.find(
        (r) => r.source_name === 'mitre-attack'
      );
      return {
        id: extRef?.external_id ?? tech.id,
        name: tech.name,
        tacticNames: tech.kill_chain_phases?.map((p) => p.phase_name) ?? [],
        isSubtechnique: tech.x_mitre_is_subtechnique ?? false,
        parentId: tech.x_mitre_is_subtechnique
          ? extRef?.external_id?.split('.')[0]
          : null,
        externalUrl: extRef?.url,
        lastSyncedAt: new Date(),
      };
    });

  await prisma.refAttackTechnique.createMany({
    data: techniques,
    skipDuplicates: true,
  });
}
```

```typescript
// packages/shared/src/types/attack-mapping.ts

export interface AttackMapping {
  techniqueId: string;    // e.g., 'T1190'
  techniqueName: string;  // e.g., 'Exploit Public-Facing Application'
  tactic: string;         // e.g., 'Initial Access'
  confidence: 'high' | 'medium' | 'low';
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 5.2.1 | Seed ATT&CK techniques from STIX data | Integration | 600+ techniques seeded with tactic mappings |
| 5.2.2 | Add ATT&CK mapping to finding | Integration | attack_mappings JSONB array updated with technique entry |
| 5.2.3 | Query findings by ATT&CK technique via GIN index | Integration | `attack_mappings @> '[{"techniqueId": "T1190"}]'` returns correct findings |
| 5.2.4 | ATT&CK technique picker UI groups by tactic | E2E | Techniques organized under tactic headings (Initial Access, Execution, etc.) |
| 5.2.5 | Remove ATT&CK mapping from finding | Integration | Technique removed from attack_mappings array |

### Task 5.3: Reusable Findings Library

**What:** Create the findings_library table for storing vetted, reusable finding templates. Implement CRUD API and an "Apply from Library" workflow that populates a new finding's details from a library template while preserving instance-specific evidence and affected assets.

**Design:**

```typescript
// apps/api/src/services/findings-library.ts

export class FindingsLibraryService {
  async applyToFinding(
    libraryFindingId: string,
    engagementId: string,
    overrides: Partial<CreateFindingInput>,
    userId: string
  ): Promise<Finding> {
    const template = await this.prisma.findingsLibrary.findUniqueOrThrow({
      where: { id: libraryFindingId },
    });

    const finding = await this.prisma.finding.create({
      data: {
        engagementId,
        organisationId: template.organisationId,
        title: overrides.title ?? template.title,
        severity: overrides.severity ?? template.defaultSeverity,
        status: 'draft',
        cweId: template.cweId,
        source: 'library',
        discoveredBy: userId,
        details: { ...template.details, ...overrides.details },
        attackMappings: template.attackMappings,
        affectedAssets: overrides.affectedAssets ?? [],
        libraryFindingId,
      },
    });

    // Increment usage counter
    await this.prisma.findingsLibrary.update({
      where: { id: libraryFindingId },
      data: { usageCount: { increment: 1 } },
    });

    return finding;
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 5.3.1 | Create library finding template | Integration | Template created with default severity, CWE, details |
| 5.3.2 | Apply library template to new finding | Integration | Finding created with template details, library_finding_id set, usage_count incremented |
| 5.3.3 | Apply template with overrides | Integration | Override fields take precedence over template defaults |
| 5.3.4 | Search library by tag | Integration | GIN index on tags array supports efficient tag search |
| 5.3.5 | Library list sorted by usage_count descending | Integration | Most-used templates appear first |
| 5.3.6 | Library UI shows template preview before applying | E2E | Preview modal displays title, severity, description, remediation |

---

## Phase 6: Report Generation & Client Portal

**Duration:** 3 weeks
**Depends on:** Phase 5

### Definition of Done
- Report template management with customizable sections and framework alignment (OWASP, PTES, NIST)
- PDF and HTML report generation from engagement findings using Handlebars templates and Puppeteer
- Report lifecycle: draft -> review -> approved -> delivered
- Executive summary section with finding severity breakdown
- Client portal with read-only access to engagement findings and reports
- Secure portal authentication separate from internal user auth

### Task 6.1: Report Template Engine

**What:** Build the report template system using Handlebars templates with configurable sections. Support OWASP, PTES, and NIST report frameworks. Store template configuration in the template_config JSONB field.

**Design:**

```typescript
// apps/api/src/services/report-generator.ts

export interface ReportContext {
  engagement: EngagementResponse;
  client: ClientResponse;
  findings: FindingResponse[];
  statistics: {
    totalFindings: number;
    bySeverity: Record<SeverityLevel, number>;
    byStatus: Record<FindingStatus, number>;
    byCwe: Array<{ cweId: number; cweName: string; count: number }>;
  };
  executiveSummary?: string;
  scopeNotes?: string;
  methodology?: string;
  generatedAt: string;
  generatedBy: string;
}

export class ReportGenerator {
  private handlebars: typeof Handlebars;

  async generatePdf(
    templateId: string,
    engagementId: string,
    options: { findingIds?: string[]; customNarratives?: Record<string, string> }
  ): Promise<Buffer> {
    const template = await this.loadTemplate(templateId);
    const context = await this.buildContext(engagementId, options);
    const html = this.renderHtml(template, context);
    const pdf = await this.htmlToPdf(html);
    return pdf;
  }

  private async htmlToPdf(html: string): Promise<Buffer> {
    const browser = await puppeteer.launch({ headless: true });
    const page = await browser.newPage();
    await page.setContent(html, { waitUntil: 'networkidle0' });
    const pdf = await page.pdf({
      format: 'A4',
      margin: { top: '2cm', bottom: '2cm', left: '2cm', right: '2cm' },
      printBackground: true,
    });
    await browser.close();
    return Buffer.from(pdf);
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 6.1.1 | Generate HTML report from OWASP template | Integration | HTML output contains engagement title, severity breakdown, all findings |
| 6.1.2 | Generate PDF report from engagement | Integration | Valid PDF file produced, openable in PDF reader |
| 6.1.3 | Report includes CVSS scores and ATT&CK mappings | Integration | Each finding section shows CVSS vector and ATT&CK techniques |
| 6.1.4 | Report excludes findings not in findingIds list | Integration | Only selected findings appear in report |
| 6.1.5 | Custom narrative overrides finding description in report | Integration | Report uses custom narrative instead of details.description |
| 6.1.6 | Report statistics match actual finding data | Unit | Severity counts, CWE groupings computed correctly |

### Task 6.2: Report Lifecycle & Delivery

**What:** Implement report CRUD with lifecycle (draft -> review -> approved -> delivered). Store generated reports in object storage. Implement approval workflow where a reviewer must approve before delivery.

**Design:**

```typescript
// apps/api/src/services/report-lifecycle.ts

export const REPORT_TRANSITIONS: Record<string, string[]> = {
  draft: ['review'],
  review: ['approved', 'draft'],
  approved: ['delivered'],
  delivered: [],
};

export class ReportLifecycleService {
  async approve(reportId: string, userId: string): Promise<Report> {
    const report = await this.prisma.report.findUniqueOrThrow({
      where: { id: reportId },
    });
    if (report.status !== 'review') {
      throw new Error('Report must be in review status to approve');
    }
    return this.prisma.report.update({
      where: { id: reportId },
      data: { status: 'approved', approvedBy: userId },
    });
  }

  async deliver(reportId: string, userId: string): Promise<Report> {
    const report = await this.prisma.report.findUniqueOrThrow({
      where: { id: reportId },
      include: { engagement: { include: { client: true } } },
    });
    if (report.status !== 'approved') {
      throw new Error('Report must be approved before delivery');
    }
    // Send notification to client contacts
    await this.notifyClientContacts(report);
    return this.prisma.report.update({
      where: { id: reportId },
      data: { status: 'delivered', deliveredAt: new Date() },
    });
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 6.2.1 | Create report from engagement | Integration | Report created with status='draft', linked to engagement |
| 6.2.2 | Submit report for review | Integration | Status transitions to 'review' |
| 6.2.3 | Approve report | Integration | Status transitions to 'approved', approved_by set |
| 6.2.4 | Deliver unapproved report (invalid) | Integration | 400 Bad Request, must be approved first |
| 6.2.5 | Download generated report PDF | Integration | PDF file returned with correct headers |

### Task 6.3: Client Portal

**What:** Build a separate client portal with its own authentication flow. Clients access the portal via a unique URL with a client-specific slug. The portal shows read-only views of engagement findings, severity breakdown, and downloadable reports. Portal authentication uses time-limited invite tokens.

**Design:**

```typescript
// apps/web/src/app/portal/[client-slug]/page.tsx

export default async function ClientPortalDashboard({
  params,
}: {
  params: { 'client-slug': string };
}) {
  const portalSession = await getPortalSession();
  const engagements = await fetchPortalEngagements(
    portalSession.clientId,
    portalSession.token
  );

  return (
    <div className="container mx-auto py-8">
      <h1 className="text-2xl font-bold mb-6">Security Assessment Portal</h1>
      <SeveritySummaryCards engagements={engagements} />
      <EngagementTimeline engagements={engagements} />
      <FindingsSummaryTable engagements={engagements} />
    </div>
  );
}
```

```typescript
// apps/api/src/routes/portal/auth.ts

export default async function portalAuthRoutes(fastify: FastifyInstance) {
  fastify.post<{ Body: { token: string } }>('/portal/auth', {
    handler: async (request) => {
      const invite = await fastify.prisma.portalInvite.findFirst({
        where: {
          token: request.body.token,
          expiresAt: { gt: new Date() },
          usedAt: null,
        },
        include: { client: true },
      });
      if (!invite) {
        throw fastify.httpErrors.unauthorized('Invalid or expired invite');
      }
      const portalToken = fastify.jwt.sign({
        clientId: invite.clientId,
        email: invite.email,
        type: 'portal',
      }, { expiresIn: '7d' });
      return { token: portalToken, clientName: invite.client.name };
    },
  });
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 6.3.1 | Generate portal invite token for client contact | Integration | Invite created with expiry, unique token generated |
| 6.3.2 | Authenticate to portal with valid invite token | Integration | Portal JWT issued, invite marked as used |
| 6.3.3 | Authenticate with expired invite token | Integration | 401 Unauthorized returned |
| 6.3.4 | Portal dashboard shows engagement severity summary | E2E | Cards display critical/high/medium/low counts |
| 6.3.5 | Portal user cannot access internal tester views | E2E | Navigation to /dashboard returns 403 |
| 6.3.6 | Portal user can download delivered reports | E2E | PDF download works for delivered reports only |
| 6.3.7 | Portal user cannot see draft or in-review reports | E2E | Only delivered reports visible in portal |

---

## Phase 7: Remediation Tracking & Retesting

**Duration:** 2 weeks
**Depends on:** Phase 6

### Definition of Done
- Remediation ticket CRUD linked to findings with external ticket reference JSONB (Jira, ServiceNow)
- Remediation lifecycle: open -> assigned -> in_progress -> pending_retest -> retesting -> verified_fixed / verified_not_fixed
- Retest results stored in retests JSONB array on remediation tickets
- Remediation dashboard with SLA tracking (days open, overdue count)
- Finding status automatically updates when remediation is verified

### Task 7.1: Remediation Ticket Management

**What:** Create the remediation_tickets table with external ticket JSONB and retests JSONB array. Implement CRUD API with lifecycle transitions. Auto-update finding status when remediation is verified.

**Design:**

```typescript
// packages/shared/src/types/remediation.ts

export interface ExternalTicket {
  system: 'jira' | 'servicenow' | 'github' | 'manual';
  ticketId: string;
  url: string;
  assignee?: string;
  priority?: string;
  lastSyncedAt?: string;
}

export interface RetestResult {
  id: string;
  testedBy: string;
  testedAt: string;
  result: 'fixed' | 'partially_fixed' | 'not_fixed' | 'regressed';
  evidence: string;
  notes?: string;
}

export type RemediationStatus =
  | 'open' | 'assigned' | 'in_progress' | 'pending_retest'
  | 'retesting' | 'verified_fixed' | 'verified_not_fixed'
  | 'risk_accepted' | 'deferred';
```

```typescript
// apps/api/src/services/remediation.ts

export class RemediationService {
  async recordRetest(
    ticketId: string,
    retest: Omit<RetestResult, 'id'>,
    userId: string
  ): Promise<RemediationTicket> {
    const ticket = await this.prisma.remediationTicket.findUniqueOrThrow({
      where: { id: ticketId },
    });

    const newRetest: RetestResult = {
      id: crypto.randomUUID(),
      ...retest,
    };

    const retests = [...(ticket.retests as RetestResult[]), newRetest];
    const newStatus = retest.result === 'fixed' ? 'verified_fixed' : 'verified_not_fixed';

    const updated = await this.prisma.remediationTicket.update({
      where: { id: ticketId },
      data: {
        retests,
        status: newStatus,
        ...(retest.result === 'fixed' ? { resolvedAt: new Date() } : {}),
      },
    });

    // Auto-update finding status
    if (retest.result === 'fixed') {
      await this.prisma.finding.update({
        where: { id: ticket.findingId },
        data: { status: 'verified_fixed' },
      });
    }

    await this.audit.log({
      organisationId: ticket.organisationId,
      userId,
      action: 'update',
      entityType: 'remediation_ticket',
      entityId: ticketId,
      changes: { retestAdded: { old: null, new: newRetest } },
    });

    return updated;
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 7.1.1 | Create remediation ticket for finding | Integration | Ticket created with status='open', linked to finding |
| 7.1.2 | Set external ticket reference (Jira) | Integration | external_ticket JSONB updated with Jira URL and ticket ID |
| 7.1.3 | Record retest result as 'fixed' | Integration | Retest appended to retests array, status='verified_fixed', finding status updated |
| 7.1.4 | Record retest result as 'not_fixed' | Integration | Retest appended, status='verified_not_fixed', finding status unchanged |
| 7.1.5 | Remediation dashboard shows days open | Integration | days_open calculated correctly from created_at |
| 7.1.6 | Remediation dashboard highlights overdue tickets | E2E | Tickets past due_date shown with overdue indicator |
| 7.1.7 | Retest evidence supports text and screenshot references | Integration | Evidence field stored in retests JSONB |

### Task 7.2: Remediation Dashboard UI

**What:** Build a remediation tracking dashboard showing all open remediation tickets across engagements with SLA indicators, severity grouping, and filterable by status, assignee, and due date. Include a timeline visualization of retest history per finding.

**Design:**

```typescript
// apps/web/src/app/(dashboard)/remediation/page.tsx

export default async function RemediationDashboard() {
  const tickets = await fetchRemediationTickets();
  const stats = computeRemediationStats(tickets);

  return (
    <div className="space-y-6">
      <RemediationStatsCards stats={stats} />
      <RemediationFilters />
      <RemediationTable tickets={tickets} />
    </div>
  );
}

interface RemediationStats {
  totalOpen: number;
  overdue: number;
  avgDaysToRemediate: number;
  verifiedFixedThisMonth: number;
  bySeverity: Record<SeverityLevel, number>;
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 7.2.1 | Dashboard displays remediation stats cards | E2E | Total open, overdue, avg days to remediate shown |
| 7.2.2 | Table filters by remediation status | E2E | Only tickets matching selected status displayed |
| 7.2.3 | Retest timeline shows chronological history | E2E | Each retest rendered with result, date, tester, evidence |
| 7.2.4 | Empty state shown when no remediation tickets exist | E2E | "No remediation tickets" message displayed |

---

## Phase 8: Scanner Integration & Import Pipeline

**Duration:** 3 weeks
**Depends on:** Phase 5

### Definition of Done
- Scanner output parsers for Burp Suite XML, Nessus .nessus XML, Qualys XML, and Nuclei JSON
- Import pipeline that normalizes scanner findings to the platform's finding schema
- Raw scanner data preserved in source_data JSONB field for future re-processing
- Deduplication logic: match imported findings against existing engagement findings by CWE + affected asset + URL path
- Import summary showing new findings, duplicates skipped, and merge suggestions
- Bulk import API endpoint accepting multipart file upload

### Task 8.1: Scanner Output Parsers

**What:** Implement scanner-specific parsers that convert Burp Suite XML, Nessus .nessus XML, Qualys CSV/XML, and Nuclei JSON outputs into the platform's normalized finding format. Preserve raw scanner data in source_data JSONB.

**Design:**

```typescript
// apps/api/src/parsers/types.ts

export interface ParsedFinding {
  title: string;
  severity: SeverityLevel;
  cweId?: number;
  cveId?: string;
  details: FindingDetails;
  affectedAssets: FindingAffectedAsset[];
  cvss?: Partial<CvssV4Metrics>;
  sourceData: Record<string, unknown>;  // raw scanner output preserved
  sourceReferenceId?: string;           // scanner's internal finding ID
}

export interface ScannerParser {
  name: string;
  supportedFormats: string[];
  parse(input: Buffer, options?: ParseOptions): Promise<ParsedFinding[]>;
}
```

```typescript
// apps/api/src/parsers/burp/parser.ts

import { XMLParser } from 'fast-xml-parser';

export class BurpParser implements ScannerParser {
  name = 'burp';
  supportedFormats = ['application/xml', 'text/xml'];

  async parse(input: Buffer): Promise<ParsedFinding[]> {
    const parser = new XMLParser({ ignoreAttributes: false });
    const xml = parser.parse(input.toString());
    const issues = xml.issues?.issue ?? [];

    return (Array.isArray(issues) ? issues : [issues]).map((issue) => ({
      title: issue.name,
      severity: this.mapSeverity(issue.severity),
      details: {
        description: issue.issueDetail ?? '',
        remediation: {
          recommendation: issue.remediationDetail ?? '',
        },
        affectedComponent: issue.path,
      },
      affectedAssets: [{
        assetId: '',  // resolved during import
        assetName: `${issue.host?.['@_ip'] ?? ''}:${issue.port ?? ''}`,
        urlPath: issue.path,
        parameter: issue.parameter ?? undefined,
      }],
      sourceData: issue,
      sourceReferenceId: issue.serialNumber,
    }));
  }

  private mapSeverity(burpSeverity: string): SeverityLevel {
    const map: Record<string, SeverityLevel> = {
      High: 'high', Medium: 'medium', Low: 'low', Information: 'informational',
    };
    return map[burpSeverity] ?? 'informational';
  }
}
```

```typescript
// apps/api/src/parsers/nessus/parser.ts

export class NessusParser implements ScannerParser {
  name = 'nessus';
  supportedFormats = ['application/xml'];

  async parse(input: Buffer): Promise<ParsedFinding[]> {
    const parser = new XMLParser({ ignoreAttributes: false });
    const xml = parser.parse(input.toString());
    const hosts = xml.NessusClientData_v2?.Report?.ReportHost ?? [];

    const findings: ParsedFinding[] = [];
    for (const host of Array.isArray(hosts) ? hosts : [hosts]) {
      const items = host.ReportItem ?? [];
      for (const item of Array.isArray(items) ? items : [items]) {
        if (item['@_severity'] === '0') continue; // skip informational
        findings.push({
          title: item['@_pluginName'],
          severity: this.mapSeverity(item['@_severity']),
          cweId: item.cwe ? parseInt(item.cwe) : undefined,
          cveId: item.cve ?? undefined,
          details: {
            description: item.description ?? '',
            remediation: { recommendation: item.solution ?? '' },
          },
          affectedAssets: [{
            assetId: '',
            assetName: host['@_name'],
            urlPath: `${item['@_protocol']}/${item['@_port']}`,
          }],
          cvss: item.cvss3_vector ? this.parseCvssVector(item.cvss3_vector) : undefined,
          sourceData: item,
          sourceReferenceId: item['@_pluginID'],
        });
      }
    }
    return findings;
  }

  private mapSeverity(nessusLevel: string): SeverityLevel {
    const map: Record<string, SeverityLevel> = {
      '4': 'critical', '3': 'high', '2': 'medium', '1': 'low',
    };
    return map[nessusLevel] ?? 'informational';
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 8.1.1 | Parse Burp Suite XML with multiple issues | Unit | Correct number of ParsedFindings produced with mapped severities |
| 8.1.2 | Parse Nessus .nessus file with multi-host scan | Unit | Findings grouped by host with correct CWE/CVE extraction |
| 8.1.3 | Parse Qualys XML report | Unit | Findings produced with severity mapping |
| 8.1.4 | Parse Nuclei JSON output | Unit | Findings with template IDs preserved in sourceData |
| 8.1.5 | Burp severity mapping (High->high, Information->informational) | Unit | All Burp severity levels mapped correctly |
| 8.1.6 | Raw scanner data preserved in sourceData | Unit | Complete original scanner record stored without modification |
| 8.1.7 | Empty or malformed scanner file produces clear error | Unit | Descriptive error message, not stack trace |

### Task 8.2: Import Pipeline & Deduplication

**What:** Implement the import pipeline that processes parsed scanner findings, resolves asset references (matching by hostname/IP to existing assets), deduplicates against existing engagement findings, and creates new finding records. Provide an import summary showing new, duplicate, and suggested-merge findings.

**Design:**

```typescript
// apps/api/src/services/import-pipeline.ts

export interface ImportResult {
  totalParsed: number;
  created: number;
  duplicatesSkipped: number;
  mergeSuggestions: Array<{
    imported: ParsedFinding;
    existingFindingId: string;
    similarity: number;
  }>;
  errors: Array<{ index: number; error: string }>;
}

export class ImportPipeline {
  async importFindings(
    engagementId: string,
    parsedFindings: ParsedFinding[],
    userId: string
  ): Promise<ImportResult> {
    const engagement = await this.prisma.engagement.findUniqueOrThrow({
      where: { id: engagementId },
    });

    const existingFindings = await this.prisma.finding.findMany({
      where: { engagementId },
      select: { id: true, title: true, cweId: true, affectedAssets: true },
    });

    const result: ImportResult = {
      totalParsed: parsedFindings.length,
      created: 0,
      duplicatesSkipped: 0,
      mergeSuggestions: [],
      errors: [],
    };

    for (const [index, parsed] of parsedFindings.entries()) {
      const duplicate = this.findDuplicate(parsed, existingFindings);
      if (duplicate) {
        result.duplicatesSkipped++;
        continue;
      }

      // Resolve asset references
      const resolvedAssets = await this.resolveAssets(
        parsed.affectedAssets,
        engagement.clientId
      );

      await this.prisma.finding.create({
        data: {
          engagementId,
          organisationId: engagement.organisationId,
          title: parsed.title,
          severity: parsed.severity,
          status: 'draft',
          cweId: parsed.cweId,
          cveId: parsed.cveId,
          source: 'scanner',
          discoveredBy: userId,
          details: parsed.details,
          affectedAssets: resolvedAssets,
          sourceData: parsed.sourceData,
          cvss: parsed.cvss ? computeCvssV4(parsed.cvss as CvssV4Metrics) : null,
        },
      });
      result.created++;
    }

    return result;
  }

  private findDuplicate(
    parsed: ParsedFinding,
    existing: Array<{ id: string; title: string; cweId: number | null; affectedAssets: unknown }>
  ): string | null {
    return existing.find((e) =>
      e.cweId === parsed.cweId &&
      this.assetsOverlap(e.affectedAssets, parsed.affectedAssets)
    )?.id ?? null;
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 8.2.1 | Import Burp XML creates findings for engagement | Integration | Findings created with source='scanner', import summary returned |
| 8.2.2 | Duplicate finding (same CWE + asset) skipped | Integration | duplicatesSkipped incremented, no duplicate created |
| 8.2.3 | Asset resolution matches by hostname | Integration | Imported finding linked to existing asset by hostname match |
| 8.2.4 | Asset resolution matches by IP address | Integration | Imported finding linked to existing asset by IP match |
| 8.2.5 | Import of 500+ findings completes within 30 seconds | Performance | Pipeline handles large scanner outputs efficiently |
| 8.2.6 | Import API accepts multipart file upload | Integration | File uploaded, parser auto-detected, import executed |
| 8.2.7 | Import summary includes merge suggestions for similar findings | Integration | Partial matches returned with similarity score |

---

## Phase 9: AI-Assisted Workflows

**Duration:** 4 weeks
**Depends on:** Phase 6, Phase 8

### Definition of Done
- AI-generated finding narratives from raw details/evidence with configurable tone (technical, executive)
- AI-generated executive summaries for engagement reports
- AI-assisted CVSS scoring suggestions given finding description
- AI-powered finding classification suggesting CWE, ATT&CK technique, and severity
- Cross-engagement finding deduplication using embedding-based similarity
- AI scoping assistant that suggests test cases from asset inventory

### Task 9.1: Finding Narrative Generation

**What:** Implement an AI service that generates professional finding descriptions and remediation recommendations from raw finding details, evidence, and PoC data. Support multiple tones: technical (for pentest report body) and executive (for executive summary). Use Claude via LangChain.js with structured output.

**Design:**

```typescript
// apps/api/src/ai/tools/finding-narrative.ts

import { ChatAnthropic } from '@langchain/anthropic';
import { z } from 'zod';

const NarrativeOutputSchema = z.object({
  description: z.string(),
  impact: z.string(),
  remediation: z.string(),
  executiveSummary: z.string(),
});

export class FindingNarrativeGenerator {
  private model: ChatAnthropic;

  constructor() {
    this.model = new ChatAnthropic({
      model: 'claude-sonnet-4-20250514',
      temperature: 0.3,
    });
  }

  async generateNarrative(
    finding: FindingResponse,
    tone: 'technical' | 'executive'
  ): Promise<z.infer<typeof NarrativeOutputSchema>> {
    const prompt = `You are a senior penetration testing consultant writing a ${tone} finding description for a client-facing pentest report.

Given the following raw finding data, produce a professional narrative:

Title: ${finding.title}
Severity: ${finding.severity}
CWE: ${finding.cweId ?? 'Not classified'}
Affected Component: ${finding.details.affectedComponent ?? 'N/A'}
Proof of Concept: ${finding.details.proofOfConcept ?? 'N/A'}
Steps to Reproduce:
${finding.details.stepsToReproduce?.join('\n') ?? 'N/A'}

Produce:
1. A clear description of the vulnerability
2. A business impact statement
3. A specific remediation recommendation with code examples where applicable
4. A one-paragraph executive summary suitable for C-level stakeholders`;

    const result = await this.model.invoke(prompt);
    return NarrativeOutputSchema.parse(JSON.parse(result.content as string));
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 9.1.1 | Generate technical narrative for SQL injection finding | Integration | Professional description with technical detail, remediation includes code example |
| 9.1.2 | Generate executive narrative for same finding | Integration | Business-focused language, no code snippets, impact quantified |
| 9.1.3 | Generate narrative for finding with minimal details | Integration | Meaningful output produced even with sparse input |
| 9.1.4 | Generated narrative does not hallucinate CVE IDs | Integration | Only CVE IDs present in source data appear in output |
| 9.1.5 | Narrative generation completes within 15 seconds | Performance | API response time acceptable for UX |

### Task 9.2: AI-Assisted Classification & CVSS Suggestion

**What:** Implement AI-assisted classification that suggests CWE ID, ATT&CK technique mappings, severity level, and CVSS base metric values given a finding's title, description, and affected component. Present suggestions with confidence scores for human review.

**Design:**

```typescript
// apps/api/src/ai/tools/finding-classifier.ts

const ClassificationOutputSchema = z.object({
  suggestedCweId: z.number().nullable(),
  suggestedCweName: z.string().nullable(),
  suggestedSeverity: z.enum(['critical', 'high', 'medium', 'low', 'informational']),
  suggestedAttackTechniques: z.array(z.object({
    techniqueId: z.string(),
    techniqueName: z.string(),
    confidence: z.enum(['high', 'medium', 'low']),
  })),
  suggestedCvssMetrics: z.object({
    attackVector: z.enum(['NETWORK', 'ADJACENT', 'LOCAL', 'PHYSICAL']),
    attackComplexity: z.enum(['LOW', 'HIGH']),
    privilegesRequired: z.enum(['NONE', 'LOW', 'HIGH']),
    userInteraction: z.enum(['NONE', 'PASSIVE', 'ACTIVE']),
  }),
  reasoning: z.string(),
});

export class FindingClassifier {
  async classify(
    title: string,
    description: string,
    affectedComponent?: string
  ): Promise<z.infer<typeof ClassificationOutputSchema>> {
    // Load CWE and ATT&CK reference data for context
    const cweContext = await this.loadCweContext();
    const attackContext = await this.loadAttackContext();

    const prompt = `You are a security vulnerability classification expert.
Given this finding, suggest the most appropriate CWE ID, MITRE ATT&CK techniques,
severity level, and CVSS 4.0 base metrics.

Title: ${title}
Description: ${description}
Affected Component: ${affectedComponent ?? 'Unknown'}

Available CWE categories: ${cweContext}
Available ATT&CK techniques: ${attackContext}

Provide your classification with reasoning.`;

    const result = await this.model.invoke(prompt);
    return ClassificationOutputSchema.parse(JSON.parse(result.content as string));
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 9.2.1 | Classify SQL injection finding | Integration | CWE-89 suggested, T1190 technique mapped, severity high/critical |
| 9.2.2 | Classify XSS finding | Integration | CWE-79 suggested, appropriate severity |
| 9.2.3 | Classify authentication bypass | Integration | CWE-287 or CWE-306 suggested |
| 9.2.4 | Classification includes reasoning | Integration | Reasoning field explains why CWE and severity were chosen |
| 9.2.5 | CVSS metric suggestions are valid enum values | Unit | All suggested values are valid CVSS 4.0 metric values |

### Task 9.3: Cross-Engagement Finding Deduplication

**What:** Implement embedding-based finding similarity computation for cross-engagement deduplication. Generate embeddings for finding titles and descriptions, compute cosine similarity, and surface potential duplicates to analysts with similarity scores. Store results for the findings library enrichment pipeline.

**Design:**

```typescript
// apps/api/src/ai/deduplication.ts

export class FindingDeduplicator {
  private embeddingModel: Embeddings;

  async computeSimilarity(
    organisationId: string,
    findingId: string,
    threshold: number = 0.85
  ): Promise<Array<{ findingId: string; title: string; similarity: number; engagementTitle: string }>> {
    const finding = await this.prisma.finding.findUniqueOrThrow({
      where: { id: findingId },
    });

    const findingText = `${finding.title}\n${(finding.details as FindingDetails).description}`;
    const targetEmbedding = await this.embeddingModel.embedQuery(findingText);

    // Get all other findings in the organisation (excluding current engagement)
    const candidates = await this.prisma.finding.findMany({
      where: {
        organisationId,
        engagementId: { not: finding.engagementId },
      },
      select: { id: true, title: true, details: true, engagementId: true },
    });

    const similarities: Array<{ findingId: string; title: string; similarity: number; engagementTitle: string }> = [];

    for (const candidate of candidates) {
      const candidateText = `${candidate.title}\n${(candidate.details as FindingDetails).description}`;
      const candidateEmbedding = await this.embeddingModel.embedQuery(candidateText);
      const similarity = cosineSimilarity(targetEmbedding, candidateEmbedding);

      if (similarity >= threshold) {
        similarities.push({
          findingId: candidate.id,
          title: candidate.title,
          similarity: Math.round(similarity * 100) / 100,
          engagementTitle: '', // resolved separately
        });
      }
    }

    return similarities.sort((a, b) => b.similarity - a.similarity);
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 9.3.1 | Identical findings across engagements detected | Integration | Similarity score > 0.95 |
| 9.3.2 | Similar but distinct findings scored correctly | Integration | Similarity between 0.7-0.9 |
| 9.3.3 | Unrelated findings have low similarity | Integration | Similarity < 0.3 |
| 9.3.4 | Deduplication API returns results within 10 seconds for 1000 findings | Performance | Response time acceptable |
| 9.3.5 | Similarity results include engagement context | Integration | Each match includes source engagement title |

### Task 9.4: Executive Summary & Report AI Generation

**What:** Build an AI service that generates complete executive summaries for engagement reports, synthesizing all findings into a narrative suitable for C-level stakeholders. Integrate with the report generation pipeline so generated summaries can be reviewed, edited, and included in final reports.

**Design:**

```typescript
// apps/api/src/ai/tools/executive-summary.ts

export class ExecutiveSummaryGenerator {
  async generate(engagementId: string): Promise<string> {
    const engagement = await this.loadEngagementWithFindings(engagementId);
    const stats = this.computeStatistics(engagement.findings);

    const prompt = `You are a senior penetration testing consultant writing an executive summary for a ${engagement.engagementType} engagement.

Client: ${engagement.client.name}
Engagement: ${engagement.title}
Period: ${engagement.startDate} to ${engagement.endDate}
Methodology: ${engagement.methodology}

Findings Summary:
- Critical: ${stats.critical} | High: ${stats.high} | Medium: ${stats.medium} | Low: ${stats.low}
- Total: ${stats.total}

Top findings by severity:
${engagement.findings
  .filter((f) => ['critical', 'high'].includes(f.severity))
  .map((f) => `- [${f.severity.toUpperCase()}] ${f.title}: ${(f.details as FindingDetails).description?.slice(0, 200)}`)
  .join('\n')}

Write a 3-5 paragraph executive summary covering:
1. Scope and approach
2. Key findings and their business impact
3. Overall security posture assessment
4. Priority remediation recommendations`;

    const result = await this.model.invoke(prompt);
    return result.content as string;
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 9.4.1 | Generate executive summary for engagement with 15 findings | Integration | 3-5 paragraph summary covering scope, findings, posture, recommendations |
| 9.4.2 | Summary mentions critical findings by name | Integration | Critical finding titles appear in summary |
| 9.4.3 | Summary does not include technical exploitation details | Integration | No PoC code, no raw request/response data |
| 9.4.4 | Generated summary integrates into report workflow | Integration | Summary stored on report content JSONB, editable before approval |

---

## Phase 10: Attack Path Visualization & Cross-Engagement Analytics

**Duration:** 3 weeks
**Depends on:** Phase 9

### Definition of Done
- Graph layer (graph_nodes, graph_edges) introduced from Data Model Suggestion 4
- Attack path visualization showing exploit chains from initial access to objective
- Cross-engagement analytics dashboard with trend lines for finding severity over time
- Mean-time-to-remediate calculation per client, severity, and CWE category
- ATT&CK coverage heat map showing tested tactics/techniques per engagement
- Blast radius analysis for compromised assets

### Task 10.1: Graph Layer Introduction

**What:** Create the graph_nodes and graph_edges tables. Build sync logic that mirrors relational findings, assets, and ATT&CK mappings into the graph layer. Implement recursive CTE queries for attack path traversal.

**Design:**

```sql
-- packages/db/migrations/010_graph_layer.sql

CREATE EXTENSION IF NOT EXISTS ltree;

CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
    entity_id       UUID,
    label           VARCHAR(500) NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    hierarchy_path  LTREE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    weight          DECIMAL(5,4) DEFAULT 1.0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_nodes_org ON graph_nodes(organisation_id);
CREATE INDEX idx_graph_nodes_type ON graph_nodes(node_type);
CREATE INDEX idx_graph_nodes_entity ON graph_nodes(entity_id);
CREATE INDEX idx_graph_nodes_properties ON graph_nodes USING GIN (properties);
CREATE INDEX idx_graph_nodes_hierarchy ON graph_nodes USING GIST (hierarchy_path);

CREATE INDEX idx_graph_edges_source ON graph_edges(source_node_id);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_node_id);
CREATE INDEX idx_graph_edges_type ON graph_edges(edge_type);
```

```typescript
// apps/api/src/services/graph-sync.ts

export class GraphSyncService {
  async syncFindingToGraph(finding: FindingResponse): Promise<void> {
    // Create or update finding node
    const findingNode = await this.upsertNode({
      organisationId: finding.organisationId,
      nodeType: 'finding',
      entityId: finding.id,
      label: finding.title,
      properties: {
        severity: finding.severity,
        cvssScore: finding.cvss?.baseScore,
        cweId: finding.cweId,
        status: finding.status,
      },
    });

    // Create edges to affected assets
    for (const asset of finding.affectedAssets as FindingAffectedAsset[]) {
      const assetNode = await this.findNodeByEntity('asset', asset.assetId);
      if (assetNode) {
        await this.upsertEdge({
          organisationId: finding.organisationId,
          sourceNodeId: findingNode.id,
          targetNodeId: assetNode.id,
          edgeType: 'affects',
          properties: { urlPath: asset.urlPath, parameter: asset.parameter },
        });
      }
    }

    // Create edges to ATT&CK techniques
    for (const mapping of finding.attackMappings as AttackMapping[]) {
      const techNode = await this.findNodeByEntity('attack_technique', mapping.techniqueId);
      if (techNode) {
        await this.upsertEdge({
          organisationId: finding.organisationId,
          sourceNodeId: findingNode.id,
          targetNodeId: techNode.id,
          edgeType: 'uses_technique',
          properties: { confidence: mapping.confidence },
        });
      }
    }
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 10.1.1 | Sync finding to graph creates node and edges | Integration | Finding node, asset edges, technique edges created |
| 10.1.2 | Recursive CTE traverses attack path 3 hops deep | Integration | All assets reachable within 3 hops returned |
| 10.1.3 | Graph query respects organisation isolation | pgTAP | Org B nodes invisible to Org A queries |
| 10.1.4 | ltree query finds all assets in DMZ zone | Integration | hierarchy_path `<@` operator returns correct assets |

### Task 10.2: Cross-Engagement Analytics Dashboard

**What:** Build analytics dashboard with finding severity trends over time, mean-time-to-remediate by client/severity/CWE, ATT&CK coverage heat map per engagement, and top recurring findings across engagements.

**Design:**

```typescript
// apps/api/src/routes/analytics/index.ts

export interface AnalyticsTimeSeriesPoint {
  date: string;
  critical: number;
  high: number;
  medium: number;
  low: number;
  informational: number;
}

export interface RemediationMetrics {
  avgDaysToRemediate: number;
  medianDaysToRemediate: number;
  bySeverity: Record<SeverityLevel, { avg: number; median: number; count: number }>;
  byClient: Array<{ clientId: string; clientName: string; avgDays: number }>;
}

export interface AttackCoverageCell {
  tacticName: string;
  techniqueId: string;
  techniqueName: string;
  findingCount: number;
  maxSeverity: SeverityLevel;
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 10.2.1 | Finding severity trend chart renders with monthly data | E2E | Line chart shows severity counts over time |
| 10.2.2 | Mean-time-to-remediate calculated correctly | Integration | Avg days matches manual calculation from test data |
| 10.2.3 | ATT&CK heat map shows coverage for engagement | E2E | Tactic columns, technique rows, color-coded by finding count |
| 10.2.4 | Top recurring findings identifies cross-engagement patterns | Integration | Findings appearing in 3+ engagements listed with frequency |
| 10.2.5 | Analytics respects client-level access control | Integration | Restricted user only sees analytics for assigned clients |

### Task 10.3: Attack Path Visualization

**What:** Build an interactive attack path visualization using a graph rendering library (e.g., React Flow or D3.js force-directed graph). Display nodes for assets and findings, edges for lateral movement and exploitation chains, with color coding by severity and asset criticality.

**Design:**

```typescript
// apps/web/src/components/analytics/AttackPathGraph.tsx

import ReactFlow, { Node, Edge, Background, Controls } from 'reactflow';

interface AttackPathGraphProps {
  nodes: GraphNodeResponse[];
  edges: GraphEdgeResponse[];
  onNodeClick: (nodeId: string) => void;
}

export function AttackPathGraph({ nodes, edges, onNodeClick }: AttackPathGraphProps) {
  const flowNodes: Node[] = nodes.map((n) => ({
    id: n.id,
    type: n.nodeType === 'asset' ? 'assetNode' : 'findingNode',
    data: {
      label: n.label,
      severity: n.properties.severity,
      criticality: n.properties.criticality,
    },
    position: { x: 0, y: 0 },  // auto-layout applied
  }));

  const flowEdges: Edge[] = edges.map((e) => ({
    id: e.id,
    source: e.sourceNodeId,
    target: e.targetNodeId,
    label: e.edgeType.replace('_', ' '),
    animated: e.edgeType === 'pivots_to',
    style: { stroke: edgeColor(e.edgeType) },
  }));

  return (
    <ReactFlow
      nodes={flowNodes}
      edges={flowEdges}
      nodeTypes={{ assetNode: AssetNode, findingNode: FindingNode }}
      onNodeClick={(_, node) => onNodeClick(node.id)}
      fitView
    >
      <Background />
      <Controls />
    </ReactFlow>
  );
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 10.3.1 | Attack path graph renders nodes and edges | E2E | All nodes visible with correct labels and colors |
| 10.3.2 | Clicking asset node shows asset details panel | E2E | Side panel displays asset properties and linked findings |
| 10.3.3 | Lateral movement edges animated | E2E | pivots_to edges show animation indicating direction |
| 10.3.4 | Graph handles 100+ nodes without performance degradation | Performance | Rendering completes within 2 seconds |

---

## Phase 11: Integrations (Jira, ServiceNow, Slack, CI/CD)

**Duration:** 3 weeks
**Depends on:** Phase 7

### Definition of Done
- Bidirectional Jira integration: create tickets from findings, sync status updates back
- ServiceNow integration: create incidents from findings, sync resolution status
- Slack integration: notifications for finding creation, engagement status changes, remediation updates
- Webhook support: configurable outgoing webhooks for finding lifecycle events
- API-first architecture with published OpenAPI 3.1 specification
- Integration configuration stored in organisation settings JSONB

### Task 11.1: Jira Integration

**What:** Implement bidirectional Jira integration using the Jira REST API. Auto-create Jira issues from findings with severity-to-priority mapping, description templating, and CWE/CVE labels. Poll for status changes and sync back to remediation tickets.

**Design:**

```typescript
// apps/api/src/services/integrations/jira.ts

export interface JiraConfig {
  baseUrl: string;
  projectKey: string;
  issueType: string;
  apiToken: string;  // encrypted at rest
  email: string;
  severityToPriority: Record<SeverityLevel, string>;
  statusMapping: Record<string, RemediationStatus>;
}

export class JiraIntegration {
  async createTicketFromFinding(
    finding: FindingResponse,
    config: JiraConfig
  ): Promise<{ ticketId: string; url: string }> {
    const description = this.formatFindingForJira(finding);
    const priority = config.severityToPriority[finding.severity];

    const response = await fetch(`${config.baseUrl}/rest/api/3/issue`, {
      method: 'POST',
      headers: {
        'Authorization': `Basic ${btoa(`${config.email}:${config.apiToken}`)}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        fields: {
          project: { key: config.projectKey },
          summary: `[${finding.severity.toUpperCase()}] ${finding.title}`,
          description: { type: 'doc', version: 1, content: description },
          issuetype: { name: config.issueType },
          priority: { name: priority },
          labels: [
            finding.cweId ? `CWE-${finding.cweId}` : null,
            finding.cveId ?? null,
          ].filter(Boolean),
        },
      }),
    });

    const data = await response.json();
    return {
      ticketId: data.key,
      url: `${config.baseUrl}/browse/${data.key}`,
    };
  }

  async syncStatus(config: JiraConfig, tickets: RemediationTicket[]): Promise<void> {
    for (const ticket of tickets) {
      if (!ticket.externalTicket?.ticketId) continue;
      const jiraIssue = await this.fetchIssue(config, ticket.externalTicket.ticketId);
      const mappedStatus = config.statusMapping[jiraIssue.fields.status.name];
      if (mappedStatus && mappedStatus !== ticket.status) {
        await this.remediationService.updateStatus(ticket.id, mappedStatus);
      }
    }
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 11.1.1 | Create Jira ticket from critical finding | Integration | Jira issue created with correct priority, title, description |
| 11.1.2 | Jira ticket includes CWE and CVE labels | Integration | Labels array contains CWE-89 and CVE-2024-XXXX |
| 11.1.3 | Status sync updates remediation ticket | Integration | Jira "Done" maps to remediation "pending_retest" |
| 11.1.4 | Invalid Jira credentials return clear error | Integration | 401 error surfaced with actionable message |
| 11.1.5 | Jira config stored encrypted in organisation settings | Integration | API token encrypted at rest in settings JSONB |

### Task 11.2: Slack Notifications

**What:** Implement Slack webhook integration for real-time notifications on finding creation, engagement status changes, remediation updates, and report delivery. Support per-user notification preferences stored in user profile JSONB.

**Design:**

```typescript
// apps/api/src/services/integrations/slack.ts

export class SlackNotifier {
  async notifyFindingCreated(finding: FindingResponse): Promise<void> {
    const color = severityToColor(finding.severity);
    await this.sendWebhook({
      attachments: [{
        color,
        title: `New ${finding.severity.toUpperCase()} Finding`,
        text: finding.title,
        fields: [
          { title: 'Engagement', value: finding.engagementTitle, short: true },
          { title: 'CWE', value: finding.cweId ? `CWE-${finding.cweId}` : 'N/A', short: true },
          { title: 'Severity', value: finding.severity, short: true },
          { title: 'Discovered By', value: finding.discoveredByName, short: true },
        ],
        actions: [{
          type: 'button',
          text: 'View Finding',
          url: `${this.baseUrl}/findings/${finding.id}`,
        }],
      }],
    });
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 11.2.1 | Finding creation triggers Slack notification | Integration | Webhook sent with finding details and severity color |
| 11.2.2 | Engagement status change triggers notification | Integration | Status transition message sent to configured channel |
| 11.2.3 | User with notifications disabled receives no alerts | Integration | Webhook not sent for users who opted out |
| 11.2.4 | Invalid webhook URL logged as error, not crash | Integration | Error logged, application continues normally |

### Task 11.3: Webhook & API Documentation

**What:** Implement configurable outgoing webhooks for finding lifecycle events. Publish OpenAPI 3.1 specification generated from Fastify route schemas. Support webhook signature verification for security.

**Design:**

```typescript
// apps/api/src/services/webhooks.ts

export interface WebhookConfig {
  url: string;
  events: string[];  // ['finding.created', 'finding.status_changed', 'engagement.completed']
  secret: string;
  isActive: boolean;
}

export class WebhookService {
  async dispatch(event: string, payload: Record<string, unknown>): Promise<void> {
    const configs = await this.getActiveWebhooks(event);
    for (const config of configs) {
      const signature = this.computeSignature(config.secret, payload);
      await fetch(config.url, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-Webhook-Signature': signature,
          'X-Webhook-Event': event,
        },
        body: JSON.stringify(payload),
      });
    }
  }

  private computeSignature(secret: string, payload: Record<string, unknown>): string {
    return crypto
      .createHmac('sha256', secret)
      .update(JSON.stringify(payload))
      .digest('hex');
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 11.3.1 | Webhook dispatched on finding creation | Integration | POST sent to configured URL with finding payload |
| 11.3.2 | Webhook signature verification passes | Unit | HMAC-SHA256 signature matches recomputed value |
| 11.3.3 | OpenAPI spec generated from route schemas | Integration | Valid OpenAPI 3.1 JSON produced, parseable by Swagger UI |
| 11.3.4 | Webhook retry on 5xx response | Integration | Failed webhook retried up to 3 times with backoff |

---

## Phase 12: Continuous Testing, Compliance Automation & Enterprise Features

**Duration:** 4 weeks
**Depends on:** Phase 11

### Definition of Done
- Scheduled engagement templates for continuous testing programmes
- Compliance framework mapping (PCI DSS v4.0, SOC 2, HIPAA, GDPR, ISO 27001) with control-to-finding linkage
- Compliance posture dashboard showing assessed controls and gaps
- SAML 2.0 SSO support via NextAuth.js + identity provider federation
- SCIM 2.0 user provisioning API for enterprise user lifecycle management
- Multi-workspace organisation support with workspace-level isolation
- API rate limiting and usage metering

### Task 12.1: Continuous Testing Orchestration

**What:** Implement scheduled engagement templates that automatically create new engagements on a defined cadence (weekly, monthly, quarterly). Support recurring scope definitions that carry forward from the previous engagement. Track continuous testing programmes with trend analysis.

**Design:**

```typescript
// apps/api/src/services/continuous-testing.ts

export interface ContinuousTestingSchedule {
  id: string;
  clientId: string;
  templateEngagementId: string;
  cadence: 'weekly' | 'biweekly' | 'monthly' | 'quarterly';
  nextRunDate: Date;
  isActive: boolean;
  config: {
    carryForwardScope: boolean;
    autoAssignTesters: boolean;
    testerIds: string[];
    notifyOnCreation: boolean;
  };
}

export class ContinuousTestingService {
  async executeScheduledEngagements(): Promise<void> {
    const dueSchedules = await this.prisma.continuousTestingSchedule.findMany({
      where: {
        isActive: true,
        nextRunDate: { lte: new Date() },
      },
    });

    for (const schedule of dueSchedules) {
      const template = await this.prisma.engagement.findUniqueOrThrow({
        where: { id: schedule.templateEngagementId },
      });

      await this.prisma.engagement.create({
        data: {
          clientId: template.clientId,
          organisationId: template.organisationId,
          title: `${template.title} - ${format(new Date(), 'yyyy-MM-dd')}`,
          engagementType: template.engagementType,
          status: 'draft',
          methodology: template.methodology,
          scope: schedule.config.carryForwardScope ? template.scope : {},
          team: schedule.config.autoAssignTesters
            ? schedule.config.testerIds.map((id) => ({ userId: id, role: 'tester', assignedAt: new Date().toISOString() }))
            : [],
          createdBy: 'system',
        },
      });

      // Advance next run date
      await this.advanceSchedule(schedule);
    }
  }
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 12.1.1 | Scheduled engagement created on due date | Integration | New engagement created from template with correct title |
| 12.1.2 | Scope carried forward from previous engagement | Integration | New engagement scope matches template scope |
| 12.1.3 | Team auto-assigned from schedule config | Integration | Team JSONB populated with configured tester IDs |
| 12.1.4 | Next run date advanced by cadence interval | Integration | Monthly schedule advances by 30 days |
| 12.1.5 | Inactive schedule skipped | Integration | No engagement created for paused schedule |

### Task 12.2: Compliance Framework Mapping

**What:** Implement compliance framework reference data (PCI DSS v4.0, SOC 2, HIPAA, GDPR, ISO 27001) with control definitions. Enable linking findings to compliance controls. Build compliance posture dashboard showing assessment status per framework.

**Design:**

```typescript
// packages/shared/src/types/compliance.ts

export interface ComplianceFramework {
  id: string;        // e.g., 'pci_dss_v4'
  name: string;      // e.g., 'PCI DSS v4.0'
  version: string;
  controls: ComplianceControl[];
}

export interface ComplianceControl {
  id: string;           // e.g., 'pci_dss_v4_11.3.1'
  controlNumber: string; // e.g., '11.3.1'
  title: string;
  description: string;
  testingRequirement: string;
}

export interface ComplianceAssessment {
  engagementId: string;
  controlId: string;
  status: 'not_assessed' | 'pass' | 'fail' | 'partial' | 'not_applicable';
  linkedFindingIds: string[];
  assessorNotes: string;
  assessedAt: string;
}
```

```typescript
// apps/api/src/routes/compliance/index.ts

export default async function complianceRoutes(fastify: FastifyInstance) {
  fastify.get<{ Params: { engagementId: string } }>(
    '/engagements/:engagementId/compliance',
    {
      preHandler: [fastify.authenticate],
      handler: async (request) => {
        const engagement = await fastify.prisma.engagement.findUniqueOrThrow({
          where: { id: request.params.engagementId },
        });

        const compliance = engagement.compliance as Record<string, unknown>;
        const frameworks = compliance.frameworks as string[] ?? [];

        const assessments = await Promise.all(
          frameworks.map(async (fwId) => {
            const controls = await fastify.prisma.complianceControl.findMany({
              where: { frameworkId: fwId },
            });
            return {
              frameworkId: fwId,
              controls: controls.map((c) => ({
                ...c,
                assessment: (compliance.controlsAssessed as ComplianceAssessment[])
                  ?.find((a) => a.controlId === c.id),
              })),
            };
          })
        );

        return assessments;
      },
    }
  );
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 12.2.1 | Load PCI DSS v4.0 framework controls | Integration | All Requirement 11.3 controls loaded with testing requirements |
| 12.2.2 | Link finding to compliance control | Integration | Control assessment updated with finding ID, status='fail' |
| 12.2.3 | Compliance posture dashboard shows pass/fail/not_assessed | E2E | Color-coded control matrix displayed |
| 12.2.4 | Compliance report includes only relevant controls | Integration | Report filtered by framework, includes linked findings |
| 12.2.5 | Multiple frameworks assessed on single engagement | Integration | PCI DSS and SOC 2 assessments stored independently |

### Task 12.3: Enterprise Authentication (SAML, SCIM)

**What:** Add SAML 2.0 SSO support via NextAuth.js identity provider federation. Implement SCIM 2.0 user provisioning API endpoint for enterprise user lifecycle management (create, update, deactivate users from IdP).

**Design:**

```typescript
// apps/api/src/routes/scim/users.ts

export default async function scimUserRoutes(fastify: FastifyInstance) {
  fastify.post<{ Body: ScimUser }>('/scim/v2/Users', {
    preHandler: [fastify.authenticateScim],
    handler: async (request) => {
      const scimUser = request.body;
      const user = await fastify.prisma.user.create({
        data: {
          organisationId: request.scimOrg.id,
          email: scimUser.userName,
          displayName: `${scimUser.name.givenName} ${scimUser.name.familyName}`,
          role: 'tester',
          authProvider: 'saml',
          isActive: scimUser.active ?? true,
        },
      });
      return formatScimResponse(user);
    },
  });

  fastify.patch<{ Params: { id: string }; Body: ScimPatch }>(
    '/scim/v2/Users/:id',
    {
      preHandler: [fastify.authenticateScim],
      handler: async (request) => {
        const operations = request.body.Operations;
        for (const op of operations) {
          if (op.path === 'active' && op.op === 'replace') {
            await fastify.prisma.user.update({
              where: { id: request.params.id },
              data: { isActive: op.value },
            });
          }
        }
      },
    }
  );
}
```

**Testing:**

| # | Test Case | Type | Expected Result |
|---|-----------|------|-----------------|
| 12.3.1 | SAML SSO login creates user on first authentication | Integration | User created with auth_provider='saml' |
| 12.3.2 | SCIM provisioning creates user in correct organisation | Integration | User created via SCIM API, linked to SCIM-authenticated org |
| 12.3.3 | SCIM deactivation disables user | Integration | PATCH with active=false sets is_active=false |
| 12.3.4 | SCIM bearer token authentication | Integration | Valid SCIM token grants access; invalid token returns 401 |
| 12.3.5 | API rate limiting enforces 100 requests/minute | Integration | 429 Too Many Requests returned at limit |

---

## Definition of Done Checklist (Full Project)

- [ ] PostgreSQL 16+ database with hybrid relational + JSONB schema deployed and migrated
- [ ] RLS policies enforce tenant isolation for all organisation-scoped tables
- [ ] Authentication supports email/password, Google OIDC, SAML 2.0, and portal invite tokens
- [ ] RBAC enforces admin, tester, analyst, and client_viewer role permissions
- [ ] Client, engagement, finding, asset, evidence CRUD fully functional
- [ ] Engagement lifecycle with validated status transitions and audit logging
- [ ] Finding lifecycle with status transitions, CVSS 4.0 scoring, and ATT&CK mapping
- [ ] Reusable findings library with "apply from library" workflow
- [ ] PDF and HTML report generation with OWASP, PTES, and NIST framework templates
- [ ] Client portal with read-only engagement and findings access
- [ ] Remediation ticket tracking with Jira/ServiceNow external ticket linking
- [ ] Retest recording with automatic finding status update on verification
- [ ] Scanner import pipeline parsing Burp, Nessus, Qualys, and Nuclei outputs
- [ ] Import deduplication matching by CWE + asset + URL path
- [ ] AI-generated finding narratives and executive summaries via Claude
- [ ] AI-assisted finding classification (CWE, ATT&CK, severity, CVSS suggestions)
- [ ] Cross-engagement finding deduplication via embedding similarity
- [ ] Graph layer with attack path visualization and blast radius analysis
- [ ] Cross-engagement analytics dashboard with severity trends and remediation metrics
- [ ] ATT&CK coverage heat map per engagement
- [ ] Jira and ServiceNow bidirectional integration
- [ ] Slack notification integration
- [ ] Configurable outgoing webhooks with HMAC signature verification
- [ ] Published OpenAPI 3.1 specification
- [ ] Continuous testing scheduling with cadence-based engagement creation
- [ ] Compliance framework mapping for PCI DSS, SOC 2, HIPAA, GDPR, ISO 27001
- [ ] SCIM 2.0 user provisioning API
- [ ] Docker Compose self-hosted deployment mode
- [ ] CI/CD pipeline with type checking, linting, unit tests, integration tests, and E2E tests
- [ ] All audit log entries immutable with partitioned storage
- [ ] Performance: findings list loads in < 2 seconds for 10,000 findings
- [ ] Security: no stored plaintext credentials, all API tokens encrypted at rest
