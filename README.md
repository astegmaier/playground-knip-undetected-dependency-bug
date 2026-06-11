# Knip undetected-dependency bug — minimal reproduction

When knip is invoked from inside a workspace package directory ("isolated
workspace" / per-package mode) and the package is installed via a hoisting
package manager (yarn classic, npm, or pnpm with `shamefully-hoist`), knip
silently fails to detect missing dependency declarations.

## Repro structure

```
packages/
  fruit/
    package.json       # name: "@scope/fruit"
    src/index.ts       # export const apple = 'apple';
  consumer/
    package.json       # devDependencies: { "knip": "..." }  ← does NOT declare @scope/fruit
    src/index.ts       # import { apple } from '@scope/fruit';   ← undeclared usage
```

Yarn classic hoists every workspace into the root `node_modules/`, so
`@scope/fruit` is reachable from `packages/consumer` via the root
`node_modules/@scope/fruit` symlink — even though `consumer/package.json`
doesn't declare it.

## How to verify

```bash
git clone <this repo>
cd playground-knip-undetected-dependency-bug
yarn install
```

### Per-package mode — **silently passes (BUG)**

```bash
cd packages/consumer
yarn knip
# → exit 0, no output
```

Expected: `Unlisted dependencies (1)  @scope/fruit  src/index.ts:1:10`.

### Per-package mode + `--strict` — still silently passes (BUG)

```bash
yarn knip --strict
# → exit 0, no output
```

The silent drop happens in `packages/knip/src/graph/build.ts` before any
strict-mode logic runs, so `--strict` doesn't help.

### Monorepo mode (default) — silently passes (in knip ≥ 6.14.0)

```bash
cd ../..
yarn knip
# → exit 0, no output
```

This is intentional per [#1742](https://github.com/webpro-nl/knip/pull/1742) ("Don't flag undeclared sibling
workspace imports as unlisted") which landed in 6.14.0, but is debatable —
sibling workspaces are still external from a `package.json` correctness
perspective.

### Monorepo mode + `--strict` — correctly detects ✓

```bash
yarn knip --strict
# → Unlisted dependencies (1)
#     @scope/fruit  packages/consumer/src/index.ts:1:10
# → exit 1
```

## See also

- `ISSUE.md` — draft of the GitHub issue for [webpro-nl/knip](https://github.com/webpro-nl/knip/issues).
- Related closed issue [#1711](https://github.com/webpro-nl/knip/issues/1711) — same triggering conditions
  (per-package invocation + hoisting), different code path (manifest loading
  vs. import-graph filtering).
