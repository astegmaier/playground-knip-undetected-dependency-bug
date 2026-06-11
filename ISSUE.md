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

Knip fails to flag an undeclared sibling-workspace import, provided the install layout hoists sibling workspaces to root `node_modules/` — i.e. npm workspaces or yarn berry with `nodeLinker: node-modules`. In `--strict` mode, it correctly flags the undeclared dependency.

The bug also surfaces in per-package mode (`cd packages/consumer && knip`), even with `--strict`. We use per-package invocation in our monorepo for the performance reasons discussed in [#1642](https://github.com/webpro-nl/knip/issues/1642) / [#1711](https://github.com/webpro-nl/knip/issues/1711) — knip's docs call it a ["last resort"](https://knip.dev/guides/performance#a-last-resort) for slow CIs — but the root-invocation case demonstrates the bug independently.

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

### Run knip from root — silently passes

```bash
cd ../..
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

I bisected this against npm-published versions using a minimal harness in the [reproduction repo](https://github.com/astegmaier/playground-knip-undetected-dependency-bug). In all cases, running `--strict` from the root correctly detects the undeclared dependency. The table below summarizes the other results:

| Range | Run knip from root | Run knip from workspace package | Run knip from workspace package + `--strict` | Trigger |
|---|---|---|---|---|
| 2.x – **5.6.1** | ✅ | ✅ | ✅ | All patterns detect |
| **5.7.0** – 5.16.x | ❌ | ❌ | ❌ | Release notes: *"Start using `resolve` as the default module resolver"* |
| 5.17.0 – 6.13.1 | ✅ | ❌ | ❌ | A silently recovered (incidental side effect of a refactor); C/D still broken |
| **6.14.0** – 6.16.1 | ❌ | ❌ | ❌ | A regressed again — commit [`e7122a1ae`](https://github.com/webpro-nl/knip/commit/e7122a1ae74d8d43f6301b8758b7348c91fb4779) deliberately suppressed sibling-workspace imports in non-strict mode |
