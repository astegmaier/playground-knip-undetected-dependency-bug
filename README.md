# Knip undetected-dependency bug — minimal reproduction

When knip is invoked from inside a workspace package directory ("isolated
workspace" / per-package mode) and the package is installed via a layout that
hoists sibling workspaces to the root `node_modules/` (npm workspaces, or
yarn berry with `nodeLinker: node-modules`), knip silently fails to detect
missing dependency declarations.

This is the same triggering pair as
[webpro-nl/knip#1711](https://github.com/webpro-nl/knip/issues/1711)
(per-package invocation + hoisting), but a different underlying code path —
manifest loading there, import-graph filtering here.

See `ISSUE.md` for the regression report formatted to fit the
[knip regression-report form](https://github.com/webpro-nl/knip/issues/new?template=regression.yml).

## Repro structure

```
packages/
  fruit/
    src/index.js       # export const apple = 'apple';
    package.json       # name: "@scope/fruit"
  consumer/
    src/index.js       # import { apple } from '@scope/fruit';   ← undeclared usage
    package.json       # name: "@scope/consumer"                  ← does NOT declare @scope/fruit
package.json           # root, workspaces: ["packages/*"], devDependencies: { "knip": "6.16.1" }
.yarnrc.yml            # nodeLinker: node-modules
```

yarn berry (with `nodeLinker: node-modules`) symlinks every workspace into the
root `node_modules/`, so `@scope/fruit` is reachable from `packages/consumer`
via the root `node_modules/@scope/fruit` symlink — even though
`consumer/package.json` doesn't declare it.

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
# ✂️  Excellent, Knip found no issues.
```

Expected: `Unlisted dependencies (1)  @scope/fruit  src/index.js:1:10`.

The `-T` (top-level) flag is needed because yarn berry restricts binary
lookups to packages declared in the current workspace's manifest. Since
`@scope/consumer` doesn't (and shouldn't) declare `knip`, `-T` tells yarn
to run the binary from the root workspace.

### Per-package mode + `--strict` — still silently passes (BUG)

```bash
yarn knip --strict
# ✂️  Excellent, Knip found no issues.
```

The silent drop happens in `packages/knip/src/graph/build.ts` before any
strict-mode logic runs, so `--strict` doesn't help.

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

All three currently-failing invocations are regressions — older knip versions
detected them. I bisected against npm-published versions using a minimal
harness:

| Range | monorepo default | monorepo `--strict` | per-pkg default | per-pkg `--strict` | Trigger |
|---|---|---|---|---|---|
| 2.x – **5.6.1** | ✅ | ✅ | ✅ | ✅ | All patterns detect |
| **5.7.0** – 5.16.x | ❌ | ❌ | ❌ | ❌ | Release notes: *"Start using `resolve` as the default module resolver"* |
| 5.17.0 – 6.13.1 | ✅ | ✅ | ❌ | ❌ | Monorepo silently recovers; per-pkg still broken |
| **6.14.0** – 6.16.1 | ❌ | ✅ | ❌ | ❌ | Monorepo default regresses again — commit [`e7122a1ae`](https://github.com/webpro-nl/knip/commit/e7122a1ae74d8d43f6301b8758b7348c91fb4779) ("Don't flag undeclared sibling workspace imports as unlisted") |

The headline regression is **5.6.1 → 5.7.0**: it broke per-package mode and
that mode has been silently broken in every release for ~15 months and
counting. The 5.17.0 / 6.14.0 movements in monorepo mode are a side note —
neither affected per-package mode.

Note on the 6.14.0 commit: its title references `(#1742)`, but issue
[#1742](https://github.com/webpro-nl/knip/issues/1742) is actually about
peer/dev dependency duplication. The sibling-workspace change appears to have
been bundled into the same PR but isn't called out in the issue thread.

### How to reproduce the bisection

The four invocations to re-run after each version swap (see [How to verify](#how-to-verify)
above for full commands and expected output):

1. `yarn knip` from repo root — monorepo default
2. `yarn knip --strict` from repo root — monorepo + `--strict`
3. `cd packages/consumer && yarn knip` — per-pkg default
4. `cd packages/consumer && yarn knip --strict` — per-pkg + `--strict`

```bash
# Bisect — swap in any version and re-run the four invocations above:
yarn add -D knip@5.6.1   # all 4 detect ✓
yarn add -D knip@5.7.0   # all 4 miss ✗ (yes, even monorepo + --strict)
yarn add -D knip@5.16.0  # all 4 miss ✗ (last version in the 5.7.0–5.16.x range)
yarn add -D knip@5.17.0  # monorepo recovers (both default and --strict) ✓, per-pkg still broken ✗
yarn add -D knip@6.13.1  # same — monorepo ✓, per-pkg ✗
yarn add -D knip@6.14.0  # monorepo default regresses ✗, but monorepo + --strict still detects ✓; per-pkg still broken ✗
yarn add -D knip@6.16.1  # same as 6.14.0 — monorepo default ✗, monorepo --strict ✓, per-pkg ✗
```

## Root cause analysis

In `packages/knip/src/graph/build.ts` (`analyzeSourceFile`), resolved imports
are added to `file.imports.external` only if the imported package is a known
workspace _or_ already declared in the current workspace's `package.json`
([build.ts#L432-L444](https://github.com/webpro-nl/knip/blob/knip%406.16.1/packages/knip/src/graph/build.ts#L432-L444)):

```ts
const wsDependencies = deputy.getDependencies(workspace.name);
for (const _import of file.imports.imports) {
  if (!_import.filePath) continue;
  const packageName = getPackageNameFromModuleSpecifier(_import.specifier);
  if (!packageName) continue;
  const isWorkspace = isInternalWorkspace(packageName);
  if (isWorkspace || wsDependencies.has(packageName)) {
    file.imports.external.add({ ..._import, specifier: packageName });
    ...
  }
}
```

The `unlisted` check in `graph/analyze.ts` only walks `file.imports.external`,
so any import that fails this gate is silently invisible to the analyzer.

When knip runs in per-package mode, only the consumer workspace's
`package.json` is loaded (the consumer doesn't declare a `workspaces` field),
so `availableWorkspacePkgNames` is empty and `isInternalWorkspace(packageName)`
returns false for every sibling. Combined with the missing declaration,
**both branches of the `if` are false → the import is silently dropped** and
never reaches the unlisted-dependency check.

For comparison, **unresolved** imports take a different code path
([build.ts#L404-L420](https://github.com/webpro-nl/knip/blob/knip%406.16.1/packages/knip/src/graph/build.ts#L404-L420))
that adds anything package-name-shaped to `file.imports.external`
unconditionally. That's why this same code _correctly_ reports missing
declarations when the repo is installed with pnpm's default (strict /
non-hoisted) layout: the import doesn't resolve, falls into the "unresolved"
branch, and is properly classified as `unlisted`.

The same asymmetry explains the **5.7.0 regression**: before 5.7.0 knip leaned
more heavily on TypeScript's own resolution, so the sibling-workspace import
landed in `file.imports.unresolved` and was caught; the
*"Start using `resolve` as the default module resolver"* change in 5.7.0
caused it to resolve through hoisting and fall into the silent-drop gate
instead.

The **6.14.0 monorepo regression** has a separate cause: commit `e7122a1ae`
added a short-circuit in `DependencyDeputy.maybeAddReferencedExternalDependency`
that silently marks any sibling-workspace import as referenced in non-strict
mode. The commit ships a test asserting this behavior.

### Observed in the wild

I observed this exact asymmetry in our monorepo: after removing a dependency
declaration from two packages, `yarn knip --continue` (per-package mode, yarn
berry with `nodeLinker: node-modules`) reported zero issues, while running the
same analysis after re-installing the repo with pnpm correctly surfaced both
packages as having unlisted dependencies — because pnpm keeps workspaces in
`.pnpm/node_modules/` rather than the hoisted root layout, so the
sibling-workspace import is genuinely unresolvable and falls into the
"unresolved" branch instead of the silent-drop gate.

## Suggested direction

Always add resolved external imports to `file.imports.external` when they
target a package that isn't part of the current workspace's transitive
type-includes set, and let `DependencyDeputy.maybeAddReferencedExternalDependency`
make the final call about whether it's `unlisted`. That collapses the
per-package vs. monorepo asymmetry and matches the path the unresolved-imports
branch already takes.

## See also

- `ISSUE.md` — draft regression report formatted for the
  [knip regression-report form](https://github.com/webpro-nl/knip/issues/new?template=regression.yml).
- Related closed issue [#1711](https://github.com/webpro-nl/knip/issues/1711)
  — same triggering conditions (per-package invocation + hoisting), different
  code path (manifest loading vs. import-graph filtering).
