# Data Model Suggestion 4: Document-Oriented with Content Trees

> Project: Documentation Platform · Created: 2026-05-20

## Philosophy

This model treats each documentation page as a rich, self-contained document with deeply nested structure — content blocks, metadata, translations, and rendering hints all stored within the page record itself. The page content uses a block-based representation (similar to Notion's block model or ProseMirror/TipTap's document tree) rather than flat Markdown text. This enables fine-grained collaborative editing, block-level comments, block-level translations, and content reuse through block references.

The approach is inspired by Notion (blocks as first-class entities), Sanity.io (structured content as portable JSON), and the Portable Text specification. Rather than storing MDX as a text blob that must be parsed on every render, content is stored as a structured tree of typed blocks — paragraphs, headings, code blocks, callouts, API playgrounds, embedded diagrams — each with its own metadata and rendering properties.

This model uses PostgreSQL but leverages JSONB heavily for the block tree, combined with a materialised path pattern (ltree) for the page hierarchy. The block structure enables features that flat Markdown cannot: block-level permissions, block-level analytics ("which code example do users copy most?"), and granular real-time collaboration (two users editing different blocks simultaneously).

**Best for:** Platforms that want a rich block-based editor (like Notion), real-time collaboration, block-level features (comments, analytics, reuse), and structured content that can be rendered to multiple output formats.

**Trade-offs:**
- Pro: Block-level granularity enables rich editor features and fine-grained analytics
- Pro: Content reuse — reference a block from another page without duplication
- Pro: ltree hierarchy queries are faster than recursive CTEs for deep trees
- Pro: Structured content is portable — render to HTML, PDF, or feed to AI without parsing Markdown
- Pro: Real-time collaboration at block level (no merge conflicts on same-page edits)
- Con: More complex content model — developers must understand the block tree structure
- Con: Importing/exporting Markdown requires block serialisation/deserialisation
- Con: Block-level storage increases row count significantly (hundreds of blocks per page)
- Con: ltree extension required — not available on all PostgreSQL hosting providers
- Con: Larger storage footprint than flat Markdown text

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenAPI 3.1 | `api_specs` stores raw spec; individual endpoints rendered as `api_playground` block type |
| CommonMark / MDX | Import/export layer converts between MDX and block tree; `blocks.properties` stores MDX-specific hints |
| Diátaxis Framework | `pages.diataxis_type` column for content categorisation |
| Portable Text | Block structure inspired by Sanity's Portable Text specification for structured rich content |
| ISO 639-1 | Language codes in `block_translations` rows |
| Mermaid / PlantUML | Dedicated `diagram` block type with `properties.syntax` = mermaid or plantuml |
| WCAG 2.2 | `image` blocks require `properties.alt_text`; heading blocks enforce hierarchy |

---

## Core Identity & Multi-Tenancy

```sql
CREATE EXTENSION IF NOT EXISTS ltree;
CREATE EXTENSION IF NOT EXISTS vector;

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
    password_hash   TEXT,
    preferences     JSONB NOT NULL DEFAULT '{}',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, user_id)
);
CREATE INDEX idx_memberships_org ON memberships(org_id);
CREATE INDEX idx_memberships_user ON memberships(user_id);
```

## Spaces & Page Hierarchy (ltree)

```sql
CREATE TABLE spaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    visibility      VARCHAR(20) NOT NULL DEFAULT 'private',
    default_lang    VARCHAR(10) NOT NULL DEFAULT 'en',
    custom_domain   VARCHAR(255),
    theme_config    JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);
CREATE INDEX idx_spaces_org ON spaces(org_id);

CREATE TABLE space_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    version_label   VARCHAR(50) NOT NULL,
    git_branch      VARCHAR(255),
    is_default      BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (space_id, version_label)
);
CREATE INDEX idx_space_versions_space ON space_versions(space_id);

CREATE TABLE pages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_version_id UUID NOT NULL REFERENCES space_versions(id) ON DELETE CASCADE,
    slug            VARCHAR(255) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    path            ltree NOT NULL,             -- e.g., 'getting_started.authentication.oauth'
    sort_order      INTEGER NOT NULL DEFAULT 0,
    page_type       VARCHAR(50) NOT NULL DEFAULT 'content',
    diataxis_type   VARCHAR(20),
    is_published    BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    meta            JSONB NOT NULL DEFAULT '{}',
    created_by      UUID REFERENCES users(id),
    updated_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (space_version_id, slug)
);
CREATE INDEX idx_pages_space_version ON pages(space_version_id);
CREATE INDEX idx_pages_path ON pages USING gist(path);
CREATE INDEX idx_pages_published ON pages(space_version_id) WHERE is_published = true;
```

## Block-Based Content Model

```sql
-- Each block is a content element within a page: paragraph, heading, code, callout, etc.
CREATE TABLE blocks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id         UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    parent_block_id UUID REFERENCES blocks(id) ON DELETE CASCADE,  -- for nested blocks (list items, columns)
    block_type      VARCHAR(50) NOT NULL,
    -- Block types:
    --   paragraph, heading, code, blockquote, callout,
    --   image, video, diagram, table, list, list_item,
    --   api_playground, embed, divider, toggle, columns, column,
    --   tab_group, tab, snippet_ref
    sort_order      INTEGER NOT NULL DEFAULT 0,
    content         TEXT,                       -- text content for text-based blocks (paragraph, heading, etc.)
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties examples by block type:
    --
    -- heading:       { "level": 2, "anchor_id": "authentication" }
    -- code:          { "language": "python", "filename": "auth.py", "highlight_lines": [3, 5], "runnable": true }
    -- callout:       { "type": "warning", "title": "Deprecation Notice" }
    -- image:         { "src": "assets/diagram.png", "alt_text": "Auth flow", "width": 800, "caption": "Figure 1" }
    -- diagram:       { "syntax": "mermaid", "source": "graph TD\n  A-->B" }
    -- api_playground:{ "spec_id": "...", "method": "POST", "path": "/users", "default_body": "{...}" }
    -- table:         { "headers": ["Field", "Type", "Required"], "rows": [["name", "string", "yes"], ...] }
    -- snippet_ref:   { "source_block_id": "...", "source_page_id": "..." }  -- content reuse!
    -- toggle:        { "summary": "Click to expand" }
    -- embed:         { "url": "https://codesandbox.io/...", "height": 400 }
    --
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_blocks_page ON blocks(page_id, sort_order);
CREATE INDEX idx_blocks_parent ON blocks(parent_block_id);
CREATE INDEX idx_blocks_type ON blocks(block_type);

-- Inline marks within text blocks (bold, italic, link, code, etc.)
-- Stored as ranges within the block's text content
CREATE TABLE inline_marks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    block_id        UUID NOT NULL REFERENCES blocks(id) ON DELETE CASCADE,
    mark_type       VARCHAR(30) NOT NULL,       -- bold, italic, code, link, strikethrough, highlight
    start_offset    INTEGER NOT NULL,
    end_offset      INTEGER NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- properties examples:
    -- link:  { "href": "https://...", "title": "API Reference" }
    -- highlight: { "color": "yellow" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_inline_marks_block ON inline_marks(block_id);
```

## Block Translations

```sql
CREATE TABLE block_translations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    block_id        UUID NOT NULL REFERENCES blocks(id) ON DELETE CASCADE,
    language_code   VARCHAR(10) NOT NULL,       -- ISO 639-1
    content         TEXT,                       -- translated text content
    properties      JSONB NOT NULL DEFAULT '{}', -- translated properties (e.g., alt_text for images)
    translated_by   VARCHAR(20) NOT NULL DEFAULT 'human',
    reviewed        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (block_id, language_code)
);
CREATE INDEX idx_block_translations_block ON block_translations(block_id);
```

## Page Revisions (Snapshot-Based)

```sql
-- Stores periodic snapshots of the full block tree for a page
CREATE TABLE page_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id         UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    snapshot_number INTEGER NOT NULL,
    title           VARCHAR(500) NOT NULL,
    block_tree      JSONB NOT NULL,             -- serialised full block tree
    -- block_tree example:
    -- [
    --   { "id": "...", "type": "heading", "content": "Getting Started", "properties": { "level": 1 },
    --     "children": [] },
    --   { "id": "...", "type": "paragraph", "content": "Welcome to...", "properties": {},
    --     "children": [],
    --     "marks": [{ "type": "bold", "start": 0, "end": 7 }] },
    --   { "id": "...", "type": "code", "content": "curl -X POST...", "properties": { "language": "bash" },
    --     "children": [] }
    -- ]
    commit_message  TEXT,
    author_id       UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (page_id, snapshot_number)
);
CREATE INDEX idx_page_snapshots_page ON page_snapshots(page_id);
```

## API Specifications

```sql
CREATE TABLE api_specs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_version_id UUID NOT NULL REFERENCES space_versions(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    spec_format     VARCHAR(20) NOT NULL,
    raw_content     TEXT NOT NULL,
    parsed_metadata JSONB,
    source_url      TEXT,
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_api_specs_space_version ON api_specs(space_version_id);

CREATE TABLE api_endpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_spec_id     UUID NOT NULL REFERENCES api_specs(id) ON DELETE CASCADE,
    method          VARCHAR(10) NOT NULL,
    path            VARCHAR(1000) NOT NULL,
    operation_id    VARCHAR(255),
    summary         TEXT,
    tags            TEXT[] NOT NULL DEFAULT '{}',
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    request_schema  JSONB,
    response_schemas JSONB,
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
    provider        VARCHAR(20) NOT NULL,
    repo_url        TEXT NOT NULL,
    branch          VARCHAR(255) NOT NULL DEFAULT 'main',
    sync_direction  VARCHAR(20) NOT NULL DEFAULT 'bidirectional',
    access_token_encrypted TEXT NOT NULL,
    last_sync_at    TIMESTAMPTZ,
    sync_status     VARCHAR(20) NOT NULL DEFAULT 'idle',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_git_connections_space ON git_connections(space_id);
```

## Collaboration

```sql
-- Comments can target a specific block, not just a page
CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id         UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    block_id        UUID REFERENCES blocks(id) ON DELETE SET NULL,  -- block-level comment!
    parent_id       UUID REFERENCES comments(id) ON DELETE CASCADE,
    author_id       UUID NOT NULL REFERENCES users(id),
    body            TEXT NOT NULL,
    resolved        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_comments_page ON comments(page_id);
CREATE INDEX idx_comments_block ON comments(block_id) WHERE block_id IS NOT NULL;

CREATE TABLE change_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_version_id UUID NOT NULL REFERENCES space_versions(id) ON DELETE CASCADE,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    author_id       UUID NOT NULL REFERENCES users(id),
    reviewer_id     UUID REFERENCES users(id),
    merged_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_change_requests_space ON change_requests(space_version_id);
```

## AI, Search & Embeddings

```sql
-- Embeddings at the block level, not page level — more granular semantic search
CREATE TABLE block_embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    block_id        UUID NOT NULL REFERENCES blocks(id) ON DELETE CASCADE,
    page_id         UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    content_text    TEXT NOT NULL,               -- denormalised for search display
    embedding       vector(1536),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (block_id)
);
CREATE INDEX idx_block_embeddings_vector ON block_embeddings USING ivfflat (embedding vector_cosine_ops);
CREATE INDEX idx_block_embeddings_page ON block_embeddings(page_id);

CREATE TABLE search_index (
    page_id         UUID PRIMARY KEY REFERENCES pages(id) ON DELETE CASCADE,
    space_version_id UUID NOT NULL,
    search_vector   tsvector NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_search_vector ON search_index USING gin(search_vector);

CREATE TABLE drift_detections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    page_id         UUID REFERENCES pages(id) ON DELETE SET NULL,
    block_id        UUID REFERENCES blocks(id) ON DELETE SET NULL,  -- drift at block level!
    commit_sha      VARCHAR(40) NOT NULL,
    drift_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    description     TEXT NOT NULL,
    suggested_fix   TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);
CREATE INDEX idx_drift_space ON drift_detections(space_id);
CREATE INDEX idx_drift_status ON drift_detections(status);
```

## Analytics

```sql
-- Block-level analytics: which blocks are read, copied, or interacted with
CREATE TABLE block_analytics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    block_id        UUID NOT NULL REFERENCES blocks(id) ON DELETE CASCADE,
    page_id         UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    event_type      VARCHAR(50) NOT NULL,       -- view, copy, expand, run_code, try_api
    visitor_id      VARCHAR(100),
    user_id         UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_block_analytics_block ON block_analytics(block_id);
CREATE INDEX idx_block_analytics_page ON block_analytics(page_id);
CREATE INDEX idx_block_analytics_created ON block_analytics(created_at);

CREATE TABLE page_views (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id         UUID NOT NULL REFERENCES pages(id) ON DELETE CASCADE,
    visitor_id      VARCHAR(100),
    user_id         UUID REFERENCES users(id),
    referrer        TEXT,
    viewed_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_page_views_page ON page_views(page_id);
CREATE INDEX idx_page_views_viewed ON page_views(viewed_at);

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    actor_id        UUID,
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_log_org ON audit_log(org_id);
CREATE INDEX idx_audit_log_created ON audit_log(created_at);
```

## Permissions & Integrations

```sql
CREATE TABLE space_permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    grantee_type    VARCHAR(20) NOT NULL,
    grantee_id      UUID NOT NULL,
    permission      VARCHAR(30) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (space_id, grantee_type, grantee_id, permission)
);
CREATE INDEX idx_space_perms_space ON space_permissions(space_id);

CREATE TABLE webhooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    secret_encrypted TEXT NOT NULL,
    events          TEXT[] NOT NULL,
    active          BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL REFERENCES spaces(id) ON DELETE CASCADE,
    filename        VARCHAR(500) NOT NULL,
    content_type    VARCHAR(100) NOT NULL,
    size_bytes      BIGINT NOT NULL,
    storage_key     TEXT NOT NULL,
    alt_text        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_assets_space ON assets(space_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity | 3 | organizations, users, memberships |
| Content Spaces | 2 | spaces, space_versions |
| Page Hierarchy | 1 | pages (with ltree path) |
| Block Content | 3 | blocks, inline_marks, block_translations |
| Revisions | 1 | page_snapshots (serialised block tree) |
| API Documentation | 2 | api_specs, api_endpoints |
| Git Integration | 1 | git_connections |
| Collaboration | 2 | comments (block-level), change_requests |
| AI & Search | 3 | block_embeddings, search_index, drift_detections |
| Analytics | 3 | block_analytics, page_views, audit_log |
| Permissions & Integrations | 3 | space_permissions, webhooks, assets |
| **Total** | **24** | Block-level granularity adds tables vs. flat Markdown |

---

## Key Design Decisions

1. **ltree for page hierarchy** — The `pages.path` column uses PostgreSQL's ltree extension for materialised path queries. Finding all descendants of "Getting Started" is `WHERE path <@ 'getting_started'` — no recursive CTE needed. ltree supports ancestor, descendant, and pattern matching queries natively.

2. **Block-based content model** — Each content element (paragraph, heading, code block, callout) is a separate row in `blocks`. This enables block-level comments, block-level analytics, block-level translations, and block-level embeddings. The trade-off is higher row count.

3. **Content reuse via snippet_ref blocks** — A `snippet_ref` block type references a block from another page. When rendered, it pulls the current content of the referenced block. This enables DRY documentation — change a snippet once, and every page that references it updates automatically.

4. **Block-level embeddings for semantic search** — Rather than chunking page text into arbitrary segments, each block gets its own embedding vector. This produces more semantically coherent search results — the AI can point users to a specific code example or callout rather than a page section.

5. **Block-level drift detection** — The `drift_detections` table can reference a specific block (e.g., "the code example in block X no longer matches the source code"). This gives technical writers precise, actionable drift alerts.

6. **Block-level analytics** — `block_analytics` tracks which blocks users interact with: which code examples they copy, which toggles they expand, which API playground calls they make. This data is invaluable for understanding documentation effectiveness.

7. **Page snapshots for revision history** — Rather than versioning individual blocks (which would be extremely complex), `page_snapshots` stores periodic serialised snapshots of the entire block tree. This provides revision history without the complexity of per-block versioning.

8. **Inline marks as separate rows** — Bold, italic, link, and other inline formatting is stored in `inline_marks` with character offset ranges. This enables rich text rendering from structured data and is the pattern used by ProseMirror, TipTap, and Slate editors.

### Example: ltree Hierarchy Queries

```sql
-- Get all pages under "guides" section
SELECT id, title, slug, path
FROM pages
WHERE space_version_id = $1
  AND path <@ 'guides'
ORDER BY path;

-- Get direct children of a page
SELECT id, title, slug
FROM pages
WHERE space_version_id = $1
  AND path ~ 'guides.authentication.*{1}'
ORDER BY sort_order;

-- Get ancestors (breadcrumb)
SELECT id, title, slug, path
FROM pages
WHERE space_version_id = $1
  AND path @> 'guides.authentication.oauth'
ORDER BY nlevel(path);
```

### Example: Render a Page's Block Tree

```sql
-- Fetch all blocks for a page, ordered for rendering
WITH RECURSIVE block_tree AS (
    SELECT id, parent_block_id, block_type, content, properties, sort_order,
           0 AS depth, ARRAY[sort_order] AS render_path
    FROM blocks
    WHERE page_id = $1 AND parent_block_id IS NULL
    UNION ALL
    SELECT b.id, b.parent_block_id, b.block_type, b.content, b.properties, b.sort_order,
           bt.depth + 1, bt.render_path || b.sort_order
    FROM blocks b
    JOIN block_tree bt ON b.parent_block_id = bt.id
)
SELECT * FROM block_tree ORDER BY render_path;

-- Include inline marks for text blocks
SELECT m.mark_type, m.start_offset, m.end_offset, m.properties
FROM inline_marks m
WHERE m.block_id = $2
ORDER BY m.start_offset;
```

### Example: Block-Level Semantic Search

```sql
-- Find the most relevant blocks across all pages in a space version
SELECT be.block_id, be.page_id, be.content_text,
       p.title AS page_title, p.slug AS page_slug,
       b.block_type,
       1 - (be.embedding <=> $2) AS similarity
FROM block_embeddings be
JOIN pages p ON be.page_id = p.id
JOIN blocks b ON be.block_id = b.id
WHERE p.space_version_id = $1
  AND p.is_published = true
ORDER BY be.embedding <=> $2
LIMIT 10;
```
