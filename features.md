# Penetration Testing Management — Feature & Functionality Survey

> Candidate #154 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| PlexTrac | Commercial SaaS | Enterprise custom pricing | https://plextrac.com |
| Dradis | Open-source/Commercial | Free CE; Pro tier | https://www.dradis.com |
| Cobalt.io | Commercial SaaS (PtaaS) | Per-engagement custom | https://cobalt.io |
| Faraday | Open-source/Commercial | Free CE; Enterprise tier | https://www.faradaysec.com |
| AttackForge | Commercial SaaS | Subscription-based | https://www.attackforge.com |
| Pentera | Commercial SaaS | Enterprise custom | https://pentera.io |

## Feature Analysis by Solution

### PlexTrac

**Core features**
- Centralized findings aggregation from pentests, scanners, and manual input
- AI-assisted reporting with reusable narrative templates
- Engagement planning and asset management
- Automated integrations with Jira and ServiceNow for remediation tracking
- Client portal for engagement visibility and findings sharing
- Remediation tracking with retest validation
- MITRE ATT&CK framework mapping for structured methodology
- Analytics dashboards for progress monitoring

**Differentiating features**
- "Report as you test" workflow: capture findings in real-time during engagement
- Native ticket integration: auto-create remediation tickets with full findings context
- Remediation validation: track fixes and validate through automated retesting
- Framework integration: built-in MITRE ATT&CK procedures for methodology consistency

**UX patterns**
- Findings-centric: navigate findings, not files or test runs
- Workflow automation: reduce manual report writing through templates and integrations
- Client-facing portal: separate viewing experience for client vs. internal team
- Real-time collaboration: multiple testers contribute to single engagement

**Integration points**
- Jira and ServiceNow for remediation ticket creation and tracking
- Slack for team notifications
- Email for client alerts and findings delivery
- API for custom integrations with other security tools
- SSO integration (Okta, Entra ID)

**Known gaps**
- Enterprise pricing model; opaque for smaller firms
- Limited self-hosting options (SaaS-only)
- Requires significant upfront training for consultants

**Licence / IP notes**
- Proprietary commercial software; no licensing concerns

---

### Dradis

**Core features**
- Customizable reporting with templates for frameworks (OWASP, PTES, OSCP)
- Issue Library: reusable findings repository across engagements
- Rules Engine: automatic mapping of scanner outputs to standardized language
- QA workflows: inline review and approval processes before report delivery
- Integration with 47+ security tools (Nessus, Qualys, Burp, etc.)
- Risk scoring: CVSSv4, DREAD, MITRE ATT&CK calculators
- Real-time findings portal for client access
- Parallel workflows: field testers and report writers work simultaneously

**Differentiating features**
- Reusable Issue Library: institutional knowledge captured and leveraged across team
- Rules Engine: automatically normalize scanner findings without manual mapping
- Data sovereignty: self-hosted CE version with full control
- Junior-to-senior quality: leverage vetted Issue Library to improve report quality
- Collaborative workflows: distributed teams can work in parallel without bottlenecks

**UX patterns**
- Library-driven: build knowledge base incrementally
- Parallel workflows: separate concerns (field testing vs. report writing)
- Progressive enrichment: add context and scoring as reports flow through QA
- Framework-guided: methodology templates reduce ambiguity

**Integration points**
- 47+ scanner integrations (Nessus, Qualys, Burp, Fortify, etc.)
- Active Directory for user management
- Email for report distribution
- REST API for custom integrations
- Webhooks for downstream automation
- Jira and ServiceNow integration (via API)

**Known gaps**
- Steeper learning curve for non-technical managers
- Smaller cloud ecosystem vs. PlexTrac
- Limited AI/ML enrichment features
- Manual effort required for engagement planning and scoping

**Licence / IP notes**
- Community Edition: Open Source; server-side code available for modification
- Pro Edition: Commercial license (proprietary)
- Dual licensing allows self-hosted OSS deployments and cloud SaaS options
- No viral licensing constraints; modifications can remain proprietary

---

### Cobalt.io

**Core features**
- Penetration Testing as a Service (PtaaS) model with on-demand testers
- Self-service test planning and scoping
- Vetted researcher community ("Cobalt Core") for engagement execution
- Human-Led, AI-Powered testing combining analyst expertise with automation
- Real-time findings delivery during engagement
- Automated remediation workflow management
- Integration with development and security tools
- Continuous testing capabilities (not just point-in-time assessments)

**Differentiating features**
- Managed testing model: leverage external experts without hiring
- Rapid engagement launch: start testing in <24 hours
- Diverse expertise: access testers with specialized skills on-demand
- Continuous model: ongoing testing rather than annual assessments
- AI augmentation: enhance manual testing with automated exploitation

**UX patterns**
- SaaS self-service: minimize pre-engagement administrative overhead
- Community-driven: testers from global vetted network
- Rapid deployment: reduce time-to-first-test significantly
- Continuous engagement: shift from one-time assessments to ongoing validation

**Integration points**
- Development platforms (GitHub, GitLab, Jira)
- Security tools (SIEM, vulnerability management platforms)
- Communication tools (Slack, email)
- Ticketing systems for remediation tracking
- API for custom workflow integration

**Known gaps**
- Less control for organizations running their own in-house engagements
- Researcher assignment may vary in expertise matching
- Limited customization of engagement scope (compared to internal teams)
- Pricing model based on engagement credits (less predictable for budgeting)

**Licence / IP notes**
- Proprietary SaaS platform; no licensing concerns
- Data residency and confidentiality agreements standard with PtaaS

---

### Faraday

**Core features**
- Central hub for aggregating findings from 80+ security tools
- Unified penetration testing reporting and centralized data
- Vulnerability management with context-driven scoring
- Team collaboration across offensive security and operations
- Full attack surface visibility in single platform
- Integration with DevSecOps pipelines
- Centralized project and engagement management

**Differentiating features**
- Broad scanner integration: 80+ tools aggregated natively
- Multi-functional: combines vulnerability management and pentest reporting
- DevSecOps integration: fits into continuous testing workflows
- Smart scoring: context-driven prioritization vs. raw vulnerability counts
- Unified visibility: reduce tool sprawl in security operations

**UX patterns**
- Tool-agnostic: accept findings from any scanner
- Context-aware: filter by asset, service, business criticality
- Collaborative: shared findings across offensive and defensive teams
- Progressive disclosure: start with high-risk findings, drill down as needed

**Integration points**
- 80+ scanner and tool integrations (Burp, Nessus, Qualys, Nuclei, etc.)
- Jira and ServiceNow for ticket creation
- Slack and email for notifications
- REST API for custom workflows
- GitLab/GitHub integration for DevSecOps pipelines
- Kubernetes and container registries

**Known gaps**
- Steeper learning curve for non-technical managers
- Less emphasis on managed PtaaS model (vs. Cobalt, Pentera)
- Limited AI-powered enrichment
- Requires significant configuration for optimal use

**Licence / IP notes**
- Community Edition: Open Source (GPL); self-hosted
- Professional/Enterprise: Commercial dual-license available
- Open-source deployment allows unlimited modifications and deployment
- Commercial variant available for organizations wanting support/SaaS

---

### AttackForge

**Core features**
- Engagement scoping and scheduling
- Comprehensive reporting with client portal
- Multi-user collaboration on pentest findings
- Integration with Jira and ServiceNow
- Methodology templates (OWASP, PTES)
- Client portal for shared visibility
- Asset tracking and management

**Differentiating features**
- Strong client-facing portal: professional presentation of findings
- Streamlined scoping: pre-engagement questionnaires and templates
- Mid-market focus: balances simplicity and power

**UX patterns**
- Client-centric: early portal access and transparent communication
- Template-driven: reduce administrative overhead in engagement setup
- Collaborative findings entry: multiple testers can contribute simultaneously

**Integration points**
- Jira and ServiceNow for ticket creation
- Email for communications
- REST API for custom integrations

**Known gaps**
- Less mature than PlexTrac for large-scale programs
- Limited open-source penetration in market
- Smaller integration ecosystem vs. Dradis/Faraday

**Licence / IP notes**
- Proprietary SaaS; no licensing concerns

---

### Pentera

**Core features**
- Automated real-world exploitation across kill chains
- Password cracking against Active Directory
- Lateral movement testing and privilege escalation validation
- Ransomware emulation against real malware families
- Validated exploitability reporting (not just vulnerability lists)
- Root-cause analysis and actionable remediation steps
- Continuous testing and revalidation after remediation
- 24/7 automated testing in live production environments
- CTEM (Continuous Threat Exposure Management) integration

**Differentiating features**
- Automated exploitation: execute complete attack chains
- Ransomware emulation: test against real-world families (LockBit, BlackCat, Play)
- Validated risk: focus on exploitable vulnerabilities, not CVE lists
- Continuous operation: ongoing testing beyond point-in-time engagements
- Remediation validation: automated retest to confirm fixes actually work

**UX patterns**
- Exploitation-first: executive reporting of "what attackers can actually exploit"
- Continuous operation: treat security validation as ongoing process
- Automated workflows: minimize manual pentest administration
- Business impact focus: risk prioritized by business criticality, not CVE severity

**Integration points**
- Cloud environments (AWS, Azure, GCP) for continuous testing
- On-premises networks and infrastructure
- Remediation platforms for automated ticketing
- Executive reporting dashboards
- API for CTEM platform integration

**Known gaps**
- Complex setup for on-premises environments
- Limited support for human-led pentest management (vs. automated)
- Requires significant infrastructure access for testing
- May generate high false-positive rates if not properly tuned

**Licence / IP notes**
- Proprietary commercial SaaS; no licensing concerns
- Exploitation techniques may touch on patent-encumbered methodologies; legal review recommended

---

## Cross-Cutting Feature Themes

### Table-Stakes Features

Any penetration testing management platform must include:

- **Findings aggregation and storage** — Centralize findings from multiple engagements, testers, and tools in structured database
- **Report generation and customization** — Create client-ready reports with customizable templates and frameworks (OWASP, PTES, NIST)
- **CVSS and risk scoring** — Standardized severity assessment; frameworks like CVSS v3/v4 and custom risk models
- **Client portal** — Secure view for clients to monitor engagement progress and review findings
- **Remediation tracking** — Link findings to tickets; track fix status and retest validation
- **Team collaboration** — Multi-user workflows; real-time synchronization of findings and report status
- **Multi-tool integration** — Aggregate findings from scanners (Burp, Nessus, Qualys, Nuclei) and manual entry
- **Engagement planning and scoping** — Define scope, methodology, rules of engagement; schedule testing windows
- **Audit logging and compliance** — Full audit trail of all changes; compliance reports for GDPR, HIPAA, SOC2
- **Access control and data isolation** — RBAC and data segregation for multi-tenant or multi-engagement environments

### Differentiating Features

Capabilities that set leaders apart:

- **Reusable issue libraries** — Institutional knowledge: accumulate vetted findings across engagements and leverage across team
- **Real-time findings delivery** — Communicate findings as they're discovered, not after engagement concludes
- **Automation and orchestration** — Auto-create tickets, send alerts, run retests; reduce manual effort
- **Continuous testing models** — Move beyond point-in-time assessments to ongoing validation
- **Framework mapping (MITRE ATT&CK)** — Align findings to attack tactics and techniques; structured threat modeling
- **AI-assisted report writing** — Auto-generate findings descriptions and remediation recommendations
- **Managed PtaaS model** — Access vetted testers without hiring; rapid engagement launch
- **Automated exploitation** — Execute complete attack chains; validate real exploitability (not just theoretical)
- **Ransomware and malware emulation** — Test against real-world threat families with legitimate malware samples
- **Remediation validation** — Automated retesting to confirm fixes actually reduce risk
- **Scanner integration breadth** — Support 50+, 80+, or 100+ tools; reduce tool sprawl

### Underserved Areas / Opportunities

Genuine gaps where a new entrant could differentiate:

- **Open-source platform with modern UX** — Dradis and Faraday are open-source but have dated UX; modern open-source pentest platform would disrupt
- **SMB-friendly pentest management** — Simplified platform for boutique firms and internal teams; avoid enterprise complexity
- **Automated scope generation** — AI-driven scope modeling from asset inventory and attack surface data
- **Predictive remediation prioritization** — ML-based correlation of finding severity, asset business value, and threat landscape
- **API-first architecture for integration** — Most platforms are web-UI-first; API-first would enable seamless DevSecOps integration
- **Cross-engagement finding deduplication** — ML-based deduplication across multiple engagements and testers
- **Fully air-gapped deployment** — Organizations in secure/air-gapped networks lack good options
- **Lightweight Docker-based deployment** — Easier deployment and scaling than traditional on-premises
- **Real-time findings visualization** — Live dashboard of testing progress (not just post-engagement reports)

### AI-Augmentation Candidates

Features currently implemented via manual work or rule-based systems where AI could excel:

- **Automated finding classification and deduplication** — Current: manual analyst review. Better: LLM classify findings across engagements; deduplicate automatically
- **Natural-language report generation** — Current: analysts write narrative descriptions manually. Better: LLM auto-generate descriptions from raw findings; improve consistency
- **Scope modelling and test case suggestion** — Current: manual scope definition. Better: LLM analyze asset inventory and attack surface; suggest test cases
- **Prioritization based on exploitability + context** — Current: CVSS scores or manual risk assessment. Better: ML combine severity + asset business value + threat landscape
- **Automated rules of engagement and NDA generation** — Current: templates filled manually. Better: LLM draft custom RoE/NDA from structured intake form
- **Attack path inference** — Current: testers manually document lateral movement. Better: LLM infer attack chains from raw findings
- **Executive brief generation** — Current: analysts summarize findings manually. Better: LLM auto-generate executive summaries with context
- **Finding root-cause analysis** — Current: manual. Better: LLM analyze vulnerability; suggest root cause and fix approach
- **Remediation recommendation ranking** — Current: static prioritization. Better: ML rank remediation by effort vs. risk impact

---

## Legal & IP Summary

**Open-source considerations:** Dradis and Faraday both offer open-source community editions. Dradis (server code available) and Faraday (GPL) allow modifications but may have copyleft implications for proprietary derivatives. Verify licensing before commercial deployment or modification.

**Patent considerations:** Automated exploitation techniques (Pentera) and advanced ML-based vulnerability analysis likely have patent implications. Organizations implementing similar capabilities should conduct patent landscape review before deployment. Lateral movement and privilege escalation exploitation methodologies may also be patent-encumbered; independent legal review recommended.

**Data handling:** PtaaS models (Cobalt.io) involve sharing infrastructure and test data with third-party researchers. Verify confidentiality agreements and data residency requirements before engagement.

**No material was omitted due to copyright uncertainty.** All sources were publicly available product documentation.

---

## Recommended Feature Scope

Based on the analysis, here's a prioritised feature scope for the project:

### Must-Have (MVP)

- **Findings database and aggregation** — Store findings from manual entry and basic scanner imports; central source of truth
- **Report generation with templates** — Customizable templates for OWASP, PTES, NIST; client-ready PDF/HTML output
- **CVSS v3 scoring** — Standardized severity assessment; audit trail of score changes
- **Basic client portal** — Read-only findings view for client stakeholders with secure access
- **Engagement scoping** — Define test scope, objectives, rules of engagement; store as metadata
- **Team management** — User roles (admin, tester, analyst, client); RBAC for findings access
- **Audit logging** — Track all findings changes, user actions, report generation events

### Should-Have (v1.1)

- **Multi-tool integration** — Import findings from Burp, Nessus, Qualys, manual entry
- **MITRE ATT&CK mapping** — Tag findings with attack techniques; filter by tactic/technique
- **Reusable findings library** — Store vetted descriptions; reuse across engagements
- **Remediation tracking** — Link findings to tickets (Jira, ServiceNow); track fix status
- **Real-time collaboration** — Multiple testers can contribute findings simultaneously
- **Remediation validation workflows** — Track retests and confirm vulnerability fixes
- **Compliance reporting** — Pre-built reports for GDPR, HIPAA, SOC2, PCI DSS

### Nice-to-Have (Backlog)

- **Automated report generation** — LLM-based narrative generation from findings
- **Continuous testing orchestration** — Schedule and automate ongoing testing campaigns
- **Ransomware emulation** — Simulate real malware families; test EDR and recovery
- **Scope generation AI** — Suggest test cases based on asset inventory and risk profile
- **Cross-engagement deduplication** — ML-based finding matching across multiple pentests
- **Managed PtaaS model** — Access vetted external testers through integrated platform
- **Real-time testing dashboard** — Live visualization of ongoing test progress
- **Predictive remediation prioritization** — ML rank remediation by effort vs. risk impact
- **Mobile application** — On-the-go engagement updates and finding triage
