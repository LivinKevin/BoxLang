# BL-2190 - cbSwagger/BoxLang Filepath Issue

## Problem Statement

- cbSwagger crashes on BoxLang when resolving file `$ref`s such as `//resources/api/auth/authenticate.response.yml`.
- Reported versions: BoxLang 1.10.1+51, bx-compat-cfml 1.31.0-snapshot, bx-orm 1.4.1-snapshot, cbSwagger 3.1.3+14.

- Reporter's reproduction:

  ```
  fpath = "//resources/api/auth/authenticate.response.yml"
  fileExists(fpath)                                                        // true
  fileRead(fpath)                                                          // works
  createObject("java", "java.io.FileInputStream").init(fpath)              // error
  createObject("java", "java.io.FileInputStream").init(expandPath(fpath))  // works
  ```

- Cause: BoxLang file functions resolve paths through `expandPath()` (mappings, web root, `//` stripping). Java's `FileInputStream` uses the raw string as a literal disk path.
  - `fileExists()` falls back to `expandPath()`: `src/main/java/ortus/boxlang/runtime/bifs/global/io/FileExists.java` lines 75-76
  - `expandPath()` strips one leading `/` from `//`: `src/main/java/ortus/boxlang/runtime/util/FileSystemUtil.java` lines 1452-1455
  - `fileRead()` also expands: `src/main/java/ortus/boxlang/runtime/dynamic/casters/BoxFileCaster.java` lines 101-102

- Where it crashes: not in cbSwagger itself, but in its dependency swagger-sdk (`coldbox-modules/swagger-sdk`, `models/OpenAPI/Parser.cfc`). It checks `fileExists(DocPath)` and then opens the raw path with `FileInputStream`. Confirmed by test T8.


## Proposed Solution

### Related upstream work (already open, awaiting review)
- swagger-sdk PR #27: https://github.com/coldbox-modules/swagger-sdk/pull/27
  Reads YAML via `fileRead()` instead of `FileInputStream`, resolves `$ref` paths with `expandPath()`, uses bx-yaml if present.
- cbSwagger PR #50: https://github.com/coldbox-modules/cbSwagger/pull/50
  Runs `expandPath()` on file paths before passing them to swagger-sdk.
- swagger-sdk's latest ForgeBox release is still 3.0.0+25 (2022), so users get the unfixed code.

### Proposed for our team
1. Test and review PRs #27 and #50 against our reproduction, and post the results on the PRs.
2. In the BoxLang repo: add tests for `fileExists()`/`expandPath()` with `//` and mapping-resolved paths.
3. In the BoxLang repo (optional): include the underlying Java error in the interop "Error invoking constructor" message (`DynamicInteropService.java` line 418).
4. Document that BoxLang file functions and Java classes resolve paths differently.

### Implemented (swagger-sdk, branch `BL-2190-filepath-fix-2`)
- `Parser.cfc` `init()`, YAML case: read the file with `fileRead( DocPath )` first, then parse the string.
  - BoxLang with bx-yaml: `yamlDeserialize( yamlText )`.
  - Otherwise: SnakeYAML `.load( yamlText )` (the jar already ships in swagger-sdk's `lib/`).
- `fileRead()` resolves `//` and mapped paths the same way `fileExists()` does, so Java never receives the raw path.
- Why not the previous code: it only worked on BoxLang when bx-yaml was installed (`yamlDeserialize not found` otherwise), and its fallback still opened the raw path with `FileInputStream`.
- Why not expand the path first (fix-1): `fileRead()` already resolves the path, so no extra expand step or `java.io.File` check is needed.

### Rejected
- Making `fileExists()` check only the literal path: it would disagree with `fileRead()` and break the ColdBox `//` workaround.
- Automatically expanding strings passed to Java: BoxLang can't tell which strings are paths, so it would break unrelated Java calls.
- Fixing the `//` built in `fetchDocumentReference()` (empty base path + `/` prefix): this is where the `//` comes from, but `fileRead()` handles it, so it isn't needed for this fix. Left as a follow-up.


## Implementation Steps

1. Got approval from the lead to work in other repos.
2. Reproduced the issue in BoxLang (T1-T10 below).
3. Cloned the team fork of swagger-sdk; branched `BL-2190-filepath-fix-2` off `development`.
4. Changed the YAML case in `models/OpenAPI/Parser.cfc` (commit d790344).
5. Added a ParserSpec test for the reporter's scenario (commit 39f16b0).
6. Ran the suite on a BoxLang 1.18 server with and without bx-yaml.
7. Next: open a PR to `development` with these results.


## Validation Plan

### Tests to Add/Update

Environment: BoxLang 1.19.0-snapshot (built locally), run from the CLI inside a fixture "webroot" folder
(BoxLang's CLI maps `/` to the current folder). `[OK]` means no exception was thrown; the value after `->` is the finding.

Command: `java -jar build/libs/boxlang-1.19.0-snapshot.jar ../repro.bxs`

```
[OK]    T1  fileExists(fpath) -> true
[OK]    T2  fileRead(fpath) -> 49 chars read
[ERROR] T3  FileInputStream(fpath) -> Error invoking constructor for class java.io.FileInputStream
[OK]    T4  FileInputStream(expandPath(fpath)) -> (java.io.FileInputStream)
[OK]    T5  fileExists(single-slash path) -> true
[ERROR] T6  FileInputStream(single-slash path) -> Error invoking constructor for class java.io.FileInputStream
[OK]    T7  java.io.File(fpath).exists() -> false
[ERROR] T8  swagger-sdk style: FileInputStream(File(fpath)) -> Error invoking constructor for class java.io.FileInputStream
[OK]    T9  fileExists('/edgecase') (directory via mapping) -> false
[OK]    T10 yamlDeserialize BIF available in core? -> false
```

Underlying Java error for T3/T6/T8:
`java.io.FileNotFoundException: /resources/api/auth/authenticate.response.yml (No such file or directory)`

Notes:
- T6: the problem is not limited to `//`; any path that only resolves through a BoxLang mapping fails in Java.
- T7: BoxLang and Java disagree about whether the same file exists.
- T10: the upstream swagger-sdk fix needs the bx-yaml module, which is not in core.

### swagger-sdk results (BoxLang 1.18, `server-boxlang@1.json`)

New test in `ParserSpec.cfc`: parses a struct with no base document and a `$ref` to
`/tests/resources/petstore/petstore.yaml`, which resolves to `//tests/...` (the reporter's path shape).

| Run | Previous code | Fix-2 |
|---|---|---|
| With bx-yaml, full suite | - | 8/8 passed |
| Without bx-yaml, ParserSpec | 1 passed, 2 errors (`yamlDeserialize not found`) | 3/3 passed |

Known limitation: without bx-yaml, `Document.asYAML()` still fails (`yamlSerialize not found`).
That writes YAML, which is out of scope here.

Local setup note: `test-harness/box.json` pins ColdBox 6, which fails on BoxLang (`server.coldfusion` missing).
Run `box install coldbox@^8 --noSave` in `test-harness` first, as CI does.

### Verification Steps
- Start the server: `box server start serverConfigFile=server-boxlang@1.json` (from the swagger-sdk root).
- Run the suite: `http://localhost:60299/tests/runner.cfm?reporter=text`, with bx-yaml and again without it.

### Success Criteria
- [x] The reporter's scenario (YAML `$ref` through a `//` path) no longer crashes.
- [x] A new test covers it, and it fails on the previous code without bx-yaml.
- [x] No existing swagger-sdk tests break.
- [ ] A swagger-sdk release on ForgeBox (still 3.0.0 from 2022) so users actually get the fix.

## Open Questions

1. Can PRs #27 and #50 be reviewed and merged, and swagger-sdk released to ForgeBox? The latest release there is still 3.0.0+25 from 2022.
2. Should BoxLang look at the isStruct/StructCasterLoose hang described in #27?
3. Why do cbSwagger's tests fail on BoxLang without the CFML compatibility module?
4. Is the // stripping in expandPath() permanent?
5. Is the clearer interop error message wanted?
