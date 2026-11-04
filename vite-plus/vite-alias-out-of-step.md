# A Dependabot update of vite-plus leaves the `vite` alias behind

## The problem

Many projects on Vite+ declare `vite` as an alias for `@voidzero-dev/vite-plus-core`, the Vite build that Vite+ ships, so that every other package that asks for `vite`, such as the Cloudflare Vite plugin, gets the same copy. Each `vite-plus` release needs the `vite-plus-core` release with exactly the same version, so the alias and `vite-plus` must always name the same version. In a pnpm workspace, both usually live in the catalog, the `catalog:` block in `pnpm-workspace.yaml` that holds version pins shared by the workspace's packages. In an npm project, the alias appears twice in `package.json`:

```yaml
# pnpm-workspace.yaml
catalog:
  vite: npm:@voidzero-dev/vite-plus-core@1.4.0
  vite-plus: 1.4.0
```

```jsonc
// package.json
"devDependencies": { "vite": "npm:@voidzero-dev/vite-plus-core@1.1.0", "vite-plus": "1.1.0" },
"overrides": { "vite": "npm:@voidzero-dev/vite-plus-core@1.1.0" }
```

Dependabot updates `vite-plus` and leaves the alias at the old version, so the install puts two copies of Vite in `node_modules`. What fails next depends on the project, and it fails late, in a step whose error doesn't point at the alias.

In decision-machines-3000, an npm project, the build failed:

```text
error: Invalid vite task command: vp with args ["build"] under cwd /home/runner/work/jev-3000/jev-3000/
* Failed to resolve vite command: GenericFailure, Expected @voidzero-dev/vite-plus-core@1.1.0, but found @voidzero-dev/vite-plus-core@0.2.8 at /home/runner/work/jev-3000/jev-3000/node_modules/vite/package.json. Run `vp migrate` to align the Vite alias, then run `vp install`.
```

In Haystack, a pnpm workspace, the type check failed first. The commit that fixed it traced the errors to the Cloudflare plugin returning plugin types from the old copy of Vite:

```text
× typescript(TS2321): Excessive stack depth comparing types '{ publicDir: string; plugins: Plugin<any>[][]; }' and 'ViteUserConfigExport'.
   ╭─[apps/worker/vite.config.ts:4:16]
× typescript(TS2322): Type 'Plugin<any>[]' is not assignable to type 'PluginOption'.
   ╭─[apps/worker/vite.config.ts:8:13]
```

A mismatch doesn't always fail at all. Haystack's first one, in [PR #51](https://github.com/schalkneethling/haystack/pull/51), failed only because those plugin types happened to clash.

## Why it happens, and why the obvious fixes don't work

- **Dependabot doesn't update aliases.** In both projects, Dependabot's own commit moved `vite-plus` and left the alias untouched ([Haystack PR #51](https://github.com/schalkneethling/haystack/pull/51), [decision-machines-3000 PR #73](https://github.com/schalkneethling/decision-machines-3000/pull/73)). [dependabot-core#15847](https://github.com/dependabot/dependabot-core/issues/15847), an open feature request, describes why. Dependabot reads an alias under its real package name only when it builds the dependency graph. Version-update jobs skip aliases.
- **A Dependabot group doesn't help.** A group decides which packages update together, but Dependabot still never updates the alias.
- **Plain `vp migrate`, which the Vite+ error suggests, can pin the wrong version.** The Vite+ docs ("[Update Vite+](https://viteplus.dev/guide/upgrade-project)") say that on a project already on Vite+, it re-pins `vite-plus`, the alias, and any `vitest` pin "to the versions bundled with the CLI running the migration". That is the `vp` you run, often the global one, not the version Dependabot chose. So a global `vp` older than the update can move the project back down. Neither project tested this. The docs also give a form that names the target version, which avoids the problem:

  ```bash
  pnpm dlx --package=vite-plus@1.4.0 vp migrate --no-interactive
  npx --package=vite-plus@1.4.0 vp migrate --no-interactive
  ```

- **A workflow that commits the fix needs care.** It would install the new, not yet reviewed release while holding a token that can write. And a commit made with the workflow's `GITHUB_TOKEN` starts the pull request's next CI run only after someone approves it. The [generated-file entry](../dependabot/generated-file-out-of-date.md) shows how to build such a workflow safely. Both projects chose a check instead, because the fix is one line and the check needs no write token.

Vite+ tracks the problem in [voidzero-dev/vite-plus#2356](https://github.com/voidzero-dev/vite-plus/issues/2356), mainly for Renovate. It proposes a Renovate preset, and a guard inside `vp` that warns or fails when the installed `vite-plus-core` differs from the CLI's version. Neither existed on 4 November 2026, when the issue was still open.

## The solution: a check that fails first, with the exact line to set

A small script compares the two versions and fails CI in seconds when they differ, with a message that says what to change. CI runs it before installing dependencies, so it fails before the install, and before the misleading error.

```text
setup-vp (run-install: false)  →  node scripts/check-vite-alias.ts  →  vp install  →  checks, tests, build
                                   └─ fails here, with the line to set
```

The code to reuse:

- **npm `package.json`:** decision-machines-3000's [`scripts/check-vite-alias.ts`](https://github.com/schalkneethling/decision-machines-3000/blob/91fdae64ed5d4de41c6b5a14bba8f7de8bf102a4/scripts/check-vite-alias.ts), which checks the alias in both `devDependencies` and `overrides`. It was merged in [PR #87](https://github.com/schalkneethling/decision-machines-3000/pull/87).
- **pnpm catalog:** Haystack's [`scripts/check-vite-alias.ts`](https://github.com/schalkneethling/haystack/blob/aed99bc621357327fe63fb4f9eb3a22f44b89a44/scripts/check-vite-alias.ts), its [tests](https://github.com/schalkneethling/haystack/blob/aed99bc621357327fe63fb4f9eb3a22f44b89a44/scripts/check-vite-alias.test.ts), and the [Quality workflow](https://github.com/schalkneethling/haystack/blob/aed99bc621357327fe63fb4f9eb3a22f44b89a44/.github/workflows/quality.yml) that runs it. It is a port of the npm version, proposed in [PR #269](https://github.com/schalkneethling/haystack/pull/269), which was still open when this entry was written. Its CI passes on a matching catalog, but no real Dependabot mismatch has run through it yet.

The design choices that matter:

- **It runs before the install, so it needs only Node.js.** `setup-vp` gets `run-install: false`, the check runs, and then `vp install --frozen-lockfile` runs as its own step. Node.js runs the TypeScript file directly, which both projects do on Node.js 24, the version their CI uses. The pnpm version can't use a YAML parser, because nothing is installed yet, so it reads the `catalog:` block line by line. It counts only keys at the catalog's own indentation, so a `vite:` nested under another entry, or the `vite@*` key under `overrides:`, is never read as the alias.
- **The message says exactly what to do.** For example: ``Set `vite: npm:@voidzero-dev/vite-plus-core@1.4.0` in the catalog, run `vp install`, and commit pnpm-workspace.yaml and pnpm-lock.yaml.``
- **Inside GitHub Actions, it also prints an error annotation,** a `::error file=…,line=…::` line that GitHub attaches to that line of the file. The two versions point it at different lines. decision-machines-3000 points it at the `vite-plus` line, which Dependabot changed, so the error shows inline in the pull request's diff. Haystack points it at the `vite` line, the one to change, but that line isn't in the diff, so the error shows only in the check run's annotations. Pick the `vite-plus` line if you want it seen in the diff.
- **It rejects a range in `vite-plus`,** such as `^1.4.0`, because the alias must name one version.
- **It fails rather than passes when it can't read the file.** The pnpm version reads only a block-style `catalog:` at the top level. A flow-style `catalog: {…}` or a named catalog makes it fail with "no `vite-plus` entry", which is the wrong reason, but it never lets a mismatch through.
- **It assumes the project declares the alias.** The Vite+ docs say a pnpm package that uses only `vite-plus` needs no `vite` entry at all. The check reports a missing alias in that case, so drop or adapt it for such a project.
- **The decision is a pure function of the file's text,** so the tests pass strings, not files. Haystack's tests cover a match, the mismatch from the real incident, a prerelease, quoted values, a range, missing entries, Windows line endings, a byte order mark, and nested keys. One test also reads the repository's own file, so the check passes on the real workspace.

The commit that sets the alias is still manual.

### The manual steps, when the check fails

1. Check out the Dependabot pull request's branch.
2. Set the alias to the version the check names, wherever the project declares it: the catalog in `pnpm-workspace.yaml`, or both `devDependencies` and `overrides` in `package.json`. If the project also pins `vitest`, the Vite+ docs say to move that pin to the bundled version too, which `vp toolchain vitest` prints. The version-pinned `vp migrate` above does all of this in one step.
3. Run `vp install` to update the lockfile.
4. Commit the manifest and the lockfile to the same branch, with `[dependabot skip]` in the message, so that Dependabot can still rebase the pull request.

## Where it came up

- **decision-machines-3000 (formerly jev-3000):** [PR #73](https://github.com/schalkneethling/decision-machines-3000/pull/73) failed its build on the bump from 0.2.8 to 1.1.0, and was fixed by hand. [PR #87](https://github.com/schalkneethling/decision-machines-3000/pull/87) added the check.
- **Haystack:** [PR #51](https://github.com/schalkneethling/haystack/pull/51) (1.0.0 to 1.1.0) and the bump to 1.4.0 ([commit eaa3bad](https://github.com/schalkneethling/haystack/commit/eaa3baddb41673447f15025dc2012c6af16565ea)) were both fixed by hand. [Issue #267](https://github.com/schalkneethling/haystack/issues/267) and [PR #269](https://github.com/schalkneethling/haystack/pull/269) port the check to a pnpm catalog.
