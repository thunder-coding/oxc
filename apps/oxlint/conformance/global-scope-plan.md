# Plan: Fix Global Scope Handling in Oxlint JS Plugins

**See also**: [Global Scope Deep Dive](./global-scope-deep-dive.md) - a comprehensive explanation of how
global scope works in ESLint vs Oxlint.

## Problem Summary

Oxlint JS plugins fail ~800 ESLint test cases, many related to global scope handling:

1. **`globals: { X: "off" }` not respected** - Disabled globals still get flagged
2. **User-configured globals not recognized** - Custom globals from config aren't added to scope
3. **`globalThis.X` not recognized** - Not handled as equivalent to global `X`

**Note**: With the current patch, lib globals (Array, Symbol, etc.) ARE properly resolved during analysis.
The issues are specifically with user-configured globals and `off` settings.

## Root Cause Analysis

### How ESLint Works

ESLint uses a two-phase approach for globals:

1. **Parse Phase**: Scope analysis creates scopes and references
   - In normal operation: `eslint-scope` provides NO built-in globals - all refs end up in `through`
   - With typescript-eslint: `lib: ["esnext"]` creates `ImplicitLibVariable` objects that ARE resolved

2. **Finalize Phase**: ESLint calls `scopeManager.addGlobals(names)` which:
   - Creates Variable objects for user-configured globals
   - Iterates `globalScope.through` and links any remaining references to variables
   - Removes resolved references from `through`
   - Populates `variable.references` arrays

### How Oxlint Currently Works

1. **Parse Phase**: Uses `@typescript-eslint/scope-manager` with `lib: ["esnext"]`
2. **Existing Patch**: Makes `ImplicitLibVariable` resolve statically even in script mode
   - With the patch: lib globals ARE resolved - both `reference.resolved` and `variable.references`
     are populated correctly
3. **No Finalize Phase**: User-configured globals are received but never used to:
   - Add custom globals that aren't in the lib
   - Remove disabled globals (`globals: { X: "off" }`)
   - Resolve any remaining references in `through`

### The Gap

The current patch **works** for lib globals. The remaining issues are:

1. **User-configured globals**: Custom globals from config aren't added to scope
2. **`globals: { X: "off" }`**: Disabled globals aren't removed from scope
3. **Any remaining `through` refs**: References to user-configured globals stay unresolved

### Two Possible Approaches

**Approach A: Keep the patch, add finalization** (Recommended - Simpler)

- Keep `lib: ["esnext"]` for built-in globals (resolved during analysis)
- Add finalization to handle user-configured globals and `off` settings
- The patch ensures lib globals work in both module and script mode

**Approach B: Remove the patch, use `lib: []`** (Cleaner but more work)

- Use `lib: []` so NO globals exist after analysis
- Finalization adds ALL globals (both built-in and user-configured)
- Unified interface with eslint-scope
- Requires maintaining a list of built-in globals

**Recommended**: Approach A - add finalization while keeping the patch and `lib: ["esnext"]`.

## Solution

### Step 1: Implement `addGlobals` and `removeGlobals` in `scope.ts`

Add functions that mimic ESLint's approach:

```typescript
import { Variable } from "@typescript-eslint/scope-manager";
import { globals, initGlobals } from "./globals.ts";

/**
 * Add global variables and resolve references to them.
 * Mimics ESLint's `scopeManager.addGlobals()` method.
 */
function addGlobals(names: string[]): void {
  debugAssertIsNonNull(tsScopeManager);
  const globalScope = tsScopeManager.scopes[0];

  // Create any missing globals
  for (const name of names) {
    if (!globalScope.set.has(name)) {
      const variable = new Variable(name, globalScope);
      globalScope.variables.push(variable);
      globalScope.set.set(name, variable);
    }
  }

  // Resolve references in `through`
  globalScope.through = globalScope.through.filter(reference => {
    const name = reference.identifier.name;
    const variable = globalScope.set.get(name);

    if (variable) {
      reference.resolved = variable;
      variable.references.push(reference);
      return false; // Remove from through
    }
    return true; // Keep in through (unresolved)
  });
}

/**
 * Remove global variables that are disabled via configuration.
 */
function removeGlobals(names: string[]): void {
  debugAssertIsNonNull(tsScopeManager);
  const globalScope = tsScopeManager.scopes[0];

  for (const name of names) {
    const variable = globalScope.set.get(name);
    if (variable) {
      globalScope.set.delete(name);
      const idx = globalScope.variables.indexOf(variable);
      if (idx !== -1) {
        globalScope.variables.splice(idx, 1);
      }
    }
  }
}

/**
 * Finalize scope globals after analysis.
 * - Adds enabled globals and resolves references to them
 * - Removes disabled globals from scope
 */
function finalizeScopeGlobals(): void {
  if (globals === null) initGlobals();
  debugAssertIsNonNull(globals);

  const enabledNames: string[] = [];
  const disabledNames: string[] = [];

  for (const [name, value] of Object.entries(globals)) {
    if (value === "off") {
      disabledNames.push(name);
    } else {
      enabledNames.push(name);
    }
  }

  // Order matters: add/resolve first, then remove disabled
  addGlobals(enabledNames);
  removeGlobals(disabledNames);
}
```

### Step 2: Call Finalization After Analysis

Modify `initTsScopeManager()` to call finalization:

```typescript
function initTsScopeManager() {
  if (ast === null) initAst();
  debugAssertIsNonNull(ast);

  analyzeOptions.sourceType = ast.sourceType;
  typeAssertIs<AnalyzeOptions>(analyzeOptions);
  tsScopeManager = analyze(ast, analyzeOptions);

  // NEW: Finalize globals - add configured globals, resolve references, remove disabled
  finalizeScopeGlobals();
}
```

### Step 3: Keep the Current Patch (For Now)

**Keep** `patches/@typescript-eslint__scope-manager.patch`. This patch:

- Makes `ImplicitLibVariable` resolve statically during analysis
- Ensures lib globals work in both module and script mode
- Sets BOTH `reference.resolved` AND `variable.references` correctly

The patch is still needed because without it, lib globals would only resolve in module mode.

**Future consideration**: If we later switch to Approach B (`lib: []`), the patch can be removed since
all globals would be added in the finalize phase.

## Files to Modify

| File                                              | Action                                                    |
| ------------------------------------------------- | --------------------------------------------------------- |
| `apps/oxlint/src-js/plugins/scope.ts`             | Add `addGlobals`, `removeGlobals`, `finalizeScopeGlobals` |
| `apps/oxlint/src-js/plugins/globals.ts`           | Already exports `initGlobals` - no changes needed         |
| `patches/@typescript-eslint__scope-manager.patch` | KEEP (no changes)                                         |

## Implementation Notes

### Import `Variable` Class

The `Variable` class is exported from `@typescript-eslint/scope-manager`. Update the imports at the top
of `scope.ts`:

```typescript
import {
  analyze,
  Variable,  // ADD THIS
  type AnalyzeOptions,
  type ScopeManager as TSESLintScopeManager,
} from "@typescript-eslint/scope-manager";
import { globals, initGlobals } from "./globals.ts";  // ADD THIS
```

The `Variable` constructor signature is `(name: string, scope: Scope)` where `scope` is the `Scope` that
this variable belongs to. For globals, this will be `globalScope` (i.e., `tsScopeManager.scopes[0]`).

The `Variable` class has `references: Reference[] = []` initialized to an empty array, which is exactly
what we need - our `addGlobals` function will populate this array.

### Order of Operations in `lintFileImpl`

1. `setupSourceForFile()` is called
2. `setGlobalsForFile()` is called - stores globals JSON
3. Rules call `create()` - may access `context.sourceCode.scopeManager`
4. First access to scopeManager triggers `initTsScopeManager()` + finalization
5. AST is walked

The finalization happens lazily when scopeManager is first accessed, which is the right timing since
globals are already set by then.

### Handle Circular Dependency

`globals.ts` and `scope.ts` need to work together. Import from `globals.ts`:

```typescript
import { globals, initGlobals } from "./globals.ts";
```

## Testing

1. Build: `cd apps/oxlint && pnpm run build-conformance`
2. Run conformance tests: `pnpm run conformance`
3. Expected to fix rules with "global scope" failures:
   - `no-array-constructor` (globals.Array: "off")
   - `no-constant-condition` (reference.resolved for undefined)
   - `no-setter-return` (globals.Reflect/Object: "off")
   - `symbol-description` (variable.references empty)
   - `no-constant-binary-expression` (undefined not recognized)
   - Many others with global scope issues

## Risks and Considerations

1. **Type mismatches**: The typescript-eslint types may not perfectly align with our Scope interface.
   Use `@ts-expect-error` where needed.

2. **Performance**: The finalization adds O(n) work where n = number of references in `through`.
   This should be acceptable as `through` is typically small.

3. **Implicit globals**: ESLint also handles `implicit.variables` cleanup. We may need to add this
   if tests fail related to implicit globals in non-strict mode. The implementation in eslint-scope is:

   ```javascript
   const { implicit } = globalScope;
   implicit.variables = implicit.variables.filter(variable => {
     if (globalScope.set.has(variable.name)) {
       implicit.set.delete(variable.name);
       return false;
     }
     return true;
   });
   ```

4. **Future patch removal**: If we later switch to Approach B (`lib: []`), we can remove the patch.
   This would require also adding all built-in globals to the finalization list.

## Not In Scope (Future Work)

1. **`globalThis.X` handling** - This is a rule-level pattern recognition issue, not a scope analysis issue
2. **Inline `/* global */` comments** - Oxlint doesn't process inline directive comments yet
3. **`/* exported */` comments** - Same as above
