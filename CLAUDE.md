# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Setup (first time)
npm run setup          # Install deps + Prisma generate + DB migrations

# Development
npm run dev            # Start dev server with Turbopack
npm run lint           # Run ESLint
npm run test           # Run all tests with Vitest
npx vitest run src/path/to/__tests__/file.test.tsx  # Run a single test file

# Database
npm run db:reset       # Reset and recreate the SQLite database

# Production
npm run build
npm run start
```

All dev/build/start commands require the `NODE_OPTIONS='--require ./node-compat.cjs'` flag (already baked into the npm scripts). Do not strip this — it provides necessary Node API shims.

Set `ANTHROPIC_API_KEY` in `.env` to use real Claude (Haiku 4.5). Without it, the app falls back to a mock provider that generates static example components.

## Code Style

Use comments sparingly — only for complicated business logic that isn't self-evident from the code itself.

## Architecture

UIGen is a Next.js 15 App Router application where users describe React components in natural language and Claude generates/edits them in real-time via a 3-panel IDE interface.

### Virtual File System

`src/lib/file-system.ts` implements an in-memory file system (`Map<string, FileNode>`) that never touches disk. All AI-driven file operations (create, edit, rename, delete) go through this. It serializes to/from JSON for database persistence. This is the core data structure — understanding it is essential for working on the file editing or preview systems.

### AI Code Generation Flow

```
User message → POST /api/chat
  → provider.ts selects Claude or MockLanguageModel
  → streams response with tool calls (str_replace_editor, file_manager)
  → file-system-context.tsx executes tool calls against VirtualFileSystem
  → PreviewFrame re-renders iframe with updated files
  → Database saves project state if user is authenticated
```

The API route (`src/app/api/chat/route.ts`) supports up to 40 tool steps per request (4 for mock), max 120s, and uses Anthropic's ephemeral prompt caching. Tool definitions live in `src/lib/tools/`.

### State Management

Two React contexts wrap the app:

- **`file-system-context.tsx`** — holds the `VirtualFileSystem` instance, exposes file CRUD, and processes tool calls from AI responses
- **`chat-context.tsx`** — wraps Vercel AI SDK's `useChat`, manages messages/streaming, dispatches tool call results back to the file system context

### Preview Rendering

`src/components/preview/PreviewFrame.tsx` renders generated code in a sandboxed iframe using `srcdoc`. Babel Standalone transforms JSX client-side. React is loaded from esm.sh via an import map. The iframe looks for an entry point in this order: `App.jsx` → `App.tsx` → `index.jsx` → `index.tsx`.

### Data Persistence

Prisma + SQLite (`prisma/dev.db`). The `Project` model stores `messages` and `data` (the serialized VirtualFileSystem) as JSON strings in text columns. Anonymous users' work is tracked in `localStorage` and can be merged into a project on sign-in.

### Authentication

JWT sessions via the `jose` library, stored as HTTP-only cookies (7-day expiry). Passwords are bcrypt-hashed. `src/middleware.ts` guards protected API routes. `src/lib/auth.ts` handles session creation/validation.

### UI Layout

`src/app/main-content.tsx` defines the 3-panel layout using `react-resizable-panels`:
- Left: Chat panel (MessageList + MessageInput)
- Right: Tabbed panel — Preview tab (PreviewFrame iframe) or Code tab (FileTree + Monaco CodeEditor)

Shadcn/ui components (New York style) live in `src/components/ui/`. Server actions for project CRUD are in `src/actions/`.
