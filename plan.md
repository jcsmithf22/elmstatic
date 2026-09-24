# Modern JavaScript and Node 24 Migration Plan

## Migration target

- Node 24 LTS, with `engines.node: ">=24.15.0"` to satisfy current jsdom requirements.
- Native ESM throughout.
- Modern Node built-ins where they remove dependencies.
- Preserve generated HTML, feeds, scaffold behavior, CLI options, and watch behavior.
- Continue supporting Linux, macOS, and Windows.

## Proposed phases

### 1. Establish behavioral coverage

Before changing implementation:

- Replace the shell-dependent smoke scripts with Node's built-in `node:test`.
- Run each test in a temporary directory.
- Cover:
  - Markdown and Elm Markup scaffold generation.
  - Expected HTML files and representative rendered content.
  - RSS, Atom, or JSON feed output.
  - Tags, sections, drafts, copied resources, and page aliases.
  - Invalid configuration and Elm compilation failures returning a nonzero exit code.
  - CLI help, version, and options.
- Add focused tests for front matter, filenames, excerpts, and draft-date boundaries.

This prevents package upgrades from silently changing generated sites.

### 2. Declare Node 24 and modernize automation

Update `package.json` and CI:

- Add `"engines": { "node": ">=24.15.0" }`.
- Add `.node-version` containing `24`.
- Use current `actions/checkout` and `actions/setup-node` releases.
- Test Node 24 on Linux, macOS, and Windows.
- Install Elm 0.19.1 through a maintained setup action or pinned platform binary.
- Use `npm ci`.
- Regenerate `npm-shrinkwrap.json` with the npm version shipped for Node 24.
- Replace Unix-specific test scripts such as `rm`, environment-prefix assignments, and `$GITHUB_WORKSPACE`.

Checkpoint: the old implementation passes the new test suite on Node 24.

### 3. Remove dependencies superseded by Node 24

Make these behavior-preserving substitutions:

- `cross-spawn` → `node:child_process.spawnSync`
- `fs-extra` → `node:fs`
  - `cpSync`
  - `mkdirSync({ recursive: true })`
  - `rmSync` followed by `mkdirSync` for an empty directory
- `glob` → `node:fs.globSync`
- Ramda → native arrays, objects, strings, `Set`, and ordinary functions

Ramda should be removed in its own commit because it touches much of the file. Tests should compare generated output before and after.

Checkpoint: four dependencies removed with no output changes.

### 4. Convert to ESM and coherent modules

Add `"type": "module"` and replace all `require()` calls.

Suggested structure:

```text
bin/
  elmstatic.js        Thin executable entry point
src/
  cli.js              Commands, options, process exit behavior
  build.js            Main build/watch orchestration
  content.js          Front matter, posts, pages, tags, drafts
  render.js           Rendering pool interface
  render-worker.js    jsdom and Elm execution
  feeds.js            RSS, Atom, and JSON generation
```

Key conversion details:

- Use `node:` prefixes for built-ins.
- Resolve scaffold and package paths using `import.meta.url` and `fileURLToPath`.
- Move worker rendering into a real worker module instead of serializing a function containing `require()`.
- Replace promise chains with `async`/`await` where it improves error propagation.
- Declare currently implicit variables such as `newPagesWithHtml`, which ESM strict mode would reject.
- Ensure rejected builds reach the CLI entry point and set a nonzero exit code.
- Preserve the executable shebang.

This is modularization by responsibility, not a broad architecture rewrite.

### 5. Upgrade retained packages

Upgrade one package or related group at a time:

1. **Commander 15**
   - Use a `Command` instance rather than the package-global object.
   - Read options through current Commander APIs.
   - Use `parseAsync()` for asynchronous commands.

2. **Chokidar 5**
   - Adopt its ESM API.
   - Verify add, change, delete, and rename-like event sequences.
   - Ensure builds do not begin before the initial event batch settles.

3. **Feed 6**
   - Update construction and serialization calls.
   - Compare generated RSS, Atom, and JSON semantically.

4. **jsdom 30**
   - Replace removed `dom.runVMScript()` with the supported VM-context API.
   - Verify all scaffold layouts and error rendering.

5. **workerpool 10**
   - Point the pool at `render-worker.js`.
   - Verify clean shutdown after `build` and continued operation during `watch`.

6. **remove-markdown 0.7**
   - Verify excerpt output against representative Markdown.

Checkpoint after each group: targeted tests plus the complete build suite.

### 6. Replace the aging front-matter stack

Replace `front-matter` with:

- A small parser for the opening `---` delimiters.
- The maintained `yaml` package for metadata parsing.

Test:

- No front matter.
- Empty front matter.
- CRLF input.
- Lists, dates, quoted strings, and multiline values.
- Malformed YAML with a useful filename-bearing error.
- Body content beginning after the closing delimiter.

This avoids carrying the old `js-yaml` 3.x dependency.

### 7. Add lightweight quality enforcement

- Add ESLint's recommended modern ESM configuration.
- Enforce undefined-variable detection, strict equality, unused imports, and modern Node globals.
- Avoid a repository-wide formatting-only rewrite.
- Add scripts such as:

```json
{
  "test": "node --test",
  "lint": "eslint bin src test",
  "check": "npm run lint && npm test"
}
```

### 8. Final compatibility verification

Run:

- Clean `npm ci`.
- Lint and complete tests.
- Both scaffold builds.
- `npm pack`, install the tarball into a temporary project, then run the packaged CLI.
- Watch-mode add/change/delete test.
- CI across all three operating systems.
- `npm audit --omit=dev`.
- Compare representative generated sites against the pre-migration output.

## Intended dependency result

Retained and upgraded:

- `chokidar`
- `commander`
- `feed`
- `jsdom`
- `remove-markdown`
- `workerpool`

Added:

- `yaml`

Removed:

- `cross-spawn`
- `front-matter`
- `fs-extra`
- `glob`
- `ramda`

That leaves seven direct runtime dependencies, down from eleven.

## Reviewable commit sequence

1. Add output and CLI regression tests.
2. Target Node 24 and modernize CI and test scripts.
3. Replace Node-superseded dependencies.
4. Remove Ramda without changing behavior.
5. Convert to ESM and split modules.
6. Upgrade retained dependencies.
7. Replace front-matter parsing.
8. Add linting, package-install verification, and documentation.

Estimated implementation time: 2–3 focused days, primarily because generated-output coverage and cross-platform verification should be established before changing the renderer.
