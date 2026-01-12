# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. It allows users to describe components in natural language and see them generated in real-time with a live preview. The application uses a virtual file system (no files written to disk) and supports both authenticated and anonymous users.

## Commands

### Setup
```bash
npm run setup
```
Installs dependencies, generates Prisma client, and runs database migrations. Use this for initial setup.

### Development
```bash
npm run dev
```
Start the development server with Turbopack. Opens at http://localhost:3000

```bash
npm run dev:daemon
```
Start dev server in background, logs written to logs.txt

### Testing
```bash
npm test
```
Run all tests with Vitest

```bash
npm test -- <path-to-test-file>
```
Run a single test file

### Database
```bash
npx prisma generate
```
Regenerate Prisma client after schema changes (output: src/generated/prisma)

```bash
npx prisma migrate dev
```
Create and apply a new migration

```bash
npm run db:reset
```
Reset database to initial state

### Build & Lint
```bash
npm run build
```
Create production build

```bash
npm run lint
```
Run ESLint checks

## Architecture

### Core Components

**Virtual File System (src/lib/file-system.ts)**
- `VirtualFileSystem` class manages an in-memory file structure
- Files stored as Map<string, FileNode> with hierarchical directory support
- Methods: createFile, updateFile, deleteFile, rename, viewFile, replaceInFile
- Serialization/deserialization for persistence via Prisma
- All paths normalized to start with '/' (e.g., "/App.jsx", "/components/Button.jsx")

**AI Integration (src/app/api/chat/route.ts)**
- Streaming API endpoint using Vercel AI SDK
- Two modes: Anthropic API (when ANTHROPIC_API_KEY is set) or MockLanguageModel
- Tools provided to AI: `str_replace_editor` (view/create/edit files) and `file_manager` (rename/delete)
- OnFinish hook saves messages and file system state to database for authenticated users
- Uses prompt caching (ephemeral cache control) for system prompt

**JSX Transformation Pipeline (src/lib/transform/jsx-transformer.ts)**
- `transformJSX`: Uses Babel standalone to transpile JSX/TSX to browser-compatible JS
- `createImportMap`: Generates ES Module import map for browser
  - Maps React to esm.sh CDN (React 19)
  - Creates blob URLs for local files
  - Handles @/ alias (maps to root directory)
  - Auto-generates placeholder modules for missing imports
  - Handles third-party packages via esm.sh
  - Collects CSS imports and inlines them
- `createPreviewHTML`: Generates sandboxed iframe HTML with import map, TailwindCSS CDN, error boundary
- Syntax errors displayed prominently in preview with file path and location

**Preview System (src/components/preview/PreviewFrame.tsx)**
- Renders components in sandboxed iframe (allow-scripts, allow-same-origin, allow-forms)
- Entry point detection: looks for /App.jsx, /App.tsx, /index.jsx, or first .jsx/.tsx file
- Auto-refresh on file system changes via refreshTrigger from FileSystemContext
- Error states: firstLoad welcome screen, no files message, syntax error display

### Authentication & Data Model

**Auth System (src/lib/auth.ts)**
- JWT-based authentication using jose library
- 7-day sessions stored in httpOnly cookies
- Functions: createSession, getSession, deleteSession, verifySession
- JWT_SECRET from env (defaults to "development-secret-key" in dev)

**Database (prisma/schema.prisma)**
- SQLite with Prisma ORM
- User model: id, email, password (bcrypt hashed), timestamps
- Project model: id, name, userId (nullable), messages (JSON), data (JSON), timestamps
- Projects can be anonymous (userId null) or user-owned
- Cascade delete: deleting user removes their projects
- Prisma client generated to src/generated/prisma (not the default location)

### State Management

**FileSystemContext (src/lib/contexts/file-system-context.tsx)**
- React Context wrapping VirtualFileSystem
- `refreshTrigger`: incremented to trigger re-renders when files change
- Methods: getFile, getAllFiles, updateFile, createFile, deleteFile, renameFile
- Initializes from project.data if provided

**ChatContext (src/lib/contexts/chat-context.tsx)**
- Manages chat messages and streaming state
- Uses `useChat` from Vercel AI SDK
- Sends file system state with each message
- Handles projectId for persistence

### UI Layout

The main interface (src/app/main-content.tsx) uses a three-panel resizable layout:
1. **Left Panel**: Chat interface for component generation
2. **Right Panel**: Switchable between Preview and Code views
   - **Preview**: Live iframe showing rendered components
   - **Code**: File tree + Monaco code editor (read-only display)

## Key Conventions

### Import Paths
- All non-library imports use `@/` alias pointing to src directory
- Example: `import { VirtualFileSystem } from '@/lib/file-system'`
- In generated components: `import Button from '@/components/Button'`

### File System Paths
- All paths must start with `/` (e.g., `/App.jsx`, `/components/Header.jsx`)
- VirtualFileSystem normalizes paths automatically

### AI Tool Architecture
- `str_replace_editor`: Text editing tool based on Zod schema (view, create, str_replace, insert commands)
- `file_manager`: File operations tool (rename, delete commands)
- Tools operate on the VirtualFileSystem instance passed to them

### Mock Provider
- When no ANTHROPIC_API_KEY: MockLanguageModel generates static components
- Creates counter/form/card components based on keywords in prompt
- Simulates streaming with delays for realistic UX
- Limited to 4 steps (maxSteps) to prevent repetition

### Testing
- Vitest with React Testing Library and jsdom
- Tests in `__tests__` directories alongside components
- Example: src/components/chat/__tests__/ChatInterface.test.tsx
- Config: vitest.config.mts with tsconfigPaths plugin for @/ alias support

## Environment Variables

Required `.env` file in root:
```
ANTHROPIC_API_KEY=your-api-key-here    # Optional: uses mock provider if not set
JWT_SECRET=your-secret-key              # Optional: defaults to "development-secret-key"
```

## Tech Stack

- **Framework**: Next.js 15 with App Router, React 19
- **Language**: TypeScript
- **Styling**: Tailwind CSS v4
- **Database**: Prisma + SQLite (dev.db in prisma directory)
- **AI**: Anthropic Claude (claude-haiku-4-5) via Vercel AI SDK
- **Code Editor**: Monaco Editor via @monaco-editor/react
- **UI Components**: Radix UI primitives (dialog, tabs, scroll-area, etc.)
- **Authentication**: JWT with jose library, bcrypt for passwords
- **Transformation**: Babel standalone for JSX/TSX transpilation
