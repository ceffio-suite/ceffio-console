# Contributing to ceffio-console

This is a **publication repository**: it carries no application code. The
console itself lives in
[`ceffio-suite/ceffio`](https://github.com/ceffio-suite/ceffio) under
`internal/console/`. What lives here is the workflow that builds the `ceffio`
binary from that repository and publishes it as a minimal image:

```
ghcr.io/ceffio-suite/ceffio-console:<version>
```

**Consequence for contributors**: a change to the console is a pull request
against `ceffio-suite/ceffio`, not here. Changes here concern the packaging
only (base image, build flags, publishing).

## GitFlow

| Branch | Role |
|---|---|
| `main` | every commit is a releasable state |
| `develop` | integration branch |
| `feature/<name>` | one change, merged into `develop` by PR |
| `release/x.y.z` | stabilisation from `develop`, tagged from here |
| `hotfix/x.y.z` | fix from `main`, merged back into both |

PR titles follow [Conventional Commits](https://www.conventionalcommits.org/).

## Releasing

A release is a tag; two workflows react in order:

1. `build.yml` clones `ceffio-suite/ceffio`, builds the binary and pushes the
   image `ghcr.io/ceffio-suite/ceffio-console:<version>` (plus `latest`);
2. `release.yml` waits for that build to be green, **pulls the image
   anonymously to prove it is public**, then creates the GitHub Release with
   generated notes.

```bash
git checkout develop && git pull
git checkout -b release/0.4.0
git tag -a v0.4.0 -m "ceffio-console 0.4.0"
git push origin release/0.4.0 && git push origin v0.4.0
# after both workflows are green:
git checkout main && git merge --no-ff release/0.4.0 && git push
git checkout develop && git merge --no-ff release/0.4.0 && git push
```

## The rules of this repository

1. **Tag = version.** The Git tag and the image tag are the same string; the
   deployment pins exact tags. Never publish an unversioned image.
2. **The image stays minimal.** `FROM scratch` + CA certificates + the static
   binary. Anything added to the image is attack surface on an administration
   console reachable over the network — justify it in the PR.
3. **The binary is built from source at a known commit**, never downloaded:
   the image must correspond exactly to a commit of `ceffio-suite/ceffio`.
4. **Secrets stay in Actions secrets** (`GHCR_TOKEN`), never in a file, never
   in a log line. The clone URL with its token is stripped right after the
   clone for that reason.
5. **A release that was not pullable was not released** — keep the anonymous
   pull check in `release.yml`.

## License

Apache License 2.0 — see [LICENSE](LICENSE).
