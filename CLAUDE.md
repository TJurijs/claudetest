# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps + DB migrations)
npm run setup

# Development server (Turbopack)
npm run dev

# Build
npm run build

# Lint
npm run lint

# Tests
npx vitest --run               # all tests (use npx, not npm test — vitest not in PATH on Windows)
npx vitest --run src/lib/__tests__/file-system.test.ts  # single test file

# Database
npx prisma migrate dev         # run new migrations
npm run db:reset               # reset and re-migrate
npx prisma generate            # regenerate client after schema changes
```

The dev server requires `NODE_OPTIONS='--require ./node-compat.cjs'` — this is already baked into all npm scripts.

## Architecture

### Overview
UIGen is a Next.js 15 App Router app where users describe React components in a chat interface and Claude generates them in real-time. The generated code lives entirely in an in-memory virtual filesystem — nothing is written to disk. Registered users get their projects (messages + filesystem state) persisted in SQLite via Prisma.

### Request/Response Flow
1. User types in chat → `ChatContext` sends to `POST /api/chat` with current VFS state (serialized)
2. API route reconstructs VFS, calls `streamText` with two tools: `str_replace_editor` and `file_manager`
3. Claude streams back tool calls that mutate the VFS
4. Tool calls are forwarded to the client via `onToolCall` → `FileSystemContext.handleToolCall`
5. On finish, if authenticated + projectId present, serialized messages + VFS are saved to `Project` in DB

### Key Abstractions

**Virtual File System** (`src/lib/file-system.ts`)
`VirtualFileSystem` class holds all generated files in a `Map<string, FileNode>`. Supports create/read/update/delete/rename, `serialize()` (for DB/API), and `deserializeFromNodes()` (to reconstruct). The singleton `fileSystem` export is used server-side in the API route; client-side a new instance is created per `FileSystemProvider`.

**FileSystemContext** (`src/lib/contexts/file-system-context.tsx`)
React context wrapping the VFS for the UI. `handleToolCall` dispatches incoming tool calls (`str_replace_editor`, `file_manager`) to mutate the VFS and triggers a re-render via `refreshTrigger`.

**ChatContext** (`src/lib/contexts/chat-context.tsx`)
Thin wrapper around Vercel AI SDK's `useChat`, wired to `/api/chat`. Serializes the current VFS state into every request body so the server has the latest file tree.

**AI Tools** (`src/lib/tools/`)
- `str_replace_editor`: Claude's primary editing tool — supports `view`, `create`, `str_replace`, `insert` commands on the VFS
- `file_manager`: Handles `delete` and `rename` operations

**JSX Transform** (`src/lib/transform/jsx-transformer.ts`)
Runs Babel standalone in the browser to transpile JSX/TSX files for the live preview. Handles missing imports by generating placeholder modules.

**Preview** (`src/components/preview/PreviewFrame`)
Renders the transformed VFS files in a sandboxed iframe. Resolves imports across virtual files using a custom module registry.

**Provider / Mock** (`src/lib/provider.ts`)
`getLanguageModel()` returns real `claude-haiku-4-5` if `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel` that generates static component code — so the app works without an API key.

### Auth
JWT-based sessions via `jose`, stored in an `httpOnly` cookie (`auth-token`). `src/lib/auth.ts` is `server-only`. Middleware (`src/middleware.ts`) guards `/api/projects` and `/api/filesystem`. Anonymous users can use the app but work is not persisted.

### Data Model (Prisma / SQLite)
```
User: id, email, password (bcrypt), projects[]
Project: id, name, userId?, messages (JSON string), data (JSON string — serialized VFS)
```
Prisma client is generated to `src/generated/prisma`. Always read `prisma/schema.prisma` before any DB work.

### UI Layout
`MainContent` (client component) renders a two-panel layout:
- Left: `ChatInterface` (chat history + input)
- Right: tabs switching between `PreviewFrame` (live preview) and a split `FileTree` / `CodeEditor` (Monaco)