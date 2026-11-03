# pnpm warns that `_auth` can't be set in the global config file

## The problem

Every pnpm command in a project prints a warning about the registry credentials in the global config file:

```text
[WARN] The following settings cannot be set in the global config file ("/Users/schalkneethling/Library/Preferences/pnpm/config.yaml") and were ignored: "_auth". Move them to a project-level pnpm-workspace.yaml. To share these settings across projects, use config dependencies: https://pnpm.io/11.x/config-dependencies
```

The warning is accurate about one thing: that pnpm ignored the setting. Its advice is wrong, though, and following it would leak the token, as the next section explains.

The global config file holds an `_auth` setting shaped like this, with the token in place of `<token>`:

```yaml
_auth:
  https://registry.npmjs.org/:
    '@':
      authToken: <token>
```

## Why it happens

`_auth` in the global `config.yaml` is a newer way to store registry credentials, and the project runs a pnpm version that predates it.

- **pnpm 11.10.0 added the setting.** It is one value keyed by registry URL, and `@` under a registry means the credentials for the whole registry. pnpm reads it only from the global `config.yaml` and from the `pnpm_config__auth` environment variable. That restriction is deliberate: a file committed to a repository can never supply the credentials, or send them to a different host.
- **pnpm 12.1.0 made `pnpm login` write the token there.** Earlier versions wrote it to `auth.ini`. So a routine `pnpm login` with a current global pnpm creates the setting, without anyone editing the file.
- **pnpm 11.9.0 and earlier don't know the setting.** They treat it as an unknown key, drop it, and print the warning. A project that pins an older pnpm in `package.json`'s `packageManager` field gets that older version, because pnpm switches to the pinned version for every command run in the project. So the global pnpm can be new and the warning still appears.

The warning's two suggestions don't apply:

- **Moving `_auth` to `pnpm-workspace.yaml` doesn't work, and it is unsafe.** Every pnpm version that reads `_auth` ignores it in a project file, by design, and a project file is usually committed, which would publish the token.
- **Config dependencies don't fit either.** They share settings through a package that a project installs, which is the wrong place for a personal token.

I confirmed the version boundary by running each installed pnpm against the same global config, from a directory with no `packageManager` pin: 11.3.0 and 11.9.0 print the warning, and 11.10.0 and 11.13.0 don't. Two things can make such a test misleading:

- **Run it outside the project.** Inside a project that pins a version, every pnpm binary switches to the pinned one, so every run shows the pinned version's behaviour.
- **Don't test with `pnpm config get`.** From 11.23.0, `pnpm config get` no longer prints config warnings, so a clean result from it proves nothing.

## What the warning means for authentication

A pnpm version that ignores `_auth` doesn't see that token at all. It still reads credentials from `~/.npmrc` (a line such as `//registry.npmjs.org/:_authToken=…`), so authentication keeps working only if `~/.npmrc` holds a valid token. The two files can hold different tokens, for example when `npm login` wrote `~/.npmrc` and `pnpm login` wrote `config.yaml`. In that case, an older pnpm uses only the `~/.npmrc` token, and a newer one also reads the `config.yaml` token, so which token a command uses can depend on the pnpm version the project pins. A token that works in one project can then fail in another.

## The solution: update the project's pinned pnpm

Update the `packageManager` field to pnpm 11.10.0 or later. The latest 11.x release is the smallest change, and a move to pnpm 12 is a bigger change that needs its own testing.

```json
"packageManager": "pnpm@11.28.8"
```

Then install, so the lockfile and any version checks agree, and run a command to confirm that the warning is gone.

If the pinned version can't change yet, the warning is harmless noise as long as `~/.npmrc` holds a valid token. Leave `_auth` where it is, because other projects with a newer pnpm use it. Don't copy it into a project file.

## Where it came up

- **create-project-calavera:** it pins `pnpm@11.3.0` ([`package.json`, line 65](https://github.com/schalkneethling/create-project-calavera/blob/813f0f1e35f8aa53bc2dc210d6923496961fd947/package.json#L65)), so every command printed the warning after a `pnpm login` with the global pnpm 12 wrote `_auth`. Its releases publish from CI with trusted publishing, so the warning didn't break a release.
- **Sources:** pnpm's changelog entries for 11.10.0 (the `_auth` setting, [pnpm/pnpm#12314](https://github.com/pnpm/pnpm/issues/12314)), 12.1.0 (`pnpm login` writing to `config.yaml`) and 11.23.0 (`pnpm config get` no longer printing warnings), and the code that prints the warning in pnpm 11.3.0's `dist/pnpm.mjs`.
