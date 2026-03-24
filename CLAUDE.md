# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server (Next.js + Turbopack)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest test suite
npm run setup        # Install deps + generate Prisma + run migrations
npm run db:reset     # Reset SQLite database
```

Run a single test file: `npx vitest run src/components/chat/chat-input.test.tsx`

## Environment

Copy `.env.example` to `.env`. `ANTHROPIC_API_KEY` is optional — without it the app falls back to a mock model that generates static demo components. `JWT_SECRET` is required for auth.

## Architecture

UIGen is a Next.js 15 (App Router) app where users describe React components in chat and Claude generates them with live preview.

### Core Data Flow

```
User chat input
  → POST /api/chat (streaming, Vercel AI SDK)
  → Claude + tool calls (str_replace_editor, file_manager)
  → VirtualFileSystem state updated
  → FileTree + CodeEditor + PreviewFrame re-render
  → (if authenticated) saved to Prisma/SQLite
```

### Key Modules

**`/src/lib/file-system.ts`** — In-memory virtual file system (~515 lines). All generated code lives here, never on disk. Serialized as JSON into the `Project.data` column.

**`/src/lib/provider.ts`** — AI model provider (~519 lines). Contains both the real Anthropic Claude integration and a `MockLanguageModel` fallback. Handles streaming for both.

**`/src/lib/tools/`** — Two Claude tools: `str_replace_editor` (file edits via string replacement) and `file_manager` (create/delete files). These are what Claude calls to modify the virtual FS.

**`/src/lib/prompts/generation.tsx`** — System prompt. Instructs Claude to use `/App.jsx` as the entry point, write Tailwind CSS, and use `@/` aliases for imports between generated files.

**`/src/lib/contexts/`** — Two React contexts:
- `ChatProvider` — wraps Vercel AI SDK's `useChat`, owns message state
- `FileSystemProvider` — owns the `VirtualFileSystem` instance, processes tool call results from the stream

**`/src/app/api/chat/route.ts`** — The only API route. Streams Claude responses, persists project state to DB for authenticated users.

**`/src/app/[projectId]/`** — Dynamic route per project. Home page redirects to the user's first project (or creates one).

### Preview Rendering

The `PreviewFrame` component renders generated JSX in an iframe using Babel Standalone for in-browser transpilation. Components are compiled and executed at runtime — no build step needed for preview.

### Authentication

JWT sessions (7-day, HTTP-only cookies) managed in `/src/lib/auth.ts`. Auth is optional — anonymous users get full functionality, with `Project.userId` left null. The anonymous work tracker (`/src/lib/anon-work-tracker.ts`) prompts users to save before leaving.

### UI Structure

`MainContent` → `ResizablePanelGroup`:
- Left panel: `ChatInterface`
- Right panel (tabbed): Preview tab (`PreviewFrame`) | Code tab (`FileTree` + `CodeEditor`)

UI primitives are Radix UI + shadcn/ui under `/src/components/ui/`.

### Path Aliases

`@/*` maps to `./src/*` (tsconfig). Generated component files also use `@/` to import from each other within the virtual FS.
