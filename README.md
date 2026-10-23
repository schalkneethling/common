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
