# Support Adobe's savecontent expression syntax

## Problem Statement

- Adobe ColdFusion lets you assign the output of a `savecontent` block directly to a variable:

  ```
  result = savecontent {
    writeOutput( "test" );
  }
  ```

- BoxLang did not support this.
- Adobe only supports it as a statement. Using it inside an expression is a parse error in Adobe too, so it is out of scope:

  ```
  writeDump( savecontent {
    writeOutput( "test" );
  } )
  ```

- What BoxLang did before the fix: the CF parser read the code as two separate statements:
  1. `result = savecontent`: an assignment from a variable named `savecontent`
  2. `{ writeOutput( "test" ); }`: a plain code block
- Result at runtime: `KeyNotFoundException: The requested key [savecontent] was not located in any scope or it's undefined`.


## Proposed Solution

### Implemented
- Recognize `<variable> = savecontent { body }` as a single statement in the CF parser, and treat it the same as the existing syntax:

  ```
  savecontent variable="result" { writeOutput( "test" ); }
  ```

- Because it becomes a normal `savecontent` component, no runtime code changes are needed.
- A small check makes sure only `name = savecontent {` uses the new rule, so other code isn't affected.

### Rejected
- Fixing it after parsing (merging an assignment of `savecontent` with the block after it): this would also wrongly merge code like `x = savecontent; { ... }`.
- Supporting it as a full expression, e.g. `writeDump( savecontent { } )`: Adobe doesn't support that either.


## Implementation Steps

1. Reproduced the issue with a locally built BoxLang jar.
2. `CFGrammar.g4`: added a new `saveContentAssignment` rule and placed it in the `statement` list so it is tried first.
3. `CFParserControl.java`: added `isSaveContentAssignment()`, which checks that the upcoming code is `name = savecontent {`.
4. `CFVisitor.java`: added `visitSaveContentAssignment()`, which turns the matched code into a normal `savecontent` component.
5. `SaveContentTest.java`: added 4 tests (below).
6. Ran `./gradlew spotlessApply` (no formatting changes).


## Validation Plan

### Tests to Add/Update

New tests in `SaveContentTest.java`, all run as CFML script:

| Test | Checks |
|---|---|
| `testCanCaptureContentAdobeAssignment` | `result = savecontent { echo( "Hello World" ); }` stores `"Hello World"` |
| `testCanCaptureContentAdobeAssignmentScoped` | `variables.result = savecontent { ... }` works |
| `testCanCaptureContentAdobeAssignmentInFunction` | Inside a function, the output goes to the local `var out`, not the `variables` scope |
| `testVariableNamedSaveContent` | A normal variable named `savecontent` still works (`savecontent = "x"; result = savecontent;`) |

### Results

Before the fix, the 3 Adobe-syntax tests fail. After the fix, all 10 `SaveContentTest` tests pass, and 1,119 related compiler and component tests still pass.

### Verification Steps
- Run the tests: `./gradlew test --tests "ortus.boxlang.runtime.components.system.SaveContentTest"`

### Success Criteria
- [x] `result = savecontent { ... }` captures the output into `result`.
- [x] Scoped names (`variables.result`) and use inside functions work.
- [x] Existing code using a variable named `savecontent` is not affected.
- [x] No existing tests break.


## Open Questions

- Only simple and dot-separated variable names are supported. `foo["x"] = savecontent { }` and `var x = savecontent { }` are not. Is that enough?
- The new syntax only works in CFML files (`.cfm`, `.cfc`, `.cfs`), not in BoxLang (`.bx`, `.bxs`) files, since the issue is about Adobe compatibility.
