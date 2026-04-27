# Technical Specification: Note Taking Web App

## 1. Overview

A web-based note-taking application that allows authenticated users to create, manage, and share rich-text notes.

### Core Features

- Authentication (sign up / login / logout)
- CRUD operations for notes
- Rich text editing (TipTap)
- Public sharing via unique links
- Toggle note visibility (private/public)

## 2. Tech Stack

### Frontend
- Next.js (App Router)
- TypeScript
- TailwindCSS
- TipTap Editor

### Backend
- Next.js Route Handlers (API layer)
- Bun runtime
- Raw SQL via Bun SQLite client

### Authentication
- better-auth

### Database
- SQLite (file-based)
- JSON storage for editor content

## 3. System Architecture

```
[ Browser ]
     |
     v
[ Next.js Frontend (App Router) ]
     |
     v
[ Route Handlers / API Layer ]
     |
     v
[ SQLite Database (Bun) ]
```

### Key Principles

- Server-side data access (no direct DB access from client)
- Stateless API routes
- JSON-based editor content
- Minimal abstraction (raw SQL)

## 4. Data Model

### 4.1 Auth Tables (managed by better-auth)

better-auth requires four tables with specific field names. These are created and managed by the library — do not rename columns.

#### `user`
```sql
CREATE TABLE user (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  emailVerified BOOLEAN NOT NULL DEFAULT FALSE,
  image TEXT,
  createdAt DATETIME NOT NULL,
  updatedAt DATETIME NOT NULL
);
```

#### `session`
```sql
CREATE TABLE session (
  id TEXT PRIMARY KEY,
  expiresAt DATETIME NOT NULL,
  token TEXT UNIQUE NOT NULL,
  createdAt DATETIME NOT NULL,
  updatedAt DATETIME NOT NULL,
  ipAddress TEXT,
  userAgent TEXT,
  userId TEXT NOT NULL REFERENCES user(id) ON DELETE CASCADE
);
```

#### `account`
```sql
CREATE TABLE account (
  id TEXT PRIMARY KEY,
  accountId TEXT NOT NULL,
  providerId TEXT NOT NULL,
  userId TEXT NOT NULL REFERENCES user(id) ON DELETE CASCADE,
  accessToken TEXT,
  refreshToken TEXT,
  idToken TEXT,
  accessTokenExpiresAt DATETIME,
  refreshTokenExpiresAt DATETIME,
  scope TEXT,
  password TEXT,
  createdAt DATETIME NOT NULL,
  updatedAt DATETIME NOT NULL
);
```

#### `verification`
```sql
CREATE TABLE verification (
  id TEXT PRIMARY KEY,
  identifier TEXT NOT NULL,
  value TEXT NOT NULL,
  expiresAt DATETIME NOT NULL,
  createdAt DATETIME,
  updatedAt DATETIME
);
```

> The `notes` table references `user(id)` (singular), matching better-auth's table name.

### 4.2 Notes Table

```sql
CREATE TABLE notes (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  title TEXT NOT NULL,
  content JSON NOT NULL,
  is_public BOOLEAN DEFAULT FALSE,
  public_id TEXT UNIQUE,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,

  FOREIGN KEY (user_id) REFERENCES users(id)
);
```

**Field Details**

| Field | Description |
|-------|-------------|
| `content` | TipTap JSON document |
| `public_id` | Random string used for share URLs |
| `is_public` | Toggles visibility |

## 5. API Design

**Base Route:** `/api`

### 5.1 Notes Endpoints

#### Create Note
```
POST /api/notes
```
Body:
```json
{
  "title": "My Note",
  "content": {}
}
```

#### Get All Notes (User)
```
GET /api/notes
```
Returns all notes for authenticated user.

#### Get Single Note
```
GET /api/notes/:id
```

#### Update Note
```
PUT /api/notes/:id
```

#### Delete Note
```
DELETE /api/notes/:id
```

#### Toggle Public Sharing
```
POST /api/notes/:id/share
```
**Behavior:**
- If not public → generate `public_id`, set `is_public = true`
- If public → set `is_public = false`, clear `public_id`

#### Get Public Note
```
GET /api/public/:public_id
```
No authentication required.

## 6. Authentication Flow

Using **better-auth**:

### Features
- Email/password login
- Session-based auth (cookies)

### Middleware
- Protect all `/api/notes/*` routes
- Public route exception: `/api/public/*`

## 7. Frontend Architecture

### App Router Structure

```
/app
  /login
  /register
  /dashboard
  /notes/[id]
  /public/[publicId]
```

### Pages

**Dashboard**
- List notes
- Create new note button

**Note Editor Page**
- Title input
- TipTap editor
- Save / delete / share toggle

**Public Note Page**
- Read-only view

## 8. TipTap Configuration

### Extensions
- StarterKit (with customization)
- Bold
- Italic
- Heading (levels 1–3)
- Code
- CodeBlock
- BulletList
- HorizontalRule

### Output Format
- JSON (stored in DB)

## 9. State Management

- Local component state for editor
- React Query or SWR (optional but recommended) for API data fetching

## 10. Security Considerations

### Input Validation
- Validate title length
- Validate JSON structure

### Authorization
Ensure user owns note before:
- read
- update
- delete

### Public Access
- Only allow access via `public_id`
- Avoid exposing internal `id`

### Rate Limiting _(optional)_
- Prevent abuse of public endpoints

## 11. Performance Considerations

```sql
CREATE INDEX idx_notes_user_id ON notes(user_id);
CREATE INDEX idx_notes_public_id ON notes(public_id);
```

- Debounced auto-save (client-side)
- Lazy loading notes list

## 12. Error Handling

Standard response format:

```json
{
  "error": "Message"
}
```

HTTP status codes:

| Code | Meaning |
|------|---------|
| `400` | Bad Request |
| `401` | Unauthorized |
| `403` | Forbidden |
| `404` | Not Found |
| `500` | Internal Server Error |

## 13. Deployment

### Environment
- Bun runtime
- Node-compatible hosting (e.g., Vercel with Bun support or custom server)

### Database
SQLite file persisted via:
- Volume (Docker), OR
- File storage on server

## 14. Future Enhancements

- Search (full-text index)
- Tags / folders
- Markdown export
- Version history
- Collaborative editing (WebSockets / CRDT)
- Rich embeds (images, links)
- Dark mode

## 15. Development Phases

### Phase 1: Core
- Auth
- Notes CRUD
- Editor integration

### Phase 2: Sharing
- Public links
- Public view page

### Phase 3: UX Improvements
- Autosave
- Loading states
- Error handling

## 16. Open Questions

- Autosave vs manual save?
- Max note size?
- Should deleting a note also invalidate public links permanently?
- Do you want soft deletes?
