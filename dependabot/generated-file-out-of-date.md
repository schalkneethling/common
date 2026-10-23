# A Dependabot pull request fails because a generated file is out of date

## The problem

A Dependabot pull request fails CI, although the update itself is fine. The failing check compares a file that a tool generates with the file in the repository, and the update changed what the tool generates.

The case this entry comes from is a Cloudflare Worker. Wrangler generates `worker-configuration.d.ts` from the Workers runtime (workerd) that it bundles. A Wrangler update brings a newer workerd, so the generated types change, and the check fails:

```text
✘ [ERROR] Types at worker-configuration.d.ts are out of date. Run `wrangler types` to regenerate.
```

The check is doing its job, because it catches a committed file that no longer matches its generator. The trouble is that every such pull request then needs someone to regenerate the file by hand.

## Why a Dependabot group doesn't fix it

Dependabot updates the version numbers in the manifest and the lockfile. It doesn't run the project's own commands, such as a code generator, so no Dependabot setting can make it regenerate the file. A group only decides which packages update together. In the Haystack case, the group that keeps `wrangler` and `@cloudflare/vite-plugin` in step was already there, and the types still failed.

The same shape came up in another project: vite-plus needs its `vite` alias moved in step, and Dependabot never moves it, so every vite-plus pull request failed its build until someone moved the alias by hand. A group couldn't fix that either.

## The solution: regenerate the file in a workflow, and document the manual steps

The automation does the work, and the documented steps are the backup for when it fails.

### The workflow

On Dependabot's pull requests, a workflow regenerates the file and commits it to the pull request's branch. Haystack's version is [`.github/workflows/dependabot-worker-types.yml`](https://github.com/schalkneethling/haystack/blob/dc883e4f6e738b0565fbed3cbb4773f0459461d0/.github/workflows/dependabot-worker-types.yml), and its design is the part worth reusing:

- **Two jobs, so the updated dependency's code never runs with a token that can write.** Installing the update runs that package's code, and a compromised release could do anything with a write token.
  - The first job has `contents: read`. It checks out the pull request's head with `persist-credentials: false`, installs, runs the generator, and uploads the file as an artifact only when it changed.
  - The second job is the only one with `contents: write`. It never checks out or installs anything. It downloads the artifact and checks it: exactly one regular file with the expected name, a plausible size, valid UTF-8 text, and the generator's header lines. Then it commits the file.
- **The commit goes through GitHub's API,** with the GraphQL `createCommitOnBranch` mutation, not a `git push`. GitHub signs a commit made this way. `expectedHeadOid` is the commit the first job checked out, so a branch that moved in the meantime isn't overwritten.
- **It runs only for Dependabot,** guarded on `github.event.pull_request.user.login == 'dependabot[bot]'` and on the branch being in the same repository. It checks the pull request's author, not `github.actor`, so a maintainer's push to the branch still gets the file regenerated.
- **Untrusted values reach the scripts only through `env:`,** and jq builds the GraphQL request, so a branch name is never parsed as shell or GraphQL.
- **The commit message contains `[dependabot skip]`.** By default, Dependabot stops rebasing a pull request once someone else has committed to it. The marker lets it keep rebasing, and the workflow regenerates the file again on the rebased branch.

Two things to know when using it:

- **The next CI run needs an approval.** A commit made with the workflow's own `GITHUB_TOKEN` starts the pull request's next run in an "approval required" state ([GITHUB_TOKEN](https://docs.github.com/en/actions/concepts/security/github_token)), so someone with write access approves it once. This happens again after each rebase.
- **A pull request that Dependabot has stopped rebasing doesn't start the workflow by itself.** Comment `@dependabot rebase` on it, or select **Update branch**.

### The manual steps, when the workflow fails

1. Check out the Dependabot pull request's branch, and install the dependencies.
2. Run the generator. In Haystack, that's `vp run @haystack/worker#types`.
3. Run the check, `vp run @haystack/worker#types:check`, to confirm the file now matches.
4. Commit only the generated file to that same branch, with `[dependabot skip]` in the message. Don't open a separate pull request: the fix belongs with the update.

## Where it came up

- **Haystack:** [PR #109](https://github.com/schalkneethling/haystack/pull/109) failed on a Wrangler update. [Issue #9](https://github.com/schalkneethling/haystack/issues/9) records the decision to automate and document, and [PR #145](https://github.com/schalkneethling/haystack/pull/145) added the workflow, with its security review. The workflow's first real run regenerated #109's types, and the next CI run passed.
- **Another project:** the vite-plus `vite` alias, described above. The same kind of workflow could fix it, with moving the alias as the step the first job runs.
