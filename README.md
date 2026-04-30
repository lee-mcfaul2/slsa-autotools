# slsa-autotools

Scaffold reproducible-build, signed-release CI for autotools projects.

## What it does

We add github workflows that build your project in a container once, capture the dependencies and then create a new build dockerfile with all build dependencies baked in. From then on, releases are built within that dockerfile. The actual release process will build two of the same artefact in paralell in two different container instances and bit-to-bit compare them before slsa verifying and sigstore signing them. The release is then made as a draft in order for maintainers to sign with their GPG key if desired.

## Why

Most autotools releases are manually uploaded from a maintainer's machine. And though they are GPG signed, they do not offer any assurances as to what exactly is in the artefacts. This means that a bad actor with maintainer access or GPG keys can upload malicious content into a perfectly legitimate release, which is very hard to detect given many build and configure setups. This tool brings that process into GitHub Actions where the source used is referenced by commit SHA and SLSA verification sets a record of when an artefact was created. We also sign with sigstore as further assurance, as well as release in draft format, offering a maintainer the opportunity to GPG sign their artefact as well. Though now the GPG signature is not the main source of verification but rather a stamp showing that a specific maintainer approved the release.

In addition to the usual verification and signature pathways, this project does something unique. We make reproducible builds... No, really. We build in SHA-tagged build containers and have a bit-to-bit comparison. Each artefact gets built in a separate instance of the build container and is then bit-to-bit compared to ensure reproducibility before continuing on to being verified and signed. We don't just set `SOURCE_DATE_EPOCH` and say it's reproducible, we prove it on every build.

## How

Two scaffolding scripts and three workflows do the work. The scripts run once per project, from this repo against a target repo — they generate workflow files, they don't continuously manage anything. The workflows then live in the target repo and run on push and tag events the same way any other GitHub Actions workflow does.

The flow:

1. `scripts/init-container` writes `scan.yml` and `build-container.yml` into the target repo on a topic branch.
2. `scan.yml` runs the project's real `make dist` under strace, maps every file path the build touched to its source apt package, pins those package versions and SHA256 hashes, and commits a `Dockerfile` back to the branch.
3. After the topic branch merges to the default branch, `build-container.yml` builds that Dockerfile, pushes the image to `ghcr.io/<owner>/<repo>/builder`, and signs it with cosign.
4. `scripts/init-release` looks up the digest of that image and writes `release.yml` pinned to it.
5. After that PR merges, pushing a tag fires `release.yml`. Two independent build jobs produce the tarball in parallel, a verify job byte-compares them, and if they match the bytes are attested, signed, and uploaded to a draft GitHub release.

## Quickstart

You'll need `git`, `gh` (with `repo` and `read:packages` scopes), `curl`, and `bash` in your local shell. Standard developer setup, nothing exotic.

Step 1 — scaffold the builder container:

```
./scripts/init-container /path/to/your-autotools-project
```

This pushes a branch named `slsa-autotools-init` to your fork and prints a PR URL. Open the PR. The scan workflow runs against the branch and adds a `Dockerfile` commit within a few minutes. Review the Dockerfile and merge.

Once the merge lands on the default branch, `build-container.yml` fires automatically and publishes the pinned image to GHCR.

Step 2 — scaffold the release workflow:

```
./scripts/init-release /path/to/your-autotools-project
```

This looks up the digest of `ghcr.io/<owner>/<repo>/builder:latest` and writes a `release.yml` pinned to that exact digest. It pushes a second branch (`slsa-autotools-init-release`) and prints the PR URL. Merge it.

Step 3 — cut a release:

```
git tag v1.0.0 && git push origin v1.0.0
```

The release workflow fires, produces the tarball, runs the two-pass byte-compare, attests provenance, signs with cosign, and creates a GitHub draft release. You review the draft, optionally add a GPG signature alongside the cosign one, and publish.

## Scope

We work with GNU Autotools projects that use `AM_INIT_AUTOMAKE` and ship a `Makefile.am`. We produce source tarballs in whichever formats the project's `make dist` recipe emits (`.tar.gz` always; `.tar.xz` and `.tar.bz2` if declared). We produce Windows binaries via `mingw-w64` cross-compile when the project ships a `windows/build.bash` script (the xz convention). We produce a Linux AppImage via `linuxdeploy` when the project ships a top-level `*.desktop` (or `*.desktop.in`) file.

We don't add or modify any file in your repo outside `Dockerfile` and `.github/workflows/`. Everything else is detected from what your project already ships.

A few things we don't do:

- Projects without automake (raw autoconf, hand-written Makefiles).
- MSVC-built Windows artefacts. Mingw cross-compile only.
- Distro packages (`.deb`, `.rpm`, Snap, Flatpak). AppImage only.
- AppImage builders other than `linuxdeploy`. Projects whose AppImage is built via `appimagetool`, `appimage-builder`, `linuxdeployqt`, etc. fall outside scope.
- Notary v2 signatures. Sigstore only.

## Under the hood

### Reproducibility

Every tarball builds twice in independent containers and we byte-compare the results. Mismatch fails the job, no artefact ever leaves the runner. Subtle non-determinism — gzip wall-clock timestamps, tar entry order, multi-threaded xz scheduling, DWARF absolute paths, translation file `POT-Creation-Date` headers, PE timestamp fields — gets pinned out by compiler wrappers and post-distdir normalisation inside the workflow itself, before the bytes ever hit the verify step.

### Per-pass container isolation

The two passes run in separate containers, both at the same absolute path. Anything that bakes the build path into output — libtool `.la` files, pkg-config `.pc` files, certain generated headers — embeds the same path in both passes by construction.

### Pinning

We swap apt sources to `snapshot.ubuntu.com/<timestamp>` (the `SNAPSHOT_TS` knob) so every apt fetch resolves to a fixed historical version. The emitted `Dockerfile` lists every package the trace touched, with both the version string and the .deb's SHA256. The builder image itself is pulled by digest, not tag.

### Signing

Every artefact gets a `cosign sign-blob` signature using keyless OIDC against the workflow's own identity. No long-lived key on a maintainer machine. Every signature lands in Rekor's public transparency log, so anyone can independently audit the issuance history without asking us for anything. SLSA build provenance is attached separately via `actions/attest-build-provenance`.

### Detection, not hooks

We don't ask projects to ship slsa-autotools-shaped scripts. The pipeline detects what your project already has:

- **Bootstrap**: `autogen.sh` if present (with `autoreconf -fi` as fallback for projects whose `autogen.sh` exits non-zero on bare invocation), `bootstrap` for gnulib-based projects, or bare `autoreconf -fi` against `configure.ac`.
- **Windows**: a cross-compile script at `windows/build.bash`. If the project's script needs `windows/COPYING.MinGW-w64-runtime.txt` and it's not in tree, we drop one in from `/usr/share/doc/mingw-w64-*/` at runtime inside the build container.
- **Linux AppImage**: a top-level `*.desktop` or `*.desktop.in` file. If present, we run `make install DESTDIR=AppDir` and hand it to a SHA-pinned `linuxdeploy`. If absent, AppImage is skipped.

Projects whose Windows or Linux build doesn't fit these shapes get skipped for that artefact and ship just the source tarball. They're not blockers.

## Status

Work in progress.
