# oxlint issues with zed and fixAll

TODO: better description
the `sort-keys` rule sort keys, but for nested keys the oxc-zed plugin tries to sort everything at once (speculation) which leads to invalid offsets for replacements which corrupts the file.

Also... is the `oxlint --fix` behaviour correct? should it really only fix one level at a time or should it fix all, like the zed extension is trying to do?

## oxc-zed

`"code_action": "source.fixAll.oxc",` in `.zed/settings.json`:

### single level input (l1.js):

```js
export const obj = {
  b: "b",
  a: "a",
};
```

works fine, nothing to mess up.

### two levels (l2.js):

```js
export const obj = {
  b: "b",
  a: {
    d: "d",
    c: "c",
  },
};
```

corrupts to:

```js
export const obj = {
  a: {
    d: "d",
    c: "c",
  },
  b: "b"c: "c",
    d: "d",
  },
};
```

Looks like it is trying to fix both levels at the same time.

### Three levels (l3.js):

```js
export const obj = {
  b: "b",
  a: {
    d: "d",
    c: {
      f: "f",
      e: "e",
    },
  },
};
```

results in:

```js
export const obj = {
  a: {
    d: "d",
    c: {
      f: "f",
      e: "e",
    },
  },
  b: "b"c: {
      f: "f",
      e: "e",
    },
    d: "d"e: "e",
      f: "f",
    },
  },
};
```

Which seems to strengthen the suspicion. Each leaf fix correct but the parent moves to a different offset while the fix is inserted at the old offset.

## oxlint --fix

Running `oxlint --fix` only fixes one level at a time:

example with three levels of unsorted keys:

```js
export const obj = {
  b: "b",
  a: {
    d: "d",
    c: {
      f: "f",
      e: "e",
    },
  },
};
```

first pass only fixes root level:

```sh
❯ pnpm lint:fix

> zed-oxc-repro@1.0.0 lint:fix /Users/johan/repos/github.com/liljedev/zed-oxc-repro
> oxlint --fix


  ⚠ eslint(sort-keys): Object keys should be sorted
    ╭─[test.js:3:6]
  2 │       b: "b",
  3 │ ╭─▶   a: {
  4 │ │       d: "d",
  5 │ │       c: {
  6 │ │         f: "f",
  7 │ │         e: "e",
  8 │ │       }
  9 │ ╰─▶   },
 10 │     };
    ╰────
  help: Replace `d: "d",
            c: {
              f: "f",
              e: "e",
            }` with `c: {
              f: "f",
              e: "e",
            },
            d: "d"`.

  ⚠ eslint(sort-keys): Object keys should be sorted
   ╭─[test.js:5:8]
 4 │         d: "d",
 5 │ ╭─▶     c: {
 6 │ │         f: "f",
 7 │ │         e: "e",
 8 │ ╰─▶     }
 9 │       },
   ╰────
  help: Replace `f: "f",
              e: "e"` with `e: "e",
              f: "f"`.

Found 2 warnings and 0 errors.
Finished in 4ms on 1 file with 94 rules using 12 threads.
```

second pass fixes second level:

```sh
❯ pnpm lint:fix

> zed-oxc-repro@1.0.0 lint:fix /Users/johan/repos/github.com/liljedev/zed-oxc-repro
> oxlint --fix


  ⚠ eslint(sort-keys): Object keys should be sorted
   ╭─[test.js:4:8]
 3 │         d: "d",
 4 │ ╭─▶     c: {
 5 │ │         f: "f",
 6 │ │         e: "e",
 7 │ ╰─▶     }
 8 │       },
   ╰────
  help: Replace `f: "f",
              e: "e"` with `e: "e",
              f: "f"`.

Found 1 warning and 0 errors.
Finished in 5ms on 1 file with 94 rules using 12 threads.
```

Third pass fixes third level:

```sh
❯ pnpm lint:fix

> zed-oxc-repro@1.0.0 lint:fix /Users/johan/repos/github.com/liljedev/zed-oxc-repro
> oxlint --fix

Found 0 warnings and 0 errors.
Finished in 4ms on 1 file with 94 rules using 12 threads.
```
