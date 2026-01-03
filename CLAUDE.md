# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. It uses Claude AI (via Anthropic API) to generate React components based on natural language descriptions. Components are created in a virtual file system and rendered in real-time in an iframe preview.

## Development Commands

### Initial Setup
```bash
npm run setup
```
Installs dependencies, generates Prisma client, and runs database migrations.

### Development
```bash
npm run dev
```
Starts Next.js development server with Turbopack on http://localhost:3000

```bash
npm run dev:daemon
```
Runs dev server in background, logs written to logs.txt

### Testing
```bash
npm test
```
Runs Vitest test suite with jsdom environment

### Database
```bash
npx prisma generate
```
Regenerates Prisma client (outputs to src/generated/prisma)

```bash
npx prisma migrate dev
```
Creates and applies new migration

```bash
npm run db:reset
```
Resets database and re-runs all migrations

### Build & Deploy
```bash
npm run build
```
Creates production build

```bash
npm start
```
Starts production server

```bash
npm run lint
```
Runs ESLint

## Architecture

### Core Concept: Virtual File System + AI Tools

The application uses a **virtual file system** (VFS) that exists only in memory and the database - no files are written to disk during component generation. The AI interacts with this VFS through two primary tools:

1. **str_replace_editor** - View, create, and edit files using string replacement operations
2. **file_manager** - Higher-level file operations (create, delete, rename, list)

The VFS is implemented as a tree structure with `FileNode` objects representing files and directories, maintained in a `Map<string, FileNode>`.

### Request Flow

1. **Client**: User sends chat message from ChatInterface component
2. **API Route** (`/api/chat/route.ts`):
   - Injects system prompt from `lib/prompts/generation.tsx`
   - Reconstructs VFS from serialized project data
   - Streams AI response using Vercel AI SDK
   - AI uses tools to manipulate VFS
   - On finish, persists updated VFS and messages to database
3. **Preview**: PreviewFrame component transforms VFS files to executable code

### Virtual File System (lib/file-system.ts)

The `VirtualFileSystem` class is the core data structure:
- Maintains a normalized path map of all files/directories
- Root is always `/`
- Supports CRUD operations: create, read, update, delete, rename
- Serialization: `serialize()` returns plain object for database storage
- Deserialization: `deserializeFromNodes()` rebuilds VFS from stored data

Key methods used by AI tools:
- `viewFile()` - Returns file contents with line numbers
- `createFileWithParents()` - Creates file and any missing parent directories
- `replaceInFile()` - String replacement editing (used by str_replace_editor)
- `insertInFile()` - Insert text at specific line number

### Preview System (lib/transform/jsx-transformer.ts)

Transforms VFS files into executable preview:
1. Uses `@babel/standalone` to transpile JSX/TSX to JavaScript
2. Creates import map for module resolution:
   - CDN imports for React, ReactDOM (esm.sh)
   - Blob URLs for user-created components
3. Generates HTML document with inline script that:
   - Imports transformed modules
   - Renders root component (App.jsx or index.jsx) using ReactDOM
4. Loaded in sandboxed iframe with `allow-scripts allow-same-origin`

### AI Provider (lib/provider.ts)

Implements dual-mode operation:
- **Production**: Uses Anthropic API (claude-haiku-4-5) when `ANTHROPIC_API_KEY` is set
- **Mock Mode**: Uses `MockLanguageModel` when no API key is present
  - Returns pre-defined component templates (Counter, Form, Card)
  - Simulates multi-step tool calling to demonstrate functionality
  - Limited to 4 steps to prevent repetition

### Database & Persistence (Prisma)

Schema: `User` ←→ `Project`
- Projects have nullable `userId` to support anonymous users
- `messages` field stores conversation as JSON array
- `data` field stores serialized VFS state as JSON object
- Prisma client generated to `src/generated/prisma` (not default location)

### Authentication (lib/auth.ts)

- JWT-based session management using `jose` library
- Sessions stored in httpOnly cookies (7-day expiration)
- Middleware protects `/api/projects` and `/api/filesystem` routes
- Anonymous usage supported - projects without `userId` are temporary

### Key Context Providers

**FileSystemContext** (`lib/contexts/file-system-context.tsx`):
- Wraps VFS instance for client components
- Provides `refreshTrigger` state to notify preview of changes
- Methods: `getAllFiles()`, `getNode()`, `createFile()`, etc.

### Component Structure

```
src/
├── app/
│   ├── api/chat/route.ts         # AI streaming endpoint
│   ├── [projectId]/page.tsx      # Project workspace page
│   └── main-content.tsx          # Main UI layout
├── components/
│   ├── chat/                     # ChatInterface, MessageList, MessageInput
│   ├── editor/                   # CodeEditor (Monaco), FileTree
│   ├── preview/                  # PreviewFrame (iframe with transformed code)
│   └── auth/                     # AuthDialog, SignInForm, SignUpForm
├── lib/
│   ├── file-system.ts            # VirtualFileSystem class
│   ├── provider.ts               # AI provider (real + mock)
│   ├── tools/                    # AI tool definitions
│   │   ├── str-replace.ts        # Text editor tool for AI
│   │   └── file-manager.ts       # File operations tool
│   ├── transform/
│   │   └── jsx-transformer.ts    # JSX→JS + import map generation
│   └── prompts/
│       └── generation.tsx        # System prompt for AI
└── actions/                      # Next.js server actions for projects
```

## Important Conventions

### File Paths in VFS
- All paths are absolute starting with `/`
- Root directory is `/`
- Example: `/App.jsx`, `/components/Button.jsx`

### Import Aliases
Generated components use `@/` import alias:
```javascript
import Button from '@/components/Button';
```
This is transformed to blob URLs in the preview system.

### Entry Points
Preview system looks for these files in order:
1. `/App.jsx`
2. `/App.tsx`
3. `/index.jsx`
4. `/index.tsx`
5. `/src/App.jsx`
6. `/src/App.tsx`

Must export default React component.

### AI Prompt Engineering
The system prompt (`lib/prompts/generation.tsx`) instructs AI to:
- Always create `/App.jsx` as entry point
- Use Tailwind CSS for styling (v4)
- Use `@/` import alias for local files
- Keep responses brief
- Operate on root route `/` of virtual filesystem

## Testing

Tests use Vitest + React Testing Library + jsdom:
- Test files located in `__tests__/` directories next to source
- Example: `src/components/chat/__tests__/MessageList.test.tsx`
- `vite-tsconfig-paths` plugin enables `@/` alias in tests

## Environment Variables

### Required for Production
None - the app runs without API key using mock provider

### Optional
- `ANTHROPIC_API_KEY` - Enables real AI generation (instead of mock responses)
- `JWT_SECRET` - Used for session signing (defaults to development-secret-key)

## Common Gotchas

1. **Prisma Client Location**: Generated to `src/generated/prisma`, not default `node_modules/.prisma/client`. Import from `@/generated/prisma`.

2. **VFS Serialization**: `FileNode.children` is a `Map` which doesn't serialize to JSON. Use `serialize()` method which strips Map children before storage.

3. **Preview Sandbox**: Iframe needs both `allow-scripts` and `allow-same-origin` for blob URL imports to work.

4. **Import Map Limitations**: Some npm packages may not work in preview due to import map resolution constraints. React, ReactDOM, and most simple libraries work via esm.sh CDN.

5. **Anonymous Projects**: Projects without `userId` are not persisted on finish. Only authenticated users get persistence.

6. **Tailwind CSS Version**: Uses Tailwind v4 (not v3), which has different configuration and usage patterns.
