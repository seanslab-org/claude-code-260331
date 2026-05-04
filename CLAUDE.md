# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

An archive of the **leaked Claude Code CLI source** (npm leak, 2026-03-31). The `src/` tree (~1,900 files, 512K+ LOC of TypeScript) is preserved as-is and **must not be modified**. The unmodified original lives on the `backup` branch. Contributions go into supporting infrastructure: `docs/`, `mcp-server/`, `scripts/`, `web/`, `prompts/`.

## Common commands

Runtime is **Bun** (`>=1.1.0`) — not Node. The `bunfig.toml` preload (`scripts/bun-plugin-shims.ts`) intercepts `bun:bundle` imports at dev time and rewrites them to `src/shims/bun-bundle.ts` so the CLI runs without the production bundler pass.

```bash
# Type-check + lint (full check)
bun run check                    # = biome check src/ && tsc --noEmit
bun run typecheck                # tsc --noEmit only
bun run lint                     # biome check src/
bun run lint:fix                 # biome check --write src/

# Build the CLI bundle (esbuild via scripts/build-bundle.ts)
bun run build                    # dev bundle
bun run build:prod               # minified
bun run build:watch              # watch mode

# Build the web target
bun run build:web                # / build:web:prod / build:web:watch

# Run the CLI directly from source (no bundle step)
bun scripts/dev.ts [args...]
```

There is **no test runner wired into package.json**. The `scripts/test-*.ts` files (`test-auth.ts`, `test-commands.ts`, `test-mcp.ts`, `test-services.ts`) are standalone smoke scripts — run them with `bun scripts/test-<name>.ts`. Don't assume a `bun test` workflow exists.

The MCP server (`mcp-server/`) has its own `package.json` and uses **npm + tsx**, not Bun: `cd mcp-server && npm install && npm run build`. The `web/` directory is a separate Next.js app with its own toolchain.

## Architecture (cross-file picture)

The CLI is a React+Ink terminal app. Key entry/wiring points to know before editing:

- **Entry**: `src/main.tsx` → Commander.js parses argv, then mounts the React/Ink renderer. Startup deliberately fires MDM read, keychain prefetch, and GrowthBook init in parallel as side-effects before heavy module evaluation — preserve this pattern when editing startup.
- **`QueryEngine.ts`** (~46K LOC): the streaming LLM loop. Owns tool-call dispatch, thinking mode, retries, token counting. Most "what does the agent do when…" questions trace back here.
- **`Tool.ts`** (~29K LOC): base types/interfaces every tool implements (input schema, permission predicate, progress state). Tools live in `src/tools/` and are registered through `src/tools.ts`.
- **`commands.ts`** (~25K LOC): slash-command registry. Commands live in `src/commands/` and are conditionally imported per environment via `bun:bundle`'s `feature(...)` macro for dead-code elimination at build time. Notable flags: `PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `DAEMON`, `VOICE_MODE`, `AGENT_TRIGGERS`, `MONITOR_TOOL`.
- **Permissions**: `src/hooks/toolPermission/` runs on every tool invocation and resolves against the active mode (`default`, `plan`, `bypassPermissions`, `auto`).
- **Subsystems** worth knowing exist: `src/bridge/` (IDE ↔ CLI protocol — see `bridgeMain.ts`, `bridgeMessaging.ts`, `bridgePermissionCallbacks.ts`), `src/coordinator/` (multi-agent orchestration), `src/services/mcp/` (MCP client), `src/services/api/` (Anthropic SDK + bootstrap), `src/services/oauth/`, `src/services/lsp/`, `src/skills/`, `src/plugins/`, `src/memdir/` (persistent memory), `src/tasks/`, `src/state/`.
- **Build resolution quirk**: the codebase imports as `import x from 'src/foo/bar.js'` (relying on `tsconfig` `baseUrl: "."`). `scripts/build-bundle.ts` ships a custom esbuild plugin (`srcResolverPlugin`) that maps those `.js` specifiers to real `.ts`/`.tsx` files. Don't rewrite to relative imports.

For deeper navigation: `docs/architecture.md`, `docs/tools.md`, `docs/commands.md`, `docs/subsystems.md`, `docs/exploration-guide.md`. The `prompts/` directory is a numbered reading order (00–16) that walks the codebase from install through bundling.

## Conventions specific to this repo

- **Don't touch `src/`.** It is the leaked source preserved verbatim. Documentation, MCP server, scripts, and web tooling are the editable surface. (See `CONTRIBUTING.md`.)
- **Formatter is Biome**, not Prettier/ESLint: tabs (width 2), 100-col lines, single quotes, semicolons-as-needed. JSON gets spaces. `noExcessiveCognitiveComplexity` and unused-import/var checks are warnings.
- **Strict TS** with `allowImportingTsExtensions` and `verbatimModuleSyntax: false`. JSX is `react-jsx`.
- **Per-file emoji commits via GitPretty**: `bash ./gitpretty-apply.sh .` rewrites the current commit's per-file metadata; `--hooks` installs it for future commits. Only run when intentionally producing the "pretty" UI on GitHub.

## Project-level engineering standard (seanslab Engineering Guide)

The user follows the seanslab "firmware bring-up discipline" workflow defined in `~/seanslab/Guides/Engineering Guide.md`. Highlights that affect day-to-day work here:

- **Plan before execute**: for any non-trivial task (≥3 steps or architectural), plan first — interfaces, failure modes, rollback, verification steps. Re-plan immediately when reality diverges.
- **Verification before "done"**: every sub-job declares its check (test, build, log, repro, benchmark, manual) up front and the check must run before the item is marked complete. No "done" without proof.
- **Devlog**: append to `devlog-YYYYMMDD.md` in the project folder as work progresses (achievements, discoveries, mistakes).
- **Tasks**: maintain `tasks/todo.md` with checkable acceptance criteria; add a review section (tests run, outcomes, limitations); record corrections as preventative rules in `tasks/lessons.md`.
- **Requirements**: log every user requirement to `Requirements.md`, newest on top, with the brief reply inline.
- **Source control**: small commits, frequent pushes, minimal-impact changes — no broad refactors unless explicitly planned.
- **Subagents**: offload research/exploration/parallel-debugging to keep the main context clean; one subagent = one mission; each delegated job carries its own success check.
- **Disk hygiene**: end-of-session, audit and delete clearly-temporary files (`__pycache__/`, build artifacts, pip/HF caches, dangling Docker images, `tmp/`); confirm before deleting anything potentially-needed (large models, datasets); report disk-before/after.
