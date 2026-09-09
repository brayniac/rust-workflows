# rust-workflows

Reusable GitHub Actions workflows shared across my Rust repositories.

Each workflow here is called with `workflow_call`, so a repository carries a
short caller instead of a copy. The parts that must agree between repositories
are defined once, in this repository; the parts that legitimately differ are
named inputs.

## `tag-release.yml`

Tags a release when a release-preparation commit lands on `main`, then opens or
pushes the next development version bump.

The flow it completes: a release PR bumps the version and updates the changelog,
its commit message names the new version, and merging it triggers this workflow
to create the tag. Anything that builds artifacts or publishes runs afterward,
from the tag.

### Use it

```yaml
name: Tag Release

on:
  push:
    branches: [main]

jobs:
  tag-release:
    uses: brayniac/rust-workflows/.github/workflows/tag-release.yml@v1
    secrets: inherit
    with:
      dev-bump: pr
```

`secrets: inherit` passes `RELEASE_TOKEN`, a PAT with `contents: write` (and
`pull-requests: write` for `dev-bump: pr`). The default `GITHUB_TOKEN` will not
work: it cannot trigger other workflows, so a tag it creates starts nothing.

**Pin a tag, never a branch.** `@v1` means a change here is adopted when you
choose. `@main` means every repository adopts it the moment it lands, which is
the drift this exists to remove, only faster.

### Inputs

| Input | Default | Set it when |
| --- | --- | --- |
| `version-manifest` | `Cargo.toml` | the root is a virtual manifest and a member carries the released version — `server/Cargo.toml`, `<name>/Cargo.toml` |
| `tag-prefix` | `v` | your tags are not `vX.Y.Z` |
| `dev-bump` | `pr` | `direct` to push the bump straight to `main`; `none` to bump by hand |
| `only-repository` | *(empty)* | the repository is forked and a fork's `main` must not tag — e.g. `iopsystems/rezolus` |

### Outputs

| Output | Meaning |
| --- | --- |
| `version` | the version read from the manifest |
| `tagged` | `"true"` when this run created a tag |

### What is fixed here

These are the differences that accumulated across repositories while each kept
its own copy. None of them was a decision, so none of them is an input.

- **Which commits count as releases.** `release: v1.2.3` and
  `release: prepare v1.2.3` are both accepted, as is a merge commit referencing
  `release/v1.2.3`, so a repository adopting this workflow does not have to
  change its release commit convention in the same change.
- **The commit must name the version the manifest holds.** This is the check
  that can fail. A release PR whose title was edited, or rebased onto a
  different bump, otherwise passes a prefix test while tagging whatever the
  manifest happens to say.
- **Version extraction is section-aware,** so a `version` key under
  `[dependencies]` cannot be mistaken for the package's.
- **The dev bump runs `cargo release version`,** which updates every manifest
  sharing the version and refreshes `Cargo.lock`. Editing one manifest leaves a
  workspace inconsistent and the lockfile stale.
- **The next development version is `MAJOR.MINOR.(PATCH+1)-alpha.0`,** with any
  existing prerelease suffix dropped first.
- **The release check is a step, not a job-level `if:`.** A job that never runs
  leaves no trace, so a release that failed to tag and a commit that was never a
  release look identical afterward. As a step, the run exists and the log says
  which it was.

### Repositories this does not fit

A workspace whose crates are versioned and tagged independently — tags like
`<crate>-v1.2.3` — releases one crate at a time in dependency order. One shared
version is the wrong model for it, and this workflow does not try to serve it.

## Versioning

Tags are `v1`, `v2`, … on this repository, and `v1` moves forward within a major
version as changes land that do not break callers. A change that requires
callers to do something different gets a new major tag.

Dual-licensed under MIT or Apache-2.0.
