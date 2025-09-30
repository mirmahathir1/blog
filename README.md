# Project Description: Blog App (Web + Android PWA)

## Goal

Ship a simple blog app as a PWA on the web (Vercel) and as an Android app via Bubblewrap/TWA. Data in a long-lived Postgres instance (Neon). Uses Docker.

## Core Features

* Account: sign up, login, logout, edit profile, delete account.
* Posts: create, edit, delete, list, view.
* Home: list all posts from all users (author and date).
* Post fields: `title` (string), `body` (string).
* User fields: `name`, `email`, `password` (hashed).
* Auth: email + password with session cookies.
* Basic validation and access control: users can only manage their own posts and account.

## Stack

* Runtime: Next.js 14+ (App Router), TypeScript.
* UI: MUI 6.
* DB: Postgres (Neon).
* ORM: Prisma for Postgres.
* Auth: Custom email/password authentication with secure cookies.
* Packaging: PWA + Bubblewrap → Trusted Web Activity (TWA) for Google Play.
* Infra: Vercel for web, Neon for DB, Google Play for Android.
* Dev: Docker for local environment.

## Architecture

* Next.js server actions/route handlers for API.
* Prisma client for data access.
* Stateless web tier on Vercel. Persistent state only in Postgres.
* Sessions stored in encrypted cookies.
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
* `GET /api/posts` → public list of posts (all users by default). Optional filters: `author=<id>`, `mine=1` (auth).
* `POST /api/posts` → create post.
* `GET /api/posts/:id` → read post.
* `PATCH /api/posts/:id` → update own post.
* `DELETE /api/posts/:id` → delete own post.

Read routes are public; all write routes require auth. Input validated server-side with Zod.

### API Conventions

- **Content-Type**: `application/json` for all request and response bodies.
- **Auth cookie**: `session` HttpOnly cookie on successful login; sent automatically on subsequent requests.
- **Timestamps**: ISO 8601 strings (UTC).
- **IDs**: Prisma `cuid()` strings.
- **Error format**:

```json
{
  "error": {
    "code": "string",          // machine-readable code
    "message": "string",       // human-readable message
    "details": { }              // optional, validation field errors, etc.
  }
}
```

### Schemas

- **User**

```json
{
  "id": "string",
  "name": "string",
  "email": "user@example.com",
  "createdAt": "2025-01-01T12:00:00.000Z",
  "updatedAt": "2025-01-01T12:00:00.000Z"
}
```

- **Post**

```json
{
  "id": "string",
  "title": "string",
  "body": "string",
  "authorId": "string",
  "createdAt": "2025-01-01T12:00:00.000Z",
  "updatedAt": "2025-01-01T12:00:00.000Z"
}
```

### Auth

- `POST /api/auth/register`
  - Request body:

```json
{
  "name": "string",
  "email": "user@example.com",
  "password": "string"          
}
```
  - Responses:
    - 201

```json
{ "user": { /* User */ } }
```
    - 400 (validation)

```json
{ "error": { "code": "VALIDATION_ERROR", "message": "...", "details": { "email": "Invalid" } } }
```
    - 409 (email exists)

```json
{ "error": { "code": "EMAIL_IN_USE", "message": "Email already registered" } }
```

- `POST /api/auth/login`
  - Request body:

```json
{ "email": "user@example.com", "password": "string" }
```
  - Responses:
    - 200 (sets `session` cookie)

```json
{ "user": { /* User */ } }
```
    - 401

```json
{ "error": { "code": "INVALID_CREDENTIALS", "message": "Invalid email or password" } }
```

- `POST /api/auth/logout`
  - Request: no body
  - Responses:
    - 204 (clears `session` cookie)
    - 401 if not authenticated

### Me

- `GET /api/me`
  - Request: none
  - Responses:
    - 200

```json
{ "user": { /* User */ } }
```
    - 401 if not authenticated

- `PATCH /api/me`
  - Request body (any subset):

```json
{ "name": "string", "email": "user@example.com" }
```
  - Responses:
    - 200

```json
{ "user": { /* User */ } }
```
    - 400 (validation)
    - 401 if not authenticated
    - 409 if email conflicts

- `DELETE /api/me`
  - Request: none
  - Responses:
    - 204
    - 401 if not authenticated

### Posts

- `GET /api/posts`
  - Query parameters:
    - `author` (string, optional): filter by author id
    - `mine` ("1", optional): if `1`, requires auth and returns only caller's posts
    - `limit` (integer, optional, default 20, max 100)
    - `cursor` (string, optional): pagination cursor (post id or encoded cursor)
  - Responses:
    - 200

```json
{
  "items": [ { /* Post */ } ],
  "nextCursor": "string|null"
}
```

- `POST /api/posts`
  - Request body:

```json
{ "title": "string", "body": "string" }
```
  - Responses:
    - 201

```json
{ "post": { /* Post */ } }
```
    - 400 (validation)
    - 401 if not authenticated

- `GET /api/posts/:id`
  - Path parameters: `id` (string)
  - Responses:
    - 200

```json
{ "post": { /* Post */ } }
```
    - 404 if not found

- `PATCH /api/posts/:id`
  - Path parameters: `id` (string)
  - Request body (any subset):

```json
{ "title": "string", "body": "string" }
```
  - Responses:
    - 200

```json
{ "post": { /* Post */ } }
```
    - 400 (validation)
    - 401 if not authenticated
    - 403 if not owner
    - 404 if not found

- `DELETE /api/posts/:id`
  - Path parameters: `id` (string)
  - Responses:
    - 204
    - 401 if not authenticated
    - 403 if not owner
    - 404 if not found

## PWA

* Make sure the app is developed as a PWS. Ensure PWA is installable and passes Lighthouse PWA checks.
* Upload to Google Play Console. Use same signing key for updates.

## Deployment

* Vercel:

  * Build command: `next build`.
  * Env vars set in Vercel Project.
  * `DATABASE_URL` points to Neon pooled connection string.
* Neon:

  * Serverless project with a primary branch.
  * Prisma migrations applied via `prisma migrate deploy`.
* Play Store:

  * TWA build pipeline uses the app's public domain only.

## Security

* Hash passwords with bcrypt (≥10 rounds).
* Session cookies: `HttpOnly`, `Secure`, `SameSite=Lax`, short TTL with rolling refresh.
* Rate limit auth endpoints (IP + user) with simple in-memory limiter on Vercel Edge or Upstash Redis if needed.
* Validate all inputs with Zod.
* Enforce ownership checks on post mutations and account updates.
* Avoid storing plaintext secrets in the repo.

## Docker

* Single container with Node LTS and `libssl` compatible with Prisma.
* Services: Postgres via Neon, so no local DB needed. Optionally run `postgres` locally via docker-compose for offline use.
* Hot reload with `next dev`.
* Example `Dockerfile` summary:

  * Base: `node:20-alpine`
  * Install deps, generate Prisma client, build, run `next`.
* Example `docker-compose.yml` summary:

  * `web` service build + port 3000.
  * Optional `db` service if not using Neon.

## Acceptance Criteria

* Web app deploys on Vercel with working Neon DB.
* PWA installable and passes Lighthouse installability.
* Android TWA installs and launches to the same origin.
* Users can register, login, logout.
* Users can edit and delete their own account.
* Users can create, edit, delete their own posts.
* Home page lists all users’ posts.
* Ownership and auth enforced server-side.
* Basic responsive MUI layout. No visual regressions on mobile and desktop.
* All endpoints validated and rate-limited.

## Initial MUI Pages

* `/` public list of all posts from all users (author and date).
* `/login`, `/register`.
* `/dashboard` user’s posts (uses `GET /api/posts?mine=1`).
* `/posts/new`, `/posts/:id/edit`, `/posts/:id`.
* `/settings` profile and account delete.
