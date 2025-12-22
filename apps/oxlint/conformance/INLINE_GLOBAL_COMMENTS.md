# Inline Global Directive Comment Processing

## Problem

ESLint supports inline directive comments that define global variables directly in source code:

```js
/* global foo */
/* global bar: writable */
/* globals foo, bar: readonly, baz: off */
```

These comments allow code to declare which global variables it expects to be available. ESLint processes
these comments and adds the declared globals to the scope manager, which affects how rules like
`no-implicit-globals`, `no-undef`, and `no-unused-vars` analyze the code.

Before this fix, Oxlint's JS plugin system did not process these inline global directive comments.
Only globals passed from the configuration (via `languageOptions.globals`) were recognized.
This caused ~20 test failures across rules that depend on global variable recognition.

## Example of the Problem

```js
/* global foo: writable */ foo = bar;
```

- **Expected (ESLint)**: Only `bar` is flagged as an undeclared global (since `foo` is declared writable)
- **Actual (Oxlint before fix)**: Both `foo` and `bar` were flagged because `foo` wasn't recognized

## Solution

Added inline global comment parsing to `scope.ts` in the `addGlobals()` function. The implementation:

1. **Parses block comments** looking for `/* global */` or `/* globals */` directives
2. **Extracts global declarations** in format: `name` (readonly), `name: writable`, `name: off`
3. **Creates scope variables** for each declared global (unless "off")
4. **Resolves references** to these globals in `globalScope.through`

### Key Code

```typescript
// Regex to match `/* global */` or `/* globals */` directive comments
const GLOBAL_DIRECTIVE_REGEX = /^(globals?)\s+(.+)$/s;

// Regex to parse individual global entries
const GLOBAL_ENTRY_REGEX = /^\s*([a-zA-Z_$][\w$]*)\s*(?::\s*(\S+))?\s*$/;

function parseInlineGlobalComments(): Globals {
  const result: Globals = {};
  const { comments } = ast;

  for (const comment of comments) {
    if (comment.type !== "Block") continue;

    const match = GLOBAL_DIRECTIVE_REGEX.exec(comment.value.trim());
    if (!match) continue;

    for (const entry of match[2].split(",")) {
      const entryMatch = GLOBAL_ENTRY_REGEX.exec(entry);
      if (entryMatch) {
        result[entryMatch[1]] = normalizeGlobalValue(entryMatch[2]);
      }
    }
  }

  return result;
}
```

## Impact

- **Tests fixed**: ~20 across multiple rules
- **Failing tests reduced**: 309 → 289
- **Affected rules**: `no-implicit-globals`, `no-global-assign`, `no-unused-vars`, `no-redeclare`,
  `prefer-regex-literals`, and others that check global variable usage

## Remaining Issues

There are still ~86 failing tests for `no-implicit-globals` due to:

1. **`globalReturn: true`** (14 tests) - Option that tells scope manager code is wrapped in a function
   (like Node.js modules). Currently hardcoded to `false` in `analyzeOptions`.

2. **`sourceType: "commonjs"`** (8 tests) - CommonJS modules also have function-wrapped scope.
   This affects how the global scope is analyzed.

These require passing parser/scope options through from the configuration, which is a separate fix.

## Files Changed

- `apps/oxlint/src-js/plugins/scope.ts` - Added inline global comment parsing
