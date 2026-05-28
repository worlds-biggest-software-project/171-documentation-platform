# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Documentation Platform · Created: 2026-05-20

## Philosophy

This model treats every change to every document as an immutable event in a central event store. The current state of any page, space, or permission is derived by replaying its event stream. Materialised read models (projections) are maintained for fast queries — a classic CQRS (Command Query Responsibility Segregation) pattern.

This approach is used by systems where complete audit trails are non-negotiable: financial ledgers, healthcare records, and legal document management. For a documentation platform, it provides perfect version history, point-in-time reconstruction ("what did this page look like on March 5th?"), and a natural foundation for AI-powered change analysis and drift detection.

The event store is the single source of truth. Read models are disposable projections that can be rebuilt from events at any time. This separation means you can add new read models (e.g., a search index, an analytics summary, a drift detection feed) without modifying the write path.

**Best for:** Platforms where full audit trails, temporal queries, and AI-driven change analysis are primary requirements. Ideal when compliance demands knowing exactly who changed what and when.

**Trade-offs:**
- Pro: Complete, immutable audit trail — every change is permanently recorded
- Pro: Point-in-time reconstruction — replay to any moment in history
- Pro: Natural fit for drift detection — events feed directly into AI analysis pipelines
- Pro: Read model flexibility — add new projections without changing the write path
- Pro: Undo/redo is trivial — just replay or skip events
- Con: Higher storage cost — events accumulate indefinitely plus read model duplication
- Con: Eventual consistency between write and read sides unless synchronous projections used
- Con: Increased complexity — developers must understand event replay, projections, and snapshotting
- Con: Debugging requires following event chains rather than inspecting current state
- Con: Schema evolution of events requires careful versioning (upcasting)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenAPI 3.1 | `ApiSpecUploaded` event stores raw spec; projection extracts endpoints into read model |
| CommonMark / MDX | Page content stored as event payload; format tracked in event metadata |
| Diátaxis Framework | `PageCategorised` event assigns Diátaxis type; queryable in page read model |
| ISO 639-1 | Language codes in `TranslationCreated` events |
| OAuth 2.0 / OIDC | `SsoConfigured` event captures IdP setup per organisation |
| ISO/IEC 27001 | Event store is inherently compliant — immutable, timestamped, actor-attributed |
| OCSF | Event structure aligns with Open Cybersecurity Schema Framework for audit events |

---

## Event Store (Source of Truth)

```sql
-- The single most important table in this model.
-- Every state change in the system is an event here.
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,              -- aggregate root ID (page, space, org, etc.)
    stream_type     VARCHAR(50) NOT NULL,        -- page, space, organization, user, api_spec
    event_type      VARCHAR(100) NOT NULL,       -- PageCreated, PageContentUpdated, SpacePublished, etc.
    event_version   INTEGER NOT NULL,            -- per-stream sequence number
    payload         JSONB NOT NULL,              -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}', -- actor_id, ip_address, correlation_id, causation_id
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)
);

-- Optimised for: replay by stream, replay by type, time-range queries
CREATE INDEX idx_events_stream ON events(stream_id, event_version);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_created ON events(created_at);
CREATE INDEX idx_events_stream_type ON events(stream_type);

-- Snapshots reduce replay cost for long-lived aggregates
CREATE TABLE snapshots (
    stream_id       UUID PRIMARY KEY,
    stream_type     VARCHAR(50) NOT NULL,
    event_version   INTEGER NOT NULL,           -- version at which snapshot was taken
    state           JSONB NOT NULL,             -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Event Type Catalogue

```
-- Page events
PageCreated              { space_version_id, parent_id, slug, title, page_type, diataxis_type }
PageContentUpdated       { body, body_format, commit_message }
PageTitleChanged         { old_title, new_title }
PageMoved                { old_parent_id, new_parent_id, old_sort_order, new_sort_order }
PagePublished            { }
PageUnpublished          { }
PageDeleted              { }
PageTranslationAdded     { language_code, title, body, translated_by }
PageTranslationUpdated   { language_code, title, body }

-- Space events
SpaceCreated             { org_id, name, slug, visibility }
SpaceSettingsUpdated     { changed_fields }
SpaceVersionCreated      { version_label, git_branch }
SpaceVersionPublished    { }
SpaceCustomDomainSet     { domain }

-- API spec events
ApiSpecUploaded          { space_version_id, name, format, raw_content, source_url }
ApiSpecUpdated           { raw_content }
ApiSpecDeleted           { }

-- Organization events
OrganizationCreated      { name, slug }
MemberInvited            { user_id, role }
MemberRoleChanged        { user_id, old_role, new_role }
MemberRemoved            { user_id }
TeamCreated              { name, slug }
TeamMemberAdded          { user_id }

-- Collaboration events
ChangeRequestOpened      { title, description, page_ids }
ChangeRequestReviewed    { reviewer_id, decision }
ChangeRequestMerged      { }
CommentPosted            { page_id, body, parent_comment_id }

-- Permission events
PermissionGranted        { space_id, grantee_type, grantee_id, permission }
PermissionRevoked        { space_id, grantee_type, grantee_id, permission }

-- AI events
DriftDetected            { page_id, commit_sha, drift_type, severity, description }
DriftResolved            { resolution }
AiQueryAsked             { session_id, question }
AiQueryAnswered          { session_id, answer, source_page_ids }

-- Git sync events
GitConnectionCreated     { space_id, provider, repo_url, branch }
GitSyncCompleted         { direction, commit_sha, files_changed }
GitSyncFailed            { error_message }
```

---

## Read Models (Projections)

These tables are rebuilt from events. They can be dropped and reconstructed at any time.

### Current Page State

```sql
CREATE TABLE rm_pages (
    id              UUID PRIMARY KEY,
    space_version_id UUID NOT NULL,
    parent_id       UUID,
    slug            VARCHAR(255) NOT NULL,
    title           VARCHAR(500) NOT NULL,
    body            TEXT,
    body_format     VARCHAR(20) NOT NULL DEFAULT 'mdx',
    page_type       VARCHAR(50) NOT NULL DEFAULT 'content',
    diataxis_type   VARCHAR(20),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    revision_count  INTEGER NOT NULL DEFAULT 0,
    last_author_id  UUID,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_pages_space_version ON rm_pages(space_version_id);
CREATE INDEX idx_rm_pages_parent ON rm_pages(parent_id);
CREATE INDEX idx_rm_pages_slug ON rm_pages(space_version_id, slug);
```

### Current Space State

```sql
CREATE TABLE rm_spaces (
    id              UUID PRIMARY KEY,
    org_id          UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    visibility      VARCHAR(20) NOT NULL DEFAULT 'private',
    default_lang    VARCHAR(10) NOT NULL DEFAULT 'en',
    custom_domain   VARCHAR(255),
    theme_config    JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_spaces_org ON rm_spaces(org_id);

CREATE TABLE rm_space_versions (
    id              UUID PRIMARY KEY,
    space_id        UUID NOT NULL,
    version_label   VARCHAR(50) NOT NULL,
    git_branch      VARCHAR(255),
    is_default      BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_space_versions_space ON rm_space_versions(space_id);
```

### Organization & Membership

```sql
CREATE TABLE rm_organizations (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE rm_org_memberships (
    org_id          UUID NOT NULL,
    user_id         UUID NOT NULL,
    role            VARCHAR(50) NOT NULL,
    joined_at       TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (org_id, user_id)
);
```

### API Endpoints (parsed from spec events)

```sql
CREATE TABLE rm_api_endpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_spec_id     UUID NOT NULL,
    method          VARCHAR(10) NOT NULL,
    path            VARCHAR(1000) NOT NULL,
    operation_id    VARCHAR(255),
    summary         TEXT,
    tags            TEXT[] NOT NULL DEFAULT '{}',
    deprecated      BOOLEAN NOT NULL DEFAULT false
);
CREATE INDEX idx_rm_api_endpoints_spec ON rm_api_endpoints(api_spec_id);
```

### Search & Embeddings

```sql
CREATE TABLE rm_search_index (
    page_id         UUID PRIMARY KEY,
    space_version_id UUID NOT NULL,
    search_vector   tsvector NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_search_vector ON rm_search_index USING gin(search_vector);

CREATE TABLE rm_embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    page_id         UUID NOT NULL,
    chunk_index     INTEGER NOT NULL,
    chunk_text      TEXT NOT NULL,
    embedding       vector(1536),
    UNIQUE (page_id, chunk_index)
);
CREATE INDEX idx_rm_embeddings_vector ON rm_embeddings USING ivfflat (embedding vector_cosine_ops);
```

### Analytics (event-derived aggregates)

```sql
CREATE TABLE rm_page_stats (
    page_id         UUID PRIMARY KEY,
    total_views     BIGINT NOT NULL DEFAULT 0,
    total_edits     INTEGER NOT NULL DEFAULT 0,
    avg_rating      NUMERIC(3,2),
    last_edited_at  TIMESTAMPTZ,
    last_viewed_at  TIMESTAMPTZ
);

CREATE TABLE rm_search_analytics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    space_id        UUID NOT NULL,
    query_text      TEXT NOT NULL,
    result_count    INTEGER NOT NULL,
    clicked_page_id UUID,
    searched_at     TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_rm_search_analytics_space ON rm_search_analytics(space_id);
```

### Permissions

```sql
CREATE TABLE rm_space_permissions (
    space_id        UUID NOT NULL,
    grantee_type    VARCHAR(20) NOT NULL,
    grantee_id      UUID NOT NULL,
    permission      VARCHAR(30) NOT NULL,
    PRIMARY KEY (space_id, grantee_type, grantee_id, permission)
);
```

---

## Projection Tracking

```sql
-- Tracks which events each projection has processed
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | events, snapshots |
| Read Model: Content | 2 | rm_pages, rm_space_versions |
| Read Model: Spaces | 1 | rm_spaces |
| Read Model: Identity | 2 | rm_organizations, rm_org_memberships |
| Read Model: API | 1 | rm_api_endpoints |
| Read Model: Search | 2 | rm_search_index, rm_embeddings |
| Read Model: Analytics | 2 | rm_page_stats, rm_search_analytics |
| Read Model: Permissions | 1 | rm_space_permissions |
| Infrastructure | 1 | projection_checkpoints |
| **Total** | **14** | 2 core + 12 projections |

---

## Key Design Decisions

1. **Single events table, not per-aggregate** — One `events` table with `stream_type` discrimination is simpler to manage than per-type event tables. The `stream_id` + `event_version` unique constraint ensures ordering within each aggregate.

2. **JSONB payloads for event data** — Event payloads use JSONB rather than typed columns, allowing new event types to be added without schema migrations. Event shape is enforced in application code.

3. **Synchronous projections for consistency** — Read models are updated in the same transaction as event insertion (not via async message queue), avoiding eventual consistency issues for a documentation platform where users expect to see their edits immediately.

4. **Snapshots for long-lived pages** — Pages with hundreds of edits benefit from periodic snapshots. The application checks for a snapshot before replaying events, reducing reconstruction time.

5. **Projections are rebuildable** — Every `rm_*` table can be dropped and rebuilt from the event store. This is the key advantage: if you add a new feature (e.g., a "most active contributors" leaderboard), you create a new projection and replay all events to populate it.

6. **Natural drift detection feed** — The event stream provides a direct input for AI drift detection. A processor watches for `GitSyncCompleted` events and compares them against recent `PageContentUpdated` events to detect divergence.

7. **Metadata envelope for compliance** — Every event includes `metadata` with `actor_id`, `ip_address`, and `correlation_id`, satisfying ISO 27001 audit requirements without a separate audit log table.

8. **Event versioning via upcasting** — When event schemas evolve, old events are upcasted during replay. For example, if `PageCreated` v1 lacked `diataxis_type`, the upcaster adds `diataxis_type: null` when replaying v1 events.

### Example: Rebuild Page State from Events

```sql
-- Get all events for a specific page, in order
SELECT event_type, payload, metadata, created_at
FROM events
WHERE stream_id = $1 AND stream_type = 'page'
ORDER BY event_version ASC;
```

```python
# Application-side replay
def rebuild_page(events):
    state = {}
    for event in events:
        match event.event_type:
            case 'PageCreated':
                state = {**event.payload, 'revision_count': 0}
            case 'PageContentUpdated':
                state['body'] = event.payload['body']
                state['revision_count'] += 1
            case 'PageTitleChanged':
                state['title'] = event.payload['new_title']
            case 'PageMoved':
                state['parent_id'] = event.payload['new_parent_id']
            case 'PagePublished':
                state['is_published'] = True
            case 'PageDeleted':
                state['deleted'] = True
    return state
```

### Example: Point-in-Time Query

```sql
-- What did this page look like on 2026-03-15?
SELECT event_type, payload, created_at
FROM events
WHERE stream_id = $1
  AND stream_type = 'page'
  AND created_at <= '2026-03-15T23:59:59Z'
ORDER BY event_version ASC;
-- Replay these events to reconstruct the page as of that date
```

### Example: Change Frequency Analysis for Drift Detection

```sql
-- Pages with the most edits in the last 30 days (potential drift candidates)
SELECT stream_id AS page_id,
       COUNT(*) AS edit_count,
       MAX(created_at) AS last_edit
FROM events
WHERE stream_type = 'page'
  AND event_type = 'PageContentUpdated'
  AND created_at > now() - INTERVAL '30 days'
GROUP BY stream_id
ORDER BY edit_count DESC
LIMIT 20;
```
