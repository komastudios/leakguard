# Releasing

Releases are cut by `.github/workflows/release.yml`, which builds the `leakguard`
CLI as a standalone binary for six targets and publishes them as GitHub Release
assets alongside a `SHA256SUMS` manifest.

Nothing is released automatically. A release happens only when you push a
`v*.*.*` tag or run the workflow by hand.

## Targets

| Target | Runner |
| --- | --- |
| `x86_64-unknown-linux-musl` | `ubuntu-latest` |
| `aarch64-unknown-linux-musl` | `ubuntu-24.04-arm` |
| `x86_64-apple-darwin` | `macos-latest` (cross) |
| `aarch64-apple-darwin` | `macos-latest` |
| `x86_64-pc-windows-msvc` | `windows-latest` |
| `aarch64-pc-windows-msvc` | `windows-latest` (cross) |

Linux binaries are statically linked against musl. All six are stripped via
`CARGO_PROFILE_RELEASE_STRIP`, which leaves `Cargo.toml` untouched so the
published crate keeps the default release profile.

The four legs whose host can execute what they built run a smoke test asserting
`leakguard --version` and `--list-kinds`. The two cross legs cannot: the arm64
macOS image has no Rosetta, and the Windows runner is x64.

## Checklist

1. Land everything you want in the release on `main`.
2. Bump `version` in `Cargo.toml`.
3. Bump the version in the README install snippet — it is hardcoded in the
   `[dependencies]` block and in the prebuilt-binary `curl` example.
4. Run `cargo build` so `Cargo.lock` picks up the new version, and commit it.
5. Move the `## [Unreleased]` entries in `CHANGELOG.md` into a new dated section,
   `## [X.Y.Z] - YYYY-MM-DD`. The workflow extracts exactly this section for the
   release notes, so anything not under that heading will not appear.
6. Commit, e.g. `Release vX.Y.Z: <summary>`.
7. Dry run first (see below) and check the rendered notes in the job summary.
8. Tag and push:

   ```sh
   git tag vX.Y.Z
   git push origin main
   git push origin vX.Y.Z
   ```

The tag push triggers the workflow, which publishes the release.

## Dry run

```sh
gh workflow run release.yml -f dry_run=true
```

This builds all six binaries, generates `SHA256SUMS`, renders the release notes
into the job summary and uploads a `release-assets` artifact — everything except
creating the release. Download the artifact to inspect the binaries before
committing to a tag.

## Releasing a specific commit

`workflow_dispatch` takes three inputs:

| Input | Meaning |
| --- | --- |
| `version` | The version to release. Empty reads it from `Cargo.toml`. |
| `ref` | The commit SHA or branch to build. Empty uses the ref selected in the UI. |
| `dry_run` | Defaults to `true`. Set `false` to actually publish. |

```sh
# Release a particular commit without tagging it first
gh workflow run release.yml -f ref=<sha> -f version=X.Y.Z -f dry_run=false
```

On this path the workflow creates the tag itself, at exactly the commit it built.

Note that GitHub's ref picker only accepts branches and tags, which is why the
commit to build is passed as the `ref` *input* rather than selected as the ref.

## Guards

The workflow refuses to publish rather than shipping something inconsistent:

- **Version mismatch.** If the tag or the `version` input disagrees with
  `Cargo.toml`, the run fails before any build starts. The binary prints
  `env!("CARGO_PKG_VERSION")`, so it can only ever report the `Cargo.toml` value;
  a release naming anything else would contradict its own assets.
- **Tag pointing elsewhere.** If the tag already exists but is not at the commit
  being built, the run fails. `gh release create --target` is silently ignored for
  an existing tag, so this would otherwise attach binaries from one commit to a
  release for another with no error.
- **Release already exists.** Checked before building, and again before
  publishing, where it exits cleanly instead of duplicating assets.
- **Wrong asset count.** Packaging asserts exactly six binaries, so a skipped
  matrix leg cannot produce a partial release.

Because the existing-release check fails fast, a run that built successfully but
died during upload needs its partial release deleted before you can re-run it.

## After a release

`cargo publish` is not automated. Publish the crate separately:

```sh
cargo publish
```

CI already runs `cargo publish --dry-run` on every push to `main`.
