# Global Scope Handling: A Deep Dive

## Executive Summary

When you write `Symbol()` in JavaScript, how does a linter know that `Symbol` refers to the built-in global
`Symbol` and not a local variable? And how can a user configure their linter to say "pretend `Array` doesn't
exist as a global"?

This document explains how ESLint handles global scope in normal operation, how it differs when using
TypeScript-ESLint, and why Oxlint's JS plugins are currently missing a critical step.

**The core insight**: Scope analysis happens in two phases. The first phase (parsing) identifies what variables
exist and where they're referenced. The second phase (finalization) links references to globals based on
configuration. Oxlint is missing the second phase.

---

## Table of Contents

1. [High-Level Overview](#high-level-overview)
2. [ESLint's Normal Operation](#eslints-normal-operation)
3. [ESLint with TypeScript-ESLint](#eslint-with-typescript-eslint)
4. [How References Get Resolved](#how-references-get-resolved)
5. [Oxlint's Current Implementation](#oxlints-current-implementation)
6. [The Gap](#the-gap)
7. [The Fix](#the-fix)
8. [Appendix: Code References](#appendix-code-references)

---

## High-Level Overview

### What is a Global?

A "global" is a variable that exists without being declared in your code. When you write:

```javascript
console.log("hello");
```

You didn't declare `console` anywhere. It's a global - provided by the JavaScript environment (browser, Node.js,
etc.). Similarly, `Array`, `Object`, `Symbol`, `undefined`, `Math`, `JSON` are all globals.

### Why Do Linters Care?

Many lint rules need to know if an identifier refers to a global:

- **`no-array-constructor`**: Flags `new Array()` because Array is a global constructor with surprising behavior
- **`no-undef`**: Flags variables that aren't defined anywhere
- **`no-global-assign`**: Flags attempts to reassign globals like `undefined = 5`
- **`symbol-description`**: Flags `Symbol()` without a description

### The Configuration Problem

Sometimes users want to tell the linter "pretend this global doesn't exist" or "pretend this exists even though
it normally wouldn't". ESLint's `languageOptions.globals` configuration handles this:

```javascript
// eslint.config.js
export default {
  languageOptions: {
    globals: {
      myCustomGlobal: "readonly",  // Treat as if it exists
      Array: "off",                 // Treat as if Array doesn't exist
      console: "writable",          // Exists and can be reassigned
    }
  }
}
```

When `Array: "off"` is set, `new Array()` should NOT be flagged by `no-array-constructor` because from the
rule's perspective, `Array` is not the built-in global anymore.

---

## ESLint's Normal Operation

In normal operation, ESLint uses **`eslint-scope`** (not typescript-eslint) for scope analysis. This is
important to understand because it works differently from typescript-eslint.

### The eslint-scope Package

`eslint-scope` is ESLint's own scope analyzer. Key characteristics:

1. **No built-in globals by default**: Unlike typescript-eslint, `eslint-scope` does NOT provide any built-in
   globals during analysis. If you analyze `console.log(Symbol())`, references to both `console` and `Symbol`
   will be completely unresolved.

2. **All global references end up in `through`**: After analysis, the global scope's `through` array contains
   ALL references that couldn't be resolved to local declarations.

3. **Post-analysis finalization required**: ESLint must explicitly call `scopeManager.addGlobals()` after
   analysis to resolve these references.

### How ESLint Defines Default Globals

ESLint defines globals based on `ecmaVersion` in `languageOptions`. The definitions are in `conf/globals.js`:

```javascript
// Simplified from eslint/conf/globals.js
module.exports = {
  es3: { Array: false, Object: false, String: false, ... },  // 37 globals
  es5: { ...es3, JSON: false },
  es2015: { ...es5, Promise: false, Map: false, Set: false, Symbol: false, ... },
  es2017: { ...es2016, Atomics: false, SharedArrayBuffer: false },
  es2020: { ...es2019, BigInt: false, globalThis: false, ... },
  es2025: { ...es2024, Float16Array: false, Iterator: false },
  // etc.
};
```

Key points:

- `false` means "readonly", `true` means "writable"
- Each ECMAScript version builds on the previous one
- The default `ecmaVersion` is `"latest"` (currently 2026)

### The Two-Phase Flow

**Phase 1: Scope Analysis**

```javascript
// eslint/lib/languages/js/index.js
function analyzeScope(ast, languageOptions, visitorKeys) {
  return eslintScope.analyze(ast, {
    ignoreEval: true,
    ecmaVersion: languageOptions.ecmaVersion,
    sourceType: languageOptions.sourceType || "script",
    // ... other options
  });
}
```

At this point, the scope manager has:

- Scopes for all blocks, functions, modules, etc.
- References for all identifier usages
- **BUT**: No globals, and all global references are unresolved in `through`

**Phase 2: Finalization**

```javascript
// eslint/lib/languages/js/source-code/source-code.js

// First, merge globals from multiple sources
applyLanguageOptions(languageOptions) {
  const configGlobals = Object.assign(
    Object.create(null),
    getGlobalsForEcmaVersion(languageOptions.ecmaVersion),  // From conf/globals.js
    languageOptions.sourceType === "commonjs" ? globals.commonjs : void 0,
    languageOptions.globals,  // User-configured globals
  );
  this.varsCache.set("configGlobals", configGlobals);
}

// Then, finalize by adding globals to scope
finalize() {
  const configGlobals = this.varsCache.get("configGlobals");
  const inlineGlobals = this.varsCache.get("inlineGlobals");  // From /* global */ comments
  addDeclaredGlobals(this.scopeManager, configGlobals, inlineGlobals);
}

// Helper that actually adds globals
function addDeclaredGlobals(scopeManager, configGlobals, inlineGlobals) {
  const finalGlobals = { ...configGlobals, ...inlineGlobals };
  const names = Object.keys(finalGlobals).filter(name => finalGlobals[name] !== "off");
  scopeManager.addGlobals(names);
}
```

### The `addGlobals` Method

In newer versions of `eslint-scope`, there's an `addGlobals` method. Here's what it does:

```javascript
// eslint-scope/lib/scope.js (GlobalScope.__addVariables)
__addVariables(names) {
  // 1. Create Variable objects for each global
  for (const name of names) {
    this.__defineGeneric(name, this.set, this.variables, null, null);
  }

  const namesSet = new Set(names);

  // 2. Resolve references in `through`
  this.through = this.through.filter(reference => {
    const name = reference.identifier.name;

    if (namesSet.has(name)) {
      const variable = this.set.get(name);
      reference.resolved = variable;     // Link reference to variable
      variable.references.push(reference); // Add reference to variable's list
      return false;  // Remove from through
    }
    return true;  // Keep in through (still unresolved)
  });

  // 3. Clean up implicit globals
  // (Variables created by assignment in non-strict mode)
  // ...
}
```

**After finalization**:

- `reference.resolved` is set for all global references
- `variable.references` contains all places the global is used
- `through` only contains truly undefined references

---

## ESLint with TypeScript-ESLint

When ESLint runs its own tests, it uses `@typescript-eslint/parser` and `@typescript-eslint/scope-manager`
instead of its normal parser and `eslint-scope`. This is different from normal operation.

### The `lib` Option

TypeScript-ESLint uses TypeScript's `lib` files to determine what globals exist:

```javascript
// Options for typescript-eslint scope analysis
const analyzeOptions = {
  lib: ["esnext"],  // Provides built-in globals like Symbol, Promise, etc.
  sourceType: "module",
  // ...
};
```

This creates `ImplicitLibVariable` objects for built-in globals **during** analysis (not after).

### How `lib` Globals Are Created

When `analyze()` is called with `lib: ["esnext"]`:

1. The `Program` node is visited
2. `populateGlobalsFromLib()` is called immediately
3. For each lib (esnext depends on es2024, es2023, ... es5), all variables are added to global scope
4. Each variable becomes an `ImplicitLibVariable` with flags:
   - `isTypeVariable: true` if valid in type context (e.g., `PropertyKey`)
   - `isValueVariable: true` if valid in value context (e.g., `Array`)
   - Some are both (e.g., `Array`, `Object`, `Promise`)
   - Some are type-only (e.g., `Partial<T>`, `Record<K,V>`)

### How References Are Resolved

When a scope closes, references are resolved. For global scope:

```javascript
// ScopeBase.js - #globalCloseRef
#globalCloseRef = (ref, scopeManager) => {
  if (this.shouldStaticallyCloseForGlobal(ref, scopeManager)) {
    this.#staticCloseRef(ref);  // Resolves: sets reference.resolved AND variable.references
  } else {
    this.#dynamicCloseRef(ref);  // Unresolved: adds to through
  }
};
```

The `shouldStaticallyCloseForGlobal()` method decides whether to resolve statically:

- **Module mode**: Always resolve statically
- **Script mode**: Only resolve let/const/class and `ImplicitLibVariable` (with Oxlint's patch)

### The `#staticCloseRef` Method

This is where the actual resolution happens:

```javascript
#staticCloseRef = (ref) => {
  const variable = this.set.get(ref.identifier.name);
  if (!variable) return delegate();  // Not found → unresolved

  // Type/value check for TypeScript
  const isValidType = ref.isTypeReference && variable.isTypeVariable;
  const isValidValue = ref.isValueReference && variable.isValueVariable;
  if (!isValidType && !isValidValue) return delegate();  // Type mismatch

  // SUCCESS: Both links are established
  variable.references.push(ref);
  ref.resolved = variable;
};
```

**Key insight**: When static resolution succeeds, BOTH links are established:

- `reference.resolved = variable` (reference → variable)
- `variable.references.push(reference)` (variable → reference)

### What Happens Without the Patch?

Without Oxlint's patch, typescript-eslint's behavior depends on `sourceType`:

**Module mode** (`sourceType: "module"`):

- `shouldStaticallyCloseForGlobal()` returns `true` via the `isModule()` check
- Lib globals ARE fully resolved - both links established
- References to `Symbol`, `Array`, etc. work correctly

**Script mode** (`sourceType: "script"`, the default):

- `shouldStaticallyCloseForGlobal()` returns `false` for lib variables
- Why? The check is `defs.length > 0`, but `ImplicitLibVariable` has no defs
- Lib globals exist in `globalScope.set` and `globalScope.variables`
- BUT references use dynamic resolution → end up in `through`
- `reference.resolved` is null, `variable.references` is empty

The key code in `ScopeBase.js`:

```javascript
shouldStaticallyCloseForGlobal(ref, scopeManager) {
  // Module mode → always resolve statically
  if (scopeManager.isModule()) {
    return true;
  }

  // Script mode → check if variable has definitions
  const variable = this.set.get(ref.identifier.name);
  const defs = variable?.defs ?? [];
  return defs.length > 0;  // FALSE for ImplicitLibVariable!
}
```

**Why does typescript-eslint take the `lib` option at all?**

1. The lib variables ARE created in the global scope (via `populateGlobalsFromLib()`)
2. In **module mode**, they ARE fully resolved
3. The default `sourceType` is `'script'`, which is the problem case
4. ESLint works around this by patching `addGlobals()` onto the scope manager

**The patch adds:**

```javascript
if (variable instanceof ImplicitLibVariable) {
  return true;  // Force static resolution for lib globals
}
```

This ensures lib globals work in both module and script mode, matching ESLint's expected behavior.

### ESLint's Workaround

ESLint patches typescript-eslint's scope manager to add an `addGlobals` method (for handling
user-configured globals that aren't in the lib):

```javascript
// eslint/tools/typescript-eslint-parser/index.js
function addGlobals(names) {
  const globalScope = this.scopes[0];

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
    const variable = globalScope.set.get(reference.identifier.name);
    if (variable) {
      reference.resolved = variable;
      variable.references.push(reference);
      return false;
    }
    return true;
  });

  // Clean up implicit globals
  // ...
}

module.exports = {
  parseForESLint(...args) {
    const result = typescriptESLintParser.parseForESLint(...args);
    result.scopeManager.addGlobals = addGlobals;  // Patch the method onto the scope manager
    return result;
  },
};
```

### Key Insight: Same Finalization Pattern

Whether ESLint uses `eslint-scope` or typescript-eslint, the finalization pattern is the same:

1. Parse and analyze (creates unresolved references in `through`)
2. Merge globals from config + inline comments
3. Call `scopeManager.addGlobals(names)` to resolve references
4. Remove disabled globals if needed

---

## Alternative: Using `lib: []` for Unified Interface

### The Question

Could we use typescript-eslint with `lib: []` (no built-in globals) and then add all globals in the
finalize phase, just like eslint-scope? This would unify the interface between the two scope managers.

### Answer: Yes, This Would Work

If we analyze with `lib: []`:

1. **During analysis**: No lib globals are created
2. **Result**: ALL global references end up in `globalScope.through`
3. **Finalize phase**: We call `addGlobals()` with our own list of globals
4. **Outcome**: Same as eslint-scope - full control over which globals exist

### Benefits of This Approach

1. **Unified interface**: Same two-phase pattern as eslint-scope
2. **Full control**: We decide exactly which globals are available
3. **Simple `off` handling**: Just don't include disabled globals in the list
4. **No patch needed**: The current patch is only needed because we use `lib: ["esnext"]` with script mode
5. **Consistent behavior**: Module and script mode work identically

### What We'd Need

We'd need a list of esnext globals. Options:

1. **Hard-code** like ESLint's `conf/globals.js` (~70 runtime globals)
2. **Extract at build time** from typescript-eslint's lib files
3. **Import at runtime** from typescript-eslint's lib modules

The runtime import is feasible:

```typescript
import { esnext } from "@typescript-eslint/scope-manager/dist/lib/lib.js";

// Recursively flatten all libs and extract variable names
function extractGlobalNames(lib: LibDefinition): string[] {
  const names: string[] = [];
  for (const [name, config] of lib.variables) {
    // Only include VALUE or TYPE_VALUE (not type-only)
    if (config.isValueVariable) {
      names.push(name);
    }
  }
  for (const dep of lib.libs) {
    names.push(...extractGlobalNames(dep));
  }
  return names;
}
```

### Type vs Value Distinction

One consideration: eslint-scope's `Variable` class doesn't distinguish types from values. If we use
`lib: []` and create regular `Variable` objects:

- **For JavaScript**: No problem - all globals are values
- **For TypeScript**: We'd lose the `isTypeVariable`/`isValueVariable` distinction

However, for linting JavaScript code (which is Oxlint's primary use case), this doesn't matter.

### Recommended Approach

For simplicity and consistency:

1. **Keep `lib: ["esnext"]`** for now - it provides all globals with correct type/value flags
2. **Implement `addGlobals()`** to handle user-configured globals and resolve any remaining `through` refs
3. **Implement `removeGlobals()`** to handle `globals: { X: "off" }`
4. **Keep the patch** for now - it ensures lib globals resolve in script mode

The patch can be removed later if we switch to `lib: []`, but that's a larger change that would require
maintaining a globals list.

---

## How References Get Resolved

This is the critical part. When you write:

```javascript
function foo() {
  console.log(x);
}
```

The reference to `console` goes through this process:

### During Scope Analysis

1. **Create Reference**: A Reference object is created for the `console` identifier
2. **Try Local Scope**: Look in the function scope's `set` for a variable named "console" - not found
3. **Bubble Up**: Reference moves to parent scope (global scope)
4. **Dynamic Resolution**: For global scope, use dynamic resolution (for `eval` compatibility)
5. **Result**: Reference goes into `globalScope.through` with `resolved = null`

### During Finalization

1. **Call addGlobals**: ESLint calls `scopeManager.addGlobals(["console", "Symbol", ...])`
2. **Create Variables**: For each name, create a Variable in the global scope if it doesn't exist
3. **Link References**: Iterate `through`, and for each reference that matches a global name:
   - Set `reference.resolved = variable`
   - Add reference to `variable.references`
   - Remove from `through`

### The `through` Array

The `globalScope.through` array is crucial:

- **Before finalization**: Contains ALL unresolved references (including to built-in globals)
- **After finalization**: Only contains truly undefined references (typos, missing imports, etc.)

---

## Oxlint's Current Implementation

### How Globals Flow Into Oxlint

1. **Rust Side**: Configuration is processed in `crates/oxc_linter/src/config/globals.rs`
   - Globals are stored as `FxHashMap<String, GlobalValue>` where GlobalValue is readonly/writable/off
   - Serialized to JSON and sent to JS side

2. **JS Side**: Received in `apps/oxlint/src-js/plugins/globals.ts`
   - `setGlobalsForFile(globalsJSON)` stores the JSON string
   - `initGlobals()` parses it into a frozen object
   - Exposed via `context.languageOptions.globals`

3. **Scope Analysis**: Done in `apps/oxlint/src-js/plugins/scope.ts`
   - Uses `@typescript-eslint/scope-manager` with `lib: ["esnext"]`
   - Built-in globals are created as `ImplicitLibVariable` objects
   - **BUT**: No finalization step is called

### The Current Patch

Oxlint has a patch at `patches/@typescript-eslint__scope-manager.patch` that:

1. Makes `ImplicitLibVariable` resolve statically during analysis (instead of dynamically)
2. Merges `isTypeVariable` and `isValueVariable` flags when duplicate lib variables are added

**Important**: The patch works correctly for lib globals. When static resolution is enabled:

- `reference.resolved` IS set correctly
- `variable.references` IS populated correctly
- Both links are established during analysis

What the patch does NOT handle:

- User-configured globals that aren't in the lib
- `globals: { X: "off" }` settings
- Any remaining references in `through`

**The patch should be kept for now** - it's needed for lib globals to work in script mode. See the
detailed explanation in [What Happens Without the Patch?](#what-happens-without-the-patch) below.

### What's Missing

The globals configuration is received and stored, but it's never used to:

1. Resolve references in `globalScope.through`
2. Populate `variable.references` arrays
3. Remove disabled globals from the scope
4. Add custom globals that aren't in the lib

---

## The Gap

Here's a concrete example of what goes wrong:

### Example 1: `reference.resolved` is null

```javascript
// Rule: no-constant-condition
if (undefined) { }  // Should flag: undefined is always falsy
```

The rule checks if `undefined` is a constant by checking `reference.resolved`:

```javascript
if (reference.resolved) {
  // Check if it's the global `undefined`
}
```

But `reference.resolved` is null, so the check fails.

### Example 2: `globals.Array: "off"` not respected

```javascript
// Config: { globals: { Array: "off" } }
// Rule: no-array-constructor
new Array(1, 2, 3);  // Should NOT flag: Array is disabled
```

The rule checks if `Array` is a global:

```javascript
const variable = globalScope.set.get("Array");
if (variable && variable.defs.length === 0) {
  // It's a built-in global, flag it
}
```

But `Array` is still in the global scope (from lib), so it gets flagged.

### Example 3: `variable.references` is empty

```javascript
// Rule: symbol-description
Symbol();  // Should flag: Symbol() without description
```

The rule finds the global `Symbol` variable and checks its references:

```javascript
const symbolVariable = globalScope.set.get("Symbol");
for (const reference of symbolVariable.references) {
  // Check each call to Symbol()
}
```

But `references` is empty, so no checks happen.

---

## The Fix

### Implementation

Add a finalization step to `scope.ts`:

```typescript
function initTsScopeManager() {
  // ... existing code ...
  tsScopeManager = analyze(ast, analyzeOptions);

  // NEW: Finalize globals
  finalizeScopeGlobals();
}

function finalizeScopeGlobals(): void {
  const globalScope = tsScopeManager.scopes[0];

  // Get configured globals
  if (globals === null) initGlobals();

  const enabledNames: string[] = [];
  const disabledNames: string[] = [];

  for (const [name, value] of Object.entries(globals)) {
    if (value === "off") {
      disabledNames.push(name);
    } else {
      enabledNames.push(name);
    }
  }

  // Add enabled globals and resolve references
  addGlobals(enabledNames);

  // Remove disabled globals
  removeGlobals(disabledNames);
}

function addGlobals(names: string[]): void {
  const globalScope = tsScopeManager.scopes[0];

  // Create missing globals
  for (const name of names) {
    if (!globalScope.set.has(name)) {
      const variable = new Variable(name, globalScope);
      globalScope.variables.push(variable);
      globalScope.set.set(name, variable);
    }
  }

  // Resolve references in `through`
  globalScope.through = globalScope.through.filter(reference => {
    const variable = globalScope.set.get(reference.identifier.name);
    if (variable) {
      reference.resolved = variable;
      variable.references.push(reference);
      return false;
    }
    return true;
  });
}

function removeGlobals(names: string[]): void {
  const globalScope = tsScopeManager.scopes[0];

  for (const name of names) {
    const variable = globalScope.set.get(name);
    if (variable) {
      globalScope.set.delete(name);
      const idx = globalScope.variables.indexOf(variable);
      if (idx !== -1) globalScope.variables.splice(idx, 1);
    }
  }
}
```

### Keep the Current Patch (For Now)

The patch at `patches/@typescript-eslint__scope-manager.patch` should be **kept**. It ensures lib globals
resolve correctly in both module and script mode. Without it, lib globals would only resolve in module mode.

**Future consideration**: If we later switch to `lib: []` (see [Alternative: Using `lib: []`](#alternative-using-lib--for-unified-interface)),
the patch can be removed since all globals would be added in the finalize phase.

### Expected Results

After this fix:

- `reference.resolved` will be set for all global references
- `variable.references` will be populated for all globals
- `globals.X: "off"` will remove the variable from scope, so rules won't see it

---

## Appendix: Code References

### ESLint

| Location                                                       | Description                                 |
| -------------------------------------------------------------- | ------------------------------------------- |
| `eslint/conf/globals.js`                                       | Default globals by ECMAScript version       |
| `eslint/lib/languages/js/index.js:45-60`                       | `analyzeScope()` - calls eslint-scope       |
| `eslint/lib/languages/js/source-code/source-code.js:946-973`   | `applyLanguageOptions()` - merges globals   |
| `eslint/lib/languages/js/source-code/source-code.js:1083-1095` | `finalize()` - calls `addDeclaredGlobals`   |
| `eslint/lib/languages/js/source-code/source-code.js:206-233`   | `addDeclaredGlobals()` - calls `addGlobals` |
| `eslint/tools/typescript-eslint-parser/index.js:33-91`         | `addGlobals()` implementation for ts-eslint |

### eslint-scope

| Location                                    | Description                       |
| ------------------------------------------- | --------------------------------- |
| `eslint-scope/lib/scope-manager.js:191-193` | `addGlobals()` method             |
| `eslint-scope/lib/scope.js:542-585`         | `__addVariables()` implementation |

### Oxlint

| Location                                          | Description                           |
| ------------------------------------------------- | ------------------------------------- |
| `apps/oxlint/src-js/plugins/scope.ts`             | Scope analysis (missing finalization) |
| `apps/oxlint/src-js/plugins/globals.ts`           | Globals handling                      |
| `apps/oxlint/src-js/plugins/lint.ts`              | Main linting flow                     |
| `patches/@typescript-eslint__scope-manager.patch` | Current patch (to be removed)         |
| `crates/oxc_linter/src/config/globals.rs`         | Rust-side globals config              |

### typescript-eslint

| Location                                                                | Description                    |
| ----------------------------------------------------------------------- | ------------------------------ |
| `@typescript-eslint/scope-manager/dist/analyze.js`                      | Entry point for scope analysis |
| `@typescript-eslint/scope-manager/dist/ScopeManager.js`                 | ScopeManager class             |
| `@typescript-eslint/scope-manager/dist/scope/GlobalScope.js`            | Global scope with `through`    |
| `@typescript-eslint/scope-manager/dist/variable/ImplicitLibVariable.js` | Built-in global variables      |

---

## Summary

1. **eslint-scope** (ESLint's normal scope analyzer) provides NO built-in globals
2. **ESLint defines globals** based on `ecmaVersion` via `conf/globals.js`
3. **typescript-eslint** provides built-in globals via `lib: ["esnext"]`:
   - **Module mode**: Lib globals ARE fully resolved (both links established)
   - **Script mode** (default): Lib globals exist but use dynamic resolution → end up in `through`
4. **Oxlint's patch** makes lib globals work in script mode by forcing static resolution for `ImplicitLibVariable`
5. **Both scope analyzers require finalization**: `scopeManager.addGlobals()` must be called to:
   - Add user-configured globals that aren't in the lib
   - Resolve any remaining references in `through`
6. **Oxlint is missing** this finalization step
7. **The fix** is to add `finalizeScopeGlobals()` that calls `addGlobals()` and `removeGlobals()`
8. **The current patch should be kept** - it's needed for lib globals to work in script mode
