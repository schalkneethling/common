# common

common issues and their fixes I run into while building projects so I do not have to reinvent the wheel each time

## How entries work

Each problem is one Markdown file, in a folder for its topic. An entry is added once its solution has worked in a real project, and it is improved whenever the problem comes up again. Each entry covers:

- **the problem,** with the exact error text, so a search for the error finds it
- **why it happens,** and why the obvious fixes don't work
- **the solution,** including the manual steps when the solution is automated
- **where it came up,** with links

## Entries

### Dependabot

- [A Dependabot pull request fails because a generated file is out of date](dependabot/generated-file-out-of-date.md): a dependency update changes a file that a tool generates, such as `worker-configuration.d.ts`, and Dependabot can't regenerate it.

### Netlify

- [A Netlify deploy fails because Netlify installs Swift for the repository's Package.swift](netlify/swift-installed-for-root-package-swift.md): a static website shares its repository with a Swift package, and Netlify tries to install Swift because the base directory is the repository root.

### pnpm

- [pnpm warns that `_auth` can't be set in the global config file](pnpm/global-auth-setting-ignored.md): a project pins a pnpm older than 11.10.0, which doesn't know the `_auth` credentials that `pnpm login` writes to the global `config.yaml` in pnpm 12.1.0 up to 12.16.0.

### Vite+

- [A Dependabot update of vite-plus leaves the `vite` alias behind](vite-plus/vite-alias-out-of-step.md): Dependabot moves `vite-plus` but not the `vite` alias to `@voidzero-dev/vite-plus-core`, so two copies of Vite are installed, and the build or the type check can fail late. A check that runs before the install fails first and names the line to set.
