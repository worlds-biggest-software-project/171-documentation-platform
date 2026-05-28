# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Documentation Platform · Created: 2026-05-20

## Philosophy

This model uses relational tables for structural data (organisations, users, spaces, page hierarchy) but delegates variable and extensible data to JSONB columns. Page metadata, theme configuration, integration settings, custom fields, and per-space configuration all live in JSONB, eliminating dozens of tables and enabling rapid feature iteration without migrations.

This is the approach favoured by modern SaaS products that need to ship quickly and adapt to diverse customer requirements. Notion's internal model uses a similar pattern — a relatively small number of core tables with rich JSON properties. Stripe's API objects are famously schema-flexible via metadata fields. The pattern works well for documentation platforms because different spaces may need different metadata (API docs need endpoint tags; tutorials need difficulty levels; changelogs need release dates).

The key insight is that not all data needs referential integrity enforced at the database level. A page's "custom_meta" field might contain `{"difficulty": "beginner", "estimated_reading_time": 5}` — this data is queried and displayed but does not participate in joins or foreign key relationships.

**Best for:** Rapid MVP development, teams that want fewer migrations, platforms where different documentation types need different metadata schemas. Ideal for small-to-medium engineering teams shipping fast.

**Trade-offs:**
- Pro: Far fewer tables (~20 vs ~33) — less migration overhead, simpler schema management
- Pro: New metadata fields require zero migrations — just start writing to the JSONB column
- Pro: Different spaces can have different metadata schemas without schema changes
- Pro: JSONB GIN indexes provide fast containment queries
- Pro: Natural fit for PostgreSQL — JSONB is a first-class citizen with mature tooling
- Con: No referential integrity on data inside JSONB columns
- Con: JSONB queries are slower than indexed relational columns for complex filtering
- Con: Schema validation must happen in application code, not the database
- Con: Harder to write reports that span JSONB fields (need `->>`  and `->>` extraction)
- Con: ORM support for JSONB varies — some ORMs treat it as an opaque blob

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenAPI 3.1 | Raw spec stored in `api_specs.raw_content`; parsed spec stored in `api_specs.parsed` as JSONB |
| CommonMark / MDX | Content format tracked in `pages.content->>'format'` |
| Diátaxis Framework | Stored as `pages.meta->>'diataxis_type'` — queryable via GIN index |
| ISO 639-1 | Language codes in `pages.translations` JSONB keys |
| JSON Schema | Optional: `spaces.meta_schema` can store a JSON Schema that validates page metadata |
| OAuth 2.0 / OIDC | SSO config stored in `organizations.settings->'sso'` JSONB path |
| SCIM 2.0 | Provisioning config in `organizations.settings->'scim'` |

---

## Core Identity

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "logo_url": "https://...",
    --   "billing_plan": "pro",
    --   "sso": { "provider": "saml", "idp_metadata_url": "..." },
    --   "scim": { "enabled": true, "token_hash": "..." },
    --   "default_theme": { "primary_color": "#2563eb", "font": "Inter" },
    --   "allowed_domains": ["example.com"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "avatar_url": "https://...",
    --   "bio": "Technical writer",
    --   "preferences": { "theme": "dark", "editor_font_size": 14 },
    --   "notification_prefs": { "email": true, "slack": false }
    -- }
    password_hash   TEXT,
    email_verified  BOOLEAN NOT NULL DEFAULT false,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    team_ids        UUID[] NOT NULL DEFAULT '{}',   -- PostgreSQL array of team UUIDs
    permissions     JSONB NOT NULL DEFAULT '{}',
    -- permissions example:
    -- {
    --   "spaces": { "<space_id>": "admin" },
    --   "global": ["manage_members", "manage_billing"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, user_id)
);
CREATE INDEX idx_memberships_org ON memberships(org_id);
CREATE INDEX idx_memberships_user ON memberships(user_id);
CREATE INDEX idx_memberships_teams ON memberships USING gin(team_ids);
```

## Content Spaces

```sql
CREATE TABLE spaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    visibility      VARCHAR(20) NOT NULL DEFAULT 'private',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "description": "API documentation for Acme Corp",
    --   "default_lang": "en",
    --   "custom_domain": "docs.acme.com",
    --   "theme": { "primary_color": "#0f172a", "logo_url": "..." },
    --   "meta_schema": { ... },   -- JSON Schema for page metadata validation
    --   "navigation": { "style": "sidebar", "collapsed_by_default": false },
    --   "seo": { "og_image": "...", "twitter_handle": "@acme" },
    --   "git": {
    --     "provider": "github",
    --     "repo": "acme/docs",
    --     "branch": "main",
    --     "sync_direction": "bidirectional"
    --   },
    --   "integrations": {
    --     "slack": { "channel_id": "C123", "notify_on": ["publish", "review"] },
    --     "analytics": { "google_tag": "G-XXXXX" }
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);
CREATE INDEX idx_spaces_org ON spaces(org_id);
CREATE INDEX idx_spaces_config ON spaces USING gin(config);

CREATE TABLE space_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    version_label   VARCHAR(50) NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    meta            JSONB NOT NULL DEFAULT '{}',
    -- meta example:
    -- { "git_branch": "release/v2.1", "release_notes": "..." }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (space_id, version_label)
);
CREATE INDEX idx_space_versions_space ON space_versions(space_id);
```

## Pages (Content Tree)

```sql
CREATE TABLE pages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_version_id UUID NOT NULL REFERENCES space_versions(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES pages(id) ON DELETE CASCADE,
    slug            VARCHAR(255) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    content         JSONB NOT NULL DEFAULT '{}',
    -- content example:
    -- {
    --   "body": "# Getting Started\n\nWelcome to...",
    --   "format": "mdx",
    --   "excerpt": "A quick introduction to...",
    --   "frontmatter": { "sidebar_label": "Quick Start", "hide_title": false }
    -- }
    meta            JSONB NOT NULL DEFAULT '{}',
    -- meta example (varies by page type):
    -- For tutorials:  { "diataxis_type": "tutorial", "difficulty": "beginner", "estimated_minutes": 15 }
    -- For API refs:   { "diataxis_type": "reference", "api_spec_id": "...", "method": "POST", "path": "/users" }
    -- For changelogs: { "diataxis_type": "explanation", "release_version": "2.1.0", "release_date": "2026-05-01" }
    translations    JSONB NOT NULL DEFAULT '{}',
    -- translations example:
    -- {
    --   "fr": { "title": "Démarrage rapide", "body": "# Démarrage...", "translated_by": "ai", "reviewed": false },
    --   "de": { "title": "Schnellstart", "body": "# Schnellstart...", "translated_by": "human", "reviewed": true }
    -- }
    is_published    BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    created_by      UUID REFERENCES users(id),
    updated_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (space_version_id, slug)
);
CREATE INDEX idx_pages_space_version ON pages(space_version_id);
CREATE INDEX idx_pages_parent ON pages(parent_id);
CREATE INDEX idx_pages_meta ON pages USING gin(meta);
CREATE INDEX idx_pages_published ON pages(space_version_id) WHERE is_published = true;
```

## Page Revisions (History)

```sql
CREATE TABLE page_revisions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id         UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    revision_number INTEGER NOT NULL,
    content         JSONB NOT NULL,             -- snapshot of pages.content at this revision
    meta            JSONB NOT NULL,             -- snapshot of pages.meta at this revision
    title           VARCHAR(500) NOT NULL,
    commit_message  TEXT,
    author_id       UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (page_id, revision_number)
);
CREATE INDEX idx_page_revisions_page ON page_revisions(page_id);
```

## API Specifications

```sql
CREATE TABLE api_specs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_version_id UUID NOT NULL REFERENCES space_versions(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    raw_content     TEXT NOT NULL,
    parsed          JSONB NOT NULL DEFAULT '{}',
    -- parsed example:
    -- {
    --   "format": "openapi_3_1",
    --   "info": { "title": "Acme API", "version": "2.1.0" },
    --   "servers": [{ "url": "https://api.acme.com/v2" }],
    --   "endpoints": [
    --     { "method": "GET", "path": "/users", "operation_id": "listUsers", "tags": ["Users"], "summary": "List all users" },
    --     { "method": "POST", "path": "/users", "operation_id": "createUser", "tags": ["Users"], "summary": "Create a user" }
    --   ],
    --   "schemas": { "User": { "type": "object", "properties": {...} } },
    --   "auth_schemes": ["bearer", "api_key"]
    -- }
    source_url      TEXT,
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_api_specs_space_version ON api_specs(space_version_id);
CREATE INDEX idx_api_specs_parsed ON api_specs USING gin(parsed);
```

## Collaboration

```sql
CREATE TABLE change_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_version_id UUID NOT NULL REFERENCES space_versions(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    author_id       UUID NOT NULL REFERENCES users(id),
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "description": "Updated authentication guide...",
    --   "pages": ["<page_id_1>", "<page_id_2>"],
    --   "diffs": { "<page_id_1>": "unified diff text..." },
    --   "reviewers": [{ "user_id": "...", "decision": "approved", "reviewed_at": "..." }],
    --   "comments": [
    --     { "id": "...", "author_id": "...", "body": "Looks good!", "created_at": "...", "resolved": false }
    --   ]
    -- }
    merged_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_change_requests_space ON change_requests(space_version_id);
CREATE INDEX idx_change_requests_status ON change_requests(status);

CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target_type     VARCHAR(20) NOT NULL,       -- page, change_request
    target_id       UUID NOT NULL,
    parent_id       UUID REFERENCES comments(id) ON DELETE CASCADE,
    author_id       UUID NOT NULL REFERENCES users(id),
    body            TEXT NOT NULL,
    resolved        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_comments_target ON comments(target_type, target_id);
```

## Assets

```sql
CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    filename        VARCHAR(500) NOT NULL,
    content_type    VARCHAR(100) NOT NULL,
    size_bytes      BIGINT NOT NULL,
    storage_key     TEXT NOT NULL,
    meta            JSONB NOT NULL DEFAULT '{}',
    -- meta example:
    -- { "alt_text": "Architecture diagram", "width": 1200, "height": 800, "uploaded_by": "..." }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_assets_space ON assets(space_id);
```

## AI & Search

```sql
CREATE TABLE embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id         UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    chunk_index     INTEGER NOT NULL,
    chunk_text      TEXT NOT NULL,
    embedding       vector(1536),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (page_id, chunk_index)
);
CREATE INDEX idx_embeddings_vector ON embeddings USING ivfflat (embedding vector_cosine_ops);

CREATE TABLE ai_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),
    messages        JSONB NOT NULL DEFAULT '[]',
    -- messages example:
    -- [
    --   { "role": "user", "content": "How do I authenticate?", "at": "2026-05-20T10:00:00Z" },
    --   { "role": "assistant", "content": "To authenticate...", "sources": ["<page_id>"], "at": "2026-05-20T10:00:01Z" }
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ai_sessions_space ON ai_sessions(space_id);

CREATE TABLE drift_alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    page_id         UUID REFERENCES pages(id) ON DELETE SET NULL,
    details         JSONB NOT NULL,
    -- details example:
    -- {
    --   "commit_sha": "abc123",
    --   "drift_type": "api_change",
    --   "severity": "high",
    --   "description": "POST /users endpoint added 'department' field not in docs",
    --   "suggested_fix": "Add 'department' field to User schema documentation",
    --   "code_diff_url": "https://github.com/acme/api/commit/abc123"
    -- }
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    resolved_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);
CREATE INDEX idx_drift_alerts_space ON drift_alerts(space_id);
CREATE INDEX idx_drift_alerts_status ON drift_alerts(status);
```

## Analytics & Audit

```sql
-- Partitioned by month for efficient time-range queries and data retention
CREATE TABLE events_log (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    actor_id        UUID,
    target_type     VARCHAR(50),
    target_id       UUID,
    data            JSONB NOT NULL DEFAULT '{}',
    -- data examples:
    -- page.viewed:    { "page_id": "...", "referrer": "https://google.com", "user_agent": "..." }
    -- page.published: { "page_id": "...", "title": "Getting Started" }
    -- search.query:   { "space_id": "...", "query": "authentication", "results": 12, "clicked": "..." }
    -- user.login:     { "ip": "1.2.3.4", "method": "sso" }
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE events_log_2026_05 PARTITION OF events_log
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE events_log_2026_06 PARTITION OF events_log
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

CREATE INDEX idx_events_log_org ON events_log(org_id);
CREATE INDEX idx_events_log_type ON events_log(event_type);
CREATE INDEX idx_events_log_created ON events_log(created_at);
CREATE INDEX idx_events_log_target ON events_log(target_type, target_id);
```

## Webhooks

```sql
CREATE TABLE webhooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    config          JSONB NOT NULL,
    -- config example:
    -- {
    --   "url": "https://hooks.acme.com/docs",
    --   "events": ["page.published", "change_request.merged"],
    --   "secret_hash": "...",
    --   "active": true,
    --   "retry_policy": { "max_attempts": 3, "backoff_seconds": [10, 60, 300] }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_webhooks_org ON webhooks(org_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity | 3 | organizations, users, memberships |
| Content Spaces | 2 | spaces, space_versions |
| Pages & History | 2 | pages, page_revisions |
| API Documentation | 1 | api_specs (endpoints live in parsed JSONB) |
| Collaboration | 2 | change_requests, comments |
| Assets | 1 | assets |
| AI & Search | 3 | embeddings, ai_sessions, drift_alerts |
| Analytics & Audit | 1 | events_log (partitioned) |
| Webhooks | 1 | webhooks |
| **Total** | **16** | Plus partition tables for events_log |

---

## Key Design Decisions

1. **JSONB for variable metadata** — `pages.meta`, `pages.content`, `spaces.config`, and `organizations.settings` use JSONB columns. This means a tutorial page and an API reference page can have completely different metadata without schema changes. GIN indexes make containment queries fast.

2. **Translations inside the page row** — Rather than a separate translations table, `pages.translations` stores a JSONB object keyed by language code. This keeps a page and all its translations in a single row, reducing join count. Trade-off: updating one translation rewrites the entire JSONB column.

3. **API endpoints parsed into JSONB, not separate table** — `api_specs.parsed` contains the full parsed spec as JSONB, including endpoints array. This avoids a separate `api_endpoints` table and keeps spec data self-contained. Endpoint search uses JSONB containment: `parsed->'endpoints' @> '[{"method": "POST"}]'`.

4. **Unified events_log replaces separate analytics tables** — Page views, search queries, audit actions, and user activities all flow into a single partitioned `events_log` table. Monthly partitions enable efficient retention policies (drop old partitions) and time-range queries.

5. **Permissions embedded in memberships** — Rather than a separate permissions table, `memberships.permissions` JSONB stores per-space and global permissions. This reduces join count for the most common query ("does user X have permission Y on space Z?").

6. **AI sessions store conversation in JSONB array** — Rather than separate `ai_conversations` + `ai_messages` tables, a single `ai_sessions` row contains the full message array. Conversations are typically short-lived and read/written as a unit.

7. **16 tables vs 33 in the normalized model** — Nearly half the table count, achieved by consolidating variable data into JSONB columns. This dramatically reduces migration burden during rapid development.

### Example: Query Pages by Diataxis Type

```sql
-- Find all published tutorials in a space version
SELECT id, title, slug, meta->>'difficulty' AS difficulty
FROM pages
WHERE space_version_id = $1
  AND is_published = true
  AND meta @> '{"diataxis_type": "tutorial"}'
ORDER BY sort_order;
```

### Example: Search API Endpoints Across Specs

```sql
-- Find all POST endpoints across all specs in a space version
SELECT s.name AS spec_name, endpoint->>'path' AS path, endpoint->>'summary' AS summary
FROM api_specs s,
     jsonb_array_elements(s.parsed->'endpoints') AS endpoint
WHERE s.space_version_id = $1
  AND endpoint->>'method' = 'POST';
```

### Example: Get Space with Git Config

```sql
-- No join needed — config is in the same row
SELECT id, name, slug,
       config->'git'->>'provider' AS git_provider,
       config->'git'->>'repo' AS git_repo,
       config->'git'->>'branch' AS git_branch
FROM spaces
WHERE org_id = $1;
```
