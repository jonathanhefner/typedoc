# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

TypeDoc is a documentation generator for TypeScript projects. It converts TypeScript source code into HTML or JSON documentation by leveraging the TypeScript compiler API.

## Build & Development Commands

| Command | Description |
|---------|-------------|
| `pnpm build` | Full build (TypeScript + locales + themes) |
| `pnpm build:tsc` | TypeScript compilation only |
| `pnpm test` | Fast tests (excludes `slow/` and `packages/` dirs) |
| `pnpm run test:full` | All tests with coverage |
| `pnpm run lint` | ESLint + dprint format check |
| `pnpx dprint fmt` | Auto-fix formatting |
| `pnpm run rebuild_specs` | Regenerate test spec files after output changes |

Run a single test file:
```bash
npx mocha --config .config/mocha.fast.json src/test/<file>.test.ts
```

Run tests matching a pattern:
```bash
npx mocha --config .config/mocha.fast.json --grep "pattern"
```

Tests run directly against TypeScript source (no build step needed) via `tsx` with the `typedoc-ts` import condition.

## Architecture

### Core Pipeline

**TypeScript Source → Converter → Reflection Model → Validation → Renderer/Serializer → Output**

The `Application` class (`src/lib/application.ts`) orchestrates this pipeline. CLI entry is `src/lib/cli.ts` which calls `convert()` → `validate()` → `generateOutputs()`.

### Key Modules

- **Converter** (`src/lib/converter/`): Transforms TypeScript compiler symbols/types into TypeDoc's reflection model. `symbols.ts` handles symbol-to-reflection conversion, `types.ts` handles type conversion. Comment parsing has its own sub-pipeline in `comments/` (lexer → parser → link resolver).

- **Models** (`src/lib/models/`): The reflection data model. Core hierarchy: `ProjectReflection` → `DeclarationReflection` → `SignatureReflection` → `ParameterReflection`. Types are a large union (`SomeType`) in `types.ts`.

- **Renderer** (`src/lib/output/`): Writes HTML from the model. Uses pluggable routers (6 built-in in `router.ts`) and themes. The renderer is event-driven; plugins hook into `PAGE_BEGIN`/`PAGE_END`/etc.

- **Default Theme** (`src/lib/output/themes/default/`): TSX-based templates using TypeDoc's **custom JSX runtime** (not React). The JSX factory is in `src/lib/utils-common/jsx.ts`. All `.tsx` files compile through this custom factory to produce HTML strings.

- **Serialization** (`src/lib/serialization/`): JSON serialization/deserialization of the reflection model, used for the Packages/Merge entry point strategies.

- **Options** (`src/lib/utils/options/`): All TypeDoc options are declared in `declaration.ts` with types, defaults, and validation. Four readers discover options from CLI args, `typedoc.json`, `package.json`, and `tsconfig.json`.

### Import Boundaries

ESLint enforces strict layering via import restrictions:
- `utils-common/` — zero external dependencies, no Node.js APIs (works in browser)
- `models/` — depends only on `#utils` and `#serialization` (no Node.js)
- `serialization/` — depends only on `#utils` and `#models`

Use the subpath imports (`#utils`, `#models`, `#serialization`, `#node-utils`) when importing across these boundaries.

### Event-Driven Plugin System

Both Converter and Renderer use `EventDispatcher`. Built-in plugins (in `converter/plugins/` and `output/plugins/`) and external plugins hook into these events. External plugins export a `load(app: Application)` function.

### Three Entry Point Strategies

1. **Normal** (Resolve/Expand) — direct conversion of entry points
2. **Packages** — converts each sub-package separately, serializes to JSON, then merges
3. **Merge** — merges pre-existing JSON files

## Code Conventions

- ESM-only (`"type": "module"`) with Node16 module resolution; internal imports use `.js` extensions
- Formatting: dprint (4-space indent, double quotes)
- TSX files use TypeDoc's custom JSX runtime, not React
- Test fixtures for converter tests are in `src/test/converter/` and `src/test/converter2/` (each with own tsconfig). The `converter2` directory is used for issue regression tests.
