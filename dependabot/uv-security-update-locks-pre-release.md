# A Dependabot security update for a uv project locks a development pre-release

## The problem

A grouped Dependabot security update for a uv project moves a package to a development pre-release, although stable releases that fix the vulnerability are on PyPI. Nothing fails: CI passes and the pull request looks ready, so the pre-release is easy to merge without noticing.

The case this entry comes from is create-project-calavera, whose only Python dependency is a lint tool, SkillSpector. Four Dependabot alerts were open for its transitive dependencies `langgraph`, `langgraph-cli`, `langgraph-sdk`, and `opentelemetry-api`. Dependabot opened one pull request for all four, titled "bump the uv group across 1 directory with 4 updates", and its description said:

```text
Updates `langgraph-cli` from 0.4.31 to 0.4.32.dev0
```

`0.4.32.dev0` is a development release, uploaded to PyPI about five hours after the stable `0.4.32`. Stable releases up to `0.4.43` were available, and SkillSpector only requires `langgraph-cli[inmem]>=0.4.14`.

Under Python's version rules (PEP 440), a `.devN` release sorts before the release it leads up to, so `0.4.32.dev0` is older than `0.4.32`, which is the alert's first patched version. You can check it with the `packaging` library:

```sh
uvx --from packaging python -c 'from packaging.version import Version as V; print(V("0.4.32.dev0") < V("0.4.32"))'
# True
```

## Why asking Dependabot again doesn't fix it

Commenting `@dependabot recreate` on the pull request rebuilt it with the same pre-release. The project's `dependabot.yml` had no `pip` or `uv` entry, so these pull requests came from Dependabot security updates, and there was no version-update configuration whose `ignore` or `allow` rules could have steered it.

Why Dependabot chose `0.4.32.dev0` isn't known. It wasn't the newest version by number, and uv itself doesn't choose it: by default, uv prefers stable versions and falls back to a pre-release only when no stable version satisfies the requirements ([uv's pre-release docs](https://docs.astral.sh/uv/concepts/resolution/#pre-release-handling)). To rule pre-releases out entirely, set `prerelease = "disallow"` under `[tool.uv]` in `pyproject.toml`, and allow the exceptions with `prerelease-package`, because a dependency that publishes only pre-releases, such as `opentelemetry-semantic-conventions` in the case below, can't resolve otherwise.

## The solution: close the pull request and regenerate the lock with uv

The fix is to let uv resolve the same packages, because uv applies the pre-release rules correctly.

1. Preview what uv would choose, without writing anything. Pass each package from the Dependabot pull request with `-P` (`--upgrade-package`), so uv upgrades only those packages, plus whatever they require:

   ```sh
   uv lock --dry-run -P langgraph-cli -P langgraph -P langgraph-sdk -P opentelemetry-api
   ```

   In create-project-calavera, this resolved stable `langgraph-cli` 0.4.43 and `langgraph` 1.2.22.

2. Open an issue and a branch, and write the lock with a current uv (see the next section for why the uv version matters):

   ```sh
   uvx --from uv==0.13.8 uv lock -P langgraph-cli -P langgraph -P langgraph-sdk -P opentelemetry-api
   ```

3. Read the lock diff. Expect some packages you didn't name to move, because the new versions require it. In create-project-calavera, three did, and the pull request listed each with its reason:

   | Package | Before | After | Why it moved |
   | --- | --- | --- | --- |
   | langgraph-checkpoint | 4.1.1 | 4.3.2 | langgraph 1.2.22 requires `>=4.3.0,<5` |
   | opentelemetry-sdk | 1.45.1 | 1.45.2 | It pins `opentelemetry-api==1.45.2` |
   | opentelemetry-semantic-conventions | 0.66b1 | 0.66b2 | The SDK pins `==0.66b2`; upstream publishes this package only as betas |

   The last row is a pre-release, and it's fine, because the package has no stable releases at all. The rule to check is that no package is on a pre-release when a stable release satisfies its requirements.

4. Check that the lock contains no unwanted pre-release, for example with `grep -nE 'version = ".*(dev|a|b|rc)[0-9]' uv.lock`, and run the project's checks.

5. Close the Dependabot pull request with a comment that links the replacement. Dependabot replies that closing a grouped pull request doesn't ignore these versions in the future, which is what you want.

6. After the replacement merges, check that the Dependabot alerts close. GitHub closes them when it rescans the default branch's lock file.

## Keep the lock file's format: regenerate with a current uv

`uv.lock` records its format version on the line `revision = N`. An older uv rewrites the whole file in the older format, so a dependency update can quietly downgrade it.

In create-project-calavera, the lock on `main` had `revision = 5`. Regenerating it with the local uv 0.10.9 wrote `revision = 3`, and so did uv 0.11.16, the version that CI pins. An adversarial review of the pull request caught the downgrade before it merged. uv 0.13.8, the newest release at the time, kept `revision = 5`, and `uvx --from uv==<version>` runs a specific uv without installing it.

Nothing broke, because the project's CI reads the lock with `uv run --frozen`, which uses the file as it is. The risk is churn: the next person who regenerates the lock with a newer uv upgrades the format again, and the diff fills with changes that have nothing to do with their update. Check the `revision` line in every lock diff, and consider raising the uv version that CI pins so it doesn't trail the format the lock already uses.

## Where it came up

- **create-project-calavera:** Dependabot [PR #913](https://github.com/schalkneethling/create-project-calavera/pull/913) locked `langgraph-cli` 0.4.32.dev0, before and after `@dependabot recreate`. [Issue #922](https://github.com/schalkneethling/create-project-calavera/issues/922) records the decision to regenerate the lock with uv, and [PR #923](https://github.com/schalkneethling/create-project-calavera/pull/923) did it, including the fix for the `revision` downgrade.
