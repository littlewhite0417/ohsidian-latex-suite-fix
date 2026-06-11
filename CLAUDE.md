# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Watch build — for development (runs esbuild with watch)
npm run build        # Production build — typecheck + esbuild minified bundle
npm run lint         # Run both tsc and eslint
npm run tsc          # TypeScript type-check (noEmit)
npm run eslint       # ESLint with auto-fix
```

`esbuild.config.mjs` is the build entry — bundles `src/main.ts` → `main.js` (CJS, ES2016). All `obsidian` and `@codemirror/*` modules are external.

## Architecture

This is an **Obsidian plugin** for LaTeX math typesetting in CodeMirror 6 (Live Preview / Source mode). The plugin class `LatexSuitePlugin` in `src/main.ts` is the entry point. On load it:

1. Loads settings (with v1.8.0→v1.8.4 migration), potentially from external snippet files
2. Registers `editorExtensions` — a dynamic array of CodeMirror extensions rebuilt whenever settings change
3. Sets up file watchers for live-reloading snippet files

### Core layers

**Extensions layer** (`src/editor_extensions/`) — CodeMirror 6 view plugins:
- `conceal.ts` — hides LaTeX markup, rendering prettified Unicode symbols via `WidgetType`
- `conceal_fns.ts` / `conceal_maps.ts` — concealment logic and symbol mappings
- `highlight_brackets.ts` — color-paired brackets and cursor-adjacent bracket highlighting
- `math_tooltip.ts` — inline math preview popup (renders MathJax in a tooltip)

**Features layer** (`src/features/`) — keyboard-driven editing behaviors coordinated in `src/latex_suite.ts`:
- `run_snippets.ts` — orchestrates snippet expansion per cursor range, with auto-enlarge bracket triggers
- `autofraction.ts` — expands `x/` → `\frac{x}{}`
- `tabout.ts` — Tab to exit equations, jump past closing brackets/`\right`
- `matrix_shortcuts.ts` — Tab→`&`, Enter→`\\`, Shift+Enter→exit matrix
- `auto_enlarge_brackets.ts` — wraps enclosing brackets with `\left`/`\right`
- `editor_commands.ts` — "Box current equation" and "Select current equation" palette commands

**Snippets engine** (`src/snippets/`):
- `snippets.ts` — abstract `Snippet<T>` class and three concrete types: `StringSnippet`, `RegexSnippet`, `VisualSnippet`. Each has a `process()` method that matches trigger text and computes a replacement
- `parse.ts` — parses user-authored snippet definitions (string/RegExp/functions) into `Snippet` objects, evaluates replacements
- `snippet_management.ts` — `expandSnippets()` applies queued snippets to the editor via CodeMirror transactions; `setSelectionToNextTabstop()` handles Tab-jumping through `$0`, `$1`, etc.
- `tabstop.ts` — tabstop spec grouping and geometry
- `codemirror/` — CodeMirror integration: `snippet_queue_state_field` (pending expansions), `tabstops_state_field` (active tabstop groups), `history` (undo/redo with snippet isolation), `config` (plugin config as a state facet)
- `luasnip_api/` — Node-based replacement API (`BaseNode`, `ArrayNode`, `SnippetTabstopOnlyNode`)
- `options.ts` — `Options` class parsing snippet option flags (m, t, A, r, v, w, etc.)
- `environment.ts` — `Environment` class for matching LaTeX environments (matrix, array, etc.)

**Context** (`src/utils/context.ts`) — two critical view plugins:
- `contextPlugin` — determines at cursor position: math/text/code mode, bounds, environment membership. The `Mode` object classifies cursor context
- `mathBoundsPlugin` — tracks all equation boundary pairs (`$…$` / `$$…$$`) visible in the viewport, using Obsidian's syntax tree

**Settings** (`src/settings/`):
- `settings.ts` — type definitions (`LatexSuitePluginSettings`, `LatexSuiteCMSettings`), defaults, `processLatexSuiteSettings()` which converts raw user settings + parsed snippets into runtime-ready `CMSettings`
- `settings_tab.ts` — Obsidian settings tab UI
- `file_watch.ts` — loads/watches snippet `.js` files from vault directories

### Key patterns

- All CodeMirror extensions are recreated when settings change via `setEditorExtensions()` → `app.workspace.updateOptions()`
- Snippet processing uses a **queue**: `runSnippets` enqueues `SnippetChangeSpec` objects, then `expandSnippets` flushes the queue in a single dispatch (with isolated history for proper undo)
- The `contextPlugin` caches computed bounds/mode per cursor position to avoid repeated tree traversal
- Snippet replacement strings use `$0`, `${1:placeholder}` syntax for tabstops; `$0` is the final cursor position
- Regex snippets must avoid negative lookbehind to support iOS ≤16.3

### Testing

There is no automated test suite. `tests/debug.ts` provides manual context-detection checks (math mode, callouts, lists) bound to `Ctrl-j` — requires running inside Obsidian's developer console.
