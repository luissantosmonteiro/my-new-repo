# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

UIGen is a Next.js 15 (App Router) app that generates React components with an LLM (Claude via Vercel AI SDK) inside a **virtual, in-memory file system** — no files are ever written to disk for generated code. Users chat with the AI, which uses tool calls to create/edit files in the virtual FS, and results render live in an in-browser preview via Babel-transformed blob URLs.

## Commands

```bash
npm run setup       # install deps, generate Prisma client, run migrations — run this first
npm run dev          # start dev server (Turbopack) at localhost:3000
npm run build        # production build
npm run lint         # next lint
npm test             # run vitest test suite (jsdom environment)
npm run db:reset      # reset the SQLite dev database (destructive)
```

Run a single test file: `npx vitest run src/lib/__tests__/file-system.test.ts`
Run tests matching a name: `npx vitest run -t "test name"`

No `ANTHROPIC_API_KEY` is required to run the app — without one, `getLanguageModel()` (`src/lib/provider.ts`) falls back to a `MockLanguageModel` that returns canned static components (counter/form/card) instead of calling Claude. This is the default dev experience unless a key is added to `.env`.

## Architecture

### Virtual file system, not disk I/O
`src/lib/file-system.ts` (`VirtualFileSystem` class) implements an in-memory tree (`Map<path, FileNode>`) with file/directory CRUD, rename, and text-editor-style operations (`viewFile`, `createFileWithParents`, `replaceInFile`, `insertInFile`) modeled after Anthropic's text-editor tool interface. This class is the single source of truth for "files" in a project — there is no real filesystem access for generated code. It gets serialized to/from JSON (`serialize`/`deserializeFromNodes`) for persistence in Postgres/SQLite and for passing across the client/server boundary on each chat request.

### Chat request lifecycle (`src/app/api/chat/route.ts`)
1. Client sends `{ messages, files, projectId }` — `files` is the serialized virtual FS.
2. Server reconstructs a `VirtualFileSystem` from `files` for this request only (stateless per-request).
3. `streamText` (Vercel AI SDK) runs with two tools bound to that FS instance:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`) — view/create/str_replace/insert, mirrors Anthropic's text editor tool.
   - `file_manager` (`src/lib/tools/file-manager.ts`) — rename/delete.
4. On finish, if `projectId` is present and the user is authenticated, the updated messages + serialized FS are persisted to the `Project` row (`prisma`).
5. System prompt lives in `src/lib/prompts/generation.tsx` and is prepended with Anthropic prompt-caching (`cacheControl: ephemeral`).

### Client-side state: two React contexts
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) owns a `VirtualFileSystem` instance client-side, exposes CRUD helpers, and applies tool calls streamed back from the AI (`handleToolCall`) to keep the UI FS in sync with what the server's tools did.
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) wraps `@ai-sdk/react`'s `useChat`, posts to `/api/chat` with the serialized FS + `projectId`, and routes `onToolCall` into `FileSystemContext`.
- Anonymous (signed-out) work is tracked in `sessionStorage` via `src/lib/anon-work-tracker.ts` so it can be recovered/attached to a project after sign-up.

### Live preview (no disk, no bundler)
`src/lib/transform/jsx-transformer.ts` transpiles each virtual file with `@babel/standalone` (React + TypeScript presets), wraps the output as a `Blob` and creates an object URL. It builds an import map (`createImportMap`) that maps local paths (with and without extensions, with `@/` alias variants) and third-party bare specifiers to `esm.sh` CDN URLs, so the browser can resolve ES module imports natively via `<script type="importmap">`. `PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) renders this into an iframe. Missing imports get placeholder modules generated on the fly rather than failing the whole preview.

### Auth & persistence
- Auth is a custom JWT-in-httpOnly-cookie implementation (`jose`) in `src/lib/auth.ts` — no external auth provider. `createSession`/`getSession`/`deleteSession` for server components/actions, `verifySession` for middleware.
- `src/middleware.ts` gates `/api/projects` and `/api/filesystem` behind a valid session.
- Data layer is Prisma with SQLite (`prisma/schema.prisma`): `User` and `Project` (project stores `messages` and `data` as JSON strings — the serialized chat history and virtual FS respectively). Generated Prisma client is checked into `src/generated/prisma` (custom `output` path in schema) — don't hand-edit it, regenerate with `npx prisma generate`.
- Server actions for project CRUD live in `src/actions/`.

### Path aliases
`@/*` maps to `src/*` (see `tsconfig.json`). shadcn/ui components live in `src/components/ui` (style: "new-york", Tailwind v4, no config file — CSS-based Tailwind config in `src/app/globals.css`).

## Testing

Vitest with jsdom (`vitest.config.mts`), using `@testing-library/react`. Tests live alongside source in `__tests__` directories (e.g. `src/lib/__tests__/`, `src/components/chat/__tests__/`). No global test setup file is configured — imports like `@testing-library/jest-dom` matchers must be brought in per-test if needed.
