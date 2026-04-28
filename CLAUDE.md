# CLAUDE.md

We're building the app described in @SPEC.md. Read that file for general architectural tasks or to check the exact database structure, tech stack or application architecture.

Keep your replies concise and focus on conveying key information. No unnessessay fluff and no long code snippets.

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
bun dev        # start dev server on http://localhost:3000
bun build      # production build
bun lint       # run ESLint
```

Use `bun` as the runtime and package manager (not npm/yarn/pnpm).

## Architecture

This is a **Next.js 16 App Router** note-taking application. The app is early-stage — the spec in `SPEC.md` defines what to build.

### Planned route structure

```
app/
  layout.tsx          # root layout (Geist fonts, TailwindCSS)
  page.tsx            # landing / redirect
  login/              # better-auth sign-in page
  register/           # better-auth sign-up page
  dashboard/          # authenticated notes list
  notes/[id]/         # TipTap editor for a single note
  public/[publicId]/  # read-only public note view
  api/
    auth/[...all]/    # better-auth handler (catch-all)
    notes/            # CRUD: GET/POST
    notes/[id]/       # GET/PUT/DELETE a note
    notes/[id]/share/ # POST: toggle public sharing
    public/[publicId] # GET: unauthenticated public note
```

### Data layer

- **SQLite** accessed via Bun's native SQLite client (`import { Database } from "bun:sqlite"`), raw SQL only — no ORM.
- Database file is local (e.g. `db.sqlite` at project root).
- Auth tables (`user`, `session`, `account`, `verification`) are managed by **better-auth** — do not rename their columns.
- `notes.content` stores TipTap's JSON output (`editor.getJSON()`).
- `notes.public_id` is a random string used in share URLs; the internal `notes.id` is never exposed on public routes.

### Authentication

**better-auth** handles email/password auth with session cookies. The catch-all route handler lives at `app/api/auth/[...all]/route.ts`. All `/api/notes/*` routes must validate the session; `/api/public/*` is unauthenticated.

### Editor

TipTap v3 (`@tiptap/react`, `@tiptap/starter-kit`) with extensions: Bold, Italic, Heading (h1–h3), Code, CodeBlock, BulletList, HorizontalRule. Content is saved as JSON, not HTML.

### Validation

Use **Zod** (already installed) for request body validation in API route handlers.
