# Security posture

slsa-autotools is build-pipeline scaffolding, not a security tool. It
does not detect or block intrusions, audit source code, sandbox builds
beyond what GitHub Actions itself provides, or replace any existing
review process.

The pipeline it scaffolds has three structural properties: pinned
inputs, in-CI byte-reproducibility check, and machine-issued
provenance and signatures. The corresponding assurances accrue only
to consumers who verify the resulting SLSA attestation and sigstore
signature before use. Consumers who do not verify gain nothing from
this scaffolding.

## Scope

In scope: properties of the release pipeline shape itself, and the
set of modifications an attacker who controls a build input can make
without producing artefacts that fail downstream verification.

Out of scope: source-level review, runtime trust in GitHub Actions,
distribution past the GitHub Releases upload, attacks against the
underlying signing or transparency infrastructure, and attacks that
do not require a verifying consumer.

A "verifying consumer" is one that, before use:

1. Fetches the artefact, its SLSA build provenance, and its cosign
   sigstore signature.
2. Validates the sigstore signature against the expected workflow
   identity — repository, ref, and workflow file path.
3. Validates that the SLSA provenance binds the artefact to the
   expected source commit and workflow run.

Subsequent maintainer GPG signature verification, where the project
publishes one, is an additional check the consumer may apply. It is
not load-bearing for any property below.

## Structural properties

These are properties of the pipeline shape. There is no detection
logic; nothing in the pipeline executes a check whose result is "this
artefact is malicious".

### Build environment is reconstructable from records

A `make dist` invocation on a maintainer machine inherits compiler
version, linker version, locale, `$PATH`, environment variables,
filesystem case-sensitivity, and `~/.gnupg` state. None of these
inputs are captured; reconstructing the produced bytes from records
alone is not generally possible.

In the scaffolded pipeline:

- Every apt input resolves through `snapshot.ubuntu.com/<timestamp>`
  to a fixed package version and SHA256.
- Compiler invocations are wrapped to remove known non-determinism
  sources (build IDs, embedded timestamps, absolute paths).
- The builder container image is referenced by digest, not tag.
- The build runs in an ephemeral container, started fresh per pass.

The Dockerfile and pinned `release.yml` together record every input
contributing to the output, modulo the GitHub Actions runtime itself.
A third party who reconstructs the environment from those records,
on equivalent runtime, produces matching bytes.

### Signing material is short-lived and externally observable

A long-lived GPG private key on a maintainer machine is a durable
target. Compromise, rotation, leak, or maintainer departure are
events whose effect extends to every release the key has signed.

The scaffolded pipeline signs with sigstore. The signing certificate
is short-lived, issued against the workflow's OIDC identity, and
valid only for the duration of the run. No long-lived signing key
exists on any maintainer machine. Each signing event lands in
Rekor's append-only transparency log; the set of signing events for
a given workflow identity is enumerable by any third party without
the project's cooperation.

### Environment-coupled tampering produces detectable byte divergence

Each release builds the same source twice, in independent containers
at the same absolute path. Outputs are byte-compared before any
signature or attestation is attached. Mismatch terminates the
release; no artefact is published, signed, or attested.

This catches non-deterministic tampering — modifications whose
effect on output bytes varies with environmental factors that
neither pass fully controls (scheduler order, thread interleaving,
ASLR-derived state, runtime nonces).

It does not catch a deterministic modification of a pinned toolchain
input — for example, a malicious compiler that targets a fixed
source-commit identifier and produces identical backdoored bytes in
both passes. Detecting that class requires independent rebuild on
infrastructure the attacker does not control (the SLSA L4 /
verifying-party model) and is not in scope here.

### Provenance binds the artefact to a commit and a workflow

A locally-built tarball with a maintainer GPG signature attests that
the maintainer signed those bytes at some point. It is not bound to
a specific commit, build environment, or workflow run.

The scaffolded pipeline emits a SLSA build provenance attestation
binding each artefact to the source commit SHA, the workflow file
path, the workflow run ID, and the builder image digest. A verifying
consumer rejects any artefact whose provenance does not match the
expected source repository, ref, and workflow file. A signature
issued by an unrelated workflow under the same identity provider, or
the same workflow operating on a different repository or branch,
does not satisfy this check.

## What does not change

### Source-code trust

The pipeline does not inspect source. A backdoor committed to the
source tree and tagged for release is reproduced, attested, and
signed. The provenance attestation correctly identifies the commit;
it makes no statement about the commit's contents. Review of source,
including `configure.ac`, `Makefile.am`, generated files committed
to the tree, and any pre-built artefacts in the repository, remains
the responsibility of maintainers and consumers.

### Dependency trust

The pipeline pins apt packages by version and SHA256, the builder
image by digest, and external tools (cosign, linuxdeploy) by
SHA256. This trusts:

- Ubuntu's package signing keys as of the snapshot timestamp.
- The pinned upstream releases of sigstore, cosign, linuxdeploy,
  and mingw-w64 not to have been tampered with at the SHAs pinned.
- `snapshot.ubuntu.com` as a faithful record of historical archive
  state.

A backdoor in any pinned upstream is reproduced into the release.
The pipeline records exactly what was used; post-incident
attribution is supported, prevention is not.

### GitHub Actions runtime trust

Builds run on GitHub-hosted runners by default. A runner-image
compromise, a cross-tenant side channel, or a malicious change to a
GitHub Actions feature affects every job those runners execute.
slsa-autotools does not, and structurally cannot, isolate against
the runtime it depends on. Self-hosted runners shift this trust to
the operator of those runners; they do not eliminate it.

### Release distribution

The signed artefact is uploaded to a GitHub Release. Distribution
past that point — mirror selection, CDN behaviour, package-manager
indices, blog posts, README hyperlinks — is outside the pipeline's
control. A consumer who fetches the artefact through a third-party
channel and does not perform the verification steps listed above
gains no property of this scaffolding, regardless of what the
pipeline did at build time.

### Active threats during the build

The two-pass diff addresses non-deterministic divergence.
Deterministic tampering inside a pinned toolchain, race conditions
exploitable identically across both passes, and hardware-level
attacks on the runner are not addressed.

## On specific upstream incidents

This document makes no claim about the prevention of any named
real-world incident. The scaffolding alters the set of artefact
properties a downstream consumer can mechanically verify, and the
cost to an attacker of producing a release that satisfies that
verification. Whether that change closes off a particular reported
attack depends on which steps of that attack relied on properties
this scaffolding alters and which relied on properties it leaves
untouched. That analysis is the consumer's to perform against their
own threat model and is not performed here.

## Reporting

Issues in slsa-autotools' own scaffolding — a template that elides
a signing or attestation step, a script that writes a credential to
a log, a default `permissions:` block looser than the surrounding
comment claims, a pinning step that silently falls back to a
floating reference — are reported as issues on this repository. The
scaffolding is fully visible to anyone reading the templates and
scripts; there is no private service to disclose against.

Issues in the underlying toolchain (sigstore, cosign, GitHub
Actions, the Ubuntu archive) are reported to those projects through
their own coordinated-disclosure processes.
