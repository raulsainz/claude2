# Technical Specification – Note Taking Web App

## 1. Overview

A web application where authenticated users can create, view, edit, delete, and publicly share rich-text notes. Notes are created with TipTap, stored as JSON in a SQLite database, and rendered in the browser with formatting.

### Core Features

- User authentication (sign up, login, logout) via better-auth
- Authenticated note management (CRUD)
- Rich text editor using TipTap with:
  - Bold, Italic
  - Heading levels (H1–H3) + normal text
  - Inline code + code blocks
  - Bullet lists
  - Horizontal rules
- Public sharing of notes via a public URL (toggle on/off)

### Tech Stack

- Next.js (App Router) + Bun runtime
- TypeScript
- TailwindCSS
- SQLite via Bun's SQLite client with raw SQL

---

## 2. Architecture

### 2.1 High-Level Architecture

- **Frontend & Backend:** Next.js (App Router)
  - Server components for data fetching
  - Client components for TipTap editor and interactive UI
  - Route Handlers (`app/api/.../route.ts`) for JSON APIs
- **Runtime:** Bun (for dev & production)
- **Database:** Single SQLite file (e.g. `data/app.db`) accessed via Bun's SQLite client
- **Auth:** better-auth integrated into Next.js (middleware + server helpers)

### 2.2 Application Layers

**Presentation layer**
- Next.js pages and components
- TailwindCSS for styling
- TipTap editor component

**API layer**
- REST-like JSON endpoints for notes CRUD & sharing

**Data access layer**
- Raw SQL queries executed via Bun's SQLite client
- A small helper module for DB access

---

## 3. Functional Requirements

### 3.1 Authentication

Users can:
- Register (email + password, minimum validation)
- Log in / log out

Authentication state is accessible on the server (for SSR) and on the client (for protected UI).

Unauthenticated users:
- Can access public shared note URLs (read-only)
- Cannot access the dashboard or personal notes

### 3.2 Notes Management (Authenticated)

**Create a new note:**
- Default title: `"Untitled note"`
- Default empty TipTap document

**View a list of own notes:**
- Show title, last updated at, shared status

**View a single note:**
- Load editor with stored TipTap JSON document

**Update a note:**
- Change title
- Change content (TipTap JSON)
- Auto-update `updated_at`

**Delete a note:**
- Hard delete

### 3.3 Note Sharing

Users can toggle note "public sharing":

**When enabled:**
- Note gets a unique public slug (e.g. `abcdef1234`)
- Accessible via `/p/{slug}` for anonymous users

**When disabled:**
- Public URL returns 404 / "Note not found"

**Public page rendering:**
- Reads note from DB by `public_slug`
- Shows title and content in read-only mode
- No editing or owner information necessary

---

## 4. Non-Functional Requirements

| Concern | Requirement |
|---------|-------------|
| **Performance** | Notes list & note view should load under ~300 ms for typical DB sizes |
| **Security** | All note operations are scoped to the authenticated user's `user_id`; public notes are read-only with no leaked private data |
| **Reliability** | Graceful handling of DB errors |
| **Maintainability** | Type-safe APIs and DB types; modularized DB and auth helpers |
| **UX** | Simple, minimal UI with keyboard-friendly editor |

---

## 5. Data Model & Database Schema (SQLite)

### 5.1 Tables

#### better-auth Core Tables

better-auth manages its own tables for authentication. The following tables must be created exactly as shown — field names are camelCase as required by better-auth's internal mapping.

##### `user`

```sql
CREATE TABLE user (
  id          TEXT    PRIMARY KEY,
  name        TEXT    NOT NULL,
  email       TEXT    NOT NULL UNIQUE,
  emailVerified INTEGER NOT NULL DEFAULT 0,
  image       TEXT,
  createdAt   TEXT    NOT NULL DEFAULT (datetime('now')),
  updatedAt   TEXT    NOT NULL DEFAULT (datetime('now'))
);
```

| Field | Type | Notes |
|-------|------|-------|
| `id` | TEXT | Primary key |
| `name` | TEXT | Display name |
| `email` | TEXT | Unique; used for login |
| `emailVerified` | INTEGER | `0` or `1` |
| `image` | TEXT | Optional avatar URL |
| `createdAt` | TEXT | ISO timestamp |
| `updatedAt` | TEXT | ISO timestamp |

##### `session`

```sql
CREATE TABLE session (
  id          TEXT NOT NULL PRIMARY KEY,
  userId      TEXT NOT NULL,
  token       TEXT NOT NULL UNIQUE,
  expiresAt   TEXT NOT NULL,
  ipAddress   TEXT,
  userAgent   TEXT,
  createdAt   TEXT NOT NULL DEFAULT (datetime('now')),
  updatedAt   TEXT NOT NULL DEFAULT (datetime('now')),
  FOREIGN KEY (userId) REFERENCES user(id) ON DELETE CASCADE
);
```

| Field | Type | Notes |
|-------|------|-------|
| `id` | TEXT | Primary key |
| `userId` | TEXT | FK → `user.id` (cascade delete) |
| `token` | TEXT | Unique session token |
| `expiresAt` | TEXT | ISO timestamp |
| `ipAddress` | TEXT | Optional |
| `userAgent` | TEXT | Optional |
| `createdAt` | TEXT | ISO timestamp |
| `updatedAt` | TEXT | ISO timestamp |

##### `account`

```sql
CREATE TABLE account (
  id                     TEXT NOT NULL PRIMARY KEY,
  userId                 TEXT NOT NULL,
  accountId              TEXT NOT NULL,
  providerId             TEXT NOT NULL,
  accessToken            TEXT,
  refreshToken           TEXT,
  accessTokenExpiresAt   TEXT,
  refreshTokenExpiresAt  TEXT,
  scope                  TEXT,
  idToken                TEXT,
  password               TEXT,
  createdAt              TEXT NOT NULL DEFAULT (datetime('now')),
  updatedAt              TEXT NOT NULL DEFAULT (datetime('now')),
  FOREIGN KEY (userId) REFERENCES user(id) ON DELETE CASCADE
);
```

| Field | Type | Notes |
|-------|------|-------|
| `id` | TEXT | Primary key |
| `userId` | TEXT | FK → `user.id` (cascade delete) |
| `accountId` | TEXT | Provider-assigned account ID (equals `userId` for credential accounts) |
| `providerId` | TEXT | e.g. `"credential"`, `"google"` |
| `accessToken` | TEXT | Optional |
| `refreshToken` | TEXT | Optional |
| `accessTokenExpiresAt` | TEXT | Optional ISO timestamp |
| `refreshTokenExpiresAt` | TEXT | Optional ISO timestamp |
| `scope` | TEXT | Optional |
| `idToken` | TEXT | Optional |
| `password` | TEXT | Hashed; only used for email/password auth |
| `createdAt` | TEXT | ISO timestamp |
| `updatedAt` | TEXT | ISO timestamp |

##### `verification`

```sql
CREATE TABLE verification (
  id         TEXT NOT NULL PRIMARY KEY,
  identifier TEXT NOT NULL,
  value      TEXT NOT NULL,
  expiresAt  TEXT NOT NULL,
  createdAt  TEXT NOT NULL DEFAULT (datetime('now')),
  updatedAt  TEXT NOT NULL DEFAULT (datetime('now'))
);
```

| Field | Type | Notes |
|-------|------|-------|
| `id` | TEXT | Primary key |
| `identifier` | TEXT | Target of the verification (e.g. email) |
| `value` | TEXT | The value/code to verify |
| `expiresAt` | TEXT | ISO timestamp |
| `createdAt` | TEXT | ISO timestamp |
| `updatedAt` | TEXT | ISO timestamp |

---

#### `notes`

```sql
CREATE TABLE notes (
  id           TEXT    NOT NULL PRIMARY KEY,
  user_id      TEXT    NOT NULL,
  title        TEXT    NOT NULL,
  content_json TEXT    NOT NULL,
  is_public    INTEGER NOT NULL DEFAULT 0,
  public_slug  TEXT    UNIQUE,
  created_at   TEXT    NOT NULL DEFAULT (datetime('now')),
  updated_at   TEXT    NOT NULL DEFAULT (datetime('now')),
  FOREIGN KEY (user_id) REFERENCES user(id) ON DELETE CASCADE
);
```

| Field | Type | Notes |
|-------|------|-------|
| `id` | TEXT | Primary key |
| `user_id` | TEXT | FK → `user.id` (cascade delete) |
| `title` | TEXT | Note title |
| `content_json` | TEXT | Stringified TipTap JSON document |
| `is_public` | INTEGER | `0` or `1` |
| `public_slug` | TEXT | Unique; `NULL` when not shared |
| `created_at` | TEXT | ISO timestamp |
| `updated_at` | TEXT | ISO timestamp |

### 5.2 Indexes

```sql
CREATE INDEX idx_notes_user_id    ON notes(user_id);
CREATE INDEX idx_notes_public_slug ON notes(public_slug);
CREATE INDEX idx_notes_is_public  ON notes(is_public);
```

---

## 6. Backend: DB & API Layer

### 6.1 Database Access Module

**File:** `lib/db.ts`

- Initialize Bun SQLite client with DB file (`app.db`)
- Export helper functions:
  - `getDb()` – returns singleton DB connection
  - `query<T>(sql, params?): T[]`
  - `get<T>(sql, params?): T | undefined`
  - `run(sql, params?)`

### 6.2 Note Repository Functions

**File:** `lib/notes.ts`

**TypeScript type:**

```typescript
export type Note = {
  id: string;
  userId: string;
  title: string;
  contentJson: string; // stringified TipTap JSON
  isPublic: boolean;
  publicSlug: string | null;
  createdAt: string;
  updatedAt: string;
};
```

**Repository functions:**

- `createNote(userId: string, data: { title?: string; contentJson?: string }): Promise<Note>`
- `getNoteById(userId: string, noteId: string): Promise<Note | null>`
- `getNotesByUser(userId: string): Promise<Note[]>`
- `updateNote(userId: string, noteId: string, data: Partial<{ title: string; contentJson: string }>): Promise<Note | null>`
- `deleteNote(userId: string, noteId: string): Promise<void>`
- `setNotePublic(userId: string, noteId: string, isPublic: boolean): Promise<Note | null>`
- `getNoteByPublicSlug(slug: string): Promise<Note | null>`

Each function enforces `user_id = ?` in SQL where applicable to prevent cross-user access.

---

## 7. API Design (Next.js Route Handlers)

Base path: `/api/notes`

### 7.1 Authentication

All `/api/notes` handlers (except public read) must check auth and return `401` if not authenticated. Implement a server helper `getCurrentUser()` / `getSession()` via better-auth.

### 7.2 Endpoints

#### `GET /api/notes`

List notes for the current user.

**Response 200:**
```json
[
  {
    "id": "note-id",
    "title": "My Note",
    "isPublic": true,
    "updatedAt": "2025-01-01T12:00:00Z"
  }
]
```

> `contentJson` is omitted from list responses for performance.

---

#### `POST /api/notes`

Create a new note.

**Request body:**
```json
{
  "title": "Optional title",
  "contentJson": { "type": "doc" }
}
```

- Defaults: `title = "Untitled note"`, `contentJson = empty TipTap doc`

**Response 201:** created Note object

---

#### `GET /api/notes/:id`

Get a single note owned by the current user.

**Responses:**
- `200` – full note including `contentJson`
- `404` – not found or not owned by user

---

#### `PUT /api/notes/:id`

Update note title and/or content.

**Request body:**
```json
{
  "title": "New title",
  "contentJson": { "type": "doc" }
}
```

**Responses:**
- `200` – updated Note object
- `404` – not found

---

#### `DELETE /api/notes/:id`

Delete a note.

**Responses:**
- `204` – success
- `404` – not found

---

#### `POST /api/notes/:id/share`

Toggle public sharing.

**Request body:**
```json
{ "isPublic": true }
```

**Behavior:**
- If `isPublic = true` and note has no `public_slug`, generate one via `nanoid()` (16+ chars)
- If `isPublic = false`, set `is_public = 0` and `public_slug = NULL`

**Response 200:**
```json
{
  "id": "note-id",
  "isPublic": true,
  "publicSlug": "abcdef1234"
}
```

### 7.3 Public Note Endpoint

#### `GET /api/public-notes/:slug`

Read-only access to a public note.

**Response 200:**
```json
{
  "title": "Public note",
  "contentJson": { "type": "doc" }
}
```

- `404` if slug not found or `is_public = 0`

> Alternatively, skip this endpoint and resolve the note directly in the `/p/[slug]` server component.

---

## 8. Frontend – Pages & Components

### 8.1 Routes

| Route | Auth | Description |
|-------|------|-------------|
| `/` | Public | Landing page with login/sign-up CTA |
| `/dashboard` | Required | List of user notes + "Create note" button |
| `/notes/[id]` | Required | Note editor (TipTap, title, share toggle, delete) |
| `/p/[slug]` | Public | Read-only public note view |

### 8.2 Layout & Navigation

- `app/layout.tsx` – global layout: header with app name, login/logout, theme toggle
- `app/(auth)/login` and `app/(auth)/register` – auth pages (if better-auth doesn't provide its own UI)

### 8.3 Components

**`components/NoteList.tsx`**
- Props: `notes: { id, title, updatedAt, isPublic }[]`
- Renders list with links to `/notes/[id]`

**`components/NoteEditor.tsx`**
- TipTap-based editor
- Controlled by parent (`onChange` updates state → API call)

**`components/ShareToggle.tsx`**
- Switch/checkbox for `isPublic`
- Shows public URL when enabled

**`components/DeleteNoteButton.tsx`**
- Confirms and calls the DELETE endpoint

**`components/PublicNoteViewer.tsx`**
- Renders TipTap content in read-only mode (`editable: false`)

---

## 9. TipTap Integration

### 9.1 Extensions

```typescript
import { useEditor, EditorContent } from '@tiptap/react';
import StarterKit from '@tiptap/starter-kit';
import Code from '@tiptap/extension-code';
import CodeBlock from '@tiptap/extension-code-block';

const editor = useEditor({
  extensions: [
    StarterKit.configure({
      heading: { levels: [1, 2, 3] },
    }),
    Code,
    CodeBlock,
  ],
  content: initialContentJson,
  onUpdate: ({ editor }) => {
    onChange(editor.getJSON());
  },
});
```

Content is stored as `JSON.stringify(json)` in the DB; load with `JSON.parse` and pass as `content`.

### 9.2 Toolbar Buttons

| Button | TipTap command |
|--------|---------------|
| Bold | `toggleBold()` |
| Italic | `toggleItalic()` |
| H1 / H2 / H3 | `toggleHeading({ level: n })` |
| Paragraph | `setParagraph()` |
| Bullet list | `toggleBulletList()` |
| Inline code | `toggleCode()` |
| Code block | `toggleCodeBlock()` |
| Horizontal rule | `setHorizontalRule()` |

---

## 10. Styling

- TailwindCSS configured in `tailwind.config.ts`
- Minimal design: neutral background, card-like note container, utility classes
- `@tailwindcss/typography` prose styles for read-only note rendering

---

## 11. Security Considerations

| Area | Approach |
|------|----------|
| **Auth enforcement** | `/dashboard` and `/notes/[id]` check auth server-side; API routes verify session |
| **Authorization** | Every note query in auth context filters by `user_id` |
| **Public slugs** | 16+ character random slug (nanoid) to prevent guessing |
| **XSS** | Primary data is TipTap JSON, not raw HTML; only TipTap's own renderer is used |
| **Rate limiting** | Optional per-IP or per-user rate limiting on API routes |

---

## 12. Development Workflow

1. Initialize Next.js app with Bun & TypeScript
2. Set up TailwindCSS
3. Integrate better-auth and session handling
4. Implement SQLite DB initialization (`scripts/init-db.ts` or a `.sql` file)
5. Build DB helpers and note repository
6. Implement `/api/notes` and sharing endpoints
7. Build dashboard and note editor pages
8. Integrate TipTap editor and toolbar
9. Implement public note page `/p/[slug]`
10. Add polish: loading states, toast messages, error handling
