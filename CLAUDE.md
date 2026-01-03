# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. It uses Claude AI to generate React components on-demand, displays them in a live preview iframe, and stores projects in a SQLite database. The app supports both anonymous and authenticated users.

## Development Commands

### Setup
```bash
npm run setup
```
Installs dependencies, generates Prisma client, and runs database migrations. Run this first.

### Development
```bash
npm run dev              # Start dev server with Turbopack
npm run dev:daemon       # Start dev server in background, logs to logs.txt
```

### Testing
```bash
npm test                 # Run Vitest tests
npm test <file>          # Run specific test file
```

### Database
```bash
npx prisma migrate dev   # Create and apply new migration
npm run db:reset         # Reset database (WARNING: deletes all data)
npx prisma studio        # Open database GUI
```

### Build
```bash
npm run build            # Production build
npm start                # Start production server
npm run lint             # Run ESLint
```

## Architecture

### Virtual File System (VFS)
The core innovation is a **virtual file system** that exists entirely in-memory and in the database. No React component files are written to disk during generation.

- **Implementation**: `src/lib/file-system.ts` - The `VirtualFileSystem` class manages an in-memory tree structure of files and directories
- **Persistence**: VFS state is serialized to JSON and stored in the database's `Project.data` field
- **Tools**: AI agent modifies the VFS via two tools:
  - `str_replace_editor` (src/lib/tools/str-replace.ts): Create, view, edit files using string replacement
  - `file_manager` (src/lib/tools/file-manager.ts): Rename and delete files

### Live Preview System
React components are rendered in a sandboxed iframe using a sophisticated transformation pipeline:

1. **Transform**: Babel (via @babel/standalone) transforms JSX/TSX to JavaScript in the browser (src/lib/transform/jsx-transformer.ts)
2. **Import Map**: Creates ES module import map with blob URLs for local files and esm.sh URLs for npm packages
3. **Render**: Generates HTML document with import map, injects into sandboxed iframe (src/components/preview/PreviewFrame.tsx)

**Key details**:
- Entry point is `/App.jsx` (or App.tsx, index.jsx, etc.)
- Local imports use `@/` alias (e.g., `import Foo from '@/components/Foo'`)
- Tailwind CSS loaded via CDN
- CSS files collected and injected as `<style>` tags
- Syntax errors displayed in preview with file location

### AI Integration
- **Route**: `src/app/api/chat/route.ts` - POST endpoint handles streaming chat
- **System Prompt**: `src/lib/prompts/generation.tsx` - Instructions for AI agent
- **Provider**: `src/lib/provider.ts` - Uses Anthropic Claude or falls back to mock provider if no API key
- **Mock Provider**: When `ANTHROPIC_API_KEY` is not set, a mock provider generates static demo components (Counter, ContactForm, Card)

### Authentication & Projects
- **JWT-based auth**: `src/lib/auth.ts` - Session stored in HTTP-only cookie
- **Database**: Prisma with SQLite (prisma/schema.prisma)
  - `User`: email, password (bcrypt hashed)
  - `Project`: name, messages (JSON), data (serialized VFS), userId (nullable for anonymous)
- **Anonymous users**: Can use the app but projects aren't persisted
- **Authenticated users**: Projects saved to database, accessible via `/{projectId}` routes

### Project State Management
Messages and VFS state flow through the system:
1. Client maintains local state of messages and VFS
2. On each AI response, both are sent to `/api/chat`
3. API reconstructs VFS from serialized nodes
4. After streaming completes, `onFinish` callback saves to database
5. On page load, project data is hydrated from database via server actions (src/actions/)

### File Structure Patterns
- **Server Actions**: `src/actions/` - Server-side functions for data fetching (getUser, getProject, createProject)
- **Components**: Organized by feature (auth/, chat/, editor/, preview/, ui/)
- **Tests**: Co-located with components in `__tests__/` directories
- **Contexts**: React contexts in `src/lib/contexts/` (file-system-context.tsx provides VFS to client components)

## Important Development Notes

### Working with the VFS
When modifying VFS logic:
- The VFS must be serializable (uses plain objects, not Map, when serialized)
- `serialize()` returns flat `Record<string, FileNode>`
- `deserializeFromNodes()` reconstructs tree structure
- Path normalization is critical (handled by `normalizePath()`)

### Working with the AI Agent
The agent only has access to two tools. If you need to add new file operations:
1. Add method to `VirtualFileSystem` class
2. Expose via existing tool or create new tool
3. Update system prompt if needed

### Preview iframe considerations
- Sandbox mode requires both `allow-scripts` and `allow-same-origin` for ES module imports to work
- Import map must include all path variations (@/, /, without extension)
- Third-party packages are loaded from esm.sh with React 19

### Database Schema
The database schema is defined in `prisma/schema.prisma`. Reference it anytime you need to understand the structure of data stored in the database.

### Database migrations
When changing Prisma schema:
```bash
npx prisma migrate dev --name <migration_name>
npx prisma generate  # Regenerate client
```

### Environment Variables
- `ANTHROPIC_API_KEY`: Optional. App works without it using mock provider.
- `JWT_SECRET`: Used fowhyr session signing. Defaults to development key.

## Testing Strategy
- **Unit tests**: Vitest with React Testing Library
- **Test files**: `**/__tests__/*.test.tsx`
- **Environment**: jsdom for DOM simulation
- Components tested: ChatInterface, MessageList, MessageInput, MarkdownRenderer, FileTree

## Code Style
- Use comments sparingly. Only comment complex code.
