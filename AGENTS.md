---
title: AI Engineer Coach Agent Instructions
description: Help AI agents understand the codebase structure, conventions, and how to contribute effectively
---

# AI Engineer Coach — Agent Instructions

This VS Code extension analyzes AI coding assistant usage by parsing session logs and visualizing insights across 11+ analytics domains. **Goal for agents**: understand the layered architecture, file naming conventions, and how to extend analyzers, parsers, or rules.

## Quick Start Commands

```bash
# Install and verify
npm install
npm run build

# Develop
npm run watch           # esbuild in watch mode (dev iteration)
npm test               # vitest run (CI mode, fast)
npm run test:watch    # vitest (interactive, dev)

# Quality gates
npm run check           # typecheck + lint + spellcheck + knip + test (all at once)
npm run typecheck
npm run lint
npm run spellcheck

# Run extension locally
code --install-extension ai-engineer-coach-*.vsix
```

Then open command palette → **AI Engineer Coach: Open Dashboard**.

## Architecture Overview

### High-Level Data Flow

```
Parsers (parse-worker.ts)
  ↓ [extracts sessions from 6+ harnesses]
Cache (in-memory + disk)
  ↓ [memoizes parsed results]
Analyzers (11 analyzer-*.ts instances)
  ↓ [compute domain-specific metrics]
Webview RPC (panel.ts)
  ↓ [marshals requests/responses]
Frontend Pages (30+ page-*.ts components)
```

### Core Components

| Component | Location | Purpose |
|-----------|----------|---------|
| **Parsers** | `src/core/parser-*.ts` | Extract sessions from VSCode, Claude, Xcode, Codex, OpenCode, CLI. Each has sync + async variants. |
| **Analyzers** | `src/core/analyzer-*.ts` | Dashboard, Production, Consumption, Timeline, Patterns, Workflows, Config, Insights, Flow, Context, Images. All inherit from `AnalyzerBase`. |
| **Rules** | `src/core/rules/*.md` | 45 markdown files defining anti-pattern detection via DSL (domain-specific language). |
| **Metrics** | `src/core/metrics/*.metric.md` | 10 computed metrics (token cost, etc.) compiled into DSL. |
| **Webview** | `src/webview/` | 30+ pages, RPC marshal (panel-rpc.ts), components. |
| **Workers** | `parse-worker.ts`, `warm-up-worker.ts`, `cache-write-worker.ts` | Offload CPU-bound work from extension host. |

### Type System

**Core types** live in `src/core/types/`:
- `session-types.ts` — Session, SessionRequest, Workspace
- `analytics-types.ts` — Dashboard output, metric results
- `config-types.ts` — Extension config
- All types are **strongly typed** (Zod schemas for validation)

## Key Patterns & Conventions

### Adding an Analyzer

1. **Create file**: `src/core/analyzer-{domain}.ts`
   - Extend `AnalyzerBase`
   - Use `this.filteredSessions(filter)` to respect date/workspace filtering
   - Implement `compute()` or `computeXyz()` method returning typed result

2. **Register**: Add to `Analyzer.ts` constructor (instantiate alongside other analyzers)

3. **Webview page**: Create `src/webview/page-{domain}.ts` to render results

4. **Test**: Colocate `analyzer-{domain}.test.ts`
   - Use `makeSession()` + `makeRequest()` test helpers
   - Fixtures allow deep field overrides

5. **Example**: See `analyzer-dashboard.ts` (orchestrates all 11 analyzers) or `analyzer-timeline.ts` (Gantt chart)

### Adding a Parser

1. **Create file**: `src/core/parser-{harness}.ts`
   - Implement `find*Dirs(): string[]` to locate session directories
   - Implement `parse*Sessions(dir: string): ParseResult[]` (sync)
   - Implement `parse*SessionsAsync()` (optional, for large datasets)
   - Return `Session[]` from results

2. **Register**: Add to `EXTERNAL_HARNESSES` array in `parser-harnesses.ts`

3. **Test**: Add `parser-{harness}.test.ts` with timestamp extraction, harness detection

4. **Example**: `parser-claude.ts`, `parser-vscode.ts` show sync/async patterns

### Adding a Rule

1. **Create file**: `src/core/rules/{name}.md`
   - Write DSL (domain-specific language) expressions that score antipatterns
   - Format: markdown with frontmatter + DSL code blocks

2. **Create metric** (if new): `src/core/metrics/{name}.metric.md` (optional)

3. **Test**: 
   - Use Rule Playground page in webview to test expressions
   - Add assertion to `antipatterns-e2e.test.ts` to verify detection

4. **Example**: `lazy-prompting.md`, `context-too-large.md`

### Naming Conventions

| Artifact | Pattern | Example |
|----------|---------|---------|
| Analyzers | `analyzer-{domain}.ts` + `.test.ts` | `analyzer-timeline.ts` |
| Parsers | `parser-{harness}.ts` + `.test.ts` | `parser-claude.ts` |
| Webview pages | `page-{domain}.ts` | `page-dashboard.ts` |
| Rules | `{antipattern-name}.md` | `lazy-prompting.md` |
| Metrics | `{metric-name}.metric.md` | `token-cost.metric.md` |
| Types | `{domain}-types.ts` | `session-types.ts` |

## Test Coverage & Quality

- **Test framework**: Vitest (fast, supports worker threads)
- **Coverage threshold**: 70% lines/functions, 60% branches
- **Pattern**: Colocate `.test.ts` next to source files
- **Fixtures**: `makeSession()`, `makeRequest()` in test helpers allow deep field overrides
- **Run tests**:
  ```bash
  npm test                    # once (CI)
  npm run test:watch        # interactive
  npm run test:coverage     # with coverage report
  ```

## Debug & Troubleshooting

### Output Logs
- Extension logs to VS Code Output channel (`AI Engineer Coach` channel)
- Check `runtime-debug.ts` for hook points

### Data Inspector
- Use **Data Explorer** page in webview to inspect parsed session fields
- Run `npm run analyze:data-inventory` to dump all parsed sessions as JSON

### Cache Issues
- Cache stored in `~/.cache/ai-coach/` (disk) + memory
- Clear memory cache: **AI Engineer Coach: Reload Data** command
- Clear disk: `rm -rf ~/.cache/ai-coach/`

### Build Issues
- **TypeScript errors**: `npm run typecheck`
- **Lint errors**: `npm run lint -- --fix`
- **Bundle size**: `npm run check-size`
- **Unused imports**: `npm run knip`

## Extension Commands

| Command | Handler | Purpose |
|---------|---------|---------|
| `aiEngineerCoach.open` | `panel.ts` | Open main dashboard |
| `aiEngineerCoach.reload` | Extension host | Reload all data (clear cache, reparse sessions) |
| `aiEngineerCoach.reviewLocalRules` | Extension host | Review & approve local rule files (trust gate) |

## Linking & Documentation

For detailed setup, see [CONTRIBUTING.md](CONTRIBUTING.md).  
For security policy, see [SECURITY.md](SECURITY.md).  
For API/extension manifest, see [package.json](package.json) `contributes` section.

---

**Key mental models**:
- **Session tree**: Workspace → Session (harness + period) → Requests (turns)
- **Filtering**: All analyzers respect optional DateFilter (fromDate, toDate, workspaceId, harness)
- **Caching**: Disk cache + memory cache. Warm-up pre-computes antipatterns async
- **RPC**: Webview ↔ Extension via `postMessage`. `panel-rpc.ts` marshals requests; long operations stream results
- **Edit location tracking**: Map<requestId, Map<file, locDelta>> tracks AI edits beyond code blocks for accurate LoC

