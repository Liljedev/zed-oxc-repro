# Repro: Zed corrupts `source.fixAll.oxc` edits for nested `sort-keys`

This repo reproduces a bug in Zed when applying `source.fixAll.oxc` for nested `eslint(sort-keys)` fixes from `oxlint`.

## Finding

The same workspace was tested in four paths:

1. `oxlint --fix` on the CLI
2. Zed with `source.fixAll.oxc`
3. VS Code with `source.fixAll.oxc`
4. A locally patched Zed build

Observed behavior:

- `oxlint --fix` works, but only applies one non-overlapping batch per invocation
- stock Zed corrupts `l2.js` and `l3.js`
- VS Code fixes the same files correctly
- a patched local Zed build also fixes the same files correctly

Conclusion: the practical bug is in Zed's LSP edit handling, not in `oxc-zed` or the `oxlint --fix` CLI path.

## Versions

- `oxlint`: `1.57.0`
- `oxfmt`: `0.42.0`
- `pnpm`: `10.32.1`

Additional local analysis used these cloned repos:

- `oxc`: `af72b802be621fbea6e6ca1fbfc9a685c978b6fc`
- `oxc-zed`: `9b82cff67c223bf929be40489e36faafa536e71c`
- this repro repo: `9118bb677f238988e189946fa325dcabe749758e`

## Files

- `l1.js`: one unsorted object level
- `l2.js`: two nested unsorted object levels
- `l3.js`: three nested unsorted object levels
- `.zed/settings.json`: Zed workspace config
- `.vscode/settings.json`: VS Code workspace config
- `.vscode/extensions.json`: recommends the Oxc VS Code extension

## Install

```sh
pnpm install
```

## Editor setup

### Zed

This repo uses:

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

### VS Code / Cursor

This repo also includes:

```json
{
  "editor.codeActionsOnSave": {
    "source.fixAll.oxc": "always"
  }
}
```

Recommended extension:

- `oxc.oxc-vscode`

## Reproduction

Open this folder in Zed with the Oxc extension enabled, then save the files below.

### `l1.js`

Input:

```js
export const obj = {
  b: 1,
  a: 1,
};
```

Observed result in Zed:

- works correctly

Expected output:

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

Expected output:

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

Expected output:

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

## Comparison across environments

### `oxlint --fix`

Running:

```sh
pnpm lint:fix
```

does not corrupt the file. It only applies one non-overlapping batch per run, so nested `sort-keys` requires repeated invocations.

### VS Code / Cursor

With the included `.vscode/settings.json` and the Oxc extension:

- `l2.js` is fixed correctly
- `l3.js` is fixed correctly
- no corruption is observed

### Patched local Zed build

A local source build of Zed was patched and tested against this repo.

Observed result:

- `l3.js` is fixed correctly
- no corruption is observed
- resulting content:

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

This confirms the practical bug is in Zed's edit-merging or edit-application path.

## Relevant Zed log

When stock Zed saves `l3.js`, it logs:

```text
2026-03-31T11:32:17+02:00 INFO  [project::lsp_store] Applying edits [(Anchor { timestamp: Lamport {<local>: 1}, offset: 23, bias: Right, buffer_id: Some(BufferId(4294967980)) }..Anchor { timestamp: Lamport {<local>: 1}, offset: 31, bias: Left, buffer_id: Some(BufferId(4294967980)) }, ""), (Anchor { timestamp: Lamport {<local>: 1}, offset: 67, bias: Right, buffer_id: Some(BufferId(4294967980)) }..Anchor { timestamp: Lamport {<local>: 1}, offset: 77, bias: Left, buffer_id: Some(BufferId(4294967980)) }, "      e: 1,\n    },\n  },\n  b: 1c: {\n      f: 1,\n      e: 1,\n    },\n    d: 1e: 1,\n      f: 1")]. Content: "export const obj = {\n  b: 1,\n  a: {\n    d: 1,\n    c: {\n      f: 1,\n      e: 1,\n    },\n  },\n};\n"
2026-03-31T11:32:17+02:00 INFO  [project::lsp_store] Applied edits. New Content: "export const obj = {\n  a: {\n    d: 1,\n    c: {\n      f: 1,\n      e: 1,\n    },\n  },\n  b: 1c: {\n      f: 1,\n      e: 1,\n    },\n    d: 1e: 1,\n      f: 1,\n    },\n  },\n};\n"
```

The malformed replacement text is already visible in Zed's logged edit application, which is what led to the local Zed source investigation.
