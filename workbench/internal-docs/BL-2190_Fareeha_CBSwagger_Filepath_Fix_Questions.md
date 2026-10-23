# BL-2190 — cbSwagger Filepath Issue

## 1. Problem Statement
**What issue are we solving?**

The issue happens when cbSwagger tries to access YAML files using certain file paths, especially ones beginning with `//`.

What makes this interesting is that BoxLang can recognize these paths through functions like `fileExists()` and `fileRead()`, but Java's `FileInputStream` doesn't always interpret them the same way. As a result, the parser can fail even though the file exists and BoxLang is able to locate it.

The goal is to fix this inconsistency so cbSwagger can reliably access referenced YAML files when running on BoxLang.

## 2. Proposed Solution
**What changes will be made and why?**

My approach was to resolve the file path before passing it to Java, rather than changing how the parser works overall.

In `models/OpenAPI/Parser.cfc`, I updated the following line:

**Before:**
```cfml
var IOFile = jLoader.create( "java.io.File" ).init( DocPath );
```

**After:**
```cfml
var IOFile = jLoader.create( "java.io.File" ).init( expandPath( DocPath ) );
```

By adding `expandPath()`, BoxLang has a chance to resolve the path using its own path-handling rules before Java attempts to open the file.

I chose this solution because it directly addresses where the mismatch occurs. It also keeps the change fairly small, without having to rewrite the existing YAML parsing logic.

## 3. Implementation Steps

**Step 1 — Understand the issue**

I started by looking into why the same file path could work with BoxLang's built-in file functions but fail when passed directly to Java. This helped narrow the problem down to how the path was being interpreted.

**Step 2 — Update the parser**

I located the YAML file-loading logic in `Parser.cfc` and changed it to use `expandPath( DocPath )` when creating the Java File object. This allows the path to be resolved before the file is opened.

**Step 3 — Add tests**

I made changes to `ParserSpec.cfc` to add test coverage around the parser. The next step is to verify that these tests specifically reproduce the filepath issue and confirm the fix.

**Step 4 — Review and commit changes**

I reviewed my changes through Git and checked for whitespace errors before committing them.

**Commit:** `601eb9880`

## 4. Validation Plan

### Tests to Add/Update

- Test whether the parser can successfully load a valid YAML file.
- Test paths beginning with `//` to see whether they resolve correctly.
- Confirm that existing JSON and YAML parsing functionality still works.
- Test an invalid or missing filepath to check how the parser handles errors.

### Verification Steps

1. Run the relevant parser tests using BoxLang.
2. Reproduce the original filepath error and compare the behavior before and after the fix.
3. Confirm that `expandPath()` resolves the path correctly before Java attempts to access the file.
4. Run the existing tests to check that the change hasn't affected other parsing functionality.

### Success Criteria

I would consider the fix successful if:

- cbSwagger can access the referenced YAML file without running into the original filepath error.
- The parser handles the affected file paths consistently.
- Existing JSON and YAML parsing tests continue to pass.
- The change doesn't introduce new errors or affect other parsing features.

### Current Status

The code change has been committed, and the Git whitespace check passed. However, I still need to verify the automated test results before confirming that the issue is fully resolved.
