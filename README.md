# Penetration Testing Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for engagement scoping, findings management, and remediation tracking across penetration testing programmes.

Penetration Testing Management gives security consultancies, MSSPs, and in-house red teams a single workspace for planning engagements, capturing findings, generating reports, and tracking remediation. It targets the gap between expensive enterprise SaaS suites and ageing open-source tooling, combining a modern UX with AI-assisted classification, reporting, and prioritisation.

---

## Why Penetration Testing Management?

- Incumbent enterprise platforms (PlexTrac, AttackForge) use opaque, custom enterprise pricing typically in the USD 20,000–100,000+ annual range, putting them out of reach for boutique firms and internal teams.
- The leading open-source options (Dradis CE, Faraday) deliver data sovereignty but have dated UX and limited AI/ML enrichment.
- PtaaS platforms (Cobalt.io) optimise for managed external testing and offer little control to organisations running their own in-house engagements.
- Automated platforms like Pentera focus on exploitation but do not manage human-led engagement workflows.
- Analyst time is dominated by manual triage, deduplication, and narrative report writing — work where modern LLMs can deliver immediate leverage.

---

## Key Features

### Findings & Engagement Core

- Findings database aggregating manual entries and scanner imports as a central source of truth
- Engagement scoping with stored objectives, rules of engagement, and methodology metadata
- CVSS v3 severity scoring with full audit trail of score changes
- Team management with RBAC roles for admin, tester, analyst, and client
- Audit logging across findings changes, user actions, and report generation

### Reporting & Client Delivery

- Customisable report templates aligned to OWASP, PTES, and NIST methodologies
- Client-ready PDF and HTML output
- Read-only client portal with secure stakeholder access to engagement findings
- Compliance-oriented reporting for GDPR, HIPAA, SOC 2, and PCI DSS

### Integrations & Collaboration

- Multi-tool finding ingestion from Burp, Nessus, Qualys, and manual entry
- MITRE ATT&CK tagging with filtering by tactic and technique
- Reusable findings library so vetted descriptions can be applied across engagements
- Remediation tracking via Jira and ServiceNow ticket linkage with fix-status updates
- Real-time collaboration for multiple testers contributing simultaneously
- Remediation validation workflows that track retests and confirm fixes

### AI-Assisted Workflows (Backlog)

- LLM-based narrative report generation from raw findings data
- AI-driven scope generation and test-case suggestion from asset inventory
- Cross-engagement finding deduplication using ML matching
- Predictive remediation prioritisation ranking fixes by effort versus risk impact
- Continuous testing orchestration for scheduled, ongoing campaigns

---

## AI-Native Advantage

AI is applied where incumbents still rely on manual analyst effort: automatic classification and deduplication of findings across engagements, natural-language generation of finding narratives and executive summaries, AI-assisted scope modelling against asset inventory and engagement history, and predictive remediation prioritisation that combines severity, business criticality, and known exploit availability. An agentic scoping assistant can also draft rules of engagement, NDAs, and test plans from a structured intake form to reduce pre-engagement administrative overhead.

---

## Tech Stack & Deployment

The project targets self-hosted deployment for data sovereignty alongside an optional cloud mode, addressing the air-gapped and lightweight Docker deployment gaps left by current open-source tools. Methodology alignment follows the dominant standards in the field: PTES, NIST SP 800-115, OSSTMM, the OWASP Testing Guide v5, and CVSS for severity scoring. An API-first architecture is intended to enable DevSecOps and CTEM integration, with native connectors for Jira, ServiceNow, Slack, and common scanners.

---

## Market Context

The penetration testing software market is estimated at USD 214.87 million in 2026, projected to reach USD 477.65 million by 2035 at a 9.2% CAGR, with the broader penetration testing services market forecast at approximately USD 3.09 billion in 2026 (Market Reports World, 2026). Enterprise management platforms typically charge USD 20,000–100,000+ annually, while PtaaS vendors price per engagement or via credit-based subscriptions. Primary buyers are heads of security at mid-to-large enterprises, MSSP and boutique pentest firm owners, and compliance officers in regulated sectors such as finance, healthcare, and government.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
