# Documentation Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source technical documentation platform with versioning, search, and a context-aware in-doc assistant — a credible alternative to GitBook, Mintlify, and Document360.

This project builds a documentation platform for engineering and product teams who publish technical docs, API references, and knowledge bases. It targets teams frustrated with rising SaaS prices, weak versioning in general-purpose tools, and the absence of a polished open-source option that combines modern AI capabilities with self-hostability.

---

## Why Documentation Platform?

- GitBook's 2025 pricing increase (2–3x) drove significant user churn and left a visible gap in the mid-market.
- Mintlify is well-designed but expensive ($300+/mo for Pro, $600+/mo Custom) and narrowly focused on API docs, locking out smaller teams and non-API content.
- Document360 removed public pricing in November 2024, signalling enterprise-only positioning and reducing transparency for buyers.
- Strong open-source options (Docusaurus, Vitepress) require engineering setup and ship without AI features, leaving non-technical writers and AI-curious teams underserved.
- Confluence and Notion offer broad workspaces but have weak versioning, no OpenAPI rendering, and poor controls for external-facing documentation.

---

## Key Features

### Authoring and Content

- Markdown and MDX editing for developer-friendly authoring
- Block-based visual editor with live preview
- Real-time multi-author collaboration
- Review and publishing workflows with approval gates
- Content reuse and DRY snippets across pages

### Versioning and Source Control

- Git sync with GitHub and GitLab for two-way synchronisation
- Built-in versioning with restoration to any historical version
- Branching and merge workflows aligned with code release cycles
- Multi-language publishing with version parity

### Search, AI Assistant, and Discovery

- Full-text search across all documentation
- AI-powered Q&A assistant embedded in the docs site
- Semantic search that finds concepts by meaning, not keyword
- AI writing suggestions for clarity, completeness, and tone
- Search analytics to surface gaps in coverage

### API Documentation

- Native OpenAPI 3.0 and 3.1 rendering
- Interactive "Try it" API explorer with code samples
- Auto-generated endpoint reference from specs
- API changelog and version tracking

### Operations and Integrations

- Role-based access control and team permissions
- Analytics for page views, search queries, and engagement
- Slack notifications for changes and approvals
- Custom domains and white-label branding
- REST API for custom integrations
- Responsive rendering across desktop, tablet, and mobile

---

## AI-Native Advantage

The platform integrates AI into the documentation lifecycle rather than bolting on a chatbot. It detects documentation drift by diffing code commits against published docs — something no incumbent does natively end-to-end. It generates first-draft documentation directly from source code, OpenAPI specs, and inline comments, and provides a context-aware in-doc assistant that understands the entire codebase rather than just static text. Reader-aware personalisation adapts content depth to inferred expertise, and multilingual auto-translation propagates terminology consistently across locales.

---

## Tech Stack & Deployment

The platform is designed to support both self-hosted and managed deployment. Content is stored in Markdown / MDX with Git as the source of truth, enabling integration with existing GitHub and GitLab workflows. OpenAPI 3.x is a first-class input format. The architecture follows the Diátaxis information-architecture framework (tutorials, how-to, reference, explanation) and supports CommonMark, MDX, Mermaid, and PlantUML. SDKs and integration points target REST APIs, Slack, and CI systems for drift detection.

---

## Market Context

The software documentation tools market was valued at approximately USD 6.3 billion in 2024 and is projected to reach USD 12.5 billion by 2033 at roughly 8% CAGR (Verified Market Reports, 2024). SaaS incumbents range from ~$40/mo for SMB tools (Archbee) to $300–$600/mo for developer platforms (Mintlify) and $30K–$100K+/yr for enterprise contracts (Document360, Confluence). Primary buyers are developer-experience engineers, technical writers, developer-relations teams, and CTOs of API-first SaaS companies.

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
