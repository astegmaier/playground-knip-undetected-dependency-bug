## Title

🔄 Regression: knip fails to detect missing dependency declarations for sibling workspace packages when there is hoisting

## Reproduction url

https://github.com/astegmaier/playground-knip-undetected-dependency-bug

## Good version

5.6.1

## Bad version

5.7.0

## Description of the regression

Knip fails to flag an undeclared sibling-workspace import, provided the install layout hoists sibling workspaces to root `node_modules/` — i.e. npm workspaces or yarn berry with `nodeLinker: node-modules`. In `--strict` mode, it correctly flags the undeclared dependency.

The bug also surfaces in per-package mode (`cd packages/consumer && knip`), even with `--strict`. We use per-package invocation in our monorepo for the performance reasons discussed in [#1642](https://github.com/webpro-nl/knip/issues/1642) / [#1711](https://github.com/webpro-nl/knip/issues/1711) — knip's docs call it a ["last resort"](https://knip.dev/guides/performance#a-last-resort) for slow CIs — but the root-invocation case demonstrates the bug independently.

## Example

Given this simplified repro structure, installed with latest yarn (`4.12.0`) with `nodeLinker: node-modules` (i.e. hoisting).

```
packages/
  fruit/
    src/index.js       # export const apple = 'apple';
    package.json       # name: "@scope/fruit"
  consumer/
    src/index.js       # import { apple } from '@scope/fruit'; ← undeclared usage
    package.json       # does NOT declare `@scope/fruit` as a dependency
package.json           # workspaces: ["packages/*"]
.yarnrc.yml            # nodeLinker: node-modules
```

You'll get these results with the latest version of knip (`6.16.1`)...

### Run knip from root — silently passes (BUG)

```bash
yarn knip
# ✂️  Excellent, Knip found no issues.
```

### Run knip from root + `--strict` — correctly detects ✓

```bash
yarn knip --strict
# → Unlisted dependencies (1)
#     @scope/fruit  packages/consumer/src/index.js:1:10
```

### Run knip from workspace package — silently passes (BUG)

```bash
cd packages/consumer
yarn knip
# ✂️  Excellent, Knip found no issues.
```

Expected: `Unlisted dependencies (1)  @scope/fruit  src/index.js:1:10`.

### Run knip from workspace package + `--strict` — silently passes (BUG)

```bash
cd packages/consumer
yarn knip --strict
# ✂️  Excellent, Knip found no issues.
```

## Regression history

I bisected this against npm-published versions using a minimal harness in the [reproduction repo](https://github.com/astegmaier/playground-knip-undetected-dependency-bug). The table below summarizes the results across all four invocations:

| Range | Run knip from root | Run knip from root + `--strict` | Run knip from workspace package | Run knip from workspace package + `--strict` | Trigger |
|---|---|---|---|---|---|
| 2.x – **5.6.1** | ✅ | ✅ | ✅ | ✅ | All patterns detect |
| **5.7.0** – 5.16.x | ❌ | ❌ | ❌ | ❌ | Release notes: *"Start using `resolve` as the default module resolver"* |
| 5.17.0 – 6.13.1 | ✅ | ✅ | ❌ | ❌ | A/B silently recovered (incidental side effect of a refactor); C/D still broken |
| **6.14.0** – 6.16.1 | ❌ | ✅ | ❌ | ❌ | A regressed again — commit [`e7122a1ae`](https://github.com/webpro-nl/knip/commit/e7122a1ae74d8d43f6301b8758b7348c91fb4779) deliberately suppressed sibling-workspace imports in non-strict mode |
