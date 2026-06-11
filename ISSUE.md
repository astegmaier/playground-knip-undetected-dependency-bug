## Title

🔄 Regression: knip silently fails to detect missing dependency declarations when run from a workspace package directory under hoisting

## Prerequisites

- [x] I've read the relevant [documentation](https://knip.dev)
- [x] I've searched for [existing issues](https://github.com/webpro-nl/knip/issues?q=is%3Aissue)
- [x] I've read the [issue reproduction guide](https://knip.dev/guides/issue-reproduction)

## Reproduction url

https://github.com/astegmaier/playground-knip-undetected-dependency-bug

## Reproduction access

- [x] I've made sure the reproduction is publicly accessible

## Good version

5.6.1

## Bad version

5.7.0

## Description of the regression

Knip silently fails to detect a missing dependency declaration when **both** of these conditions are true:

1. knip is invoked from a workspace package directory (we still rely on this pattern in our monorepo for the performance reasons discussed in [#1642](https://github.com/webpro-nl/knip/issues/1642#issuecomment-4136489174) / [#1711](https://github.com/webpro-nl/knip/issues/1711)).
2. the repo uses an install layout that hoists sibling workspaces to the root `node_modules/` (npm workspaces, or yarn berry with `nodeLinker: node-modules`), so the missing dependency is still reachable from the consumer's `node_modules/@scope/fruit` lookup.

This is the same triggering pair as [#1711](https://github.com/webpro-nl/knip/issues/1711), but a different underlying code path.

## Example

Given this simplified repro structure:

```
packages/
  fruit/
    src/index.js       # export const apple = 'apple';
    package.json       # name: "@scope/fruit"
  consumer/
    src/index.js       # import { apple } from '@scope/fruit'; ← undeclared usage
    package.json       # does NOT declare `@scope/fruit` as a dependency
package.json           # root, workspaces: ["packages/*"], devDependencies: { "knip": "6.16.1" }
.yarnrc.yml            # nodeLinker: node-modules
```

`@scope/fruit` is a sibling workspace. yarn berry (with `nodeLinker: node-modules`) symlinks it to root `node_modules/@scope/fruit` (pointing at `packages/fruit`), so node module resolution finds the import from `packages/consumer` without complaint.

You'll get these results...

### Per-package mode — **silently passes (BUG)**

```bash
cd packages/consumer
yarn knip
# ✂️  Excellent, Knip found no issues.
```

Expected: `Unlisted dependencies (1)  @scope/fruit  src/index.js:1:10`.

### Per-package mode + `--strict` — still silently passes (BUG)

```bash
yarn knip --strict
# ✂️  Excellent, Knip found no issues.
```

### Monorepo mode (default) — silently passes (in knip ≥ 6.14.0)

```bash
cd ../..
yarn knip
# ✂️  Excellent, Knip found no issues.
```

### Monorepo mode + `--strict` — correctly detects ✓

```bash
yarn knip --strict
# → Unlisted dependencies (1)
#     @scope/fruit  packages/consumer/src/index.js:1:10
```

## Regression history

I bisected this against npm-published versions using a minimal harness in the [reproduction repo](https://github.com/astegmaier/playground-knip-undetected-dependency-bug). The four invocations from the Example section above are labelled A/B/C/D below (B — monorepo + `--strict` — is omitted from the table since it correctly detects in every version 2.x – 6.16.1):

| Range | A monorepo default | C per-pkg default | D per-pkg `--strict` | Trigger |
|---|---|---|---|---|
| 2.x – **5.6.1** | ✅ | ✅ | ✅ | All patterns detect |
| **5.7.0** – 5.16.x | ❌ | ❌ | ❌ | Release notes: *"Start using `resolve` as the default module resolver"* |
| 5.17.0 – 6.13.1 | ✅ | ❌ | ❌ | A silently recovered (incidental side effect of a refactor); C/D still broken |
| **6.14.0** – 6.16.1 | ❌ | ❌ | ❌ | A regressed again — commit [`e7122a1ae`](https://github.com/webpro-nl/knip/commit/e7122a1ae74d8d43f6301b8758b7348c91fb4779) deliberately suppressed sibling-workspace imports in non-strict mode |

The main story is the **5.6.1 → 5.7.0** regression: it broke per-package mode (C/D) and that mode has been silently broken in every release for ~15 months and counting. The 5.17.0 / 6.14.0 movements in monorepo mode (A) are a side note — neither affected per-package mode.
