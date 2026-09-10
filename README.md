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
    secrets:
      RELEASE_TOKEN: ${{ secrets.RELEASE_TOKEN }}
    with:
      dev-bump: pr
```

`RELEASE_TOKEN` is a PAT with `contents: write` (and `pull-requests: write`
for `dev-bump: pr`). The default `GITHUB_TOKEN` will not work: it cannot
trigger other workflows, so a tag it creates starts nothing.

**Pass the secret by name, not with `secrets: inherit`.** GitHub carries
inherited secrets only between repositories in the same organization or
enterprise. A caller in any other organization that writes `secrets: inherit`
passes validation, passes the gate, and then fails at checkout with `Input
required and not supplied: token` — the release commit is on `main` and no tag
exists. The explicit form works from anywhere.

**The caller needs no `permissions:` block.** Every write in this workflow
authenticates with `RELEASE_TOKEN`, so the job's own `GITHUB_TOKEN` only needs
`contents: read` — which a repository grants by default even when its default
workflow permissions are read-only. A called workflow may not request more than
its caller holds, so a workflow that asked for `contents: write` here would fail
validation in exactly those repositories, before running a single step.

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
| `dev-bump-package` | *(empty)* | `version-manifest` is one member of a workspace whose other members version on their own cadence — the bump is then `cargo release version -p <name>` and touches only that manifest and `Cargo.lock` |
| `dev-bump-style` | `alpha` | `patch` for a bare `MAJOR.MINOR.(PATCH+1)`, where the next release finalizes the version in place or path dependencies carry caret requirements a prerelease would not satisfy |

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
  existing prerelease suffix dropped first — or the bare `MAJOR.MINOR.(PATCH+1)`
  under `dev-bump-style: patch`. Either way the shape is the workflow's, not a
  hand-edit's.
- **The release check is a step, not a job-level `if:`.** A job that never runs
  leaves no trace, so a release that failed to tag and a commit that was never a
  release look identical afterward. As a step, the run exists and the log says
  which it was.

### Repositories this does not fit

A workspace whose crates are versioned and tagged independently — tags like
`<crate>-v1.2.3` — releases one crate at a time in dependency order. One shared
version is the wrong model for it, and this workflow does not try to serve it.

A hybrid workspace — a product tagged `v1.2.3` alongside satellite crates that
release on their own cadence — fits for the product's releases: point
`version-manifest` at the product's manifest and set `dev-bump-package` to it,
so the dev bump does not drag the satellites to the product's version. The
satellites' own releases are outside this workflow; tag them by hand, and let
the repository's publish workflow act on the tag.

## Versioning

Tags are `v1`, `v2`, … on this repository, and `v1` moves forward within a major
version as changes land that do not break callers. A change that requires
callers to do something different gets a new major tag.

Dual-licensed under MIT or Apache-2.0.
