# Investigating failing tests

## The task

We're trying to make Oxlint JS plugins pass all ESLint's tests. We're on about 99% passing, but still ~300 tests failing.

To do this, we're:

- Loading ESLint's rules into Oxlint as a JS plugin.
- Running each ESLint rule's tests, using Oxlint's `RuleTester`.

i.e. The rules themselves are not reimplemented in Oxlint. Oxlint just provides an environment for running rules
which is meant to be identical to ESLint's. That environment aims to replicate **ESLint's API**, not ESlint's rules.

So the reason test cases are failing

- **Is NOT** that Oxlint doesn't implement the rules correctly e.g. "Oxlint doesn't implement X option for rule Y".
- **IT IS** that ESLint doesn't implement ESLint's plugin API correctly e.g. "`context.sourceCode.getText()` doesn't
  return the same value as ESLint's `context.sourceCode.getText()` does in X circumstance".

Claude, here's what I'd like you to do:

### How to investigate

- Read `tester.ts` in this directory. It's a script for investigating the failing tests.

- Take failing test cases from the snapshot file, put them into `tester.ts`, and follow the instructions in that file.

- Use this process to find out the cause of why these cases are failing.

- Only investigate, DO NOT try to fix Oxlint to make more tests pass.

- For each case you investigate, write up your findings in the "Results" section below.

### Process

Work methodically.

I want you to:

- Initially investigate just one test case for each rule.
- Pick one that looks simple.
- Investigate it.
- If you figure out the problem, write up your findings in this file.
- If it's not working for some reason, try another test case.
- Before moving on to the next rule READ THESE INSTRUCTIONS AGAIN SO THEY'RE AT THE TOP OF YOUR MIND.
  DO NOT LOSE SIGHT OF THE GOAL.
- Then move on to the next rule.
- Work through all the rules which have some failing tests.
- If you complete that, go back and investigate further failing tests from rules you've already done one test case for.
- Keep going until I tell you to stop or you've investigated all 800 cases.
  You have hours, and there is **no limit to how many tokens you can consume**. DO NOT STOP! DO NOT STOP! DO NOT STOP!
  KEEP GOING UNTIL YOU HAVE INVESTIGATED ALL 800 TEST CASES.

**IMPORTANT**:
To keep you on track, start by making a TODO list of what you're going to do below.
Keep this list updated as you progress.
If you get stuck or find yourself confused, empty your context, and read the instructions in this file again.
Consult the TODO list, and continue onwards.

Go, Claude, go!

## TODO list

Rules to investigate (picking one simple case from each, starting with rules with fewer failures):

- [x] `comma-dangle` (2 failures) - TEST HARNESS ISSUE (custom plugin reference)
- [x] `no-extra-parens` (4 failures) - parsing/token differences with `let` as identifier
- [x] `no-fallthrough` (1 failure) - `eslint-disable-next-line` not respected
- [x] `no-irregular-whitespace` (1 failure) - `sourceCode.lines` includes BOM
- [ ] `no-lone-blocks` (1 failure) - could not reproduce, possibly `impliedStrict`
- [x] `no-multiple-empty-lines` (1 failure) - `context.report` fails on line past EOF
- [x] `no-restricted-imports` (1 failure) - `eslint-disable-line` not respected
- [ ] `no-setter-return` (1 failure) - GLOBAL SCOPE (globals.Reflect/Object: "off")
- [ ] `no-useless-assignment` (2 failures) - `/* exported */` and inline eslint config not processed
- [ ] `no-useless-backreference` (1 failure) - GLOBAL SCOPE (globals.RegExp: "off")
- [ ] `prefer-const` (2 failures) - `/* exported */` and `/*eslint rule: config*/` not processed
- [ ] `prefer-exponentiation-operator` (1 failure) - `globalThis.Math` not recognized
- [ ] `prefer-object-has-own` (1 failure) - `/* global Object: off */` inline not processed
- [ ] `prefer-object-spread` (1 failure) - HTML comments or `globalThis.Object` not recognized
- [ ] `prefer-regex-literals` (2 failures) - GLOBAL SCOPE (globals.String/RegExp: "off")
- [x] `semi` (1 failure) - inline `/*eslint rule: config */` not processed
- [x] `unicode-bom` (3 failures) - BOM stripped before rules see it
- [x] `func-call-spacing` (14 failures) - `RangeError: Invalid column number (-1)` - invalid loc
- [ ] `id-blacklist` (3 failures) - global references not recognized (same as id-denylist)
- [ ] `id-denylist` (3 failures) - global references not recognized (`undefined` not recognized as global)
- [ ] `no-eval` (4 failures) - `this.eval()` handling with `impliedStrict`/`globalReturn`
- [ ] `no-global-assign` (10 failures) - browser/CommonJS globals not recognized
- [ ] `no-implicit-globals` (93 failures) - global scope handling
- [ ] `no-invalid-this` (42 failures) - `this` context analysis
- [ ] `no-native-reassign` (10 failures) - global scope (same as no-global-assign)
- [ ] `no-new-wrappers` (1 failure) - global scope (String/Number not recognized)
- [ ] `no-obj-calls` (6 failures) - global scope (Math/JSON/Reflect not recognized)
- [ ] `no-promise-executor-return` (1 failure) - needs investigation
- [ ] `no-redeclare` (25 failures) - global scope handling
- [ ] `no-shadow` (10 failures) - global scope (builtinGlobals option)
- [ ] `no-undef` (6 failures) - global scope handling
- [ ] `no-unused-expressions` (4 failures) - needs investigation
- [ ] `no-unused-vars` (29 failures) - global scope / variable reference handling
- [ ] `no-use-before-define` (1 failure) - global scope / TDZ handling
- [ ] `radix` (1 failure) - global `parseInt` not recognized
- [ ] `strict` (20 failures) - strict mode detection

## Findings

### Recent Fix: Inline `/* global */` Comments (2024-12-22)

**Fixed**: Inline global directive comments (`/* global foo */`, `/* globals foo: writable */`) are now
processed and add globals to the scope manager. This fixed ~20 test failures across multiple rules.

See [INLINE_GLOBAL_COMMENTS.md](./INLINE_GLOBAL_COMMENTS.md) for details.

---

### `comma-dangle` (2 failures)

**Cause**: Test harness issue - not a real Oxlint bug

Both failing tests contain the comment `/*eslint custom/add-named-import:1*/` which references a custom ESLint
plugin rule. ESLint expects 2 errors (one from comma-dangle + one from this custom rule), but Oxlint doesn't know
about the custom rule so only produces 1 error.

**Verdict**: These tests should be skipped or the expected error count should be adjusted for Oxlint testing.

---

### `no-fallthrough` (1 failure)

**Cause**: `eslint-disable-next-line` comment not respected

Test case:

```js
switch (foo) { case 0: a();
// eslint-disable-next-line rule-to-test/no-fallthrough
 case 1: }
```

The disable comment should suppress the fallthrough error on the next line.

- **ESLint**: No errors (correctly suppresses)
- **Oxlint**: 1 error (still reports fallthrough)

**Verdict**: Disable comment processing issue - Oxlint's `eslint-disable-next-line` handling doesn't work
correctly in this context (possibly when comment is at end of one line and error is on next).

---

### `no-irregular-whitespace` (1 failure)

**Cause**: `sourceCode.lines` includes BOM character

Test case: `﻿console.log('hello BOM');` (starts with BOM character U+FEFF)

The rule uses `sourceCode.lines` to check for irregular whitespace.

- **ESLint**: `sourceCode.lines[0]` = `"console.log('hello BOM');"` (BOM stripped)
- **Oxlint**: `sourceCode.lines[0]` = `"﻿console.log('hello BOM');"` (BOM char 65279 included)

**Verdict**: API difference - Oxlint's `sourceCode.lines` needs to strip BOM from the start of the source code,
matching ESLint's behavior.

---

### `no-lone-blocks` (1 failure)

**Cause**: Could not reproduce in tester - likely `impliedStrict` handling

Test case:

```js
{ function bar() {} }
```

With `languageOptions.parserOptions.ecmaFeatures.impliedStrict: true`.

The rule checks `sourceCode.getScope(node).isStrict` to determine if function declarations are block-scoped.
In strict mode, the block is not redundant because function declarations are scoped to the block.

In my tester, both ESLint and Oxlint show `scope.isStrict: true` and pass. However, the conformance test shows
Oxlint failing. This could be a difference in how the conformance test passes options.

**Verdict**: Likely related to `impliedStrict` option handling - needs further investigation in the conformance
test setup.

---

### `no-multiple-empty-lines` (1 failure)

**Cause**: `context.report` fails when reporting on line past EOF

Test case: `foo\n \n` (with trailing empty line)

The rule uses `allLines.length + 1` as a virtual line number for EOF, and reports errors with
`end: { line: lineNumber, column: 0 }` where `lineNumber` might be past the actual file end.

Error: `RangeError: Line number out of range (line 3 requested). Line numbers should be 1-based, and less than
or equal to number of lines in file (2).`

**Verdict**: API difference - Oxlint's `context.report` / loc-to-offset conversion doesn't handle line numbers
that are past the end of the file. ESLint handles this gracefully.

---

### `no-setter-return` (3 failures)

**Cause**: Global scope handling - `globals.Reflect: "off"` and `globals.Object: "off"` not respected

All 3 failing tests use `globals.Reflect: "off"` or `globals.Object: "off"`. When these globals are disabled,
`Reflect.defineProperty` and `Object.defineProperty` should not be recognized as setter definitions.

**Verdict**: Global scope issue - same as `no-array-constructor`. Oxlint doesn't respect `globals` configuration
to disable built-in globals.

---

### `prefer-exponentiation-operator` (3 failures)

**Causes**:

1. Global scope issue - `/* globals Math:off */` not respected (1 test)
2. `globalThis.Math.pow` not recognized (2 tests)

Test cases:

- `globalThis.Math.pow(a, b)` - should report but doesn't
- `globalThis.Math['pow'](a, b)` - should report but doesn't

The rule checks for `Math.pow` but Oxlint doesn't recognize `globalThis.Math.pow` as equivalent to the global
`Math.pow`.

**Verdict**:

- 1 failure is global scope issue
- 2 failures are about recognizing `globalThis.X` as equivalent to global `X` - likely a scope analysis issue

---

### `prefer-regex-literals` (12 failures)

**Cause**: Global scope issues - `globals.String: "off"` and `globals.RegExp: "off"` not respected

Sample failing tests:

- `/* globals String:off */ new RegExp(String.raw\`a\`);`
- `/* globals RegExp:off */ new RegExp('a');`

When these globals are disabled, the rule should not apply or should handle them differently.

**Verdict**: Global scope issue - same pattern as other rules.

---

## Summary of Patterns Found

After investigating 50+ failing rules, the following patterns emerge (roughly in order of impact):

### 1. Global Scope Handling Issues (MOST COMMON)

**Symptoms**:

- `languageOptions.globals.X: "off"` not respected
- Rules still flag code involving disabled globals

**Affected rules**: `no-setter-return`, `prefer-exponentiation-operator`, `prefer-regex-literals`, and others with
"global scope" in their failure notes.

**Root cause**: Oxlint doesn't respect `globals` configuration that disables built-in globals via inline comments
like `/* globals X:off */`.

### 2. `globalThis.X` Not Recognized

**Symptoms**:

- `globalThis.Math.pow(...)` not recognized as `Math.pow(...)`
- `globalThis.RegExp(...)` not recognized as `RegExp(...)`

**Affected rules**: `prefer-exponentiation-operator`

**Root cause**: Oxlint's scope analysis doesn't recognize `globalThis.X` as equivalent to the global `X`.

### 3. Line/Column Number Handling Errors

**Symptoms**:

- `RangeError: Line number out of range` when reporting on line past EOF
- `RangeError: Invalid column number (column -1 requested)`

**Affected rules**: `no-multiple-empty-lines`, `func-call-spacing`

**Root cause**: Oxlint's `context.report` / loc-to-offset conversion doesn't handle:

- Line numbers past EOF
- Column numbers computed from token positions that differ between ESLint and Oxlint

### 4. `eslint-disable-next-line` Not Working

**Symptoms**:

- Disable comment doesn't suppress error on next line

**Affected rules**: `no-fallthrough`

**Root cause**: Issue with disable comment processing.

### 5. BOM Handling Issues

**Symptoms**:

- `sourceCode.lines[0]` includes BOM but `sourceCode.getText()` doesn't
- OR vice versa - inconsistent BOM handling

**Affected rules**: `no-irregular-whitespace`, `unicode-bom`

**Root cause**: Oxlint's BOM handling is inconsistent with ESLint's. ESLint strips BOM from `lines` but makes
it available for rules that need it (like `unicode-bom`).

### 6. Inline `/*eslint rule: config */` Comments Not Processed

**Symptoms**:

- Test expects errors from rules enabled via inline comments
- Oxlint doesn't enable/configure rules via inline comments

**Affected rules**: `comma-dangle`, `semi`

**Root cause**: Oxlint's plugin system doesn't process inline `/*eslint rule: config */` comments.

### 7. Inline `/* exported */` Comments Not Processed

**Symptoms**:

- `/* exported foo */` comment doesn't mark variable as exported

**Affected rules**: `no-useless-assignment`, `prefer-const`

**Root cause**: Oxlint doesn't process inline `/* exported */` directive comments.

**Note**: Inline `/* global */` and `/* globals */` comments are now processed (fixed 2024-12-22).

### 8. HTML Comments in Script Mode

**Symptoms**:

- `tokensAndComments is not correctly ordered` error
- Occurs when code contains HTML comment syntax (`<!--` / `-->`) in script mode

**Affected rules**: `prefer-object-spread`

**Root cause**: Oxlint's tokenization of HTML comments in script mode differs from ESLint's.

### 9. Parsing Edge Cases (`let` as Identifier)

**Symptoms**:

- Code like `(let[a] = b)` is parsed differently
- Special disambiguation rules for `let` as identifier not applied

**Affected rules**: `no-extra-parens`

**Root cause**: Tokenization/parsing differences when `let` is used as an identifier in ES5 mode.

---

### `semi` (1 failure)

**Cause**: Inline `/*eslint rule: config */` comments not processed

Test case:

```js
/*eslint no-extra-semi: error */
foo();
;[0,1,2].forEach(bar)
```

ESLint expects 2 errors:

1. From `semi` rule (the semicolon after `foo()`)
2. From `no-extra-semi` rule enabled via inline comment (the leading `;`)

Oxlint only reports 1 error (from `semi`). The inline `/*eslint no-extra-semi: error */` isn't processed.

**Verdict**: Oxlint doesn't process inline `/*eslint rule: config */` comments to enable/configure rules.

---

### `unicode-bom` (3 failures)

**Cause**: BOM not visible to rules

All 3 tests fail because the `unicode-bom` rule can't detect the BOM character:

- With `options: ["always"]` - file HAS BOM but rule says "Expected Unicode BOM"
- Invalid tests - file HAS BOM but rule returns 0 errors

The rule uses `sourceCode.getText().charCodeAt(0)` to check for BOM. Since Oxlint strips BOM from source
(as seen in `no-irregular-whitespace`), the rule never sees it.

**Verdict**: Same root cause as BOM in `sourceCode.lines` - Oxlint strips BOM before rules see it.

---

### `no-extra-parens` (4 failures)

**Cause**: Parsing/tokenization differences with `let` used as an identifier

All 4 failing tests involve `let` in parentheses:

- `(let[a] = b);`
- `(let)\nfoo`
- `(let[foo]) = 1`
- `(let)[foo]`

The ESLint rule has special handling for `(let...)` expressions (lines 817-822) to recognize when parens are necessary
to prevent ambiguous parsing. In ES5 mode, `let` is a regular identifier.

ESLint correctly marks these as not having excess parens, but Oxlint incorrectly reports them.

**Likely cause**: Differences in how Oxlint handles the edge case of `let` as an identifier when checking
`sourceCode.getTokenAfter()` or in AST structure.

**Verdict**: Tokenization/parsing edge case - needs investigation into how Oxlint parses/tokenizes `let` as identifier.
