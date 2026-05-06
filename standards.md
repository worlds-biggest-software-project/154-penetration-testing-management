# Standards & API Reference

> Project: Penetration Testing Management · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 27001:2022** — Information security management systems; Annex A control A.8.8 (Management of technical vulnerabilities) and A.5.36 (Compliance with policies, rules, and standards for information security) mandate periodic security testing including penetration testing as a core assurance activity. URL: https://www.iso.org/standard/82875.html

- **ISO/IEC 27002:2022** — Implementation guidance; Section 8.8 advises organisations to conduct penetration tests as part of vulnerability management, document results, and track remediation through a formal process. URL: https://www.iso.org/standard/75652.html

- **ISO/IEC 29147:2018 — Vulnerability Disclosure** — Defines processes for receiving, handling, and disclosing vulnerabilities discovered during penetration tests; relevant for coordinating responsible disclosure between pentest teams and vendors. URL: https://www.iso.org/standard/72311.html

### W3C & IETF Standards

- **RFC 9110 — HTTP Semantics** — Governs the REST API design of pentest management platform management interfaces; relevant to API-based submission of findings, asset definitions, and engagement metadata. URL: https://datatracker.ietf.org/doc/html/rfc9110

- **RFC 6749 — OAuth 2.0** — Standard authorization framework used by pentest management platforms (Cobalt, PlexTrac) for API authentication and for integrating with CI/CD pipelines and SOAR tools. URL: https://datatracker.ietf.org/doc/html/rfc6749

- **RFC 7519 — JSON Web Token (JWT)** — PlexTrac uses JWT for API session management; commonly used across pentest management APIs for stateless authenticated requests. URL: https://datatracker.ietf.org/doc/html/rfc7519

### Data Model & API Specifications

- **CVSS 4.0 — Common Vulnerability Scoring System** — FIRST.org standard for rating vulnerability severity (Base, Threat, Environmental, Supplemental metric groups); the universal severity scoring system used in pentest findings reports and remediation prioritisation. URL: https://www.first.org/cvss/

- **PTES — Penetration Testing Execution Standard** — Open practitioner framework defining seven engagement phases: Pre-engagement, Intelligence Gathering, Threat Modelling, Vulnerability Analysis, Exploitation, Post-Exploitation, and Reporting; widely used to structure pentest management workflows. URL: http://www.pentest-standard.org/

- **OWASP Web Security Testing Guide (WSTG) v4.2** — Comprehensive methodology for web application penetration testing; defines test cases by category (authentication, authorisation, session management, injection, etc.) used as checklist templates in pentest management platforms. URL: https://owasp.org/www-project-web-security-testing-guide/

- **OWASP API Security Top 10 (2023)** — Defines the ten most critical API security risks; used as a structured test case library for API-focused penetration tests managed on PTaaS platforms. URL: https://owasp.org/API-Security/

- **MITRE ATT&CK** — Adversary tactics, techniques, and procedures (TTP) knowledge base in STIX 2.1 format; used in pentest management platforms to map exploitation findings to ATT&CK techniques for threat-intelligence-aligned reporting. URL: https://attack.mitre.org/

- **OpenAPI 3.1** — REST API specification format used by pentest management platforms (Cobalt, PlexTrac) for their management APIs; enables Swagger-based exploration and SDK code generation. URL: https://spec.openapis.org/oas/latest.html

### Security & Authentication Standards

- **NIST SP 800-115 — Technical Guide to Information Security Testing and Assessment** — Foundational US government reference for planning and executing security assessments including penetration tests; defines planning, test types (network, system, application), data collection, and reporting phases; frequently cited in compliance mandates and SoW templates. URL: https://csrc.nist.gov/pubs/sp/800/115/final

- **NIST SP 800-53 Rev. 5 — CA-8 (Penetration Testing)** — Federal control explicitly requiring penetration testing at an organisation-defined frequency and scope; commonly referenced in FedRAMP, FISMA, and DoD assessments. URL: https://csrc.nist.gov/publications/detail/sp/800-53/5/final

- **PCI DSS v4.0 — Requirement 11.3 (Penetration Testing)** — Mandates penetration testing at least annually and after any significant infrastructure or application changes; defines scope, methodology, and remediation verification requirements for cardholder data environments. URL: https://www.pcisecuritystandards.org/

- **CREST — Council for Registered Ethical Security Testers** — UK and international professional body for pentest certification (CREST Certified Tester); pentest management platforms often include CREST-aligned report templates and tester credential verification. URL: https://www.crest-approved.org/

- **OSCP / OffSec Certifications** — Industry-standard offensive security certifications (OSCP, OSEP, OSED) from Offensive Security; recognised pentest team credential standards used in PTaaS platform tester vetting processes. URL: https://www.offsec.com/

- **TIBER-EU — Threat Intelligence-Based Ethical Red Teaming** — European Central Bank framework for red team testing of financial institutions; requires structured engagement lifecycle management aligned with threat intelligence; supported by pentest management platforms in the EU financial sector. URL: https://www.ecb.europa.eu/paym/cyber-resilience/tiber-eu/html/index.en.html

- **GDPR Article 32 — Security of Processing** — Penetration testing is cited in supervisory authority guidance as a technical measure for demonstrating security of processing; pentest management platforms must handle findings data (which can contain sensitive vulnerability details) with appropriate access controls. URL: https://gdpr-info.eu/art-32-gdpr/

- **FedRAMP High Authorization** — NodeZero (Horizon3.ai) received FedRAMP High Authorization in May 2025, making it the first fully autonomous penetration testing platform certified for US federal cloud environments. URL: https://www.fedramp.gov/

- **OWASP Top 10 (2021)** — Defines the ten most critical web application security risks; used as a baseline test scope in pentest management platform engagement templates. URL: https://owasp.org/Top10/

### MCP Server Specifications

Penetration testing management is beginning to intersect with the MCP ecosystem for AI-assisted security workflows:

- **AI Pentesting Agents (2026)** — 39+ AI pentesting agent tools catalogued as of 2026; platforms like NodeZero and Pentera use autonomous AI agents to execute test cases; MCP is emerging as an integration protocol for connecting AI pentest agents to vulnerability databases, ticketing systems, and reporting platforms. URL: https://appsecsanta.com/research/ai-pentesting-agents-2026

- **Adversarial Exposure Validation (Gartner, 2025)** — Gartner's emerging category combining breach and attack simulation (BAS), automated penetration testing, and automated red teaming into a unified exposure validation capability; by 2027, 40% of organisations are projected to run formal exposure validation programmes. URL: https://www.gartner.com/

---

## Similar Products — Developer Documentation & APIs

### Cobalt (PTaaS + Pentest Management Platform)

- **Description:** Pentest-as-a-Service platform combining an on-demand marketplace of vetted pentesters (Cobalt Core) with a Pentest Management Platform (PMP) for in-house pentest teams; covers the full pentest lifecycle from planning through remediation tracking.
- **API Documentation:** https://docs.cobalt.io/cobalt-api/
- **Developer Guide:** https://docs.cobalt.io/ and https://www.cobalt.io/developer-solutions
- **SDKs/Libraries:** REST API (JSON); DAST target data access; Cobalt API blog overview
- **Standards:** REST/JSON, OpenAPI
- **Authentication:** API token (retrieved from profile dropdown; Bearer token in Authorization header)

### PlexTrac

- **Description:** Pentest reporting and exposure management platform used by security consultancies and in-house red teams; centralises findings capture, evidence management, narrative report generation, and remediation workflow; integrates with Cobalt, Pentera, NodeZero, Nessus, Burp Suite, and more.
- **API Documentation:** https://api-docs.plextrac.com/
- **Developer Guide:** https://docs.plextrac.com/plextrac-documentation/api-documentation/overview
- **SDKs/Libraries:** REST API (JSON); GraphQL API (listed in API docs); Brinqa connector; BlinkOps connector
- **Standards:** REST/JSON, GraphQL, OpenAPI
- **Authentication:** JWT token (obtained via `/authenticate` endpoint; required for all API endpoints)

### HackerOne

- **Description:** Vulnerability disclosure and bug bounty programme management platform; provides coordinated disclosure (CVD) workflows, triage services, and bounty payment automation; expanding into AI Red Teaming for LLM and agentic application testing.
- **API Documentation:** https://api.hackerone.com/
- **SDKs/Libraries:** Ruby gem (hackerone-client); Python client; Postman collection
- **Developer Guide:** https://api.hackerone.com/
- **Standards:** REST/JSON (JSON:API spec), OpenAPI
- **Authentication:** API credentials (username + token, HTTP Basic Auth)

### Synack (Synack Red Team Platform)

- **Description:** Managed PTaaS platform with a curated global network of vetted security researchers (Synack Red Team); combines researcher-driven testing with automated scanning; strong in enterprise application and AI/LLM testing.
- **API Documentation:** https://platform.synack.com/api (requires account)
- **SDKs/Libraries:** REST API; webhook integrations; JIRA/ServiceNow connectors
- **Developer Guide:** Synack platform portal (account required)
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0
- **Authentication:** OAuth 2.0 (platform SSO); API key for programme integrations

### NodeZero (Horizon3.ai)

- **Description:** Fully autonomous penetration testing platform (FedRAMP High authorized, May 2025); executes continuous attack paths against production environments; first AI to solve the Active Directory Game of GOAD benchmark (14 minutes); strong for hybrid cloud and AD environments.
- **API Documentation:** https://docs.horizon3.ai/ (requires account)
- **SDKs/Libraries:** REST API; SIEM/SOAR integrations; PlexTrac integration available
- **Developer Guide:** Horizon3.ai developer portal (account required)
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0
- **Authentication:** API key / OAuth 2.0

### Pentera

- **Description:** Automated security validation platform performing continuous production-safe penetration testing; validates exploitable attack paths and confirms exposure reduction; widely used by large enterprises and global banks for hybrid infrastructure testing.
- **API Documentation:** https://docs.pentera.io/ (requires account)
- **SDKs/Libraries:** REST API; integration connectors for Jira, ServiceNow, Splunk
- **Developer Guide:** Pentera platform documentation (account required)
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0
- **Authentication:** API key; OAuth 2.0 for SSO

### Metasploit Framework (Open Source)

- **Description:** Open-source (BSD licence) exploitation framework and the most widely used penetration testing toolkit; Metasploit Pro adds management features (campaign management, report generation, team collaboration) with a REST API.
- **API Documentation:** https://docs.metasploit.com/ (Metasploit Pro REST API)
- **SDKs/Libraries:** metasploit-framework Ruby libraries; msfrpc client libraries (Python: python-msfrpc, pymetasploit3)
- **Developer Guide:** https://github.com/rapid7/metasploit-framework/wiki
- **Standards:** REST/JSON (Pro), MSGPACK-RPC (Framework console API), OpenAPI (partial)
- **Authentication:** Token-based authentication for REST API; username/password for console API

### Burp Suite Enterprise Edition (PortSwigger)

- **Description:** Enterprise web application security testing platform extending Burp Suite Professional with centralised scanning, CI/CD integration, and REST API for managing scan configurations, scheduling, and results across multiple applications.
- **API Documentation:** https://portswigger.net/burp/documentation/enterprise/api
- **SDKs/Libraries:** Java client library; GraphQL API; Burp Extensions (Montoya API for custom plugin development)
- **Developer Guide:** https://portswigger.net/burp/documentation/enterprise
- **Standards:** GraphQL, REST/JSON, OpenAPI; Burp Collaborator protocol
- **Authentication:** API key for REST/GraphQL; SSO (SAML 2.0, LDAP) for platform access

---

## Notes

- **PTES + OWASP + ATT&CK combination**: The practitioner consensus for 2025-2026 is to blend PTES for engagement lifecycle management, OWASP WSTG for web/API test case coverage, and MITRE ATT&CK for TTP-aligned adversary simulation reporting — a "polymethodologist" approach.

- **Adversarial Exposure Validation (AEV)**: Gartner's 2025 category consolidating BAS, automated pentesting, and automated red teaming is driving platform convergence; pentest management platforms (PlexTrac, Cobalt) are integrating with AEV tools (NodeZero, Pentera) via APIs.

- **FedRAMP High for autonomous pentesting (2025)**: NodeZero's May 2025 FedRAMP High Authorization validates autonomous AI pentesting for the most sensitive US federal workloads — a significant milestone for the category.

- **AI Red Teaming for LLMs (2025-2026)**: HackerOne and Synack are expanding into adversarial testing of AI/LLM applications; OWASP has published the LLM Top 10 (2025) as the emerging standard for AI application pentest scope.

- **Open-source landscape**: Metasploit Framework (BSD) is the dominant open-source pentest toolkit; Faraday (GPL) and Dradis (open-source community edition) provide open-source pentest management and collaboration; AttackForge has a community edition for reporting.
