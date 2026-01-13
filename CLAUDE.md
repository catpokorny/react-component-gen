# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (uses Turbopack)
npm run dev

# Run tests
npm test

# Run a single test file
npx vitest path/to/test.ts

# Run tests in watch mode
npx vitest --watch

# Lint
npm run lint

# Build for production
npm run build

# Reset database
npm run db:reset
```

## Architecture

UIGen is an AI-powered React component generator that creates components in a virtual file system and renders them in a live preview iframe.

### Core Flow

1. **User sends message** via ChatInterface -> ChatProvider -> `/api/chat` route
2. **AI generates code** using Claude (or MockLanguageModel if no API key) with `str_replace_editor` and `file_manager` tools
3. **Tool calls update VirtualFileSystem** - files never touch disk, stored in memory
4. **Preview renders** by transforming JSX via Babel, creating blob URLs, and loading in sandboxed iframe with import maps

### Key Abstractions

**VirtualFileSystem** (`src/lib/file-system.ts`): In-memory file system with CRUD operations, serialization for persistence, and text editor commands (view, create, str_replace, insert). All file paths are normalized to start with `/`.

**AI Tools** (`src/lib/tools/`):
- `str_replace_editor`: Create files, replace strings, insert lines (Claude-style editing)
- `file_manager`: Rename and delete files/folders

**JSX Transformer** (`src/lib/transform/jsx-transformer.ts`): Converts user JSX/TSX to executable code using Babel. Creates import maps with blob URLs for local files and esm.sh URLs for npm packages. Handles `@/` alias resolution.

**Provider** (`src/lib/provider.ts`): Returns Anthropic Claude (claude-haiku-4-5) if `ANTHROPIC_API_KEY` is set, otherwise returns MockLanguageModel with hardcoded responses.

### Context Providers

- **FileSystemProvider**: Manages VirtualFileSystem state, handles tool calls from AI
- **ChatProvider**: Wraps `@ai-sdk/react` useChat, sends file system state with each request, tracks anonymous work

### Data Persistence

- Authenticated users: Projects stored in SQLite via Prisma (messages + file system data as JSON strings)
- Anonymous users: Work tracked in sessionStorage for sign-up prompts
- Database schema is defined in `prisma/schema.prisma` - reference it to understand data structure

### Preview System

PreviewFrame loads generated components in a sandboxed iframe:
- Entry point detection: `/App.jsx` -> `/App.tsx` -> `/index.jsx` -> first `.jsx/.tsx` file
- Tailwind via CDN (`cdn.tailwindcss.com`)
- React 19 via esm.sh
- ErrorBoundary catches runtime errors

### Generated Code Requirements

The AI is prompted to:
- Always create `/App.jsx` as the entry point with default export
- Use `@/` import alias for local files (e.g., `@/components/Button`)
- Style with Tailwind CSS, not inline styles
- No HTML files - App.jsx is the entry point

## Code Style

- Use comments sparingly. Only comment complex code.
