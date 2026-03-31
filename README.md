# Repro: `source.fixAll.oxc` corrupts nested `sort-keys` fixes in Zed

This repo reproduces a bug when using the Oxc Zed extension with:

- `code_action: "source.fixAll.oxc"`
- `oxlint`'s `eslint(sort-keys)` rule
- nested object literals

The same nested input does not get corrupted when running the `oxlint` CLI with `--fix`. Instead, the CLI applies only one non-overlapping batch of fixes per invocation and converges after multiple passes.

The same `source.fixAll.oxc` setup was also tested in VS Code with the official Oxc extension and worked correctly there.

A local patched build of Zed was also tested against this repo and fixed `l3.js` correctly, which strongly suggests the bug was in Zed's edit-merging/application path.

## Summary

There appear to be four separate behaviors:

1. `oxlint --fix` on the CLI
   It applies one non-overlapping batch of fixes per run. For nested `sort-keys`, this means repeated runs are required.

2. `source.fixAll.oxc` through the language server in Zed
   Saving `l2.js` and `l3.js` corrupts the file.

3. `source.fixAll.oxc` through the language server in VS Code
   The same files are fixed correctly.

4. `source.fixAll.oxc` through a locally patched Zed build
   The same files are fixed correctly.

Current conclusion: the bug was in Zed's handling of the returned workspace edit rather than in `oxc-zed`.

## Versions

- `oxlint`: `1.57.0`
- `oxfmt`: `0.42.0`
- `pnpm`: `10.32.1`

Additional local analysis was done against these cloned repos:

- `oxc`: `af72b802be621fbea6e6ca1fbfc9a685c978b6fc`
- `oxc-zed`: `9b82cff67c223bf929be40489e36faafa536e71c`
- this repro repo: `9118bb677f238988e189946fa325dcabe749758e`

## Repo contents

- `l1.js`: one unsorted object level
- `l2.js`: two nested unsorted object levels
- `l3.js`: three nested unsorted object levels
- `.zed/settings.json`: config that triggers `source.fixAll.oxc`
- `.vscode/settings.json`: config that triggers `source.fixAll.oxc` in VS Code
- `.vscode/extensions.json`: recommends the official Oxc VS Code extension

## Zed configuration

This repo uses the following Zed settings:

```json
{
  "languages": {
    "JavaScript": {
      "format_on_save": "on",
      "formatter": [
        {
          "code_action": "source.fixAll.oxc"
        }
      ]
    }
  },
  "lsp": {
    "oxlint": {
      "initialization_options": {
        "settings": {
          "disableNestedConfig": false,
          "fixKind": "safe_fix",
          "run": "onType",
          "unusedDisableDirectives": "deny"
        }
      }
    }
  }
}
```

## Install

```sh
pnpm install
```

## VS Code setup

This repo also includes a VS Code workspace configuration so the same LSP `fixAll` path can be tested outside Zed.

Recommended extension:

- `oxc.oxc-vscode`

Workspace setting:

```json
{
  "editor.codeActionsOnSave": {
    "source.fixAll.oxc": "always"
  }
}
```

This matches the current Oxc editor setup docs for VS Code.

## Reproducing the Zed corruption

Open this folder in Zed with the Oxc extension enabled. Then save each file and let the configured `source.fixAll.oxc` action run.

### Relevant Zed save log

When saving `l3.js`, Zed logs the following:

```text
2026-03-31T11:32:17+02:00 INFO  [project::lsp_store] Applying edits [(Anchor { timestamp: Lamport {<local>: 1}, offset: 23, bias: Right, buffer_id: Some(BufferId(4294967980)) }..Anchor { timestamp: Lamport {<local>: 1}, offset: 31, bias: Left, buffer_id: Some(BufferId(4294967980)) }, ""), (Anchor { timestamp: Lamport {<local>: 1}, offset: 67, bias: Right, buffer_id: Some(BufferId(4294967980)) }..Anchor { timestamp: Lamport {<local>: 1}, offset: 77, bias: Left, buffer_id: Some(BufferId(4294967980)) }, "      e: 1,\n    },\n  },\n  b: 1c: {\n      f: 1,\n      e: 1,\n    },\n    d: 1e: 1,\n      f: 1")]. Content: "export const obj = {\n  b: 1,\n  a: {\n    d: 1,\n    c: {\n      f: 1,\n      e: 1,\n    },\n  },\n};\n"
2026-03-31T11:32:17+02:00 INFO  [project::lsp_store] Applied edits. New Content: "export const obj = {\n  a: {\n    d: 1,\n    c: {\n      f: 1,\n      e: 1,\n    },\n  },\n  b: 1c: {\n      f: 1,\n      e: 1,\n    },\n    d: 1e: 1,\n      f: 1,\n    },\n  },\n};\n"
```

This is the strongest evidence collected so far because the malformed replacement text is already visible in Zed's logged edit application.

## Reproducing in VS Code

Open this folder in VS Code or Cursor with the Oxc extension enabled.

Then:

1. install dependencies with `pnpm install`
2. make sure the workspace recommendation `oxc.oxc-vscode` is installed
3. open `l2.js` or `l3.js`
4. save the file so `editor.codeActionsOnSave` runs `source.fixAll.oxc`

Observed result:

- `l2.js` and `l3.js` are fixed correctly
- no corruption was observed

This comparison is the main reason the current suspicion has shifted away from `oxc-zed` and toward Zed's application of the returned workspace edit.

## Reproducing in a patched local Zed build

A local source build of Zed with a patch to its LSP edit-merging/application logic was tested against this repo.

Observed result:

- `l3.js` is fixed correctly on save
- no corruption was observed
- the resulting file content is:

```js
export const obj = {
  a: {
    c: {
      e: 1,
      f: 1,
    },
    d: 1,
  },
  b: 1,
};
```

This provides a strong confirmation that the practical bug is in Zed's edit handling, not in the Oxc Zed extension wrapper.

### `l1.js`

Input:

```js
export const obj = {
  b: 1,
  a: 1,
};
```

Result:

This case works. There is only one fix range, so nothing overlaps.

Expected final output:

```js
export const obj = {
  a: 1,
  b: 1,
};
```

### `l2.js`

Input:

```js
export const obj = {
  b: 1,
  a: {
    d: 1,
    c: 1,
  },
};
```

Observed result in Zed:

```js
export const obj = {
  a: {
    d: 1,
    c: 1,
  },
  b: 1c: 1,
    d: 1,
  },
};
```

Expected logical result:

```js
export const obj = {
  a: {
    c: 1,
    d: 1,
  },
  b: 1,
};
```

### `l3.js`

Input:

```js
export const obj = {
  b: 1,
  a: {
    d: 1,
    c: {
      f: 1,
      e: 1,
    },
  },
};
```

Observed result in Zed:

```js
export const obj = {
  a: {
    d: 1,
    c: {
      f: 1,
      e: 1,
    },
  },
  b: 1c: {
      f: 1,
      e: 1,
    },
    d: 1e: 1,
      f: 1,
    },
  },
};
```

Expected logical result:

```js
export const obj = {
  a: {
    c: {
      e: 1,
      f: 1,
    },
    d: 1,
  },
  b: 1,
};
```

## CLI behavior with `oxlint --fix`

The CLI does not have a `--fixAll` flag. The relevant flags are:

- `--fix`
- `--fix-suggestions`
- `--fix-dangerously`

For this repro, `pnpm lint:fix` runs:

```sh
oxlint --fix
```

### Important observation

`oxlint --fix` does not corrupt the file. It applies one non-overlapping batch of fixes per invocation, so nested `sort-keys` requires repeated runs.

### Example: `l3.js`

Start with:

```js
export const obj = {
  b: 1,
  a: {
    d: 1,
    c: {
      f: 1,
      e: 1,
    },
  },
};
```

#### First run

```sh
pnpm lint:fix
```

After run 1:

```js
export const obj = {
  a: {
    d: 1,
    c: {
      f: 1,
      e: 1,
    },
  },
  b: 1,
};
```

The remaining diagnostics are for the nested objects only.

#### Second run

```sh
pnpm lint:fix
```

After run 2:

```js
export const obj = {
  a: {
    c: {
      f: 1,
      e: 1,
    },
    d: 1,
  },
  b: 1,
};
```

#### Third run

```sh
pnpm lint:fix
```

After run 3:

```js
export const obj = {
  a: {
    c: {
      e: 1,
      f: 1,
    },
    d: 1,
  },
  b: 1,
};
```

So the CLI converges, but it does not fix all nested levels in one invocation.

## Why this likely is not an `oxc-zed` bug

The Zed extension code appears to be a thin wrapper around `oxlint --lsp`:

- it advertises `source.fixAll.oxc`
- it launches `oxlint --lsp`
- it does not appear to rewrite or merge edits on its own

That makes the likely bug location the `oxlint` language-server `fixAll` path, not the Zed extension itself.

After also testing the same scenario in VS Code, the more precise conclusion is:

- `oxc-zed` still does not look like the root cause
- the difference is more likely between how Zed and VS Code apply the returned `WorkspaceEdit`
- if both clients are requesting the same `source.fixAll.oxc` action, then the likely fault is in Zed itself or in an editor-client assumption Zed does not currently match

## Relevant `oxlint` context

In the `oxc` repo, the LSP `fixAll` implementation collects multiple fixes into one `WorkspaceEdit`.

There is already a comment in the code acknowledging the problem:

```rust
// when source.fixAll.oxc we collect all changes at ones
// and return them as one workspace edit.
// it is possible that one fix will change the range for the next fix
// see oxc-project/oxc#10422
```

That still looks relevant because nested `sort-keys` replacements can interfere with later ranges.

By contrast, the CLI fixer sorts fixes by span and skips overlapping edits within a run, which explains why repeated `--fix` invocations converge without corruption.

However, because VS Code applies the same `source.fixAll.oxc` result correctly, this repo no longer points cleanly to `oxlint` as the primary culprit.

## Relation to `oxc` issue `#10422` and PR `#10428`

This repro appears to be related to, but not the same as, the earlier bug:

- issue: `oxc-project/oxc#10422`
- fix: `oxc-project/oxc#10428`

Important distinction:

- `#10428` fixed the specific `#10422` VSCode reproduction
- it did not make `source.fixAll.oxc` generally safe for mutually interfering edits

From local inspection of the historical commit, `#10428` changed the language-server behavior so that `source.fixAll.oxc` returns one batched `WorkspaceEdit` for the file. That solved the original case, but it did not add conflict resolution or iterative re-linting between edits.

The current implementation still batches fixes that can invalidate the ranges of later fixes. So this repo likely demonstrates a new manifestation of the same underlying problem area, rather than evidence that `#10428` was never merged or never worked.

In other words:

- this repro is probably not a duplicate of `#10422`
- it is likely a new bug in the same `fixAll` batching strategy
- `#10422` and `#10428` are still relevant prior context and should be linked from any new upstream issue

## Current conclusion

- `oxlint --fix`:
  likely working as currently designed, even if the one-level-per-run behavior may be surprising
- `source.fixAll.oxc` in VS Code:
  works correctly for this repro
- patched local Zed:
  works correctly for this repro
- `source.fixAll.oxc` in stock Zed:
  corrupts the file for this repro
- `oxc-zed`:
  likely not the root cause
- `zed`:
  is the repo where the practical bug lives
