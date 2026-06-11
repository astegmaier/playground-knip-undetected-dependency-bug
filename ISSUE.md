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
2. the repo uses a "hoisting" install strategy (yarn classic, npm, or pnpm with `shamefully-hoist`), so the missing dependency is still reachable via the root `node_modules`.

This is the same triggering pair as [#1711](https://github.com/webpro-nl/knip/issues/1711), but a different underlying code path.

### Symptom: undeclared sibling-workspace import not reported

**myRepo/packages/consumer/package.json** (see [simplified reproduction](https://github.com/astegmaier/playground-knip-undetected-dependency-bug/blob/main/packages/consumer/package.json))

```json
{
  "name": "@scope/consumer",
  "private": true,
  "devDependencies": {
    "knip": "6.16.1"
  }
}
```

**myRepo/packages/consumer/src/index.ts**

```ts
import { apple } from '@scope/fruit'; // ← @scope/fruit is NOT declared in consumer's package.json
export const x = apple;
```

`@scope/fruit` is a sibling workspace. Yarn classic hoists it to the root `node_modules/@scope/fruit` (as a symlink to `packages/fruit`), so TS resolves the import from `packages/consumer` without complaint.

Running knip from the workspace root with `--strict` will (correctly) report the missing declaration:

```
yarn knip --strict

Unlisted dependencies (1)
@scope/fruit  packages/consumer/src/index.ts:1:10
```

But running knip from the consumer directory will (incorrectly) report nothing:

```
cd packages/consumer
yarn knip
# → exit 0, no output

yarn knip --strict
# → exit 0, no output
```

Note that `--strict` does **not** fix the per-package case — the silent drop happens before any strict-mode logic runs.

### Regression history

I bisected this against npm-published versions using a minimal harness in the [reproduction repo](https://github.com/astegmaier/playground-knip-undetected-dependency-bug). The three failure modes I tested:

- **A** = per-package mode, default (`cd packages/consumer && knip`)
- **B** = per-package mode, `--strict`
- **C** = monorepo mode, default (`knip` from root)

Every release from 2.x through **5.6.1** detected the bug in all three modes. **5.7.0** broke all three, and per-package mode (A and B) has been silently broken in every release since. The release notes for 5.7.0 read *"Start using `resolve` as the default module resolver"* — this matches the root-cause analysis below.

A side note for completeness: a refactor in **5.17.0** unintentionally surfaced the bug again in monorepo mode (C) only — that window lasted until **6.14.0** (commit [`e7122a1ae`](https://github.com/webpro-nl/knip/commit/e7122a1ae74d8d43f6301b8758b7348c91fb4779), bundled into a different PR) deliberately suppressed sibling-workspace imports in non-strict mode and re-broke C. Per-package mode (A and B) was unaffected by either of those changes and has been silently passing this entire time.

The only currently-reliable invocation is **monorepo mode + `--strict`**, which detects the bug in every version 2.x – 6.16.1.

### Root cause analysis

In `packages/knip/src/graph/build.ts` (`analyzeSourceFile`), resolved imports are added to `file.imports.external` only if the imported package is a known workspace _or_ already declared in the current workspace's `package.json` ([build.ts#L432-L444](https://github.com/webpro-nl/knip/blob/knip%406.16.1/packages/knip/src/graph/build.ts#L432-L444)):

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

The `unlisted` check in `graph/analyze.ts` only walks `file.imports.external`, so any import that fails this gate is silently invisible to the analyzer.

When knip runs in per-package mode, only the consumer workspace's `package.json` is loaded (the consumer doesn't declare a `workspaces` field), so `availableWorkspacePkgNames` is empty and `isInternalWorkspace(packageName)` returns false for every sibling. Combined with the missing declaration, **both branches of the `if` are false → the import is silently dropped** and never reaches the unlisted-dependency check.

For comparison, **unresolved** imports take a different code path ([build.ts#L404-L420](https://github.com/webpro-nl/knip/blob/knip%406.16.1/packages/knip/src/graph/build.ts#L404-L420)) that adds anything package-name-shaped to `file.imports.external` unconditionally. That's why this same code _correctly_ reports missing declarations when the repo is installed with pnpm's default (strict / non-hoisted) layout: the import doesn't resolve, falls into the "unresolved" branch, and is properly classified as `unlisted`. The same asymmetry explains the 5.7.0 regression — before 5.7.0 knip leaned more heavily on TypeScript's own resolution, so the sibling-workspace import landed in `file.imports.unresolved` and was caught; the *"Start using `resolve` as the default module resolver"* change in 5.7.0 caused it to resolve through hoisting and fall into the silent-drop gate instead.

I observed this exact asymmetry in our monorepo: after removing a dependency declaration from two packages, `yarn knip --continue` (per-package mode, yarn classic hoisting) reported zero issues, while running the same analysis after re-installing with pnpm correctly surfaced both packages as having unlisted dependencies.

### Suggested direction

Always add resolved external imports to `file.imports.external` when they target a package that isn't part of the current workspace's transitive type-includes set, and let `DependencyDeputy.maybeAddReferencedExternalDependency` make the final call about whether it's `unlisted`. That collapses the per-package vs. monorepo asymmetry and matches the path the unresolved-imports branch already takes.

I'm happy to take a stab at the PR if you're open to the change.
