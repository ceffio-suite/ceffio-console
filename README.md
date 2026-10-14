<p align="center">
  <img src="assets/ceffio-banner-1280.png" alt="Ceffio" width="100%">
</p>

<p align="center">
  <a href="https://github.com/ceffio-suite/ceffio-console/actions/workflows/build.yml"><img src="https://img.shields.io/github/actions/workflow/status/ceffio-suite/ceffio-console/build.yml?label=build&logo=github" alt="build status"></a>
  <a href="https://github.com/ceffio-suite/ceffio-console/pkgs/container/ceffio-console"><img src="https://img.shields.io/badge/GHCR-ceffio--console-2496ED?logo=docker&logoColor=white" alt="GHCR image"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-AGPL--3.0--or--later-blue" alt="AGPL-3.0-or-later license"></a>
</p>

# ceffio-console

Container image of the **Ceffio administration console**.

    ghcr.io/ceffio-suite/ceffio-console:0.2.0

This is a **publication repository**, not a code repository: the console lives
in the main [`ceffio-suite/ceffio`](https://github.com/ceffio-suite/ceffio)
repository, under `internal/console/`. This repository exists for one precise
reason — to make the image **addressable and versioned**, so that a deployment
can pull it without rebuilding anything on the target machine.

## Why a separate image

The console is not a standalone service: it **embeds the `ceffio` binary**,
which carries both the web page and the administration logic (reading the
cluster state, Keycloak account management). It therefore cannot be built from
a simple `Dockerfile` in its folder — the full binary is required.

Rather than extracting the console into a separate code repository (which would
duplicate the project, or create a cross-dependency between two repositories),
this repository carries **the workflow**: it fetches the sources, builds the
binary, packages it and publishes it.

## What the workflow does

```
1. Build the Ceffio binary       from github.com/ceffio-suite/ceffio
2. Package it as a minimal image FROM scratch, static binary only
3. Publish to GHCR               ghcr.io/ceffio-suite/ceffio-console:<version>
```

The image starts from an **empty** system. The binary is static
(`CGO_ENABLED=0`), it needs no library; only the certificate authorities are
added, because the console calls the Kubernetes API and Keycloak over HTTPS.
For an administration console reachable over the network, reducing the attack
surface to almost nothing is not a detail.

## Publish a version

Via tag — this is the normal way, and it guarantees that the Git tag and the
image tag match:

```bash
git tag -a v0.2.0 -m "Console 0.2.0"
git push origin v0.2.0
```

Or manually, to republish an existing version:

```bash
gh workflow run build.yml -f version=0.2.0
```

## Usage

The image is meant to run **inside the Ceffio cluster**, where it reads its
state directly from the Kubernetes API.

```yaml
containers:
- name: console
  image: ghcr.io/ceffio-suite/ceffio-console:0.2.0
  args: ["console", "--domain", "example.com", "--addr", ":8088"]
  env:
  # Console access key (the interface asks for it, then sends it as a header).
  - name: CEFFIO_KEY
    valueFrom: {secretKeyRef: {name: ceffio-console-key, key: key}}
  # Keycloak administration credentials, for SSO account management.
  - name: KC_ADMIN
    valueFrom: {secretKeyRef: {name: keycloak-admin, key: username}}
  - name: KC_ADMIN_PASSWORD
    valueFrom: {secretKeyRef: {name: keycloak-admin, key: password}}
  ports:
  - {name: http, containerPort: 8088}
```

In practice it is not written by hand: `ceffio install` (phase 11) deploys the
console and **imports the image locally** when it is not available from a
registry. This published image serves installations that prefer pulling over
building.

## Permissions required in the cluster

The console needs a `ServiceAccount` that can **read** pods, services,
deployments, ingresses and certificates. It does not write to the cluster:
deployment remains the CLI's business. A Ceffio installation creates this role
automatically.

## Verify an image

An existing tag proves nothing. To verify that an image is actually published
and downloadable:

```bash
docker pull --platform linux/amd64 ghcr.io/ceffio-suite/ceffio-console:0.2.0
```

`--platform` is required from an ARM Mac: the image targets **amd64**, like
the Linux servers it equips.

## License

To be defined (open source — license pending decision).