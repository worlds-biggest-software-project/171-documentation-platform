# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Documentation Platform · Created: 2026-05-20

## Philosophy

This model follows classic third-normal-form relational design. Every concept gets its own table with explicit foreign keys, junction tables for many-to-many relationships, and lookup/reference tables for controlled vocabularies. The content tree uses an adjacency list with a `parent_id` self-reference, and versioning is handled through a dedicated `page_revisions` table that stores every historical snapshot of page content.

This is the approach used by mature CMS platforms like WordPress (wp_posts, wp_postmeta, wp_terms) and Confluence (content, spaces, bodycontent). It prioritises data integrity, query flexibility, and compatibility with standard ORM tooling. Every relationship is explicit and enforceable at the database level.

**Best for:** Teams that value data integrity, need complex cross-entity reporting, and prefer well-understood relational patterns with mature tooling support.

**Trade-offs:**
- Pro: Strong referential integrity — cascading deletes and foreign key constraints prevent orphaned data
- Pro: Easy to reason about — every entity has a clear table, every relationship a clear join
- Pro: Works with any ORM (Prisma, Drizzle, TypeORM, SQLAlchemy)
- Pro: Standard PostgreSQL — no extensions or exotic features required
- Con: High table count (~35-40 tables) increases migration complexity
- Con: Content tree queries (ancestors, descendants) require recursive CTEs
- Con: Schema changes for new page metadata fields require migrations
- Con: Multi-language content doubles the row count in content tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenAPI 3.1 | `api_specs` table stores raw OpenAPI YAML/JSON; parsed endpoints stored in `api_endpoints` for indexed search |
| CommonMark / MDX | `page_revisions.body` stores raw MDX content; `page_revisions.body_format` tracks format variant |
| Diátaxis Framework | `pages.diataxis_type` enum column categorises content as tutorial / how-to / reference / explanation |
| ISO 639-1 | `languages.code` uses two-letter language codes for i18n |
| OAuth 2.0 / OIDC | `sso_connections` table stores IdP configuration per organisation |
| SCIM 2.0 | `scim_tokens` table enables automated user provisioning |
| WCAG 2.2 | `accessibility_audits` table tracks automated accessibility scan results per page |
| RFC 7763 | Content served with `text/markdown` MIME type tracked in `page_revisions.body_format` |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    logo_url        TEXT,
    billing_plan    VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    password_hash   TEXT,                       -- NULL for SSO-only users
    email_verified  BOOLEAN NOT NULL DEFAULT false,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE org_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- owner, admin, member, viewer
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, user_id)
);
CREATE INDEX idx_org_memberships_org ON org_memberships(org_id);
CREATE INDEX idx_org_memberships_user ON org_memberships(user_id);

CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE TABLE team_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, user_id)
);
```

## Authentication & SSO

```sql
CREATE TABLE sso_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    provider        VARCHAR(50) NOT NULL,       -- saml, oidc, github, google
    config          JSONB NOT NULL,             -- IdP metadata, client IDs, etc.
    enabled         BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id) ON DELETE SET NULL,
    name            VARCHAR(255) NOT NULL,
    token_hash      TEXT NOT NULL UNIQUE,
    scopes          TEXT[] NOT NULL DEFAULT '{}',
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_api_tokens_org ON api_tokens(org_id);
```

## Content Spaces & Pages

```sql
CREATE TABLE spaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    visibility      VARCHAR(20) NOT NULL DEFAULT 'private',  -- public, private, internal
    default_lang    VARCHAR(10) NOT NULL DEFAULT 'en',
    custom_domain   VARCHAR(255),
    favicon_url     TEXT,
    theme_config    JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);
CREATE INDEX idx_spaces_org ON spaces(org_id);
CREATE INDEX idx_spaces_custom_domain ON spaces(custom_domain) WHERE custom_domain IS NOT NULL;

CREATE TABLE space_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    version_label   VARCHAR(50) NOT NULL,       -- e.g., "v2.1", "latest"
    git_branch      VARCHAR(255),               -- linked Git branch
    is_default      BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (space_id, version_label)
);
CREATE INDEX idx_space_versions_space ON space_versions(space_id);

CREATE TABLE pages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_version_id UUID NOT NULL REFERENCES space_versions(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES pages(id) ON DELETE CASCADE,
    slug            VARCHAR(255) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    page_type       VARCHAR(50) NOT NULL DEFAULT 'content',  -- content, api_reference, changelog, link
    diataxis_type   VARCHAR(20),                -- tutorial, how_to, reference, explanation
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (space_version_id, slug)
);
CREATE INDEX idx_pages_space_version ON pages(space_version_id);
CREATE INDEX idx_pages_parent ON pages(parent_id);
CREATE INDEX idx_pages_slug ON pages(space_version_id, slug);

CREATE TABLE page_revisions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id         UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    revision_number INTEGER NOT NULL,
    body            TEXT NOT NULL,               -- MDX/Markdown content
    body_format     VARCHAR(20) NOT NULL DEFAULT 'mdx',  -- mdx, markdown, rst
    commit_message  TEXT,
    author_id       UUID REFERENCES users(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (page_id, revision_number)
);
CREATE INDEX idx_page_revisions_page ON page_revisions(page_id);
CREATE INDEX idx_page_revisions_created ON page_revisions(created_at);
```

## Translations (i18n)

```sql
CREATE TABLE languages (
    code            VARCHAR(10) PRIMARY KEY,    -- ISO 639-1: en, fr, de, ja
    name            VARCHAR(100) NOT NULL,
    native_name     VARCHAR(100)
);

CREATE TABLE page_translations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id         UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    language_code   VARCHAR(10) NOT NULL REFERENCES languages(code),
    title           VARCHAR(500) NOT NULL,
    body            TEXT NOT NULL,
    body_format     VARCHAR(20) NOT NULL DEFAULT 'mdx',
    translated_by   VARCHAR(20) NOT NULL DEFAULT 'human',  -- human, ai
    reviewed        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (page_id, language_code)
);
CREATE INDEX idx_page_translations_page ON page_translations(page_id);
```

## API Documentation

```sql
CREATE TABLE api_specs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_version_id UUID NOT NULL REFERENCES space_versions(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    spec_format     VARCHAR(20) NOT NULL,       -- openapi_3_0, openapi_3_1, asyncapi_3_0, graphql_sdl
    raw_content     TEXT NOT NULL,               -- original YAML/JSON/SDL
    parsed_metadata JSONB,                      -- extracted info, servers, auth schemes
    source_url      TEXT,                        -- if imported from URL
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_api_specs_space_version ON api_specs(space_version_id);

CREATE TABLE api_endpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_spec_id     UUID NOT NULL REFERENCES api_specs(id) ON DELETE CASCADE,
    method          VARCHAR(10) NOT NULL,        -- GET, POST, PUT, DELETE, PATCH
    path            VARCHAR(1000) NOT NULL,
    operation_id    VARCHAR(255),
    summary         TEXT,
    description     TEXT,
    tags            TEXT[] NOT NULL DEFAULT '{}',
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    request_schema  JSONB,
    response_schemas JSONB,                     -- keyed by status code
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_api_endpoints_spec ON api_endpoints(api_spec_id);
CREATE INDEX idx_api_endpoints_tags ON api_endpoints USING gin(tags);
```

## Git Integration

```sql
CREATE TABLE git_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    provider        VARCHAR(20) NOT NULL,       -- github, gitlab, bitbucket
    repo_url        TEXT NOT NULL,
    branch          VARCHAR(255) NOT NULL DEFAULT 'main',
    sync_direction  VARCHAR(20) NOT NULL DEFAULT 'bidirectional',  -- to_git, from_git, bidirectional
    access_token_encrypted TEXT NOT NULL,
    webhook_secret_encrypted TEXT,
    last_sync_at    TIMESTAMPTZ,
    sync_status     VARCHAR(20) NOT NULL DEFAULT 'idle',  -- idle, syncing, error
    sync_error      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_git_connections_space ON git_connections(space_id);

CREATE TABLE git_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    git_connection_id UUID NOT NULL REFERENCES git_connections(id) ON DELETE CASCADE,
    direction       VARCHAR(10) NOT NULL,       -- push, pull
    commit_sha      VARCHAR(40),
    files_changed   INTEGER NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL,       -- success, error
    error_message   TEXT,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);
CREATE INDEX idx_git_sync_log_connection ON git_sync_log(git_connection_id);
```

## Collaboration & Reviews

```sql
CREATE TABLE change_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_version_id UUID NOT NULL REFERENCES space_versions(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',  -- open, in_review, approved, merged, closed
    author_id       UUID NOT NULL REFERENCES users(id),
    reviewer_id     UUID REFERENCES users(id),
    merged_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_change_requests_space_version ON change_requests(space_version_id);
CREATE INDEX idx_change_requests_status ON change_requests(status);

CREATE TABLE change_request_pages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    change_request_id UUID NOT NULL REFERENCES change_requests(id) ON DELETE CASCADE,
    page_id         UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    diff_body       TEXT NOT NULL,              -- unified diff of changes
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id         UUID REFERENCES pages(id) ON DELETE CASCADE,
    change_request_id UUID REFERENCES change_requests(id) ON DELETE CASCADE,
    author_id       UUID NOT NULL REFERENCES users(id),
    parent_id       UUID REFERENCES comments(id) ON DELETE CASCADE,
    body            TEXT NOT NULL,
    resolved        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    CHECK (page_id IS NOT NULL OR change_request_id IS NOT NULL)
);
CREATE INDEX idx_comments_page ON comments(page_id);
CREATE INDEX idx_comments_change_request ON comments(change_request_id);
```

## Media & Assets

```sql
CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    filename        VARCHAR(500) NOT NULL,
    content_type    VARCHAR(100) NOT NULL,
    size_bytes      BIGINT NOT NULL,
    storage_key     TEXT NOT NULL,              -- S3/R2 object key
    alt_text        TEXT,
    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_assets_space ON assets(space_id);
```

## AI Features

```sql
CREATE TABLE drift_detections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    page_id         UUID REFERENCES pages(id) ON DELETE SET NULL,
    commit_sha      VARCHAR(40) NOT NULL,
    drift_type      VARCHAR(50) NOT NULL,       -- api_change, code_change, dependency_update
    severity        VARCHAR(20) NOT NULL,       -- low, medium, high, critical
    description     TEXT NOT NULL,
    suggested_fix   TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',  -- open, acknowledged, fixed, dismissed
    resolved_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);
CREATE INDEX idx_drift_detections_space ON drift_detections(space_id);
CREATE INDEX idx_drift_detections_status ON drift_detections(status);

CREATE TABLE ai_conversations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id),
    session_id      VARCHAR(100) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ai_messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES ai_conversations(id) ON DELETE CASCADE,
    role            VARCHAR(20) NOT NULL,       -- user, assistant
    content         TEXT NOT NULL,
    sources         JSONB,                     -- referenced page IDs and snippets
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ai_messages_conversation ON ai_messages(conversation_id);

CREATE TABLE embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id         UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    chunk_index     INTEGER NOT NULL,
    chunk_text      TEXT NOT NULL,
    embedding       vector(1536),              -- pgvector; dimension depends on model
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (page_id, chunk_index)
);
CREATE INDEX idx_embeddings_page ON embeddings(page_id);
CREATE INDEX idx_embeddings_vector ON embeddings USING ivfflat (embedding vector_cosine_ops);
```

## Search & Analytics

```sql
CREATE TABLE search_index (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id         UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    space_version_id UUID NOT NULL REFERENCES space_versions(id) ON DELETE CASCADE,
    search_vector   tsvector NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_search_index_vector ON search_index USING gin(search_vector);
CREATE INDEX idx_search_index_page ON search_index(page_id);

CREATE TABLE page_views (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id         UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    visitor_id      VARCHAR(100),              -- anonymous hash or user ID
    user_id         UUID REFERENCES users(id),
    referrer        TEXT,
    user_agent      TEXT,
    viewed_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_page_views_page ON page_views(page_id);
CREATE INDEX idx_page_views_viewed_at ON page_views(viewed_at);

CREATE TABLE search_queries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    query_text      TEXT NOT NULL,
    results_count   INTEGER NOT NULL DEFAULT 0,
    clicked_page_id UUID REFERENCES pages(id),
    user_id         UUID REFERENCES users(id),
    searched_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_search_queries_space ON search_queries(space_id);
CREATE INDEX idx_search_queries_searched_at ON search_queries(searched_at);

CREATE TABLE feedback (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id         UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    rating          SMALLINT CHECK (rating BETWEEN 1 AND 5),
    comment         TEXT,
    user_id         UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_feedback_page ON feedback(page_id);
```

## Permissions

```sql
CREATE TABLE space_permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    grantee_type    VARCHAR(20) NOT NULL,       -- user, team, org
    grantee_id      UUID NOT NULL,
    permission      VARCHAR(30) NOT NULL,       -- read, write, admin, publish
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (space_id, grantee_type, grantee_id, permission)
);
CREATE INDEX idx_space_permissions_space ON space_permissions(space_id);
CREATE INDEX idx_space_permissions_grantee ON space_permissions(grantee_type, grantee_id);
```

## Integrations & Webhooks

```sql
CREATE TABLE webhooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    secret_encrypted TEXT NOT NULL,
    events          TEXT[] NOT NULL,            -- page.published, page.updated, etc.
    active          BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE slack_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    workspace_id    VARCHAR(100) NOT NULL,
    channel_id      VARCHAR(100) NOT NULL,
    access_token_encrypted TEXT NOT NULL,
    notify_on       TEXT[] NOT NULL DEFAULT '{page.published,change_request.merged}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    actor_id        UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,      -- page.created, user.invited, space.deleted
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_log_org ON audit_log(org_id);
CREATE INDEX idx_audit_log_created ON audit_log(created_at);
CREATE INDEX idx_audit_log_action ON audit_log(action);
CREATE INDEX idx_audit_log_resource ON audit_log(resource_type, resource_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 5 | organizations, users, org_memberships, teams, team_memberships |
| Authentication & SSO | 2 | sso_connections, api_tokens |
| Content Spaces & Pages | 4 | spaces, space_versions, pages, page_revisions |
| Translations | 2 | languages, page_translations |
| API Documentation | 2 | api_specs, api_endpoints |
| Git Integration | 2 | git_connections, git_sync_log |
| Collaboration | 3 | change_requests, change_request_pages, comments |
| Media | 1 | assets |
| AI Features | 4 | drift_detections, ai_conversations, ai_messages, embeddings |
| Search & Analytics | 4 | search_index, page_views, search_queries, feedback |
| Permissions | 1 | space_permissions |
| Integrations & Audit | 3 | webhooks, slack_connections, audit_log |
| **Total** | **33** | |

---

## Key Design Decisions

1. **Adjacency list for page hierarchy** — `pages.parent_id` is simple and well-understood. Tree queries use recursive CTEs, which PostgreSQL handles efficiently for typical documentation trees (rarely deeper than 5-6 levels).

2. **Separate page_revisions table** — Every edit creates a new row in `page_revisions`, providing full history without temporal table complexity. The current revision is the highest `revision_number` for a given `page_id`.

3. **Space versions decouple doc versions from content** — A space can have multiple versions (v1.0, v2.0, latest), each with its own page tree. This mirrors how GitBook and Mintlify handle versioned documentation.

4. **pgvector for semantic search** — The `embeddings` table uses the pgvector extension for AI-powered semantic search, keeping the vector store collocated with relational data for transactional consistency.

5. **Polymorphic permissions with grantee_type** — Rather than separate user_permissions and team_permissions tables, a single `space_permissions` table uses `grantee_type` to distinguish between users, teams, and entire organisations.

6. **Audit log as append-only** — The `audit_log` table has no UPDATE or DELETE expected — it is insert-only for compliance and forensic purposes.

7. **Translations as separate rows, not columns** — `page_translations` stores one row per language per page, rather than adding `title_fr`, `body_de` columns. This scales to any number of languages without schema changes.

8. **OpenAPI specs stored raw plus parsed** — `api_specs.raw_content` preserves the original spec for re-rendering, while `api_endpoints` provides indexed, searchable individual endpoints.

### Example: Recursive CTE for Page Tree

```sql
-- Fetch the full page tree for a space version
WITH RECURSIVE page_tree AS (
    SELECT id, parent_id, title, slug, sort_order, 0 AS depth,
           ARRAY[sort_order] AS path
    FROM pages
    WHERE space_version_id = $1 AND parent_id IS NULL
    UNION ALL
    SELECT p.id, p.parent_id, p.title, p.slug, p.sort_order, pt.depth + 1,
           pt.path || p.sort_order
    FROM pages p
    JOIN page_tree pt ON p.parent_id = pt.id
)
SELECT * FROM page_tree ORDER BY path;
```
