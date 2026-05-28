# Documentation Platform — Phased Development Plan

> Project: 171-documentation-platform · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (full-stack) | Frontend-heavy platform with rich editor, real-time collaboration, and API; TypeScript provides type safety across the entire stack and strong MDX/React ecosystem alignment |
| API Framework | Next.js 15 (App Router) + tRPC | Server-rendered documentation pages for SEO; tRPC provides end-to-end type safety between API and frontend; App Router enables React Server Components for fast doc page rendering |
| Database | PostgreSQL 16 + pgvector | Relational integrity for multi-tenant content hierarchy; pgvector for semantic search embeddings; tsvector for full-text search; JSONB for flexible metadata — all in one engine |
| ORM | Drizzle ORM | Type-safe SQL with zero runtime overhead; excellent PostgreSQL support including JSONB, arrays, and pgvector; migration generation from schema |
| Task Queue | BullMQ (Redis-backed) | Async workloads: Git sync, AI embedding generation, drift detection, webhook delivery, OpenAPI parsing; Redis also serves as real-time pub/sub for collaboration |
| Frontend | React 19 + Next.js 15 | React Server Components for fast doc rendering; client components for editor and interactive features; massive ecosystem of editor libraries |
| Editor | TipTap (ProseMirror-based) | Block-based rich text editor with Markdown input/output; extensible with custom blocks (callouts, code, API playground); real-time collaboration via Yjs |
| Real-time Collaboration | Yjs + y-websocket | CRDT-based collaborative editing; integrates natively with TipTap; conflict-free without OT server complexity |
| Search | PostgreSQL tsvector + pgvector | Full-text keyword search via tsvector; semantic search via pgvector embeddings; avoids external search dependency (Algolia/Meilisearch) for self-hosted deployments |
| AI/LLM | OpenAI API (GPT-4o) with provider abstraction | Writing suggestions, drift detection, code-to-docs generation, Q&A assistant; abstraction layer allows swapping to Anthropic, local models, or Azure OpenAI |
| Object Storage | S3-compatible (AWS S3 / MinIO / R2) | Image and asset uploads; MinIO for self-hosted; R2 or S3 for managed deployments |
| Authentication | NextAuth.js v5 (Auth.js) | Email/password + OAuth (GitHub, Google) + SAML/OIDC for enterprise SSO; integrates with Next.js App Router |
| Containerisation | Docker + docker-compose | Self-hosted deployment target; compose orchestrates app, PostgreSQL, Redis, MinIO in one command |
| Testing | Vitest + Playwright | Vitest for unit/integration tests (fast, native TypeScript); Playwright for E2E browser tests of editor and published docs |
| Code Quality | ESLint + Prettier + tsc strict | Linting, formatting, type checking; enforced via pre-commit hooks and CI |
| Package Manager | pnpm | Fast, disk-efficient; strict node_modules layout prevents phantom dependencies |
| Monorepo | Turborepo | If the project grows to separate packages (e.g., shared types, CLI, docs renderer); starts as a single Next.js app |
| Markdown Processing | unified (remark + rehype) + MDX | Industry-standard Markdown/MDX processing pipeline; remark for parsing, rehype for HTML rendering; MDX for React component embedding |
| OpenAPI Rendering | @readme/openapi-parser + custom renderer | Parse OpenAPI 3.0/3.1 specs; render interactive endpoint documentation with "Try it" playground |
| Diagram Rendering | mermaid-js | Client-side rendering of Mermaid diagrams embedded in documentation pages |
| CSS | Tailwind CSS 4 | Utility-first styling; theme customisation for white-label branding; consistent design system |

### Data Model Decision

Adopting **Data Model Suggestion 1 (Entity-Centric Normalized Relational)** as the primary schema, with selective JSONB usage from Suggestion 3 for truly variable metadata fields (`spaces.theme_config`, `organizations.settings`). The normalised model provides strong referential integrity essential for a multi-tenant documentation platform, works with standard ORM tooling, and keeps the schema explicit and auditable. The 33-table count is manageable with Drizzle migrations.

Key adaptations:
- Use Suggestion 1's explicit table structure for all core entities (pages, revisions, translations, permissions)
- Adopt Suggestion 3's JSONB pattern for `spaces.theme_config`, `organizations.settings`, and `api_specs.parsed_metadata`
- Use Suggestion 1's pgvector `embeddings` table for semantic search
- Add Suggestion 4's block-level comment targeting as an optional enhancement in Phase 10

### Project Structure

```
documentation-platform/
├── package.json
├── pnpm-lock.yaml
├── tsconfig.json
├── next.config.ts
├── tailwind.config.ts
├── drizzle.config.ts
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── .eslintrc.cjs
├── .prettierrc
├── vitest.config.ts
├── playwright.config.ts
├── src/
│   ├── app/                          # Next.js App Router pages
│   │   ├── (auth)/                   # Login, register, SSO callback
│   │   ├── (dashboard)/              # Org dashboard, space management
│   │   │   ├── [orgSlug]/
│   │   │   │   ├── spaces/
│   │   │   │   ├── settings/
│   │   │   │   └── members/
│   │   ├── (editor)/                 # Page editor views
│   │   │   └── [orgSlug]/[spaceSlug]/[...pagePath]/edit/
│   │   ├── (docs)/                   # Published documentation reader
│   │   │   └── [orgSlug]/[spaceSlug]/[versionLabel]/[...pagePath]/
│   │   ├── api/                      # REST API routes
│   │   │   ├── trpc/[trpc]/
│   │   │   ├── webhooks/
│   │   │   ├── git/
│   │   │   └── v1/                   # Public REST API
│   │   └── layout.tsx
│   ├── server/                       # Server-side logic
│   │   ├── db/
│   │   │   ├── schema/               # Drizzle schema definitions
│   │   │   │   ├── organizations.ts
│   │   │   │   ├── users.ts
│   │   │   │   ├── spaces.ts
│   │   │   │   ├── pages.ts
│   │   │   │   ├── revisions.ts
│   │   │   │   ├── translations.ts
│   │   │   │   ├── api-specs.ts
│   │   │   │   ├── git.ts
│   │   │   │   ├── collaboration.ts
│   │   │   │   ├── ai.ts
│   │   │   │   ├── search.ts
│   │   │   │   ├── analytics.ts
│   │   │   │   ├── permissions.ts
│   │   │   │   ├── integrations.ts
│   │   │   │   └── index.ts
│   │   │   ├── migrations/
│   │   │   └── client.ts             # Drizzle client initialisation
│   │   ├── trpc/
│   │   │   ├── router.ts             # Root tRPC router
│   │   │   ├── context.ts
│   │   │   └── routers/
│   │   │       ├── spaces.ts
│   │   │       ├── pages.ts
│   │   │       ├── revisions.ts
│   │   │       ├── search.ts
│   │   │       ├── ai.ts
│   │   │       ├── git.ts
│   │   │       ├── analytics.ts
│   │   │       └── admin.ts
│   │   ├── services/                 # Business logic
│   │   │   ├── auth.ts
│   │   │   ├── pages.ts
│   │   │   ├── search.ts
│   │   │   ├── ai/
│   │   │   │   ├── embeddings.ts
│   │   │   │   ├── assistant.ts
│   │   │   │   ├── drift-detection.ts
│   │   │   │   ├── writing-suggestions.ts
│   │   │   │   └── provider.ts       # LLM provider abstraction
│   │   │   ├── git-sync.ts
│   │   │   ├── openapi-parser.ts
│   │   │   ├── markdown.ts
│   │   │   ├── webhooks.ts
│   │   │   └── permissions.ts
│   │   ├── queue/                    # BullMQ workers
│   │   │   ├── workers/
│   │   │   │   ├── git-sync.worker.ts
│   │   │   │   ├── embeddings.worker.ts
│   │   │   │   ├── drift.worker.ts
│   │   │   │   └── webhooks.worker.ts
│   │   │   └── queues.ts
│   │   └── lib/                      # Shared server utilities
│   │       ├── config.ts
│   │       ├── logger.ts
│   │       └── errors.ts
│   ├── components/                   # React components
│   │   ├── editor/
│   │   │   ├── Editor.tsx
│   │   │   ├── Toolbar.tsx
│   │   │   ├── extensions/           # TipTap custom extensions
│   │   │   │   ├── callout.ts
│   │   │   │   ├── code-block.ts
│   │   │   │   ├── api-playground.ts
│   │   │   │   ├── diagram.ts
│   │   │   │   └── snippet-ref.ts
│   │   │   └── CollaborationCursor.tsx
│   │   ├── docs/                     # Published doc reader components
│   │   │   ├── DocLayout.tsx
│   │   │   ├── Sidebar.tsx
│   │   │   ├── TableOfContents.tsx
│   │   │   ├── SearchDialog.tsx
│   │   │   ├── AiAssistant.tsx
│   │   │   ├── ApiReference.tsx
│   │   │   └── VersionSwitcher.tsx
│   │   ├── dashboard/
│   │   │   ├── SpaceList.tsx
│   │   │   ├── MemberManagement.tsx
│   │   │   └── AnalyticsDashboard.tsx
│   │   └── ui/                       # Shared UI primitives
│   │       ├── Button.tsx
│   │       ├── Dialog.tsx
│   │       ├── Input.tsx
│   │       └── ...
│   ├── lib/                          # Shared client utilities
│   │   ├── trpc.ts                   # tRPC client
│   │   ├── utils.ts
│   │   └── constants.ts
│   └── types/                        # Shared TypeScript types
│       ├── api.ts
│       ├── editor.ts
│       └── index.ts
├── tests/
│   ├── unit/
│   │   ├── services/
│   │   ├── db/
│   │   └── lib/
│   ├── integration/
│   │   ├── api/
│   │   ├── services/
│   │   └── db/
│   ├── e2e/
│   │   ├── auth.spec.ts
│   │   ├── editor.spec.ts
│   │   ├── docs-reader.spec.ts
│   │   └── search.spec.ts
│   └── fixtures/
│       ├── sample-openapi.yaml
│       ├── sample-page.mdx
│       └── seed-data.ts
└── scripts/
    ├── seed.ts
    ├── migrate.ts
    └── generate-embeddings.ts
```

---

## Phase 1: Foundation — Project Scaffolding, Database Schema, and Authentication

### Purpose
Establish the project skeleton, database connection, core schema tables, and user authentication. After this phase, users can register, log in, create organisations, and manage team membership. This is the foundation every subsequent phase builds upon.

### Tasks

#### 1.1 — Project Initialisation and Tooling

**What**: Scaffold the Next.js 15 project with TypeScript, Tailwind CSS, ESLint, Prettier, Vitest, and Docker configuration.

**Design**:

```typescript
// next.config.ts
import type { NextConfig } from 'next';

const config: NextConfig = {
  experimental: {
    serverActions: { bodySizeLimit: '10mb' },
    typedRoutes: true,
  },
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: '**.s3.amazonaws.com' },
    ],
  },
};

export default config;
```

```typescript
// src/server/lib/config.ts
import { z } from 'zod';

export const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url().default('redis://localhost:6379'),
  NEXTAUTH_URL: z.string().url(),
  NEXTAUTH_SECRET: z.string().min(32),
  S3_ENDPOINT: z.string().url().optional(),
  S3_BUCKET: z.string().default('docplatform-assets'),
  S3_ACCESS_KEY: z.string().optional(),
  S3_SECRET_KEY: z.string().optional(),
  OPENAI_API_KEY: z.string().optional(),
  LLM_PROVIDER: z.enum(['openai', 'anthropic', 'azure']).default('openai'),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});

export type Env = z.infer<typeof envSchema>;
export const env = envSchema.parse(process.env);
```

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports: ["3000:3000"]
    env_file: .env
    depends_on: [db, redis]
  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: docplatform
      POSTGRES_USER: docplatform
      POSTGRES_PASSWORD: dev_password
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
volumes:
  pgdata:
```

**Testing**:
- `Unit: envSchema.parse with all required vars → returns typed Env object`
- `Unit: envSchema.parse with missing DATABASE_URL → throws ZodError with field name`
- `Unit: envSchema.parse with defaults → REDIS_URL and LOG_LEVEL populated`
- `Integration: docker-compose up → all services healthy within 30 seconds`
- `Integration: Next.js dev server starts → responds 200 at localhost:3000`

#### 1.2 — Database Schema: Identity and Multi-Tenancy

**What**: Define Drizzle schema for organisations, users, org memberships, teams, and team memberships; generate and run initial migration.

**Design**:

```typescript
// src/server/db/schema/organizations.ts
import { pgTable, uuid, varchar, text, jsonb, timestamp, uniqueIndex } from 'drizzle-orm/pg-core';

export const organizations = pgTable('organizations', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  logoUrl: text('logo_url'),
  billingPlan: varchar('billing_plan', { length: 50 }).notNull().default('free'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  displayName: varchar('display_name', { length: 255 }).notNull(),
  avatarUrl: text('avatar_url'),
  passwordHash: text('password_hash'),
  emailVerified: timestamp('email_verified', { withTimezone: true }),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const orgMemberships = pgTable('org_memberships', {
  id: uuid('id').primaryKey().defaultRandom(),
  orgId: uuid('org_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  role: varchar('role', { length: 50 }).notNull().default('member'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueOrgUser: uniqueIndex('idx_org_memberships_unique').on(table.orgId, table.userId),
}));

export const teams = pgTable('teams', {
  id: uuid('id').primaryKey().defaultRandom(),
  orgId: uuid('org_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueOrgSlug: uniqueIndex('idx_teams_org_slug').on(table.orgId, table.slug),
}));

export const teamMemberships = pgTable('team_memberships', {
  id: uuid('id').primaryKey().defaultRandom(),
  teamId: uuid('team_id').notNull().references(() => teams.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueTeamUser: uniqueIndex('idx_team_memberships_unique').on(table.teamId, table.userId),
}));
```

**Testing**:
- `Unit: Drizzle schema compiles → no TypeScript errors`
- `Integration: drizzle-kit generate → produces SQL migration file`
- `Integration: drizzle-kit migrate → migration applies to empty database without errors`
- `Integration: insert organization → returns UUID, slug is unique-constrained`
- `Integration: insert duplicate org_membership → raises unique constraint violation`
- `Integration: delete organization → cascades to org_memberships`

#### 1.3 — Authentication with NextAuth.js

**What**: Configure NextAuth.js v5 with email/password credentials and GitHub OAuth; session management with JWT; protected route middleware.

**Design**:

```typescript
// src/server/services/auth.ts
import NextAuth from 'next-auth';
import Credentials from 'next-auth/providers/credentials';
import GitHub from 'next-auth/providers/github';
import { DrizzleAdapter } from '@auth/drizzle-adapter';
import { db } from '@/server/db/client';
import bcrypt from 'bcryptjs';
import { z } from 'zod';

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

export const { handlers, signIn, signOut, auth } = NextAuth({
  adapter: DrizzleAdapter(db),
  session: { strategy: 'jwt' },
  providers: [
    GitHub({
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
    }),
    Credentials({
      credentials: {
        email: { type: 'email' },
        password: { type: 'password' },
      },
      async authorize(credentials) {
        const { email, password } = loginSchema.parse(credentials);
        const user = await db.query.users.findFirst({
          where: (u, { eq }) => eq(u.email, email),
        });
        if (!user?.passwordHash) return null;
        const valid = await bcrypt.compare(password, user.passwordHash);
        return valid ? { id: user.id, email: user.email, name: user.displayName } : null;
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) token.userId = user.id;
      return token;
    },
    async session({ session, token }) {
      session.user.id = token.userId as string;
      return session;
    },
  },
});
```

```typescript
// src/middleware.ts
import { auth } from '@/server/services/auth';

export default auth((req) => {
  const isAuth = !!req.auth;
  const isAuthPage = req.nextUrl.pathname.startsWith('/login') ||
                     req.nextUrl.pathname.startsWith('/register');
  const isDocsPage = req.nextUrl.pathname.match(/^\/[^/]+\/[^/]+\/docs/);
  const isApiRoute = req.nextUrl.pathname.startsWith('/api/v1');

  // Public doc pages and public API don't require auth
  if (isDocsPage || isApiRoute) return;

  if (!isAuth && !isAuthPage) {
    return Response.redirect(new URL('/login', req.nextUrl));
  }
});

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
};
```

**Testing**:
- `Unit: loginSchema.parse with valid email/password → returns parsed object`
- `Unit: loginSchema.parse with short password → throws ZodError`
- `Integration (mocked db): authorize with correct credentials → returns user object`
- `Integration (mocked db): authorize with wrong password → returns null`
- `Integration (mocked db): authorize with non-existent email → returns null`
- `E2E: visit /dashboard without session → redirects to /login`
- `E2E: login with valid credentials → redirects to /dashboard, session cookie set`
- `E2E: visit published docs without auth → renders normally (public)`

#### 1.4 — Organisation Management API

**What**: tRPC router for creating organisations, inviting members, managing roles, and listing memberships.

**Design**:

```typescript
// src/server/trpc/routers/admin.ts
import { z } from 'zod';
import { router, protectedProcedure } from '../context';
import { organizations, orgMemberships, users } from '@/server/db/schema';
import { eq, and } from 'drizzle-orm';
import { TRPCError } from '@trpc/server';

export const adminRouter = router({
  createOrg: protectedProcedure
    .input(z.object({
      name: z.string().min(1).max(255),
      slug: z.string().min(2).max(100).regex(/^[a-z0-9-]+$/),
    }))
    .mutation(async ({ ctx, input }) => {
      const org = await ctx.db.insert(organizations).values(input).returning();
      await ctx.db.insert(orgMemberships).values({
        orgId: org[0].id,
        userId: ctx.session.user.id,
        role: 'owner',
      });
      return org[0];
    }),

  listOrgs: protectedProcedure
    .query(async ({ ctx }) => {
      return ctx.db.query.orgMemberships.findMany({
        where: eq(orgMemberships.userId, ctx.session.user.id),
        with: { organization: true },
      });
    }),

  inviteMember: protectedProcedure
    .input(z.object({
      orgId: z.string().uuid(),
      email: z.string().email(),
      role: z.enum(['admin', 'member', 'viewer']),
    }))
    .mutation(async ({ ctx, input }) => {
      // Verify caller is owner or admin of the org
      const membership = await ctx.db.query.orgMemberships.findFirst({
        where: and(
          eq(orgMemberships.orgId, input.orgId),
          eq(orgMemberships.userId, ctx.session.user.id),
        ),
      });
      if (!membership || !['owner', 'admin'].includes(membership.role)) {
        throw new TRPCError({ code: 'FORBIDDEN' });
      }
      // Find or create user, add membership
      let user = await ctx.db.query.users.findFirst({
        where: eq(users.email, input.email),
      });
      if (!user) {
        const [created] = await ctx.db.insert(users).values({
          email: input.email,
          displayName: input.email.split('@')[0],
        }).returning();
        user = created;
      }
      await ctx.db.insert(orgMemberships).values({
        orgId: input.orgId,
        userId: user.id,
        role: input.role,
      });
      return { userId: user.id, role: input.role };
    }),

  updateMemberRole: protectedProcedure
    .input(z.object({
      orgId: z.string().uuid(),
      userId: z.string().uuid(),
      role: z.enum(['admin', 'member', 'viewer']),
    }))
    .mutation(async ({ ctx, input }) => {
      await ctx.db.update(orgMemberships)
        .set({ role: input.role })
        .where(and(
          eq(orgMemberships.orgId, input.orgId),
          eq(orgMemberships.userId, input.userId),
        ));
    }),
});
```

**Testing**:
- `Unit: createOrg input validation → rejects slug with uppercase`
- `Unit: createOrg input validation → rejects empty name`
- `Integration: createOrg → inserts org + owner membership, returns org with UUID`
- `Integration: createOrg with duplicate slug → raises unique constraint error`
- `Integration: listOrgs → returns only orgs the caller belongs to`
- `Integration: inviteMember as owner → creates user if needed, adds membership`
- `Integration: inviteMember as viewer → returns FORBIDDEN`
- `Integration: updateMemberRole → updates role in database`

---

## Phase 2: Content Spaces and Page Management

### Purpose
Implement documentation spaces, the page tree hierarchy, page creation/editing with Markdown content, and the revision history system. After this phase, users can create documentation spaces, organise pages in a tree, edit Markdown content, and browse revision history.

### Tasks

#### 2.1 — Database Schema: Spaces, Pages, and Revisions

**What**: Define Drizzle schema for spaces, space_versions, pages, and page_revisions tables.

**Design**:

```typescript
// src/server/db/schema/spaces.ts
import { pgTable, uuid, varchar, text, boolean, integer, jsonb, timestamp, uniqueIndex, index } from 'drizzle-orm/pg-core';
import { organizations } from './organizations';

export const spaces = pgTable('spaces', {
  id: uuid('id').primaryKey().defaultRandom(),
  orgId: uuid('org_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull(),
  description: text('description'),
  visibility: varchar('visibility', { length: 20 }).notNull().default('private'),
  defaultLang: varchar('default_lang', { length: 10 }).notNull().default('en'),
  customDomain: varchar('custom_domain', { length: 255 }),
  themeConfig: jsonb('theme_config').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueOrgSlug: uniqueIndex('idx_spaces_org_slug').on(table.orgId, table.slug),
  orgIdx: index('idx_spaces_org').on(table.orgId),
}));

export const spaceVersions = pgTable('space_versions', {
  id: uuid('id').primaryKey().defaultRandom(),
  spaceId: uuid('space_id').notNull().references(() => spaces.id, { onDelete: 'cascade' }),
  versionLabel: varchar('version_label', { length: 50 }).notNull(),
  gitBranch: varchar('git_branch', { length: 255 }),
  isDefault: boolean('is_default').notNull().default(false),
  publishedAt: timestamp('published_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueSpaceVersion: uniqueIndex('idx_space_versions_unique').on(table.spaceId, table.versionLabel),
}));
```

```typescript
// src/server/db/schema/pages.ts
import { pgTable, uuid, varchar, text, boolean, integer, timestamp, uniqueIndex, index } from 'drizzle-orm/pg-core';
import { spaceVersions } from './spaces';
import { users } from './organizations';

export const pages = pgTable('pages', {
  id: uuid('id').primaryKey().defaultRandom(),
  spaceVersionId: uuid('space_version_id').notNull().references(() => spaceVersions.id, { onDelete: 'cascade' }),
  parentId: uuid('parent_id').references((): any => pages.id, { onDelete: 'cascade' }),
  slug: varchar('slug', { length: 255 }).notNull(),
  title: varchar('title', { length: 500 }).notNull(),
  sortOrder: integer('sort_order').notNull().default(0),
  pageType: varchar('page_type', { length: 50 }).notNull().default('content'),
  diatasisType: varchar('diataxis_type', { length: 20 }),
  isPublished: boolean('is_published').notNull().default(false),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueSlug: uniqueIndex('idx_pages_slug').on(table.spaceVersionId, table.slug),
  spaceVersionIdx: index('idx_pages_space_version').on(table.spaceVersionId),
  parentIdx: index('idx_pages_parent').on(table.parentId),
}));

export const pageRevisions = pgTable('page_revisions', {
  id: uuid('id').primaryKey().defaultRandom(),
  pageId: uuid('page_id').notNull().references(() => pages.id, { onDelete: 'cascade' }),
  revisionNumber: integer('revision_number').notNull(),
  body: text('body').notNull(),
  bodyFormat: varchar('body_format', { length: 20 }).notNull().default('mdx'),
  commitMessage: text('commit_message'),
  authorId: uuid('author_id').references(() => users.id, { onDelete: 'set null' }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueRevision: uniqueIndex('idx_page_revisions_unique').on(table.pageId, table.revisionNumber),
  pageIdx: index('idx_page_revisions_page').on(table.pageId),
}));
```

**Testing**:
- `Integration: migration applies spaces, space_versions, pages, page_revisions tables`
- `Integration: insert space with unique org+slug → succeeds`
- `Integration: insert page with parent_id → creates child page`
- `Integration: delete parent page → cascades to children`
- `Integration: insert page_revision with sequential revision_number → succeeds`
- `Integration: insert duplicate revision_number for same page → raises unique constraint`

#### 2.2 — Space Management API

**What**: tRPC router for CRUD operations on spaces and space versions.

**Design**:

```typescript
// src/server/trpc/routers/spaces.ts
import { z } from 'zod';
import { router, protectedProcedure } from '../context';

export const spacesRouter = router({
  create: protectedProcedure
    .input(z.object({
      orgId: z.string().uuid(),
      name: z.string().min(1).max(255),
      slug: z.string().min(2).max(100).regex(/^[a-z0-9-]+$/),
      visibility: z.enum(['public', 'private', 'internal']).default('private'),
      description: z.string().max(1000).optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      // Verify org membership, create space, create default version "latest"
    }),

  list: protectedProcedure
    .input(z.object({ orgId: z.string().uuid() }))
    .query(async ({ ctx, input }) => {
      // Return spaces for org where user has access
    }),

  get: protectedProcedure
    .input(z.object({ spaceId: z.string().uuid() }))
    .query(async ({ ctx, input }) => {
      // Return space with versions, page count
    }),

  update: protectedProcedure
    .input(z.object({
      spaceId: z.string().uuid(),
      name: z.string().min(1).max(255).optional(),
      description: z.string().max(1000).optional(),
      visibility: z.enum(['public', 'private', 'internal']).optional(),
      themeConfig: z.record(z.unknown()).optional(),
    }))
    .mutation(async ({ ctx, input }) => { /* ... */ }),

  createVersion: protectedProcedure
    .input(z.object({
      spaceId: z.string().uuid(),
      versionLabel: z.string().min(1).max(50),
      copyFromVersionId: z.string().uuid().optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      // Create new version, optionally copy page tree from existing version
    }),

  delete: protectedProcedure
    .input(z.object({ spaceId: z.string().uuid() }))
    .mutation(async ({ ctx, input }) => { /* ... */ }),
});
```

**Testing**:
- `Integration: create space → inserts space + default "latest" version`
- `Integration: create space with duplicate slug in same org → returns error`
- `Integration: list spaces → returns only spaces in the specified org`
- `Integration: createVersion with copyFromVersionId → deep-copies page tree`
- `Integration: delete space → cascades to versions, pages, revisions`

#### 2.3 — Page Tree Management API

**What**: tRPC router for creating, moving, reordering, and deleting pages within a space version; includes recursive tree fetch using CTE.

**Design**:

```typescript
// src/server/trpc/routers/pages.ts
import { z } from 'zod';
import { router, protectedProcedure } from '../context';
import { sql } from 'drizzle-orm';

export const pagesRouter = router({
  getTree: protectedProcedure
    .input(z.object({ spaceVersionId: z.string().uuid() }))
    .query(async ({ ctx, input }) => {
      // Recursive CTE to fetch full page tree
      const result = await ctx.db.execute(sql`
        WITH RECURSIVE page_tree AS (
          SELECT id, parent_id, title, slug, sort_order, page_type, diataxis_type,
                 is_published, 0 AS depth, ARRAY[sort_order] AS path
          FROM pages
          WHERE space_version_id = ${input.spaceVersionId} AND parent_id IS NULL
          UNION ALL
          SELECT p.id, p.parent_id, p.title, p.slug, p.sort_order, p.page_type,
                 p.diataxis_type, p.is_published, pt.depth + 1, pt.path || p.sort_order
          FROM pages p
          JOIN page_tree pt ON p.parent_id = pt.id
        )
        SELECT * FROM page_tree ORDER BY path
      `);
      return result.rows;
    }),

  create: protectedProcedure
    .input(z.object({
      spaceVersionId: z.string().uuid(),
      parentId: z.string().uuid().nullable(),
      title: z.string().min(1).max(500),
      slug: z.string().min(1).max(255).regex(/^[a-z0-9-]+$/),
      pageType: z.enum(['content', 'api_reference', 'changelog', 'link']).default('content'),
      diatasisType: z.enum(['tutorial', 'how_to', 'reference', 'explanation']).nullable().optional(),
      body: z.string().default(''),
      bodyFormat: z.enum(['mdx', 'markdown']).default('mdx'),
    }))
    .mutation(async ({ ctx, input }) => {
      // Insert page, create initial revision (revision_number = 1)
    }),

  update: protectedProcedure
    .input(z.object({
      pageId: z.string().uuid(),
      title: z.string().min(1).max(500).optional(),
      body: z.string().optional(),
      bodyFormat: z.enum(['mdx', 'markdown']).optional(),
      commitMessage: z.string().max(500).optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      // Update page, create new revision with incremented revision_number
    }),

  move: protectedProcedure
    .input(z.object({
      pageId: z.string().uuid(),
      newParentId: z.string().uuid().nullable(),
      newSortOrder: z.number().int(),
    }))
    .mutation(async ({ ctx, input }) => { /* ... */ }),

  publish: protectedProcedure
    .input(z.object({ pageId: z.string().uuid() }))
    .mutation(async ({ ctx, input }) => {
      // Set is_published = true
    }),

  delete: protectedProcedure
    .input(z.object({ pageId: z.string().uuid() }))
    .mutation(async ({ ctx, input }) => { /* ... */ }),
});
```

**Testing**:
- `Integration: getTree returns pages ordered by depth and sort_order`
- `Integration: getTree for empty space version → returns empty array`
- `Integration: create page → inserts page + revision_number 1`
- `Integration: create page with duplicate slug → raises unique constraint error`
- `Integration: update page body → creates new revision with revision_number + 1`
- `Integration: move page → updates parent_id and sort_order`
- `Integration: delete page with children → cascades to children and their revisions`
- `Integration: publish page → sets is_published = true`

#### 2.4 — Revision History API

**What**: tRPC endpoints for listing revisions, viewing a specific revision, and restoring to a previous revision.

**Design**:

```typescript
// src/server/trpc/routers/revisions.ts
export const revisionsRouter = router({
  list: protectedProcedure
    .input(z.object({
      pageId: z.string().uuid(),
      limit: z.number().int().min(1).max(100).default(20),
      cursor: z.number().int().optional(),
    }))
    .query(async ({ ctx, input }) => {
      // Return revisions descending by revision_number, with author info
    }),

  get: protectedProcedure
    .input(z.object({
      pageId: z.string().uuid(),
      revisionNumber: z.number().int(),
    }))
    .query(async ({ ctx, input }) => {
      // Return specific revision body and metadata
    }),

  restore: protectedProcedure
    .input(z.object({
      pageId: z.string().uuid(),
      revisionNumber: z.number().int(),
    }))
    .mutation(async ({ ctx, input }) => {
      // Copy revision body into a new revision (don't delete history)
    }),

  diff: protectedProcedure
    .input(z.object({
      pageId: z.string().uuid(),
      fromRevision: z.number().int(),
      toRevision: z.number().int(),
    }))
    .query(async ({ ctx, input }) => {
      // Return unified diff between two revisions
    }),
});
```

**Testing**:
- `Integration: list revisions → returns descending by revision_number`
- `Integration: get specific revision → returns body and author`
- `Integration: restore revision 3 when current is 7 → creates revision 8 with body from 3`
- `Integration: diff between revisions → returns unified diff string`

---

## Phase 3: Markdown/MDX Editor and Document Renderer

### Purpose
Build the writing experience: a TipTap-based rich editor with Markdown/MDX input, and a server-rendered documentation reader for published pages. After this phase, authors can write in a modern editor and readers can view published docs with syntax highlighting, table of contents, and responsive layout.

### Tasks

#### 3.1 — TipTap Editor with Markdown Support

**What**: React component wrapping TipTap with custom extensions for documentation authoring: headings, code blocks with syntax highlighting, callouts, tables, and Markdown shortcuts.

**Design**:

```typescript
// src/components/editor/Editor.tsx
'use client';

import { useEditor, EditorContent } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';
import CodeBlockLowlight from '@tiptap/extension-code-block-lowlight';
import { Markdown } from 'tiptap-markdown';
import { CalloutExtension } from './extensions/callout';
import { DiagramExtension } from './extensions/diagram';

interface EditorProps {
  initialContent: string;
  format: 'mdx' | 'markdown';
  onSave: (content: string, commitMessage?: string) => Promise<void>;
  readOnly?: boolean;
}

export function Editor({ initialContent, format, onSave, readOnly }: EditorProps) {
  const editor = useEditor({
    extensions: [
      StarterKit.configure({ codeBlock: false }),
      CodeBlockLowlight.configure({ lowlight }),
      Markdown.configure({ html: false, transformPastedText: true }),
      CalloutExtension,
      DiagramExtension,
    ],
    content: initialContent,
    editable: !readOnly,
  });

  // Export to Markdown for storage
  const getMarkdown = (): string => {
    return editor?.storage.markdown.getMarkdown() ?? '';
  };

  return (
    <div className="editor-container">
      <Toolbar editor={editor} />
      <EditorContent editor={editor} className="prose max-w-none" />
    </div>
  );
}
```

```typescript
// src/components/editor/extensions/callout.ts
import { Node, mergeAttributes } from '@tiptap/core';

export type CalloutType = 'info' | 'warning' | 'danger' | 'tip' | 'note';

export const CalloutExtension = Node.create({
  name: 'callout',
  group: 'block',
  content: 'block+',
  defining: true,

  addAttributes() {
    return {
      type: { default: 'info' as CalloutType },
      title: { default: null },
    };
  },

  parseHTML() {
    return [{ tag: 'div[data-callout]' }];
  },

  renderHTML({ HTMLAttributes }) {
    return ['div', mergeAttributes(HTMLAttributes, { 'data-callout': '' }), 0];
  },
});
```

**Testing**:
- `Unit: Editor renders with initialContent → displays formatted text`
- `Unit: type "# Hello" → creates H1 heading`
- `Unit: type triple backtick → creates code block`
- `Unit: getMarkdown → returns valid Markdown string`
- `Unit: callout extension → renders callout div with correct type attribute`
- `Integration: paste Markdown → converts to TipTap nodes correctly`
- `E2E: create new page → editor loads, type content, save → revision created`

#### 3.2 — MDX Processing Pipeline

**What**: unified/remark/rehype pipeline for parsing MDX content and rendering to HTML with syntax highlighting, Mermaid diagrams, and custom components.

**Design**:

```typescript
// src/server/services/markdown.ts
import { unified } from 'unified';
import remarkParse from 'remark-parse';
import remarkMdx from 'remark-mdx';
import remarkGfm from 'remark-gfm';
import remarkRehype from 'remark-rehype';
import rehypeHighlight from 'rehype-highlight';
import rehypeSlug from 'rehype-slug';
import rehypeAutolinkHeadings from 'rehype-autolink-headings';
import rehypeStringify from 'rehype-stringify';

export interface RenderResult {
  html: string;
  headings: Array<{ id: string; text: string; level: number }>;
  frontmatter: Record<string, unknown>;
}

export async function renderMdx(source: string): Promise<RenderResult> {
  const headings: RenderResult['headings'] = [];

  const file = await unified()
    .use(remarkParse)
    .use(remarkMdx)
    .use(remarkGfm)
    .use(remarkRehype, { allowDangerousHtml: true })
    .use(rehypeSlug)
    .use(rehypeAutolinkHeadings, { behavior: 'wrap' })
    .use(rehypeHighlight)
    .use(extractHeadings, { headings })
    .use(rehypeStringify)
    .process(source);

  return {
    html: String(file),
    headings,
    frontmatter: {},
  };
}

export function extractTableOfContents(headings: RenderResult['headings']): TocNode[] {
  // Build nested TOC tree from flat headings list
}
```

**Testing**:
- `Unit: renderMdx("# Hello") → html contains <h1 id="hello">`
- `Unit: renderMdx with code block → applies syntax highlighting classes`
- `Unit: renderMdx with GFM table → renders HTML table`
- `Unit: renderMdx with Mermaid block → wraps in mermaid container div`
- `Unit: extractTableOfContents → builds nested tree from flat headings`
- `Unit: renderMdx with malformed MDX → returns error, does not throw`
- `Fixture: sample-page.mdx → renders complete HTML matching snapshot`

#### 3.3 — Documentation Reader (Published Docs)

**What**: Server-rendered Next.js pages for the public documentation site with sidebar navigation, table of contents, version switcher, breadcrumbs, and responsive layout.

**Design**:

```typescript
// src/app/(docs)/[orgSlug]/[spaceSlug]/[versionLabel]/[...pagePath]/page.tsx
import { db } from '@/server/db/client';
import { renderMdx } from '@/server/services/markdown';
import { DocLayout } from '@/components/docs/DocLayout';
import { notFound } from 'next/navigation';

interface DocsPageProps {
  params: {
    orgSlug: string;
    spaceSlug: string;
    versionLabel: string;
    pagePath: string[];
  };
}

export default async function DocsPage({ params }: DocsPageProps) {
  const { orgSlug, spaceSlug, versionLabel, pagePath } = params;
  const slug = pagePath[pagePath.length - 1];

  // Fetch space, version, page, and sidebar tree
  const space = await findSpaceBySlug(orgSlug, spaceSlug);
  if (!space) notFound();

  const version = await findVersion(space.id, versionLabel);
  if (!version) notFound();

  const page = await findPageBySlug(version.id, slug);
  if (!page || !page.isPublished) notFound();

  const latestRevision = await getLatestRevision(page.id);
  const { html, headings } = await renderMdx(latestRevision.body);
  const pageTree = await getPageTree(version.id);
  const versions = await getVersions(space.id);

  return (
    <DocLayout
      space={space}
      pageTree={pageTree}
      currentPage={page}
      versions={versions}
      currentVersion={version}
      headings={headings}
    >
      <article
        className="prose prose-slate max-w-none"
        dangerouslySetInnerHTML={{ __html: html }}
      />
    </DocLayout>
  );
}

export async function generateMetadata({ params }: DocsPageProps) {
  // Return title, description, og:image for SEO
}
```

**Testing**:
- `E2E: visit published doc page → renders HTML with sidebar, TOC, content`
- `E2E: visit non-existent page → returns 404`
- `E2E: visit unpublished page → returns 404`
- `E2E: version switcher → navigates to same page in different version`
- `E2E: sidebar navigation → highlights current page, expands ancestors`
- `E2E: mobile view → sidebar collapses to hamburger menu`
- `Integration: generateMetadata → returns correct title and description`

#### 3.4 — Editor Page Integration

**What**: Wire the editor component into a Next.js page with save/publish actions, revision history sidebar, and page metadata editing.

**Design**:

```typescript
// src/app/(editor)/[orgSlug]/[spaceSlug]/[...pagePath]/edit/page.tsx
'use client';

import { Editor } from '@/components/editor/Editor';
import { trpc } from '@/lib/trpc';
import { useState } from 'react';

export default function EditPage({ params }: EditPageProps) {
  const { data: page } = trpc.pages.get.useQuery({ /* ... */ });
  const { data: revision } = trpc.revisions.latest.useQuery({ pageId: page?.id });
  const updateMutation = trpc.pages.update.useMutation();

  const handleSave = async (content: string, commitMessage?: string) => {
    await updateMutation.mutateAsync({
      pageId: page!.id,
      body: content,
      commitMessage,
    });
  };

  return (
    <div className="flex h-screen">
      <PageTreeSidebar spaceVersionId={page?.spaceVersionId} />
      <div className="flex-1">
        <EditorHeader page={page} onPublish={handlePublish} />
        {revision && (
          <Editor
            initialContent={revision.body}
            format={revision.bodyFormat}
            onSave={handleSave}
          />
        )}
      </div>
      <MetadataSidebar page={page} />
    </div>
  );
}
```

**Testing**:
- `E2E: navigate to edit page → loads editor with current content`
- `E2E: edit content and save → creates new revision, shows success toast`
- `E2E: click publish → page becomes visible in published docs`
- `E2E: page tree sidebar → shows page hierarchy, clicking navigates to other pages`

---

## Phase 4: Full-Text Search and Navigation

### Purpose
Add keyword-based full-text search across all documentation within a space, plus search analytics to track what users search for. After this phase, readers can search docs and admins can see what queries produce zero results (indicating content gaps).

### Tasks

#### 4.1 — Database Schema: Search Index and Search Queries

**What**: Create the search_index and search_queries tables with tsvector-based full-text search.

**Design**:

```typescript
// src/server/db/schema/search.ts
import { pgTable, uuid, timestamp, index, customType } from 'drizzle-orm/pg-core';

const tsvector = customType<{ data: string }>({
  dataType() { return 'tsvector'; },
});

export const searchIndex = pgTable('search_index', {
  id: uuid('id').primaryKey().defaultRandom(),
  pageId: uuid('page_id').notNull().references(() => pages.id, { onDelete: 'cascade' }),
  spaceVersionId: uuid('space_version_id').notNull().references(() => spaceVersions.id, { onDelete: 'cascade' }),
  searchVector: tsvector('search_vector').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  vectorIdx: index('idx_search_vector').using('gin', table.searchVector),
  pageIdx: index('idx_search_index_page').on(table.pageId),
}));

export const searchQueries = pgTable('search_queries', {
  id: uuid('id').primaryKey().defaultRandom(),
  spaceId: uuid('space_id').notNull().references(() => spaces.id, { onDelete: 'cascade' }),
  queryText: text('query_text').notNull(),
  resultsCount: integer('results_count').notNull().default(0),
  clickedPageId: uuid('clicked_page_id').references(() => pages.id),
  userId: uuid('user_id').references(() => users.id),
  searchedAt: timestamp('searched_at', { withTimezone: true }).notNull().defaultNow(),
});
```

**Testing**:
- `Integration: migration creates search_index with GIN index on tsvector`
- `Integration: insert search_index row with tsvector → queryable with ts_query`

#### 4.2 — Search Indexing Service

**What**: Service that builds and updates the tsvector search index when pages are created, updated, or deleted.

**Design**:

```typescript
// src/server/services/search.ts
import { db } from '@/server/db/client';
import { sql } from 'drizzle-orm';

export async function indexPage(pageId: string, spaceVersionId: string): Promise<void> {
  const revision = await getLatestRevision(pageId);
  const page = await getPage(pageId);
  if (!revision || !page) return;

  // Strip Markdown formatting to get plain text
  const plainText = stripMarkdown(revision.body);

  // Build weighted tsvector: title (A weight), body (B weight)
  await db.execute(sql`
    INSERT INTO search_index (page_id, space_version_id, search_vector)
    VALUES (
      ${pageId},
      ${spaceVersionId},
      setweight(to_tsvector('english', ${page.title}), 'A') ||
      setweight(to_tsvector('english', ${plainText}), 'B')
    )
    ON CONFLICT (page_id) DO UPDATE SET
      search_vector = EXCLUDED.search_vector,
      updated_at = now()
  `);
}

export async function searchPages(
  spaceVersionId: string,
  query: string,
  limit: number = 20
): Promise<SearchResult[]> {
  const results = await db.execute(sql`
    SELECT p.id, p.title, p.slug, p.page_type,
           ts_rank(si.search_vector, websearch_to_tsquery('english', ${query})) AS rank,
           ts_headline('english', pr.body, websearch_to_tsquery('english', ${query}),
                       'StartSel=<mark>, StopSel=</mark>, MaxWords=50, MinWords=20') AS snippet
    FROM search_index si
    JOIN pages p ON si.page_id = p.id
    JOIN LATERAL (
      SELECT body FROM page_revisions
      WHERE page_id = p.id
      ORDER BY revision_number DESC LIMIT 1
    ) pr ON true
    WHERE si.space_version_id = ${spaceVersionId}
      AND p.is_published = true
      AND si.search_vector @@ websearch_to_tsquery('english', ${query})
    ORDER BY rank DESC
    LIMIT ${limit}
  `);
  return results.rows as SearchResult[];
}

export interface SearchResult {
  id: string;
  title: string;
  slug: string;
  pageType: string;
  rank: number;
  snippet: string;
}
```

**Testing**:
- `Unit: stripMarkdown removes headings, links, code blocks → plain text`
- `Integration: indexPage → creates tsvector with title (A) and body (B) weights`
- `Integration: searchPages("authentication") → returns pages mentioning auth, ranked by relevance`
- `Integration: searchPages with no results → returns empty array`
- `Integration: update page and re-index → search returns updated content`
- `Integration: ts_headline → returns snippet with <mark> tags around matched terms`

#### 4.3 — Search UI Component

**What**: Command-palette-style search dialog (Cmd+K) for the documentation reader, with instant results and keyboard navigation.

**Design**:

```typescript
// src/components/docs/SearchDialog.tsx
'use client';

import { useState, useEffect, useCallback } from 'react';
import { trpc } from '@/lib/trpc';
import { useDebounce } from '@/lib/hooks';

interface SearchDialogProps {
  spaceVersionId: string;
  open: boolean;
  onClose: () => void;
}

export function SearchDialog({ spaceVersionId, open, onClose }: SearchDialogProps) {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 200);
  const [selectedIndex, setSelectedIndex] = useState(0);

  const { data: results } = trpc.search.query.useQuery(
    { spaceVersionId, query: debouncedQuery },
    { enabled: debouncedQuery.length >= 2 }
  );

  // Keyboard navigation: ArrowUp, ArrowDown, Enter, Escape
  const handleKeyDown = useCallback((e: KeyboardEvent) => { /* ... */ }, []);

  return (
    <dialog open={open} className="search-dialog">
      <input
        type="search"
        placeholder="Search documentation..."
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        autoFocus
      />
      <ul role="listbox">
        {results?.map((result, i) => (
          <li key={result.id} role="option" aria-selected={i === selectedIndex}>
            <a href={`/${result.slug}`}>
              <span className="title">{result.title}</span>
              <span className="snippet" dangerouslySetInnerHTML={{ __html: result.snippet }} />
            </a>
          </li>
        ))}
      </ul>
    </dialog>
  );
}
```

**Testing**:
- `E2E: press Cmd+K → search dialog opens with focus on input`
- `E2E: type "auth" → results appear after debounce, matching pages shown`
- `E2E: arrow down + Enter → navigates to selected result page`
- `E2E: Escape → closes search dialog`
- `E2E: type query with no results → shows "No results found" message`
- `Unit: useDebounce(200) → delays value by 200ms`

#### 4.4 — Search Analytics

**What**: Log search queries and clicked results; tRPC endpoint for admins to view top queries and zero-result queries.

**Design**:

```typescript
// src/server/trpc/routers/search.ts (addition)
export const searchRouter = router({
  query: publicProcedure /* ... (from 4.2) */,

  logClick: publicProcedure
    .input(z.object({
      searchQueryId: z.string().uuid(),
      clickedPageId: z.string().uuid(),
    }))
    .mutation(async ({ ctx, input }) => { /* ... */ }),

  analytics: protectedProcedure
    .input(z.object({
      spaceId: z.string().uuid(),
      days: z.number().int().min(1).max(90).default(30),
    }))
    .query(async ({ ctx, input }) => {
      // Return: top queries, zero-result queries, click-through rate
      return {
        topQueries: [], // [{ query, count, avgResults }]
        zeroResultQueries: [], // [{ query, count, lastSearched }]
        totalSearches: 0,
        avgClickThroughRate: 0,
      };
    }),
});
```

**Testing**:
- `Integration: log search query → inserts into search_queries`
- `Integration: logClick → updates clicked_page_id on search query`
- `Integration: analytics → returns top queries sorted by frequency`
- `Integration: analytics → returns zero-result queries`

---

## Phase 5: Permissions and Access Control

### Purpose
Implement role-based access control for spaces: who can read, write, publish, and administrate each space. After this phase, organisations can control who sees and edits which documentation.

### Tasks

#### 5.1 — Database Schema: Permissions

**What**: Create space_permissions table and implement permission checking service.

**Design**:

```typescript
// src/server/db/schema/permissions.ts
export const spacePermissions = pgTable('space_permissions', {
  id: uuid('id').primaryKey().defaultRandom(),
  spaceId: uuid('space_id').notNull().references(() => spaces.id, { onDelete: 'cascade' }),
  granteeType: varchar('grantee_type', { length: 20 }).notNull(), // 'user' | 'team' | 'org'
  granteeId: uuid('grantee_id').notNull(),
  permission: varchar('permission', { length: 30 }).notNull(), // 'read' | 'write' | 'admin' | 'publish'
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueGrant: uniqueIndex('idx_space_perms_unique').on(
    table.spaceId, table.granteeType, table.granteeId, table.permission
  ),
}));
```

```typescript
// src/server/services/permissions.ts
export type Permission = 'read' | 'write' | 'publish' | 'admin';

export async function checkPermission(
  userId: string,
  spaceId: string,
  required: Permission
): Promise<boolean> {
  const hierarchy: Record<Permission, Permission[]> = {
    admin: ['admin', 'publish', 'write', 'read'],
    publish: ['publish', 'write', 'read'],
    write: ['write', 'read'],
    read: ['read'],
  };

  // Check: org owner/admin → full access
  // Check: direct user permission
  // Check: team membership → team permission
  // Check: org-wide permission
  // Check: space visibility (public → read for everyone)
  const grants = await db.query.spacePermissions.findMany({
    where: and(eq(spacePermissions.spaceId, spaceId)),
  });

  // ... evaluate grants against user's memberships and teams
  return false;
}

export function requirePermission(permission: Permission) {
  return async (ctx: TRPCContext, spaceId: string) => {
    const allowed = await checkPermission(ctx.session.user.id, spaceId, permission);
    if (!allowed) throw new TRPCError({ code: 'FORBIDDEN' });
  };
}
```

**Testing**:
- `Unit: permission hierarchy → admin implies read, write, publish`
- `Integration: org owner → has admin permission on all org spaces`
- `Integration: user with write permission → can write but not publish`
- `Integration: team with read permission → team members can read`
- `Integration: public space → anonymous read access`
- `Integration: no permission → checkPermission returns false`
- `Integration: requirePermission throws FORBIDDEN for unauthorized user`

#### 5.2 — Permission Management API

**What**: tRPC endpoints for granting, revoking, and listing permissions on spaces.

**Design**:

```typescript
export const permissionsRouter = router({
  grant: protectedProcedure
    .input(z.object({
      spaceId: z.string().uuid(),
      granteeType: z.enum(['user', 'team', 'org']),
      granteeId: z.string().uuid(),
      permission: z.enum(['read', 'write', 'publish', 'admin']),
    }))
    .mutation(async ({ ctx, input }) => { /* admin-only */ }),

  revoke: protectedProcedure
    .input(z.object({
      spaceId: z.string().uuid(),
      granteeType: z.enum(['user', 'team', 'org']),
      granteeId: z.string().uuid(),
      permission: z.enum(['read', 'write', 'publish', 'admin']),
    }))
    .mutation(async ({ ctx, input }) => { /* admin-only */ }),

  list: protectedProcedure
    .input(z.object({ spaceId: z.string().uuid() }))
    .query(async ({ ctx, input }) => { /* admin-only, return all grants */ }),
});
```

**Testing**:
- `Integration: grant permission → creates space_permission row`
- `Integration: grant duplicate permission → idempotent, no error`
- `Integration: revoke permission → deletes space_permission row`
- `Integration: list permissions → returns all grants with grantee details`
- `Integration: non-admin tries to grant → returns FORBIDDEN`

---

## Phase 6: API Documentation (OpenAPI)

### Purpose
Add first-class OpenAPI 3.0/3.1 support: upload specs, parse endpoints, render interactive API reference pages with a "Try it" playground. After this phase, teams can upload their API specs and get beautiful, interactive API documentation.

### Tasks

#### 6.1 — Database Schema: API Specs and Endpoints

**What**: Create api_specs and api_endpoints tables.

**Design**:

```typescript
// src/server/db/schema/api-specs.ts
export const apiSpecs = pgTable('api_specs', {
  id: uuid('id').primaryKey().defaultRandom(),
  spaceVersionId: uuid('space_version_id').notNull().references(() => spaceVersions.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  specFormat: varchar('spec_format', { length: 20 }).notNull(), // openapi_3_0, openapi_3_1, asyncapi_3_0
  rawContent: text('raw_content').notNull(),
  parsedMetadata: jsonb('parsed_metadata'),
  sourceUrl: text('source_url'),
  lastSyncedAt: timestamp('last_synced_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const apiEndpoints = pgTable('api_endpoints', {
  id: uuid('id').primaryKey().defaultRandom(),
  apiSpecId: uuid('api_spec_id').notNull().references(() => apiSpecs.id, { onDelete: 'cascade' }),
  method: varchar('method', { length: 10 }).notNull(),
  path: varchar('path', { length: 1000 }).notNull(),
  operationId: varchar('operation_id', { length: 255 }),
  summary: text('summary'),
  description: text('description'),
  tags: text('tags').array().notNull().default([]),
  deprecated: boolean('deprecated').notNull().default(false),
  requestSchema: jsonb('request_schema'),
  responseSchemas: jsonb('response_schemas'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});
```

**Testing**:
- `Integration: migration creates api_specs and api_endpoints tables`
- `Integration: insert api_spec → succeeds`
- `Integration: delete api_spec → cascades to endpoints`

#### 6.2 — OpenAPI Parser Service

**What**: Parse uploaded OpenAPI 3.0/3.1 YAML/JSON specs, validate them, extract endpoints, and store in the database.

**Design**:

```typescript
// src/server/services/openapi-parser.ts
import SwaggerParser from '@apidevtools/swagger-parser';

export interface ParsedSpec {
  format: 'openapi_3_0' | 'openapi_3_1';
  title: string;
  version: string;
  servers: Array<{ url: string; description?: string }>;
  endpoints: ParsedEndpoint[];
  schemas: Record<string, unknown>;
}

export interface ParsedEndpoint {
  method: string;
  path: string;
  operationId?: string;
  summary?: string;
  description?: string;
  tags: string[];
  deprecated: boolean;
  parameters: Array<{ name: string; in: string; required: boolean; schema: unknown }>;
  requestBody?: { contentType: string; schema: unknown; required: boolean };
  responses: Record<string, { description: string; schema?: unknown }>;
}

export async function parseOpenApiSpec(rawContent: string): Promise<ParsedSpec> {
  const api = await SwaggerParser.validate(rawContent);
  // Extract and normalise endpoints, schemas, servers
  return { /* ... */ };
}

export async function syncSpecToDb(
  specId: string,
  parsed: ParsedSpec
): Promise<void> {
  // Delete existing endpoints for spec, insert new ones
  await db.delete(apiEndpoints).where(eq(apiEndpoints.apiSpecId, specId));
  for (const endpoint of parsed.endpoints) {
    await db.insert(apiEndpoints).values({
      apiSpecId: specId,
      method: endpoint.method.toUpperCase(),
      path: endpoint.path,
      operationId: endpoint.operationId,
      summary: endpoint.summary,
      tags: endpoint.tags,
      deprecated: endpoint.deprecated,
      requestSchema: endpoint.requestBody?.schema,
      responseSchemas: endpoint.responses,
    });
  }
}
```

**Testing**:
- `Unit: parseOpenApiSpec with valid petstore.yaml → returns endpoints array`
- `Unit: parseOpenApiSpec with OpenAPI 3.1 → detects format correctly`
- `Unit: parseOpenApiSpec with invalid spec → throws validation error`
- `Unit: parseOpenApiSpec extracts parameters, request body, responses`
- `Integration: syncSpecToDb → creates endpoint rows matching spec`
- `Integration: re-sync spec → old endpoints deleted, new ones created`
- `Fixture: sample-openapi.yaml → parses to expected endpoint count`

#### 6.3 — API Reference Renderer

**What**: React components for rendering interactive API documentation: endpoint list grouped by tag, parameter tables, request/response schemas, and "Try it" playground.

**Design**:

```typescript
// src/components/docs/ApiReference.tsx
interface ApiReferenceProps {
  spec: ParsedSpec;
  endpoints: ParsedEndpoint[];
}

export function ApiReference({ spec, endpoints }: ApiReferenceProps) {
  const grouped = groupByTag(endpoints);
  return (
    <div className="api-reference">
      <ApiOverview title={spec.title} version={spec.version} servers={spec.servers} />
      {Object.entries(grouped).map(([tag, eps]) => (
        <ApiTagGroup key={tag} tag={tag} endpoints={eps} />
      ))}
    </div>
  );
}

// src/components/docs/ApiPlayground.tsx
interface ApiPlaygroundProps {
  endpoint: ParsedEndpoint;
  servers: Array<{ url: string }>;
}

export function ApiPlayground({ endpoint, servers }: ApiPlaygroundProps) {
  const [selectedServer, setSelectedServer] = useState(servers[0]?.url);
  const [params, setParams] = useState<Record<string, string>>({});
  const [body, setBody] = useState('');
  const [response, setResponse] = useState<{ status: number; body: string } | null>(null);

  const execute = async () => {
    const url = buildUrl(selectedServer, endpoint.path, params);
    const res = await fetch('/api/v1/proxy', {
      method: 'POST',
      body: JSON.stringify({ url, method: endpoint.method, body, headers: {} }),
    });
    setResponse(await res.json());
  };

  return (
    <div className="api-playground">
      <ParameterForm parameters={endpoint.parameters} values={params} onChange={setParams} />
      {endpoint.requestBody && <RequestBodyEditor schema={endpoint.requestBody.schema} value={body} onChange={setBody} />}
      <button onClick={execute}>Try it</button>
      {response && <ResponseViewer status={response.status} body={response.body} />}
    </div>
  );
}
```

**Testing**:
- `Unit: ApiReference renders endpoint list grouped by tag`
- `Unit: ApiPlayground renders parameter form based on endpoint params`
- `E2E: upload OpenAPI spec → API reference page renders with all endpoints`
- `E2E: click "Try it" → sends request through proxy, displays response`
- `E2E: deprecated endpoint → shows deprecation badge`

#### 6.4 — API Spec Management API

**What**: tRPC endpoints for uploading, updating, and deleting OpenAPI specs.

**Design**:

```typescript
export const apiSpecsRouter = router({
  upload: protectedProcedure
    .input(z.object({
      spaceVersionId: z.string().uuid(),
      name: z.string().min(1).max(255),
      rawContent: z.string().min(1),
      sourceUrl: z.string().url().optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      const parsed = await parseOpenApiSpec(input.rawContent);
      const [spec] = await ctx.db.insert(apiSpecs).values({
        spaceVersionId: input.spaceVersionId,
        name: input.name,
        specFormat: parsed.format,
        rawContent: input.rawContent,
        parsedMetadata: parsed,
        sourceUrl: input.sourceUrl,
      }).returning();
      await syncSpecToDb(spec.id, parsed);
      return spec;
    }),

  resync: protectedProcedure
    .input(z.object({ specId: z.string().uuid() }))
    .mutation(async ({ ctx, input }) => {
      // Re-fetch from sourceUrl if set, re-parse, re-sync endpoints
    }),

  delete: protectedProcedure
    .input(z.object({ specId: z.string().uuid() }))
    .mutation(async ({ ctx, input }) => { /* ... */ }),
});
```

**Testing**:
- `Integration: upload valid spec → creates api_spec + endpoints`
- `Integration: upload invalid spec → returns validation error, no rows created`
- `Integration: resync → re-fetches, re-parses, replaces endpoints`
- `Integration: delete spec → cascades to endpoints`

---

## Phase 7: Git Integration and Docs-as-Code

### Purpose
Enable bidirectional sync between the documentation platform and Git repositories (GitHub, GitLab). After this phase, teams can push docs to Git and pull changes from Git, maintaining the docs-as-code workflow described in standards.md.

### Tasks

#### 7.1 — Database Schema: Git Connections and Sync Log

**What**: Create git_connections and git_sync_log tables.

**Design**:

```typescript
// src/server/db/schema/git.ts
export const gitConnections = pgTable('git_connections', {
  id: uuid('id').primaryKey().defaultRandom(),
  spaceId: uuid('space_id').notNull().references(() => spaces.id, { onDelete: 'cascade' }),
  provider: varchar('provider', { length: 20 }).notNull(), // github, gitlab
  repoUrl: text('repo_url').notNull(),
  branch: varchar('branch', { length: 255 }).notNull().default('main'),
  syncDirection: varchar('sync_direction', { length: 20 }).notNull().default('bidirectional'),
  accessTokenEncrypted: text('access_token_encrypted').notNull(),
  webhookSecretEncrypted: text('webhook_secret_encrypted'),
  lastSyncAt: timestamp('last_sync_at', { withTimezone: true }),
  syncStatus: varchar('sync_status', { length: 20 }).notNull().default('idle'),
  syncError: text('sync_error'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const gitSyncLog = pgTable('git_sync_log', {
  id: uuid('id').primaryKey().defaultRandom(),
  gitConnectionId: uuid('git_connection_id').notNull().references(() => gitConnections.id, { onDelete: 'cascade' }),
  direction: varchar('direction', { length: 10 }).notNull(), // push, pull
  commitSha: varchar('commit_sha', { length: 40 }),
  filesChanged: integer('files_changed').notNull().default(0),
  status: varchar('status', { length: 20 }).notNull(),
  errorMessage: text('error_message'),
  startedAt: timestamp('started_at', { withTimezone: true }).notNull().defaultNow(),
  completedAt: timestamp('completed_at', { withTimezone: true }),
});
```

**Testing**:
- `Integration: migration creates git_connections and git_sync_log tables`
- `Integration: insert git_connection → succeeds with encrypted token`

#### 7.2 — Git Sync Service

**What**: Service for pushing documentation content to Git (as Markdown files) and pulling changes from Git into the page tree.

**Design**:

```typescript
// src/server/services/git-sync.ts
import { Octokit } from '@octokit/rest';

export interface GitSyncResult {
  direction: 'push' | 'pull';
  commitSha: string;
  filesChanged: number;
  status: 'success' | 'error';
  errorMessage?: string;
}

export async function pushToGit(connectionId: string): Promise<GitSyncResult> {
  const connection = await getGitConnection(connectionId);
  const octokit = new Octokit({ auth: decrypt(connection.accessTokenEncrypted) });

  // 1. Fetch all published pages for the space version
  // 2. Convert page tree to directory structure with Markdown files
  // 3. Create/update files in the repo via GitHub Contents API
  // 4. Create commit, log result
}

export async function pullFromGit(connectionId: string): Promise<GitSyncResult> {
  const connection = await getGitConnection(connectionId);
  const octokit = new Octokit({ auth: decrypt(connection.accessTokenEncrypted) });

  // 1. Fetch file tree from repo
  // 2. Parse Markdown files
  // 3. Diff against existing pages
  // 4. Create/update/delete pages and revisions
  // 5. Log result
}

export function pageTreeToFileStructure(pages: PageWithContent[]): Map<string, string> {
  // Convert page hierarchy to file paths: /getting-started/index.md, /getting-started/auth.md
  const files = new Map<string, string>();
  for (const page of pages) {
    const path = buildFilePath(page);
    const content = addFrontmatter(page);
    files.set(path, content);
  }
  return files;
}
```

**Testing**:
- `Unit: pageTreeToFileStructure → converts page tree to flat file map`
- `Unit: buildFilePath → nested pages get nested directory paths`
- `Unit: addFrontmatter → prepends YAML frontmatter with title, diataxis_type`
- `Integration (mocked Octokit): pushToGit → creates commit with correct files`
- `Integration (mocked Octokit): pullFromGit → creates pages from repo files`
- `Integration (mocked Octokit): pullFromGit with deleted file → marks page as deleted`
- `Integration: sync log → records commit SHA, files changed, status`

#### 7.3 — Git Sync BullMQ Worker and Webhook Handler

**What**: Background worker for async Git sync operations; webhook endpoint for GitHub/GitLab push events to trigger auto-sync.

**Design**:

```typescript
// src/server/queue/workers/git-sync.worker.ts
import { Worker } from 'bullmq';

export const gitSyncWorker = new Worker('git-sync', async (job) => {
  const { connectionId, direction } = job.data;
  if (direction === 'push') {
    return pushToGit(connectionId);
  } else {
    return pullFromGit(connectionId);
  }
}, { connection: redis });

// src/app/api/webhooks/git/route.ts
export async function POST(req: Request) {
  const signature = req.headers.get('x-hub-signature-256');
  const body = await req.text();

  // Verify webhook signature
  // Find git_connection by repo URL
  // Enqueue pull sync job
  return Response.json({ ok: true });
}
```

**Testing**:
- `Integration (mocked): git-sync worker processes push job → calls pushToGit`
- `Integration (mocked): git-sync worker processes pull job → calls pullFromGit`
- `Integration: webhook with valid signature → enqueues sync job`
- `Integration: webhook with invalid signature → returns 401, no job enqueued`
- `Integration: webhook for unknown repo → returns 404`

---

## Phase 8: AI Features — Semantic Search, Assistant, and Writing Suggestions

### Purpose
Add the AI-native differentiators: semantic search via embeddings, an in-doc Q&A assistant, and real-time writing suggestions. After this phase, readers get AI-powered answers and authors get writing assistance — the features that distinguish this platform from Docusaurus and Vitepress.

### Tasks

#### 8.1 — Database Schema: AI Tables

**What**: Create embeddings, ai_conversations, ai_messages, and drift_detections tables with pgvector.

**Design**:

```typescript
// src/server/db/schema/ai.ts
import { pgTable, uuid, integer, text, varchar, jsonb, timestamp, uniqueIndex, index, customType } from 'drizzle-orm/pg-core';

const vector = customType<{ data: number[] }>({
  dataType() { return 'vector(1536)'; },
});

export const embeddings = pgTable('embeddings', {
  id: uuid('id').primaryKey().defaultRandom(),
  pageId: uuid('page_id').notNull().references(() => pages.id, { onDelete: 'cascade' }),
  chunkIndex: integer('chunk_index').notNull(),
  chunkText: text('chunk_text').notNull(),
  embedding: vector('embedding'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueChunk: uniqueIndex('idx_embeddings_unique').on(table.pageId, table.chunkIndex),
  vectorIdx: index('idx_embeddings_vector').using('ivfflat', table.embedding),
}));

export const aiConversations = pgTable('ai_conversations', {
  id: uuid('id').primaryKey().defaultRandom(),
  spaceId: uuid('space_id').notNull().references(() => spaces.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').references(() => users.id),
  sessionId: varchar('session_id', { length: 100 }).notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export const aiMessages = pgTable('ai_messages', {
  id: uuid('id').primaryKey().defaultRandom(),
  conversationId: uuid('conversation_id').notNull().references(() => aiConversations.id, { onDelete: 'cascade' }),
  role: varchar('role', { length: 20 }).notNull(), // user, assistant
  content: text('content').notNull(),
  sources: jsonb('sources'), // referenced page IDs and snippets
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export const driftDetections = pgTable('drift_detections', {
  id: uuid('id').primaryKey().defaultRandom(),
  spaceId: uuid('space_id').notNull().references(() => spaces.id, { onDelete: 'cascade' }),
  pageId: uuid('page_id').references(() => pages.id, { onDelete: 'set null' }),
  commitSha: varchar('commit_sha', { length: 40 }).notNull(),
  driftType: varchar('drift_type', { length: 50 }).notNull(),
  severity: varchar('severity', { length: 20 }).notNull(),
  description: text('description').notNull(),
  suggestedFix: text('suggested_fix'),
  status: varchar('status', { length: 20 }).notNull().default('open'),
  resolvedBy: uuid('resolved_by').references(() => users.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  resolvedAt: timestamp('resolved_at', { withTimezone: true }),
});
```

**Testing**:
- `Integration: migration enables pgvector extension and creates tables`
- `Integration: insert embedding with 1536-dimension vector → succeeds`
- `Integration: cosine similarity query → returns nearest neighbors`

#### 8.2 — Embedding Generation Service

**What**: Chunk page content, generate embeddings via OpenAI, and store in pgvector. BullMQ worker triggers on page update.

**Design**:

```typescript
// src/server/services/ai/embeddings.ts
import { OpenAI } from 'openai';

const CHUNK_SIZE = 500; // tokens
const CHUNK_OVERLAP = 50;

export interface Chunk {
  index: number;
  text: string;
  pageId: string;
}

export function chunkPageContent(pageId: string, title: string, body: string): Chunk[] {
  const plainText = `${title}\n\n${stripMarkdown(body)}`;
  const sentences = splitIntoSentences(plainText);
  const chunks: Chunk[] = [];
  let current = '';
  let index = 0;

  for (const sentence of sentences) {
    if (tokenCount(current + sentence) > CHUNK_SIZE && current.length > 0) {
      chunks.push({ index, text: current.trim(), pageId });
      index++;
      // Overlap: keep last few sentences
      const overlap = getOverlapText(current, CHUNK_OVERLAP);
      current = overlap + sentence;
    } else {
      current += sentence;
    }
  }
  if (current.trim()) {
    chunks.push({ index, text: current.trim(), pageId });
  }
  return chunks;
}

export async function generateAndStoreEmbeddings(pageId: string): Promise<void> {
  const page = await getPage(pageId);
  const revision = await getLatestRevision(pageId);
  if (!page || !revision) return;

  const chunks = chunkPageContent(pageId, page.title, revision.body);

  // Delete existing embeddings for this page
  await db.delete(embeddings).where(eq(embeddings.pageId, pageId));

  // Generate embeddings in batch
  const openai = new OpenAI();
  const response = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: chunks.map(c => c.text),
  });

  // Store embeddings
  for (let i = 0; i < chunks.length; i++) {
    await db.insert(embeddings).values({
      pageId,
      chunkIndex: chunks[i].index,
      chunkText: chunks[i].text,
      embedding: response.data[i].embedding,
    });
  }
}
```

**Testing**:
- `Unit: chunkPageContent splits long text into ~500-token chunks`
- `Unit: chunkPageContent preserves sentence boundaries`
- `Unit: chunkPageContent includes overlap between chunks`
- `Unit: chunkPageContent on short text → single chunk`
- `Integration (mocked OpenAI): generateAndStoreEmbeddings → creates embedding rows`
- `Integration (mocked OpenAI): re-generate → deletes old embeddings, creates new`

#### 8.3 — Semantic Search

**What**: Combine keyword search (tsvector) with semantic search (pgvector) for hybrid search results.

**Design**:

```typescript
// src/server/services/ai/semantic-search.ts
export async function hybridSearch(
  spaceVersionId: string,
  query: string,
  limit: number = 10
): Promise<SearchResult[]> {
  // Generate embedding for the query
  const openai = new OpenAI();
  const queryEmbedding = await openai.embeddings.create({
    model: 'text-embedding-3-small',
    input: query,
  });
  const vector = queryEmbedding.data[0].embedding;

  // Reciprocal Rank Fusion of keyword + semantic results
  const keywordResults = await searchPages(spaceVersionId, query, limit * 2);
  const semanticResults = await db.execute(sql`
    SELECT e.page_id, e.chunk_text,
           1 - (e.embedding <=> ${JSON.stringify(vector)}::vector) AS similarity
    FROM embeddings e
    JOIN pages p ON e.page_id = p.id
    WHERE p.space_version_id = ${spaceVersionId}
      AND p.is_published = true
    ORDER BY e.embedding <=> ${JSON.stringify(vector)}::vector
    LIMIT ${limit * 2}
  `);

  return reciprocalRankFusion(keywordResults, semanticResults.rows, limit);
}

function reciprocalRankFusion(
  keywordResults: SearchResult[],
  semanticResults: any[],
  limit: number,
  k: number = 60
): SearchResult[] {
  const scores = new Map<string, number>();
  keywordResults.forEach((r, i) => {
    scores.set(r.id, (scores.get(r.id) ?? 0) + 1 / (k + i + 1));
  });
  semanticResults.forEach((r, i) => {
    scores.set(r.page_id, (scores.get(r.page_id) ?? 0) + 1 / (k + i + 1));
  });
  // Sort by fused score, return top N
  return [...scores.entries()]
    .sort((a, b) => b[1] - a[1])
    .slice(0, limit)
    .map(([id]) => /* fetch full result */);
}
```

**Testing**:
- `Unit: reciprocalRankFusion merges two ranked lists correctly`
- `Unit: reciprocalRankFusion with disjoint lists → combines both`
- `Integration (mocked OpenAI): hybridSearch returns combined keyword + semantic results`
- `Integration (mocked OpenAI): hybridSearch with exact match → keyword ranks higher`
- `Integration (mocked OpenAI): hybridSearch with conceptual query → semantic ranks higher`

#### 8.4 — AI Q&A Assistant

**What**: In-doc chatbot that answers questions using RAG (retrieval-augmented generation) over the documentation.

**Design**:

```typescript
// src/server/services/ai/assistant.ts
import { OpenAI } from 'openai';

export interface AssistantResponse {
  answer: string;
  sources: Array<{ pageId: string; title: string; slug: string; snippet: string }>;
}

export async function askAssistant(
  spaceId: string,
  spaceVersionId: string,
  question: string,
  conversationHistory: Array<{ role: string; content: string }> = []
): Promise<AssistantResponse> {
  // 1. Retrieve relevant chunks via semantic search
  const relevant = await hybridSearch(spaceVersionId, question, 5);

  // 2. Build context from retrieved chunks
  const context = relevant.map(r =>
    `[Source: ${r.title}]\n${r.snippet}`
  ).join('\n\n---\n\n');

  // 3. Call LLM with context + conversation history
  const openai = new OpenAI();
  const response = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [
      {
        role: 'system',
        content: `You are a documentation assistant. Answer questions based ONLY on the provided documentation context. If the answer is not in the context, say "I don't have enough information in the documentation to answer that." Always cite your sources by referring to the source document titles.

Context:
${context}`,
      },
      ...conversationHistory.map(m => ({ role: m.role as 'user' | 'assistant', content: m.content })),
      { role: 'user', content: question },
    ],
    temperature: 0.1,
    max_tokens: 1000,
  });

  return {
    answer: response.choices[0].message.content ?? '',
    sources: relevant.map(r => ({
      pageId: r.id,
      title: r.title,
      slug: r.slug,
      snippet: r.snippet,
    })),
  };
}
```

```typescript
// src/components/docs/AiAssistant.tsx
'use client';

export function AiAssistant({ spaceId, spaceVersionId }: AiAssistantProps) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState('');
  const askMutation = trpc.ai.ask.useMutation();

  const handleSubmit = async () => {
    const question = input;
    setInput('');
    setMessages(prev => [...prev, { role: 'user', content: question }]);

    const response = await askMutation.mutateAsync({
      spaceId, spaceVersionId, question,
      history: messages,
    });

    setMessages(prev => [...prev, {
      role: 'assistant',
      content: response.answer,
      sources: response.sources,
    }]);
  };

  return (
    <div className="ai-assistant">
      <MessageList messages={messages} />
      <input value={input} onChange={e => setInput(e.target.value)} placeholder="Ask about the docs..." />
      <button onClick={handleSubmit}>Ask</button>
    </div>
  );
}
```

**Testing**:
- `Integration (mocked OpenAI): askAssistant → returns answer with sources`
- `Integration (mocked OpenAI): askAssistant with no relevant context → returns "I don't have enough information"`
- `Integration (mocked OpenAI): askAssistant stores conversation in ai_conversations + ai_messages`
- `E2E: open AI assistant widget → type question → answer appears with source links`
- `E2E: click source link → navigates to referenced documentation page`

#### 8.5 — AI Writing Suggestions

**What**: Real-time writing feedback for authors: clarity, completeness, tone, and terminology consistency.

**Design**:

```typescript
// src/server/services/ai/writing-suggestions.ts
export interface WritingSuggestion {
  type: 'clarity' | 'completeness' | 'tone' | 'terminology' | 'structure';
  severity: 'info' | 'warning';
  message: string;
  range?: { from: number; to: number };
  suggestion?: string;
}

export async function getWritingSuggestions(
  content: string,
  pageType: string,
  diatasisType: string | null
): Promise<WritingSuggestion[]> {
  const openai = new OpenAI();
  const response = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [
      {
        role: 'system',
        content: `You are a technical writing coach following the Diataxis framework. Analyze the documentation and return suggestions as JSON array.
Content type: ${pageType}
Diataxis type: ${diatasisType ?? 'unknown'}
Return format: [{ "type": "clarity|completeness|tone|terminology|structure", "severity": "info|warning", "message": "...", "suggestion": "..." }]`,
      },
      { role: 'user', content },
    ],
    response_format: { type: 'json_object' },
    temperature: 0.3,
  });

  return JSON.parse(response.choices[0].message.content ?? '[]');
}
```

**Testing**:
- `Integration (mocked OpenAI): getWritingSuggestions returns array of suggestions`
- `Integration (mocked OpenAI): tutorial content → suggests step-by-step structure`
- `Integration (mocked OpenAI): reference content → suggests parameter documentation`
- `Unit: suggestion types are valid enum values`

---

## Phase 9: Collaboration — Change Requests and Comments

### Purpose
Add review workflows for documentation changes: change requests (similar to pull requests), inline comments, and approval gates. After this phase, teams can review and discuss documentation changes before publishing.

### Tasks

#### 9.1 — Database Schema: Collaboration Tables

**What**: Create change_requests, change_request_pages, and comments tables.

**Design**:

```typescript
// src/server/db/schema/collaboration.ts
export const changeRequests = pgTable('change_requests', {
  id: uuid('id').primaryKey().defaultRandom(),
  spaceVersionId: uuid('space_version_id').notNull().references(() => spaceVersions.id, { onDelete: 'cascade' }),
  title: varchar('title', { length: 500 }).notNull(),
  description: text('description'),
  status: varchar('status', { length: 20 }).notNull().default('open'), // open, in_review, approved, merged, closed
  authorId: uuid('author_id').notNull().references(() => users.id),
  reviewerId: uuid('reviewer_id').references(() => users.id),
  mergedAt: timestamp('merged_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const changeRequestPages = pgTable('change_request_pages', {
  id: uuid('id').primaryKey().defaultRandom(),
  changeRequestId: uuid('change_request_id').notNull().references(() => changeRequests.id, { onDelete: 'cascade' }),
  pageId: uuid('page_id').notNull().references(() => pages.id, { onDelete: 'cascade' }),
  diffBody: text('diff_body').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export const comments = pgTable('comments', {
  id: uuid('id').primaryKey().defaultRandom(),
  pageId: uuid('page_id').references(() => pages.id, { onDelete: 'cascade' }),
  changeRequestId: uuid('change_request_id').references(() => changeRequests.id, { onDelete: 'cascade' }),
  authorId: uuid('author_id').notNull().references(() => users.id),
  parentId: uuid('parent_id').references((): any => comments.id, { onDelete: 'cascade' }),
  body: text('body').notNull(),
  resolved: boolean('resolved').notNull().default(false),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```

**Testing**:
- `Integration: migration creates collaboration tables`
- `Integration: insert change request → succeeds with status 'open'`
- `Integration: threaded comments → parent_id references work`
- `Integration: delete change_request → cascades to change_request_pages`

#### 9.2 — Change Request Workflow API

**What**: tRPC endpoints for creating, reviewing, approving, and merging change requests.

**Design**:

```typescript
export const collaborationRouter = router({
  createChangeRequest: protectedProcedure
    .input(z.object({
      spaceVersionId: z.string().uuid(),
      title: z.string().min(1).max(500),
      description: z.string().optional(),
      pages: z.array(z.object({
        pageId: z.string().uuid(),
        newBody: z.string(),
      })),
    }))
    .mutation(async ({ ctx, input }) => {
      // Create CR, compute diffs for each page, store in change_request_pages
    }),

  review: protectedProcedure
    .input(z.object({
      changeRequestId: z.string().uuid(),
      decision: z.enum(['approve', 'request_changes']),
      comment: z.string().optional(),
    }))
    .mutation(async ({ ctx, input }) => { /* ... */ }),

  merge: protectedProcedure
    .input(z.object({ changeRequestId: z.string().uuid() }))
    .mutation(async ({ ctx, input }) => {
      // Apply diffs as new revisions, update status to 'merged'
    }),

  addComment: protectedProcedure
    .input(z.object({
      changeRequestId: z.string().uuid().optional(),
      pageId: z.string().uuid().optional(),
      parentId: z.string().uuid().optional(),
      body: z.string().min(1),
    }))
    .mutation(async ({ ctx, input }) => { /* ... */ }),

  resolveComment: protectedProcedure
    .input(z.object({ commentId: z.string().uuid() }))
    .mutation(async ({ ctx, input }) => { /* ... */ }),
});
```

**Testing**:
- `Integration: create CR with 2 pages → creates CR + 2 change_request_pages with diffs`
- `Integration: review CR → updates status, records reviewer`
- `Integration: merge CR → applies diffs as new revisions, status = merged`
- `Integration: merge already-merged CR → returns error`
- `Integration: add threaded comment → parent_id links correctly`
- `Integration: resolve comment → sets resolved = true`

---

## Phase 10: Internationalisation, Analytics, and Integrations

### Purpose
Add multi-language support for documentation, page view analytics, Slack notifications, webhooks, and audit logging. After this phase, the platform supports international teams, provides usage insights, and integrates with external notification systems.

### Tasks

#### 10.1 — Translation Support

**What**: Database schema for languages and page_translations; API for adding/updating translations; UI for side-by-side translation editing.

**Design**:

```typescript
// src/server/db/schema/translations.ts
export const languages = pgTable('languages', {
  code: varchar('code', { length: 10 }).primaryKey(), // ISO 639-1
  name: varchar('name', { length: 100 }).notNull(),
  nativeName: varchar('native_name', { length: 100 }),
});

export const pageTranslations = pgTable('page_translations', {
  id: uuid('id').primaryKey().defaultRandom(),
  pageId: uuid('page_id').notNull().references(() => pages.id, { onDelete: 'cascade' }),
  languageCode: varchar('language_code', { length: 10 }).notNull().references(() => languages.code),
  title: varchar('title', { length: 500 }).notNull(),
  body: text('body').notNull(),
  bodyFormat: varchar('body_format', { length: 20 }).notNull().default('mdx'),
  translatedBy: varchar('translated_by', { length: 20 }).notNull().default('human'), // human, ai
  reviewed: boolean('reviewed').notNull().default(false),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniquePageLang: uniqueIndex('idx_page_translations_unique').on(table.pageId, table.languageCode),
}));
```

**Testing**:
- `Integration: add translation → creates page_translation row`
- `Integration: update translation → updates body and updatedAt`
- `Integration: duplicate page+language → raises unique constraint`
- `E2E: language switcher → renders page in selected language`
- `E2E: missing translation → falls back to default language`

#### 10.2 — Page View Analytics

**What**: Track page views, record search queries, and provide an analytics dashboard for space admins.

**Design**:

```typescript
// src/server/db/schema/analytics.ts
export const pageViews = pgTable('page_views', {
  id: uuid('id').primaryKey().defaultRandom(),
  pageId: uuid('page_id').notNull().references(() => pages.id, { onDelete: 'cascade' }),
  visitorId: varchar('visitor_id', { length: 100 }),
  userId: uuid('user_id').references(() => users.id),
  referrer: text('referrer'),
  userAgent: text('user_agent'),
  viewedAt: timestamp('viewed_at', { withTimezone: true }).notNull().defaultNow(),
});

export const feedback = pgTable('feedback', {
  id: uuid('id').primaryKey().defaultRandom(),
  pageId: uuid('page_id').notNull().references(() => pages.id, { onDelete: 'cascade' }),
  rating: integer('rating'), // 1-5
  comment: text('comment'),
  userId: uuid('user_id').references(() => users.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});
```

```typescript
// src/server/trpc/routers/analytics.ts
export const analyticsRouter = router({
  trackView: publicProcedure
    .input(z.object({
      pageId: z.string().uuid(),
      referrer: z.string().optional(),
    }))
    .mutation(async ({ ctx, input }) => { /* insert page_view */ }),

  getSpaceAnalytics: protectedProcedure
    .input(z.object({
      spaceId: z.string().uuid(),
      days: z.number().int().min(1).max(90).default(30),
    }))
    .query(async ({ ctx, input }) => ({
      totalViews: 0,
      uniqueVisitors: 0,
      topPages: [], // [{ pageId, title, views }]
      viewsByDay: [], // [{ date, views }]
      topReferrers: [],
      avgFeedbackRating: 0,
    })),

  submitFeedback: publicProcedure
    .input(z.object({
      pageId: z.string().uuid(),
      rating: z.number().int().min(1).max(5),
      comment: z.string().max(1000).optional(),
    }))
    .mutation(async ({ ctx, input }) => { /* insert feedback */ }),
});
```

**Testing**:
- `Integration: trackView → inserts page_view row`
- `Integration: getSpaceAnalytics → returns aggregated view counts`
- `Integration: getSpaceAnalytics topPages → sorted by view count descending`
- `Integration: submitFeedback → inserts feedback with rating`

#### 10.3 — Slack Integration and Webhooks

**What**: Slack notifications for page publishing, change request activity; generic webhooks for external integrations.

**Design**:

```typescript
// src/server/db/schema/integrations.ts
export const webhooks = pgTable('webhooks', {
  id: uuid('id').primaryKey().defaultRandom(),
  orgId: uuid('org_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  url: text('url').notNull(),
  secretEncrypted: text('secret_encrypted').notNull(),
  events: text('events').array().notNull(),
  active: boolean('active').notNull().default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const slackConnections = pgTable('slack_connections', {
  id: uuid('id').primaryKey().defaultRandom(),
  orgId: uuid('org_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  workspaceId: varchar('workspace_id', { length: 100 }).notNull(),
  channelId: varchar('channel_id', { length: 100 }).notNull(),
  accessTokenEncrypted: text('access_token_encrypted').notNull(),
  notifyOn: text('notify_on').array().notNull().default(['page.published', 'change_request.merged']),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const auditLog = pgTable('audit_log', {
  id: uuid('id').primaryKey().defaultRandom(),
  orgId: uuid('org_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  actorId: uuid('actor_id').references(() => users.id),
  action: varchar('action', { length: 100 }).notNull(),
  resourceType: varchar('resource_type', { length: 50 }).notNull(),
  resourceId: uuid('resource_id').notNull(),
  metadata: jsonb('metadata').notNull().default({}),
  ipAddress: text('ip_address'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});
```

```typescript
// src/server/services/webhooks.ts
export async function dispatchEvent(
  orgId: string,
  event: string,
  payload: Record<string, unknown>
): Promise<void> {
  // 1. Find active webhooks subscribed to this event
  // 2. Enqueue delivery jobs via BullMQ
  // 3. Find Slack connections subscribed to this event
  // 4. Send Slack message via Slack API
  // 5. Log to audit_log
}
```

**Testing**:
- `Integration: dispatchEvent → enqueues webhook delivery jobs for matching webhooks`
- `Integration: dispatchEvent → sends Slack message for matching connections`
- `Integration: webhook delivery with retry → retries on 5xx, gives up after 3 attempts`
- `Integration: audit_log → records action with actor, resource, metadata`
- `Integration: inactive webhook → not triggered`

#### 10.4 — Audit Log API

**What**: tRPC endpoint for querying the audit log with filters.

**Design**:

```typescript
export const auditRouter = router({
  list: protectedProcedure
    .input(z.object({
      orgId: z.string().uuid(),
      action: z.string().optional(),
      resourceType: z.string().optional(),
      actorId: z.string().uuid().optional(),
      startDate: z.date().optional(),
      endDate: z.date().optional(),
      limit: z.number().int().min(1).max(100).default(50),
      cursor: z.string().uuid().optional(),
    }))
    .query(async ({ ctx, input }) => {
      // Paginated, filtered audit log query
    }),
});
```

**Testing**:
- `Integration: list audit log → returns entries sorted by created_at desc`
- `Integration: filter by action → returns only matching entries`
- `Integration: filter by date range → returns entries within range`
- `Integration: pagination cursor → returns next page of results`

---

## Phase 11: Drift Detection and Code-to-Docs

### Purpose
Implement the AI-native differentiators that no incumbent offers end-to-end: automatic detection of documentation drift when code changes, and generation of first-draft documentation from source code. After this phase, the platform proactively alerts when docs are outdated.

### Tasks

#### 11.1 — Drift Detection Service

**What**: Analyse Git commits against published documentation to detect when code changes make docs outdated.

**Design**:

```typescript
// src/server/services/ai/drift-detection.ts
export interface DriftAlert {
  pageId: string | null;
  commitSha: string;
  driftType: 'api_change' | 'code_change' | 'dependency_update';
  severity: 'low' | 'medium' | 'high' | 'critical';
  description: string;
  suggestedFix: string | null;
}

export async function detectDrift(
  spaceId: string,
  commitSha: string,
  changedFiles: Array<{ path: string; diff: string }>
): Promise<DriftAlert[]> {
  // 1. Filter to relevant files (source code, API specs, configs)
  const relevantFiles = changedFiles.filter(f =>
    f.path.match(/\.(ts|js|py|go|rs|yaml|json)$/) &&
    !f.path.includes('test') && !f.path.includes('spec')
  );
  if (relevantFiles.length === 0) return [];

  // 2. For each changed file, find documentation pages that reference it
  //    (via semantic search on file paths, function names, etc.)

  // 3. Call LLM to analyse whether the code change invalidates the documentation
  const openai = new OpenAI();
  const response = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [
      {
        role: 'system',
        content: `You are a documentation drift detector. Given a code diff and related documentation, determine if the documentation is now outdated. Return JSON array of drift alerts.
Format: [{ "pageId": "...", "driftType": "api_change|code_change|dependency_update", "severity": "low|medium|high|critical", "description": "...", "suggestedFix": "..." }]`,
      },
      {
        role: 'user',
        content: `Code changes:\n${relevantFiles.map(f => `--- ${f.path} ---\n${f.diff}`).join('\n\n')}\n\nRelated documentation:\n${relatedDocs}`,
      },
    ],
    response_format: { type: 'json_object' },
    temperature: 0.1,
  });

  return JSON.parse(response.choices[0].message.content ?? '[]');
}
```

```typescript
// src/server/queue/workers/drift.worker.ts
import { Worker } from 'bullmq';

export const driftWorker = new Worker('drift-detection', async (job) => {
  const { spaceId, commitSha, changedFiles } = job.data;
  const alerts = await detectDrift(spaceId, commitSha, changedFiles);

  for (const alert of alerts) {
    await db.insert(driftDetections).values({
      spaceId,
      pageId: alert.pageId,
      commitSha,
      driftType: alert.driftType,
      severity: alert.severity,
      description: alert.description,
      suggestedFix: alert.suggestedFix,
    });
  }

  // Dispatch notifications for high/critical severity
  const severe = alerts.filter(a => ['high', 'critical'].includes(a.severity));
  if (severe.length > 0) {
    await dispatchEvent(spaceId, 'drift.detected', { alerts: severe });
  }
}, { connection: redis });
```

**Testing**:
- `Integration (mocked OpenAI): detectDrift with API endpoint change → returns api_change alert`
- `Integration (mocked OpenAI): detectDrift with test file change → returns empty (filtered out)`
- `Integration: drift worker → stores alerts in drift_detections table`
- `Integration: high severity drift → dispatches notification event`
- `Integration: drift dashboard API → returns open alerts sorted by severity`

#### 11.2 — Drift Detection Dashboard

**What**: UI for viewing, acknowledging, and resolving drift alerts.

**Design**:

```typescript
// src/server/trpc/routers/ai.ts (addition)
export const aiRouter = router({
  // ... existing endpoints

  listDriftAlerts: protectedProcedure
    .input(z.object({
      spaceId: z.string().uuid(),
      status: z.enum(['open', 'acknowledged', 'fixed', 'dismissed']).optional(),
    }))
    .query(async ({ ctx, input }) => { /* ... */ }),

  resolveDrift: protectedProcedure
    .input(z.object({
      driftId: z.string().uuid(),
      status: z.enum(['acknowledged', 'fixed', 'dismissed']),
    }))
    .mutation(async ({ ctx, input }) => {
      await ctx.db.update(driftDetections).set({
        status: input.status,
        resolvedBy: ctx.session.user.id,
        resolvedAt: new Date(),
      }).where(eq(driftDetections.id, input.driftId));
    }),
});
```

**Testing**:
- `E2E: drift dashboard → shows open alerts with severity badges`
- `E2E: click "Acknowledge" → status changes to acknowledged`
- `E2E: click "Dismiss" → alert removed from open list`
- `Integration: resolveDrift → updates status, resolvedBy, resolvedAt`

#### 11.3 — Code-to-Docs Generation

**What**: Generate first-draft documentation from source code files, OpenAPI specs, or inline comments.

**Design**:

```typescript
// src/server/services/ai/code-to-docs.ts
export interface GeneratedDoc {
  title: string;
  body: string;
  bodyFormat: 'mdx';
  suggestedDiatasisType: 'tutorial' | 'how_to' | 'reference' | 'explanation';
}

export async function generateDocsFromCode(
  sourceCode: string,
  language: string,
  targetType: 'reference' | 'tutorial' | 'how_to'
): Promise<GeneratedDoc> {
  const openai = new OpenAI();
  const response = await openai.chat.completions.create({
    model: 'gpt-4o',
    messages: [
      {
        role: 'system',
        content: `You are a technical documentation writer following the Diataxis framework. Generate ${targetType} documentation from the provided source code.
Output format: MDX with appropriate headings, code examples, parameter tables, and callouts.
Do NOT invent functionality — only document what exists in the code.`,
      },
      { role: 'user', content: `Language: ${language}\n\n${sourceCode}` },
    ],
    temperature: 0.3,
    max_tokens: 4000,
  });

  return {
    title: extractTitle(response.choices[0].message.content ?? ''),
    body: response.choices[0].message.content ?? '',
    bodyFormat: 'mdx',
    suggestedDiatasisType: targetType,
  };
}

export async function generateDocsFromOpenApi(
  spec: string
): Promise<GeneratedDoc[]> {
  const parsed = await parseOpenApiSpec(spec);
  const docs: GeneratedDoc[] = [];

  // Generate overview page
  docs.push(await generateOverviewDoc(parsed));

  // Generate per-tag group pages
  for (const tag of getUniqueTags(parsed.endpoints)) {
    docs.push(await generateTagDoc(tag, parsed.endpoints.filter(e => e.tags.includes(tag))));
  }

  return docs;
}
```

**Testing**:
- `Integration (mocked OpenAI): generateDocsFromCode with Python function → returns MDX reference doc`
- `Integration (mocked OpenAI): generateDocsFromCode with TypeScript class → returns class documentation`
- `Integration (mocked OpenAI): generateDocsFromOpenApi → returns overview + per-tag docs`
- `Unit: extractTitle → pulls first H1 from MDX content`
- `Unit: generateDocsFromOpenApi returns correct number of docs for tag count`

---

## Phase 12: Custom Domains, White-Label Branding, and Public REST API

### Purpose
Add enterprise-ready features: custom domains for published doc sites, theme customisation for white-label branding, and a public REST API for programmatic access. After this phase, the platform is ready for production use by organisations that need branded documentation portals.

### Tasks

#### 12.1 — Custom Domain Support

**What**: Configure custom domains per space; Next.js middleware for multi-tenant domain routing.

**Design**:

```typescript
// src/middleware.ts (extended)
export default auth(async (req) => {
  const hostname = req.headers.get('host') ?? '';

  // Check if this is a custom domain
  if (!hostname.endsWith(process.env.BASE_DOMAIN!)) {
    const space = await findSpaceByCustomDomain(hostname);
    if (space) {
      // Rewrite to docs route
      const url = req.nextUrl.clone();
      url.pathname = `/${space.org.slug}/${space.slug}/docs${req.nextUrl.pathname}`;
      return NextResponse.rewrite(url);
    }
    return NextResponse.json({ error: 'Unknown domain' }, { status: 404 });
  }
  // ... existing auth logic
});
```

**Testing**:
- `Integration: request to custom domain → rewrites to correct space docs`
- `Integration: request to unknown domain → returns 404`
- `Integration: custom domain with path → resolves to correct page`
- `E2E: visit docs.example.com → shows space documentation with custom branding`

#### 12.2 — Theme Customisation

**What**: UI for customising documentation site appearance: colours, fonts, logo, favicon, code theme.

**Design**:

```typescript
// Theme configuration schema
export const themeConfigSchema = z.object({
  primaryColor: z.string().regex(/^#[0-9a-fA-F]{6}$/).default('#2563eb'),
  accentColor: z.string().regex(/^#[0-9a-fA-F]{6}$/).default('#7c3aed'),
  backgroundColor: z.string().regex(/^#[0-9a-fA-F]{6}$/).default('#ffffff'),
  textColor: z.string().regex(/^#[0-9a-fA-F]{6}$/).default('#0f172a'),
  fontFamily: z.enum(['inter', 'roboto', 'open-sans', 'source-sans', 'system']).default('inter'),
  codeFontFamily: z.enum(['fira-code', 'jetbrains-mono', 'source-code-pro', 'system-mono']).default('fira-code'),
  codeTheme: z.enum(['github-dark', 'github-light', 'dracula', 'one-dark', 'nord']).default('github-dark'),
  logoUrl: z.string().url().optional(),
  faviconUrl: z.string().url().optional(),
  sidebarStyle: z.enum(['default', 'minimal', 'grouped']).default('default'),
  hideFooter: z.boolean().default(false),
  customCss: z.string().max(10000).optional(),
});

export type ThemeConfig = z.infer<typeof themeConfigSchema>;
```

**Testing**:
- `Unit: themeConfigSchema validates correct config → passes`
- `Unit: themeConfigSchema with invalid hex color → fails`
- `E2E: update theme → docs site reflects new colors and fonts`
- `E2E: custom logo → appears in sidebar header`
- `E2E: custom CSS → applied to docs pages`

#### 12.3 — Public REST API

**What**: Versioned REST API (v1) for programmatic access to spaces, pages, search, and specs. OpenAPI 3.1 spec auto-generated.

**Design**:

```typescript
// src/app/api/v1/spaces/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { authenticateApiToken } from '@/server/services/auth';

export async function GET(req: NextRequest) {
  const auth = await authenticateApiToken(req);
  if (!auth) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });

  const spaces = await listSpacesForOrg(auth.orgId);
  return NextResponse.json({
    data: spaces.map(s => ({
      id: s.id,
      name: s.name,
      slug: s.slug,
      visibility: s.visibility,
      created_at: s.createdAt,
    })),
  });
}

// API Token authentication
// src/server/db/schema/integrations.ts (api_tokens already defined in data model)
export const apiTokens = pgTable('api_tokens', {
  id: uuid('id').primaryKey().defaultRandom(),
  orgId: uuid('org_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').references(() => users.id, { onDelete: 'set null' }),
  name: varchar('name', { length: 255 }).notNull(),
  tokenHash: text('token_hash').notNull().unique(),
  scopes: text('scopes').array().notNull().default([]),
  expiresAt: timestamp('expires_at', { withTimezone: true }),
  lastUsedAt: timestamp('last_used_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});
```

Public API endpoints:
- `GET /api/v1/spaces` — list spaces
- `GET /api/v1/spaces/:id` — get space details
- `GET /api/v1/spaces/:id/pages` — list pages in a space version
- `GET /api/v1/pages/:id` — get page content
- `PUT /api/v1/pages/:id` — update page content
- `POST /api/v1/pages` — create page
- `DELETE /api/v1/pages/:id` — delete page
- `GET /api/v1/search` — search docs
- `POST /api/v1/specs` — upload OpenAPI spec

**Testing**:
- `Integration: GET /api/v1/spaces with valid token → returns 200 with spaces list`
- `Integration: GET /api/v1/spaces without token → returns 401`
- `Integration: GET /api/v1/spaces with expired token → returns 401`
- `Integration: GET /api/v1/pages/:id → returns page content as JSON`
- `Integration: PUT /api/v1/pages/:id → updates content, creates revision`
- `Integration: token scopes → read-only token cannot PUT`
- `E2E: generate API token → use in curl command → returns data`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (Auth, Schema, Org Mgmt)      ─── required by everything
    │
Phase 2: Spaces & Pages                           ─── requires Phase 1
    │
Phase 3: Editor & Doc Renderer                    ─── requires Phase 2
    │
    ├── Phase 4: Search & Navigation               ─── requires Phase 2 (can parallel with Phase 3)
    │
    └── Phase 5: Permissions & Access Control       ─── requires Phase 1 (can parallel with Phases 3-4)
         │
Phase 6: OpenAPI / API Documentation              ─── requires Phases 2, 3
    │
Phase 7: Git Integration                          ─── requires Phase 2 (can parallel with Phase 6)
    │
Phase 8: AI Features (Search, Assistant, Writing)  ─── requires Phases 2, 4
    │
Phase 9: Collaboration (CRs, Comments)            ─── requires Phases 2, 5
    │
    ├── Phase 10: i18n, Analytics, Integrations    ─── requires Phases 2, 5 (can parallel with 8, 9)
    │
    └── Phase 11: Drift Detection & Code-to-Docs   ─── requires Phases 7, 8
         │
Phase 12: Custom Domains, Branding, REST API       ─── requires Phases 2, 5, 6
```

**Parallelism opportunities:**
- Phases 3, 4, and 5 can be developed concurrently after Phase 2
- Phases 6 and 7 can be developed concurrently after Phase 3
- Phases 8, 9, and 10 can be developed concurrently (they share Phase 2 dependency but cover independent features)
- Phase 12 can begin as soon as Phases 5 and 6 are complete

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented.
2. All unit tests pass (`pnpm test`).
3. All integration tests pass (`pnpm test:integration`).
4. ESLint passes with zero errors (`pnpm lint`).
5. Prettier formatting passes (`pnpm format:check`).
6. TypeScript compiles with zero errors (`pnpm typecheck`).
7. Docker build succeeds (`docker build .`).
8. All new features work end-to-end in the dev environment.
9. Database migrations generated and applied without errors (`pnpm db:migrate`).
10. New environment variables documented in `.env.example`.
11. New API endpoints documented (tRPC routers are self-documenting; REST endpoints have JSDoc).
12. No regressions in existing tests from previous phases.
