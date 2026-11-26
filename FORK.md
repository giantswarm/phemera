# The Giant Swarm phemera line

This repository builds and publishes container images of
[phemera-dev/phemera](https://github.com/phemera-dev/phemera) `main`, so that phemera can be tried on a
cluster without building it first. The line is upstream `main` as it is, plus the files listed under
[The fork's own files](#the-forks-own-files): no change to upstream's code is carried, and the images are
built from upstream's Containerfiles unchanged. Giant Swarm's Team Shield owns it.

This repository is public, and so is this file: it names no customer, installation or private repository.

## Branches

| Branch | Content | Moved by |
|---|---|---|
| `main` | Read-only mirror of upstream `main`, fast-forwarded | the sync only |
| `giantswarm` (default) | Upstream `main` plus the fork's own commits on top | the sync (a rebase onto upstream `main`), and pull requests for the fork's own files |
| `sync/main` | The sync's candidate while its gate runs | the sync only; deleted once landed |
| `upstream/<slug>` | A change prepared for upstream, one commit on the mirror's `main` | see [Contributing](#contributing) |

`giantswarm` is protected by a ruleset that requires the check `ci-ok` on every update for every actor:
a pull request's head passes it before the merge, and the sync's candidate passes it before the sync
pushes exactly that commit. The upstream commit the line is on is `git merge-base giantswarm main`.

## What is published

| | |
|---|---|
| Images | `gsoci.azurecr.io/giantswarm/phemera/authority` from `authority/Containerfile`, `gsoci.azurecr.io/giantswarm/phemera/ssh-proxy` from `ssh-proxy/Containerfile` |
| Platforms | `linux/amd64` and `linux/arm64` in one index, each built natively on a machine of its architecture (the Containerfiles compile for the platform they build on) |
| Tags | Every push to `giantswarm`: gitsemver's dev version of the commit, `X.Y.Z-r588f3d76t<YYYYMMDDHHMMSS>h<sha7>` (`r588f3d76` is the hash of the branch name, then the commit's committer date in UTC and its short hash), and `latest` |
| Upstream commit | The index annotation `io.giantswarm.upstream.version=main@<full upstream sha>`, and the index's `org.opencontainers.image.revision` names the line's commit |
| Signatures | cosign keyless under the project's CircleCI OIDC identity, with SLSA provenance and an SPDX SBOM per platform: `cosign verify --certificate-oidc-issuer https://oidc.circleci.com --certificate-identity-regexp '^https://circleci\.com/api/v2/projects/[a-f0-9-]+/pipeline-definitions/[a-f0-9-]+$' <image>` |

The images are public: no login is needed to pull them. The committer date grows with every sync, so the
newest build has the highest version: a Flux `OCIRepository` follows the line with the range `>=0.0.0-0`
and the filter `^.*-r588f3d76t[0-9]{14}h[0-9a-f]{7}$`.

Once upstream cuts releases, the line builds them too, and `latest` moves to them.

## Publishing

`.circleci/config.yml` is the pipeline, hand-written and ours to keep; devctl generates nothing in this
repository. It uses the Giant Swarm architect orb's stock jobs: `build-image` per image and platform,
`push-to-registries` to join the digests into the signed index, then `verify` (pulls both indexes without
credentials and checks the two platforms and the upstream annotation) and `scan` (Trivy; a fixable HIGH or
CRITICAL finding fails the pipeline, and its fix goes upstream). Nothing is published from GitHub Actions
or by hand, and no GitHub Actions workflow holds a registry credential. Pull-request branches run the same
builds without pushing.

## Sync

`.github/workflows/sync-upstream.yaml` runs every hour and on demand. It fast-forwards `main` to upstream
`main`; when upstream moved, it rebases the fork's own commits of `giantswarm` onto it, pushes the result
to `sync/main`, waits for `ci-ok` on it and pushes exactly that commit to `giantswarm`, which publishes it.
A fork commit that conflicts with upstream, a red gate or a line that moved meanwhile fail the run and
leave `giantswarm` where it was; the next run starts again. A conflict is resolved by hand: rebase
`giantswarm` onto `main`, push the result to `sync/main`, and once `ci-ok` passed there push that commit
to `giantswarm`. The sync pushes as the org's HeraldBot GitHub App, a bypass actor of the line's pull-request
rule but not of its gate.

## The fork's own files

`ci-ok` (`.github/workflows/fork-ci.yaml`) fails a line that differs from upstream `main` in any other
file:

- `FORK.md` — this file
- `.circleci/config.yml` — the publishing pipeline
- `.github/workflows/fork-ci.yaml` — the gate: the line unchanged against upstream, both Containerfiles
  building
- `.github/workflows/sync-upstream.yaml` — the sync
- `renovate.json5` — Renovate off: upstream owns the dependency graph
- `.trivyignore` — the scan's accepted findings, each with its reason and expiry

Upstream's workflows stay as they are. On this repository the ones that publish or campaign on upstream's
behalf are disabled in its Actions settings: `docs.yaml` (deploys upstream's site to GitHub Pages),
`scorecard.yaml` (publishes an OpenSSF Scorecard), `fuzz-batch.yaml` and `fuzz-prune.yaml` (upstream's
nightly fuzzing). `ci.yaml`, `codeql.yaml` and `reuse.yaml` run on the mirror's `main`, and with `dco.yaml`,
`pr-title.yaml` and `fuzz.yaml` on pull requests against `giantswarm`.

## Contributing

What we find running phemera goes upstream. A change is prepared as one upstream-ready commit (upstream's
conventions and tests, DCO sign-off, nothing Giant Swarm-specific) on a branch `upstream/<slug>` cut from
the mirror's `main`, and a maintainer of this line opens the pull request upstream. The line carries no
change to upstream's code; carrying one would be a decision that extends `ci-ok`'s list above and records
the change, its upstream pull request and its exit in a file of its own under `fork/`.
