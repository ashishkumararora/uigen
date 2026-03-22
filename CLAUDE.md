# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Development
npm run dev          # Start Next.js dev server with Turbopack
npm run build        # Production build
npm run lint         # ESLint

# Database
npm run setup        # Install deps + prisma generate + migrate dev
npm run db:reset     # Reset SQLite database (destructive)

# Testing
npm test                        # Run all tests (Vitest)
npm test -- file-system         # Run tests matching pattern
npm test -- --watch             # Watch mode
```

Node must be started with `NODE_OPTIONS='--require ./node-compat.cjs'` (already in all npm scripts via `node-compat.cjs` shim for Windows path compatibility).

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface; Claude generates and edits code in an in-memory virtual file system, which is immediately rendered in a sandboxed iframe.

### Data Flow

```
User chat message
  → ChatProvider (useAIChat from @ai-sdk/react)
    → POST /api/chat  { messages: UIMessage[], files: serialized VFS, projectId? }
      → streamText() with Claude + two tools
        → tool calls streamed back → FileSystemContext.handleToolCall()
          → VirtualFileSystem updated
            → PreviewFrame re-renders (Babel transpile → srcdoc iframe)
              → on finish: project saved to SQLite if authenticated
```

### Virtual File System

`src/lib/file-system.ts` — `VirtualFileSystem` is the core abstraction. All code lives in-memory; nothing is written to disk. The serialized state (JSON) is sent with every chat request so Claude always has the current file tree, and saved to the `projects.data` column on finish.

Claude writes to the VFS via two AI tools:
- `str_replace_editor` (`src/lib/tools/str-replace.ts`) — create, view, str_replace, insert
- `file_manager` (`src/lib/tools/file-manager.ts`) — rename, delete

### Preview System

`src/lib/transform/jsx-transformer.ts` transpiles the VFS using `@babel/standalone` (browser-side Babel). It builds an ES module import map where each file becomes a blob URL, then injects a self-contained HTML document into a sandboxed iframe via `srcdoc`. Entry point is `/App.jsx` by convention.

### State Management

Two React contexts:
- `FileSystemProvider` (`src/lib/contexts/file-system-context.tsx`) — owns the VFS instance, exposes `handleToolCall` which dispatches AI tool results to the VFS
- `ChatProvider` (`src/lib/contexts/chat-context.tsx`) — wraps `useAIChat` from `@ai-sdk/react` v3; manages `input` state locally (the v3 hook no longer returns `input`/`handleSubmit`); uses `DefaultChatTransport` with a function body so the latest serialized VFS is captured per request

### AI Provider

`src/lib/provider.ts` — returns `claude-haiku-4-5` when `ANTHROPIC_API_KEY` is set, otherwise falls back to a `MockLanguageModel` that generates a hardcoded component (useful for dev without an API key).

### Authentication & Persistence

JWT-based sessions via `src/lib/auth.ts`. Users/projects stored in SQLite via Prisma. `project.messages` and `project.data` are JSON strings (UIMessage[] and serialized VFS respectively). Anonymous users can generate components; their work is tracked in sessionStorage (`src/lib/anon-work-tracker.ts`) and can be migrated on sign-up.

## Key Conventions

- **Message format**: The app uses `ai` v6 which uses `UIMessage` (with `parts: []`) instead of the old `Message` type (with `content: string`). All message handling must use `UIMessage`.
- **API route**: `/api/chat` receives `UIMessage[]`, converts via `convertToModelMessages()` before passing to `streamText`, and returns `toUIMessageStreamResponse()`.
- **Imports**: Path alias `@/*` maps to `src/*`.
- **Styling**: Tailwind CSS v4 (PostCSS plugin). No inline styles.
- **Generated code convention**: Claude is instructed to always use `/App.jsx` as entry point and `@/` for cross-file imports within the virtual FS.

## Environment Variables

```
ANTHROPIC_API_KEY    # Optional — omit to use mock provider
JWT_SECRET           # Session signing key (has dev default)
```
