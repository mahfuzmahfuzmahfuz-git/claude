# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Run all commands from the project root (`uigen/`):

```bash
npm run setup                        # First-time setup: install, generate Prisma client, run migrations
NODE_OPTIONS='--require ./node-compat.cjs' npx next dev --turbopack  # Start dev server (http://localhost:3000) — use this instead of npm run dev on Windows
npm run build                        # Production build
npm test                             # Run tests in watch mode
npm test -- --watchAll=false         # Run tests once (CI mode)
npm test -- --testPathPattern=Chat   # Run a single test file
npm run db:reset                     # Force reset the SQLite database
```

> `npm run dev` fails on Windows because the script uses Unix env-var syntax. Run the `NODE_OPTIONS=...` command directly instead.

## Architecture

UIGen is an AI-powered React component generator. Users describe a UI in natural language; Claude generates React + Tailwind components into a **virtual file system** and the result is rendered live in a sandboxed iframe.

### Request flow

1. User types a prompt → `ChatContext` (wraps Vercel AI SDK `useChat`) sends `POST /api/chat`
2. `/api/chat/route.ts` calls `streamText` with the Claude model and two tools: `str_replace_editor` and `file_manager`
3. Tool calls stream back to the client and are executed against the **FileSystemContext** (in-memory virtual FS)
4. On stream completion the API persists updated messages + file state to the project row in SQLite
5. `PreviewFrame` re-renders: files are Babel-transpiled client-side (via `@babel/standalone`), imports are rewritten to `esm.sh` CDN URLs, and the result is injected into an iframe via `srcdoc`

### Key directories

| Path | Purpose |
|------|---------|
| `src/app/` | Next.js 15 App Router — pages and API routes |
| `src/app/api/chat/route.ts` | AI streaming endpoint; tool execution; project persistence |
| `src/app/[projectId]/page.tsx` | Protected project page; loads project and hydrates contexts |
| `src/components/chat/` | Chat UI (interface, message list, input, markdown renderer) |
| `src/components/editor/` | Monaco code editor and file tree |
| `src/components/preview/` | Live iframe preview |
| `src/lib/contexts/` | `FileSystemContext` (virtual FS state) and `ChatContext` (AI SDK integration) |
| `src/lib/tools/` | AI tool implementations: `str-replace.ts`, `file-manager.ts` |
| `src/lib/transform/` | Babel JSX transform + CDN import mapping for preview |
| `src/lib/prompts/` | System prompt for AI generation |
| `src/lib/auth.ts` | JWT session management (HS256, 7-day, HTTP-only cookie) |
| `src/actions/` | Next.js Server Actions: auth (signUp/signIn/signOut), project CRUD |
| `prisma/` | SQLite schema; `dev.db` is the database file |

### Virtual file system

All generated files live in memory (`src/lib/file-system.ts`). Paths are always absolute (rooted at `/`). The FS serialises to/from JSON for database persistence. `FileSystemContext` exposes file operations to components and handles incoming tool-call results from the AI stream.

### AI model & fallback

- **Real:** `claude-haiku-4-5` via `@ai-sdk/anthropic` — requires `ANTHROPIC_API_KEY` in `.env`
- **Mock:** `MockLanguageModel` in `src/lib/provider.ts` — used automatically when no API key is set; generates static sample components

The AI is instructed to always create `/App.jsx` as the entry point and use `@/` path aliases and Tailwind for styling.

### Authentication

JWT-based with bcrypt password hashing. Sessions stored in HTTP-only cookies. Anonymous users get a temporary project (stored in sessionStorage); signing up converts it to a permanent project.

### Database

SQLite via Prisma. Two models: `User` and `Project`. `Project.messages` and `Project.data` are JSON strings holding chat history and file system state respectively. `userId` is nullable (anonymous projects).

### Environment variables

| Variable | Required | Notes |
|----------|----------|-------|
| `ANTHROPIC_API_KEY` | No | Falls back to mock provider if absent |
| `JWT_SECRET` | No | Defaults to a hardcoded dev key |

### Testing

Vitest + jsdom + React Testing Library. Tests live alongside source in `__tests__/` subdirectories. Coverage includes chat components, editor, contexts, virtual FS, and JSX transformer.
