# A Netlify deploy fails because Netlify installs Swift for the repository's Package.swift

## The problem

A Netlify deploy of a static website fails before it publishes anything, in the "Install dependencies" stage, because Netlify tries to install a Swift toolchain. The website doesn't use Swift. The repository does, because it also holds a Swift package, such as a macOS app, with its `Package.swift` at the root.

The case this entry comes from is dotdrawer, a macOS menu bar app whose website lives in `site/`. The deploy log showed:

```text
Failed during stage 'Install dependencies': dependency_installation script returned non-zero exit code: 1
Downloading https://download.swift.org/swift-5.4-release/ubuntu2404/swift-5.4-RELEASE/swift-5.4-RELEASE-ubuntu24.04.tar.gz
```

Swift 5.4 has no build for Netlify's Ubuntu 24.04 build image, so the download fails and the deploy stops.

## Why it happens

Netlify installs a language's toolchain when it finds that language's dependency file in the site's base directory. For Swift, the [Manage build dependencies](https://docs.netlify.com/build/configure-builds/manage-dependencies/) page says that when neither a `SWIFT_VERSION` environment variable nor a `.swift-version` file picks a version, but a `Package.swift` exists in the base directory, Netlify installs the build image's default Swift version.

The base directory defaults to the repository root. So with no `base` set, Netlify finds the app's `Package.swift` there, even though `publish` points at a folder with nothing but HTML in it:

```text
repository root   ← base directory (default)
├── Package.swift ← Netlify sees this and installs Swift
└── site/         ← what Netlify publishes
```

The obvious fixes don't work:

- **Netlify has no setting to skip the install.** The docs don't document one.
- **Pinning a Swift version that exists only moves the problem.** `SWIFT_VERSION` or a `.swift-version` file would make the download succeed, but the build would still install a toolchain the website doesn't need. The same docs page says the build then runs `swift build` for the package's dependencies. A macOS app is unlikely to build on Linux, but dotdrawer never tried this route.
- **Moving or deleting `Package.swift` breaks the app.** It belongs at the root for Swift Package Manager and Xcode.

## The solution: make the website's folder the base directory

Set `base` to the website's folder in the `netlify.toml` at the repository root. `Package.swift` is then outside the base directory, so Netlify doesn't install Swift. A `base` in a root-level `netlify.toml` overrides the base directory set in Netlify's UI ([Configure builds › Set the base directory](https://docs.netlify.com/build/configure-builds/overview/)).

Changing `base` changes how two other settings resolve, so both change with it:

- **`publish` resolves from the base directory,** so `publish = "."` publishes the website's folder itself. Netlify's docs show `publish` only for a subfolder of the base, such as `publish = "build-output/"`, not for the base itself. `"."` worked in dotdrawer. Avoid `"/"`, which the same docs use for the repository root.
- **The `ignore` command runs from the base directory** ([Ignore builds](https://docs.netlify.com/build/configure-builds/ignore-builds/)), so its Git paths are relative to that folder. Git resolves pathspecs from the folder it runs in, so `.` is the website and `../netlify.toml` reaches the config file at the root.

dotdrawer's `netlify.toml`, from [PR #11](https://github.com/schalkneethling/dotdrawer/pull/11):

```toml
[build]
  base = "site"
  publish = "."
  # Skip the deploy when a commit changes nothing in site/ or in this file.
  # `git diff --quiet` exits 0 when nothing changed, which tells Netlify to skip.
  ignore = "git diff --quiet $CACHED_COMMIT_REF $COMMIT_REF -- . ../netlify.toml"
```

Netlify still finds a `netlify.toml` that sits at the root: it looks in the package directory, then the base directory, then the root ([Monorepos › How Netlify finds your configuration files](https://docs.netlify.com/build/configure-builds/monorepos/)). The website's folder has no `netlify.toml` of its own, so the root one applies.

### Checking it

1. Before pushing, run the `ignore` command's `git diff` from inside the website's folder, with two real commits in place of the variables. It should exit 1 for a commit that changed the website and 0 for one that didn't.
2. After pushing, the deploy log should show no Swift download, and the deploy should publish the website's folder.

## Where it came up

- **dotdrawer:** the first deploy of the website from [PR #10](https://github.com/schalkneethling/dotdrawer/pull/10) failed with the error above. [PR #11](https://github.com/schalkneethling/dotdrawer/pull/11) set the base directory to `site/`, and the next deploy published [dotdrawer.schalkneethling.com](https://dotdrawer.schalkneethling.com/).
