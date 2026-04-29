# Security posture

slsa-autotools is not a security tool. It does not detect attacks,
does not block bad actors, does not sandbox builds beyond what GitHub
Actions already provides, does not audit dependencies, and does not
replace any existing security review.

What it does is move an autotools project's release flow off the
maintainer's local machine into a pinned-input CI pipeline with
attached provenance and signatures. This removes some
specific attack avenues that exist with laptop builds.

## What goes away by being in CI/CD with this shape

These are properties of the workflow shape, not active defences.
Nothing here is "slsa-autotools detected and blocked X." The shape
simply doesn't have the corresponding surface.

### Maintainer-machine state in the build environment

When `make dist` runs on a maintainer's laptop, the tarball bytes
depend on whatever is installed there: compiler version, linker
version, locale, `$PATH` ordering, environment variables, filesystem
case-sensitivity, the contents of `~/.gnupg`. None of that is
recorded; reproducing the bytes a year later usually fails.

In the slsa-autotools shape, every byte the builder consumes comes from a pinned
`snapshot.ubuntu.com` mirror at a fixed timestamp, the compiler is
invoked through wrappers that strip its non-determinism (build IDs,
timestamps, absolute paths), and the build runs in an ephemeral
container the maintainer never touched. The bytes are reproducible
from the recorded inputs; they don't depend on a machine state nobody
can replay.

### Long-lived signing keys

Local releases sign with a maintainer's GPG key. That key is a
durable target — its loss, rotation, leak, or the maintainer's
departure from the project all become security-relevant events for
every release the key has ever signed and every future release.

With slsa-autotools, signatures come from sigstore. Each one binds to a
short-lived certificate issued against the workflow's own OIDC
identity for the duration of that one run. There is no long-lived
key for an attacker to steal, no rotation schedule to get wrong, no
off-boarding event that puts past releases in question.

### Unverifiable claims about what built what

A locally-produced tarball is bytes plus, at best, a GPG signature
attesting "I, the maintainer, signed these bytes." It is not bound
to a specific commit, a specific build environment, or a specific
time, except by the maintainer's word.

In the CI shape, every artifact carries a SLSA build provenance
attestation tying it to the exact commit, workflow file, and runner
image that produced it, plus a Rekor transparency log entry that any
third party can fetch and verify without asking the maintainer for
anything.

### Undetected non-determinism

A locally-produced tarball is whatever bytes `make dist` happens to
emit on that one machine. Drift across runs goes unnoticed until
someone tries to reproduce it and fails.

The CI shape produces the tarball twice in independent containers
and byte-compares. If the runs differ, the release fails. This is a
build-quality property in itself, but it transitively closes the
door on "non-determinism is hiding tampering" failure modes — bytes
that won't match across two clean runs of the same commit are bytes
nobody should be signing.

## What still exists

Everything else.

### Source-code trust

slsa-autotools does not look at the source. If the maintainer
commits a backdoor and tags a release, the pipeline cheerfully
reproduces, attests, and signs the backdoor. The provenance
attestation correctly says which commit was built; it does not vouch
for what's in the commit. Code review is unaffected and unreplaced.

### Dependency trust

The pipeline pins apt packages to `snapshot.ubuntu.com` and pins
external tools by SHA256 inside the emitted `release.yml` (cosign,
linuxdeploy, plus the project's own toolchain via the pinned builder
image). That trusts:

- Ubuntu's package signing infrastructure.
- The snapshot mirror to actually serve the bytes its index claims.
- The pinned upstream releases of sigstore / cosign / linuxdeploy /
  mingw-w64 not to have been tampered with at the SHAs we pinned.

A compromised upstream package, or a backdoor in any of those
toolchains, lands in the release. The pipeline records *what was
used* (pinned versions, SHA256s), so post-incident
forensics is possible; there is no pipeline-level prevention.

### GitHub Actions runtime trust

The build runs on shared GitHub-hosted runners by default. A
runner-image compromise, a side-channel attack across tenants, or a
malicious GitHub Actions feature affects every job the runner
executes. slsa-autotools does not isolate against the runtime it
depends on.

### Release distribution

The signed tarball lands in a GitHub Release. From there, downstream
consumers fetch it through whatever channel they use. The pipeline
does not control the distribution path past the upload — mirror
poisoning, CDN substitution, or out-of-band channels (a blog post
linking to a malicious mirror) are unaffected. Verifying the cosign
signature and SLSA attestation at consumption time is the only
defence on that side, and it has to be done by the consumer.

### Active threats during the build

The two-pass diff catches non-determinism. It does not catch a
deterministic backdoor — for example, malicious code in a pinned
toolchain that targets a fixed commit identifier and produces the
same bytes both passes. Detection of that requires an independent
rebuild on fully different infrastructure (the SLSA Level 4 /
verifying-party model) and is out of scope here.

## Reporting

If something in slsa-autotools' own scaffolding scripts or emitted
workflow templates is broken in a way that has security
consequences for projects that adopt it (a template silently
eliding a signature step, a script writing a token to a log, a
default that grants more workflow permissions than the comment
claims, etc.), open an issue on this repository describing the
failure mode and a reproduction. There is nothing private to
report against — this repository contains scaffolding and
templates, not a running service, and the scaffolding's behaviour
is fully visible to anyone who reads the templates.

For security issues in the underlying tools (sigstore, cosign,
GitHub Actions, the Ubuntu archive), report directly upstream;
those projects have their own coordinated disclosure processes.
