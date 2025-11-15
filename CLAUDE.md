# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Gemini CLI is an open-source AI agent that brings the power of Gemini directly into the terminal. It's a TypeScript monorepo using npm workspaces, with a React-based terminal UI (Ink) for the frontend and a Node.js backend that interfaces with the Google Gemini API.

## Essential Build & Test Commands

**Before any commit:**
```bash
npm run preflight
```
This runs the complete validation suite: build, tests, type checking, and linting. All changes must pass this before submission.

**Development workflow:**
```bash
npm install              # Install dependencies
npm run build            # Build all packages
npm start                # Run CLI from source
npm run build:all        # Build including sandbox container
```

**Testing:**
```bash
npm run test                                 # Unit tests only
npm run test:e2e                            # Integration tests (no sandbox)
npm run test:integration:sandbox:docker     # Integration tests with Docker sandbox
npm run test:integration:sandbox:podman     # Integration tests with Podman sandbox
npm run test:scripts                        # Test the scripts directory
```

**Run a single test file:**
```bash
cd packages/cli  # or packages/core
npx vitest run path/to/test-file.test.ts
```

**Code quality:**
```bash
npm run lint         # Run ESLint
npm run lint:fix     # Auto-fix ESLint issues and format with Prettier
npm run format       # Format with Prettier only
npm run typecheck    # TypeScript type checking across all packages
```

**Debugging:**
```bash
npm run debug        # Start CLI with Node debugger (--inspect-brk)
DEV=true npm start   # Start with React DevTools support
DEBUG=1 gemini       # Debug sandbox container
```

## Repository Structure

```
packages/
├── cli/                    # Frontend: User-facing CLI, UI rendering, history
│   └── src/
│       ├── ui/            # React components (Ink-based terminal UI)
│       ├── commands/      # CLI command handlers
│       ├── services/      # CLI-specific services
│       └── gemini.tsx     # Main CLI entry point
│
├── core/                   # Backend: Gemini API client, tool execution, orchestration
│   └── src/
│       ├── tools/         # Built-in tools (file system, shell, web fetch, etc.)
│       ├── agents/        # Agent implementations
│       ├── services/      # Core services
│       ├── prompts/       # Prompt construction
│       ├── mcp/           # Model Context Protocol integration
│       └── index.ts       # Core API exports
│
├── a2a-server/            # Experimental: Agent-to-Agent server
├── test-utils/            # Testing utilities for temp file systems
└── vscode-ide-companion/  # VS Code extension for Gemini CLI

docs/                       # All project documentation
scripts/                    # Build, test, and development automation scripts
integration-tests/          # End-to-end integration tests
```

## Architecture Flow

**User Interaction Flow:**
1. **User Input** → `packages/cli` handles terminal input and displays UI
2. **CLI → Core** → CLI sends user prompt to core package
3. **Core → Gemini API** → Core constructs prompt with context, history, and tool definitions
4. **Gemini Response** → API returns text response or tool execution request
5. **Tool Execution** → Core executes tools (with user approval for write operations)
6. **Tool Result → API** → Results sent back to Gemini for final response
7. **Core → CLI** → Final response returned to CLI
8. **Display to User** → CLI renders formatted output in terminal

**Key Separation:**
- `packages/cli`: Owns user experience, rendering, input/output, theming
- `packages/core`: Owns API communication, tool registration/execution, state management
- Tools are modular extensions in `packages/core/src/tools/`

## TypeScript & React Conventions

**Type Safety:**
- Avoid `any` types—use `unknown` and type narrowing instead
- Avoid type assertions (`as Type`) except when absolutely necessary
- Use the `checkExhaustive()` helper in switch default clauses (from `packages/cli/src/utils/checks.ts`)

**Code Structure:**
- Prefer plain objects with TypeScript interfaces over classes
- Use ES module exports for public APIs; unexported = private
- Leverage array operators (`.map()`, `.filter()`, `.reduce()`) for immutability

**React (Ink UI):**
- Use functional components with Hooks only
- Keep render logic pure—no side effects in component body
- Use `useEffect` only for synchronization, not for state updates
- Never call `setState` inside `useEffect` (degrades performance)
- Include all dependencies in `useEffect` dependency arrays
- Omit `useMemo`, `useCallback`, `React.memo`—React Compiler handles optimization
- Use `useState` functional updates for state based on previous state: `setState(prev => prev + 1)`

**Testing (Vitest):**
- Test files co-located with source: `*.test.ts` (logic) or `*.test.tsx` (React components)
- Use `vi.mock()` for ES modules; place critical mocks at top of file
- Use `vi.hoisted()` for mock functions needed before `vi.mock` factory
- For React components, use `render()` from `ink-testing-library` and assert with `lastFrame()`
- Mock internal modules and external SDKs frequently: `@google/genai`, `@modelcontextprotocol/sdk`, `fs`, `os`, `child_process`
- Use `vi.useFakeTimers()` and `vi.advanceTimersByTimeAsync()` for async timer testing

## Important Notes

**Main Branch:** `main`

**Node.js Version:**
- Development: Use Node.js `~20.19.0` (specific version required due to dev dependencies; use nvm)
- Production: Any version `>=20` is acceptable

**Sandboxing:**
- macOS: Uses Seatbelt (`sandbox-exec`) by default
- Cross-platform: Set `GEMINI_SANDBOX=docker` or `GEMINI_SANDBOX=podman`
- Build sandbox: `npm run build:sandbox` (included in `build:all`)

**Import Restrictions:**
- ESLint enforces package boundaries—respect relative import restrictions between packages

**Flag Naming:**
- Use hyphens in flag names: `--my-flag` (not `--my_flag`)

**Comments:**
- Only write high-value comments; avoid using comments to communicate with users

**Documentation:**
- Update `/docs` for any user-facing changes (new commands, modified behavior, flags)
- Follow [Google Developer Documentation Style Guide](https://developers.google.com/style)

## Additional Context

**GEMINI.md vs CLAUDE.md:**
- This codebase uses `GEMINI.md` for detailed coding conventions and testing patterns. Consult it for:
  - Detailed TypeScript/React guidelines
  - Comprehensive testing patterns and mocking strategies
  - ES module best practices
  - Type narrowing and exhaustiveness checking
- `CLAUDE.md` (this file) focuses on high-level architecture and common commands for productivity.

**Contributing:**
- All PRs must link to an existing issue
- Keep PRs small and focused on a single change
- Run `npm run preflight` before submitting
- Follow [Conventional Commits](https://www.conventionalcommits.org/) for commit messages
- Use `/review-frontend <PR_NUMBER>` for automated frontend review (experimental)
