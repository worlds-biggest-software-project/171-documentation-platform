# Standards & API Reference

> Project: Documentation Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 26514:2022 — Systems and Software Documentation** — ISO standard for requirements for designers and developers of documentation; defines documentation process requirements and documentation product requirements relevant to developer documentation platforms. URL: https://www.iso.org/standard/80716.html

- **ISO/IEC 27001:2022** — Information security management; governs access controls, audit logging, and data handling for documentation platforms that may host confidential API references, internal process documentation, and pre-release product information. URL: https://www.iso.org/standard/82875.html

### W3C & IETF Standards

- **RFC 7763 / RFC 7764 — MIME Type for Markdown** — IETF RFCs registering the `text/markdown` MIME type; relevant to documentation platforms that serve Markdown content via API or store Markdown files in Git repositories. URL: https://datatracker.ietf.org/doc/html/rfc7763

- **CommonMark Specification** — Unambiguous Markdown specification (spec.commonmark.org); the universal Markdown standard for documentation platforms (Docusaurus, MkDocs, GitBook, Mintlify); OpenAPI requires CommonMark 0.27 minimum for description field rendering. URL: https://spec.commonmark.org/

- **MDX (Markdown + JSX)** — Extended Markdown format allowing React component embedding in documentation pages; used natively by Docusaurus (Meta) and Mintlify; enables interactive code examples, callouts, and embedded demo components in documentation. URL: https://mdxjs.com/

- **RFC 6749 — OAuth 2.0** — Authorization framework used by hosted documentation platforms (ReadMe, GitBook, Mintlify) for SSO integration with enterprise identity providers and API key-based access to documentation management APIs. URL: https://datatracker.ietf.org/doc/html/rfc6749

- **RFC 7519 — JSON Web Token (JWT)** — Used for API-token-based access to documentation management APIs and for authenticating readers to private/internal documentation portals. URL: https://datatracker.ietf.org/doc/html/rfc7519

- **RFC 4287 — Atom Syndication Format** — Used by documentation platforms for publishing changelog and release notes feeds that can be consumed by feed readers and developer notification systems. URL: https://datatracker.ietf.org/doc/html/rfc4287

- **W3C WCAG 2.2 — Web Content Accessibility Guidelines** — Governs the accessibility of documentation web interfaces; technical documentation must meet WCAG 2.2 AA (keyboard navigation, colour contrast, semantic headings, alt text for diagrams). URL: https://www.w3.org/TR/WCAG22/

### Data Model & API Specifications

- **OpenAPI Specification 3.1 (OAS 3.1)** — The primary machine-readable API description standard; documentation platforms (ReadMe, Mintlify, Redocly, GitBook) consume OpenAPI specs to generate interactive API reference documentation, "Try it" playgrounds, and SDK code samples; OpenAPI 3.2 in development as of 2026. URL: https://spec.openapis.org/oas/v3.2.0.html

- **AsyncAPI 3.0** — API description standard for event-driven APIs (WebSocket, MQTT, Kafka, SNS/SQS); documentation platforms that serve developer portals for message-driven architectures must support AsyncAPI alongside OpenAPI. URL: https://www.asyncapi.com/docs/reference/specification/v3.0.0

- **JSON Schema** — Standard for describing JSON data structures; used in OpenAPI schemas for request/response body documentation; documentation platforms render JSON Schema visually for developer reference. URL: https://json-schema.org/

- **GraphQL SDL (Schema Definition Language)** — The type system syntax for GraphQL APIs; documentation platforms increasingly generate GraphQL reference docs from SDL alongside OpenAPI REST docs. URL: https://graphql.org/learn/schema/

- **OpenAPI 3.1** — Used by ReadMe, Mintlify, Redocly, and GitBook to describe their own management REST APIs; enables programmatic content creation, navigation management, and deployment automation. URL: https://spec.openapis.org/oas/latest.html

- **Diátaxis Documentation Framework** — Systematic approach to structuring technical documentation around four quadrants: tutorials, how-to guides, explanations, and reference; widely adopted as the standard methodology for organising developer documentation; integrated into MCP skill implementations (mcpmarket.com). URL: https://diataxis.fr/

- **Docs-as-Code** — Methodology for treating documentation like software code: stored in Git, reviewed in pull requests, built by CI/CD pipelines, and deployed automatically; the standard workflow for engineering-owned documentation platforms (Docusaurus, MkDocs, Sphinx). URL: https://www.writethedocs.org/guide/docs-as-code/

### Security & Authentication Standards

- **GDPR Article 32 — Security of Processing** — Documentation platforms hosting internal documentation containing personal data, customer case studies, or confidential API documentation must implement appropriate access controls and encryption. URL: https://gdpr-info.eu/art-32-gdpr/

- **SOC 2 Type II** — Required enterprise compliance certification for SaaS documentation platforms; ReadMe, GitBook, and Mintlify maintain SOC 2 Type II compliance for enterprise customers. URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2

- **SAML 2.0 / OIDC** — Required for enterprise SSO integration in documentation platforms; ensures documentation access is tied to enterprise identity lifecycle for private/internal documentation portals. URL: https://docs.oasis-open.org/security/saml/v2.0/

- **SCIM 2.0 (RFC 7643/7644)** — Used for automated user provisioning and deprovisioning in enterprise documentation platform deployments. URL: https://datatracker.ietf.org/doc/html/rfc7643

- **Content Security Policy (CSP)** — W3C standard for controlling resources loaded by documentation pages; important for documentation platforms that embed interactive demos, live code editors, and API playgrounds to prevent XSS attacks. URL: https://www.w3.org/TR/CSP3/

- **OWASP API Security Top 10 (2023)** — Governs the REST API security of documentation management platforms; API2 (Broken Authentication) is critical for ensuring only authorised users can publish or update documentation. URL: https://owasp.org/API-Security/

### MCP Server Specifications

Documentation platforms are at the forefront of the MCP ecosystem in 2025-2026:

- **Mintlify MCP Server Auto-Generation** — Mintlify automatically generates Model Context Protocol (MCP) servers from documentation; enables AI applications to query docs in real-time and execute API calls on behalf of users; turns static documentation into a queryable resource for AI agents. URL: https://www.mintlify.com/blog/generate-mcp-servers-for-your-docs

- **Context7 / docs-mcp-server** — Open-source (MIT) "Grounded Docs MCP Server" (arabold/docs-mcp-server, upstash/context7); fetches official documentation from websites, GitHub, npm, PyPI, and local files to solve AI hallucination by providing AI with exact, version-specific documentation. URL: https://github.com/arabold/docs-mcp-server

- **OpenAI Docs MCP Server** — OpenAI hosts a public MCP server for developers.openai.com and platform.openai.com documentation; enables AI clients to query OpenAI's documentation in real time. URL: https://developers.openai.com/learn/docs-mcp

- **Microsoft Learn MCP Server** — Remote MCP server (Streamable HTTP) for Microsoft Learn documentation; compatible with GitHub Copilot and other MCP clients. URL: https://learn.microsoft.com/en-us/training/support/mcp-developer-reference

- **AWS Documentation MCP Server** — AWS Labs open-source MCP server for AWS documentation; enables AI agents to query AWS service documentation. URL: https://awslabs.github.io/mcp/servers/aws-documentation-mcp-server

- **MCP Servers for Documentation Sites (April 2026)** — Fern published a framework pattern for generating MCP servers from documentation sites; MCP servers for docs expose API schemas, code repositories, and content in a format AI clients can query, replacing the need for RAG pipelines over static docs. URL: https://buildwithfern.com/post/mcp-servers-documentation-sites

---

## Similar Products — Developer Documentation & APIs

### ReadMe

- **Description:** Complete developer hub SaaS platform with API reference (OpenAPI), guides, changelogs, and community forums; live "Try it" playground for real-time authenticated API testing; deep API analytics (usage, errors, developer behaviour); enterprise governance and audit logs.
- **API Documentation:** https://docs.readme.com/main/reference
- **SDKs/Libraries:** ReadMe Management API (REST/JSON); GitHub Actions sync; rdme CLI (Node.js)
- **Developer Guide:** https://docs.readme.com/
- **Standards:** REST/JSON, OpenAPI 3.1 (consumption), CommonMark, OAuth 2.0, API token auth, SAML 2.0
- **Authentication:** API key (Authorization header); OAuth 2.0 for SSO

### Mintlify

- **Description:** AI-native documentation platform with MDX-based content, bi-directional Git sync, auto-generated MCP servers, and AI writing assistance; preferred by developer-focused startups; automatically generates MCP servers from docs for AI agent integration.
- **API Documentation:** https://mintlify.com/docs/api-reference/overview
- **SDKs/Libraries:** Mintlify REST API; mintlify CLI; MDX components library; MCP server auto-generation
- **Developer Guide:** https://mintlify.com/docs
- **Standards:** MDX (Markdown + JSX), CommonMark, OpenAPI 3.1 (consumption), REST/JSON, OAuth 2.0, API token auth, Git-based sync
- **Authentication:** API key; OAuth 2.0 for team SSO

### GitBook

- **Description:** Hosted documentation platform with no-build-step publishing; sign up → write Markdown → docs are live; Git sync for code-first teams; REST API for content management; AI assistant (GitBook AI); used by 50,000+ teams.
- **API Documentation:** https://developer.gitbook.com/
- **SDKs/Libraries:** GitBook REST API (JSON); @gitbook/api (TypeScript); Webhooks; GitHub/GitLab sync
- **Developer Guide:** https://developer.gitbook.com/
- **Standards:** REST/JSON, OpenAPI, CommonMark, OAuth 2.0, API token auth, Webhooks, SAML 2.0 (Enterprise)
- **Authentication:** API token; OAuth 2.0 for integrations; SAML 2.0 (Enterprise)

### Docusaurus (Open Source)

- **Description:** Open-source (MIT) static documentation site generator from Meta; Markdown and MDX support; React-based; versioning; i18n (internationalisation); plugin ecosystem; docs-as-code workflow via Git; widely used by open-source projects (React, TypeScript, Redwood).
- **API Documentation:** https://docusaurus.io/docs/api/docusaurus-config
- **SDKs/Libraries:** @docusaurus/core (npm); @docusaurus/plugin-content-docs; community plugins; MDX ecosystem
- **Developer Guide:** https://docusaurus.io/docs
- **Standards:** MDX (Markdown + JSX), CommonMark, React, OpenAPI (via Redocly/docusaurus-openapi plugin), Algolia DocSearch, MIT licence
- **Authentication:** Not applicable (static site); Netlify/Vercel auth layers for private deployments

### MkDocs (Open Source)

- **Description:** Open-source (BSD) Python-based static documentation site generator; simple YAML configuration; strong Python ecosystem adoption; MkDocs Material theme is the reference design standard; entered maintenance mode in November 2025 (bug fixes/security patches only, no new features).
- **API Documentation:** https://www.mkdocs.org/user-guide/
- **SDKs/Libraries:** mkdocs (PyPI); mkdocs-material (PyPI, maintenance mode); mkdocstrings (API reference auto-generation from docstrings)
- **Developer Guide:** https://www.mkdocs.org/
- **Standards:** CommonMark, Python-Markdown extensions, YAML config, Algolia/lunr.js search, BSD licence
- **Authentication:** Not applicable (static site)

### Redocly

- **Description:** API-first documentation platform; OpenAPI-native; Redoc (open-source API reference renderer, MIT); Redocly CLI for OpenAPI linting, bundling, and docs generation; Redocly Portal for full developer portal SaaS; strong in enterprise API governance.
- **API Documentation:** https://redocly.com/docs/developer-portal/
- **SDKs/Libraries:** Redoc (React component, npm: redoc); @redocly/cli (npm); @redocly/openapi-core; Redocly Portal APIs
- **Developer Guide:** https://redocly.com/docs/
- **Standards:** OpenAPI 3.0/3.1 (primary), AsyncAPI, CommonMark, REST/JSON, OAuth 2.0
- **Authentication:** API key; OAuth 2.0 for enterprise SSO

### Sphinx (Open Source)

- **Description:** Open-source (BSD) Python documentation generator; the standard for Python library documentation (ReadTheDocs ecosystem); reStructuredText (RST) and MyST Markdown; auto-generates API reference from Python docstrings; used by Django, Python, NumPy, SQLAlchemy.
- **API Documentation:** https://www.sphinx-doc.org/en/master/usage/restructuredtext/
- **SDKs/Libraries:** Sphinx (PyPI); MyST-Parser; sphinx-autodoc; ReadTheDocs hosting; sphinx-needs (requirements docs)
- **Developer Guide:** https://www.sphinx-doc.org/
- **Standards:** reStructuredText (RST), MyST Markdown, Docutils, Python docstring conventions (NumPy, Google, Sphinx style), BSD licence
- **Authentication:** Not applicable (static site); ReadTheDocs/Netlify auth for private docs

### Context7 / docs-mcp-server (Open Source)

- **Description:** Open-source (MIT) MCP server that fetches official documentation from websites, GitHub, npm, PyPI, and local files; solves AI hallucination by providing AI tools with exact, version-specific documentation; alternative to Context7, Nia, and Ref.Tools commercial services.
- **API Documentation:** https://github.com/arabold/docs-mcp-server
- **SDKs/Libraries:** Node.js/TypeScript MCP server; Docker; npm package
- **Developer Guide:** https://github.com/arabold/docs-mcp-server
- **Standards:** MCP (Anthropic Model Context Protocol), REST (documentation fetching), CommonMark, MIT licence
- **Authentication:** Not applicable (local MCP server)

---

## Notes

- **MCP as the defining 2026 documentation technology**: The emergence of MCP servers for documentation (Mintlify auto-generation, Context7, OpenAI Docs MCP, AWS Docs MCP) represents a fundamental shift — documentation is becoming a live, queryable data source for AI agents rather than static web pages; this is now a baseline expectation for developer documentation platforms targeting AI-native teams.

- **Diátaxis as the standard documentation methodology**: The Diátaxis framework (tutorials, how-to guides, explanations, reference) has become the de facto standard methodology for structuring developer documentation; documentation platforms should support content organisation aligned with Diátaxis quadrants.

- **OpenAPI 3.1 as the API documentation backbone**: All API documentation platforms (ReadMe, Mintlify, Redocly, GitBook) consume OpenAPI 3.1 specifications to generate interactive reference docs; new developer portals must provide excellent OpenAPI rendering as table stakes.

- **MDX replacing plain Markdown**: MDX (Markdown + React JSX) is rapidly replacing plain Markdown for developer documentation that requires interactive components; Mintlify and Docusaurus are MDX-native; GitBook and MkDocs support MDX via plugins.

- **MkDocs Material maintenance mode (November 2025)**: The widely-used MkDocs Material theme entered maintenance mode in November 2025; teams building new documentation platforms should consider Docusaurus, Mintlify, or Redocly as active alternatives.

- **Open-source landscape**: Docusaurus (MIT, Meta), MkDocs (BSD), Sphinx (BSD), and Redoc (MIT) are the leading open-source documentation tools; Context7/docs-mcp-server (MIT) is the leading open-source documentation MCP server; all major open-source options require self-hosting for the documentation site itself.
