# Project Description: Blog App (Web + Android PWA)

## Goal

Ship a simple blog app as a PWA on the web (Vercel) and as an Android app via Bubblewrap/TWA. Data in a long-lived Postgres instance (Neon). Development in Docker.

## Core Features

* Account: sign up, login, logout, edit profile, delete account.
* Posts: create, edit, delete, list, view.
* Post fields: `title` (string), `body` (string).
* User fields: `name`, `email`, `password` (hashed).
* Auth: email + password with session cookies.
* Basic validation and access control: users can only manage their own posts and account.

## Non-Goals

* Rich text, media uploads, comments, roles, drafts, tags, likes. Omit until later.

## Stack

* Runtime: Next.js 14+ (App Router), TypeScript.
* UI: MUI 6.
* DB: Postgres (Neon).
* ORM: Prisma for Postgres.
* Auth: NextAuth Credentials provider or custom auth with secure cookies.
* Packaging: PWA + Bubblewrap → Trusted Web Activity (TWA) for Google Play.
* Infra: Vercel for web, Neon for DB, Google Play for Android.
* Dev: Docker for local dev environment.

## Architecture

* Next.js server actions/route handlers for API.
* Prisma client for data access.
* Stateless web tier on Vercel. Persistent state only in Postgres.
* Sessions stored in encrypted cookies; optional DB session table for invalidation.
* PWA with offline shell for public pages; gated content requires online auth.

## Data Model (Prisma)

```prisma
model User {
  id        String   @id @default(cuid())
  name      String
  email     String   @unique
  password  String   // bcrypt hash
  posts     Post[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Post {
  id        String   @id @default(cuid())
  title     String
  body      String
  authorId  String
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([authorId])
}
```

### SQL constraints (effective via Prisma)

* Unique email.
* FK `Post.authorId → User.id` with `ON DELETE CASCADE`.

## API (Next.js route handlers)

* `POST /api/auth/register` → create user.
* `POST /api/auth/login` → issue session cookie.
* `POST /api/auth/logout` → revoke session.
* `GET /api/me` → current user profile.
* `PATCH /api/me` → update `name` or `email`.
* `DELETE /api/me` → delete account.
* `GET /api/posts` → list current user’s posts; query `?author=<id>` for public listing.
* `POST /api/posts` → create post.
* `GET /api/posts/:id` → read post.
* `PATCH /api/posts/:id` → update own post.
* `DELETE /api/posts/:id` → delete own post.

All write routes require auth. Input validated server-side with Zod.

## PWA

* `manifest.webmanifest`: name, icons, theme/background colors, `start_url`, `scope`, `display: standalone`.
* Service Worker: precache app shell and static assets; network-first for API calls to avoid stale auth.
* HTTPS required on production.

## Android (Bubblewrap/TWA)

1. Ensure PWA is installable and passes Lighthouse PWA checks.
2. Set `assetlinks.json` on `https://<domain>/.well-known/assetlinks.json`.
3. `npx @bubblewrap/cli init` → configure package id, app name, icon, signing.
4. `bubblewrap build` → generate `.apk/.aab`.
5. Upload to Google Play Console. Use same signing key for updates.

## Deployment

* Vercel:

  * Build command: `next build`.
  * Env vars set in Vercel Project.
  * `DATABASE_URL` points to Neon pooled connection string.
* Neon:

  * Serverless project with a primary branch.
  * Prisma migrations applied via `prisma migrate deploy`.
* Play Store:

  * TWA build pipeline uses production domain only.

## Environment Variables

* `DATABASE_URL` (Neon Postgres connection string).
* `NEXTAUTH_SECRET` (if using NextAuth).
* `NEXTAUTH_URL` (prod and preview).
* `SESSION_SECRET` (if using custom auth).
* `NODE_ENV`, `VERCEL`, etc.

## Security

* Hash passwords with bcrypt (≥10 rounds).
* Session cookies: `HttpOnly`, `Secure`, `SameSite=Lax`, short TTL with rolling refresh.
* Rate limit auth endpoints (IP + user) with simple in-memory limiter on Vercel Edge or Upstash Redis if needed.
* Validate all inputs with Zod.
* Enforce ownership checks on post mutations and account updates.
* Avoid storing plaintext secrets in the repo.

## Docker (Dev)

* Single container with Node LTS and `libssl` compatible with Prisma.
* Services: Postgres via Neon, so no local DB needed. Optionally run `postgres` locally via docker-compose for offline dev.
* Hot reload with `next dev`.
* Example `Dockerfile` summary:

  * Base: `node:20-alpine`
  * Install deps, generate Prisma client, build, run `next`.
* Example `docker-compose.yml` summary:

  * `web` service build + port 3000.
  * Optional `db` service if not using Neon in dev.

## CI/CD

* GitHub Actions:

  * Lint, typecheck, test on PR.
  * On push to `main`: run `prisma migrate deploy`, then Vercel deploy via token.
* Block deploy if migrations fail.

## Testing

* Unit: Zod schemas, auth utilities.
* Integration: API routes with Supertest.
* E2E: Playwright for critical flows (register, login, CRUD posts).
* Lighthouse PWA and performance checks.

## Monitoring/Logs

* Vercel logs and errors.
* Prisma query logging in non-prod only.
* Optional Sentry for API and client.

## Acceptance Criteria

* Web app deploys on Vercel with working Neon DB.
* PWA installable and passes Lighthouse installability.
* Android TWA installs and launches to the same origin. Deep links verified via `assetlinks.json`.
* Users can register, login, logout.
* Users can edit and delete their own account.
* Users can create, edit, delete their own posts.
* Ownership and auth enforced server-side.
* Basic responsive MUI layout. No visual regressions on mobile and desktop.
* All endpoints validated and rate-limited.
* CI tests pass on `main`. Migrations are idempotent.

## Initial MUI Pages

* `/` public list of posts (author and date).
* `/login`, `/register`.
* `/dashboard` user’s posts.
* `/posts/new`, `/posts/:id/edit`, `/posts/:id`.
* `/settings` profile and account delete.

## Timeline Outline

* Week 1: scaffold Next.js, Prisma schema, auth, Neon setup.
* Week 2: CRUD UI with MUI, validation, tests.
* Week 3: PWA hardening, Bubblewrap TWA, Play Console submission.
* Week 4: polish, rate limiting, monitoring, docs.

## README Checklist

* Prereqs, env vars, local run with Docker, migration commands, deploy steps, Bubblewrap commands, Play Store notes, and rollback steps.
