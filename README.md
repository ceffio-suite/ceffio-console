> **This repository is archived and no longer maintained.**
> Its content is kept for reference only. Everything below describes a scheme
> that has since been replaced — see the notice at the top for what is
> current. Do not follow the instructions in this README.

# Why this repository was archived

This repository published the administration console as its own image,
`ghcr.io/ceffio-suite/ceffio-console`. That scheme has been replaced: the
console image is now **published by the main repository**, alongside every other
Ceffio component, as

```
ghcr.io/ceffio-suite/ceffio:<version>
```

**What changed.** `ceffio-suite/ceffio` gained its own release workflow, which
builds the binary, packages it and publishes one image per release, plus the
release archives for `linux/amd64` and `linux/arm64`. Keeping a second
publication path for one component meant two ways to build the same thing, two
places to fix a defect in the packaging, and a version number that could drift
from the product it belonged to — this repository was last pushed while the
product was at 0.2.0, and the console has since been deployed at 0.7.1 from the
main repository's image.

**Nothing needs this repository to build the console.** The console lives in
`internal/console/` of [`ceffio-suite/ceffio`](https://github.com/ceffio-suite/ceffio),
and `ceffio install` deploys it. The released image is what a deployment pulls;
`ceffio` itself carries the image reference and never refers to this repository.

Existing tags and the GHCR package remain available. If you were pulling
`ghcr.io/ceffio-suite/ceffio-console`, switch to
`ghcr.io/ceffio-suite/ceffio:<version>` — the same binary, published with the
product.

---

The rest of this file is the original README, kept unchanged for reference.

<p align="center">
  <img src="assets/ceffio-banner-1280.png" alt="Ceffio" width="100%">
</p>

# ceffio-console (archived)

Container image of the **Ceffio administration console** — superseded, see the
notice above.

This is a **publication repository**, not a code repository: the console lives
in the main [`ceffio-suite/ceffio`](https://github.com/ceffio-suite/ceffio)
repository, under `internal/console/`. This repository existed to make the image
**addressable and versioned**, so that a deployment could pull it without
rebuilding anything on the target machine.

## Why a separate image

The console is not a standalone service: it **embeds the `ceffio` binary**,
which carries both the web page and the administration logic (reading the
cluster state, Keycloak account management). It therefore cannot be built from
a simple `Dockerfile` in its folder — the full binary is required.

Rather than extracting the console into a separate code repository (which would
duplicate the project, or create a cross-dependency between two repositories),
this repository carried **the workflow**: it fetched the sources, built the
binary, packaged it and published it.

## What the workflow did

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
console and imports the image locally when it is not available from a registry.

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