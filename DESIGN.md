# Design notes

Collected rationale for the non-obvious architectural choices in
slsa-autotools. Material here is intended as source for the
eventual README / public docs rather than as standalone
documentation.

## Windows binaries: mingw cross-compile only, no MSVC / no native Windows runners

slsa-autotools produces Windows binaries by running `mingw-w64`
cross-compile on the same pinned Ubuntu builder image the source
tarball build uses. It does **not** use GitHub's
`runs-on: windows-latest`, does not install Visual Studio or the
MSVC Build Tools, and does not produce MSVC-linked `.lib` / `.dll` /
`.pdb` artefacts.

This is a deliberate trade-off. Three reasons:

1. **The `windows-latest` runner image is mutable.** Microsoft
   updates it out of band — there is no equivalent of
   `snapshot.ubuntu.com` that lets you say "use the VS 17.x
   toolchain as it existed at timestamp T." The whole input-pinning
   story this project is built on collapses the moment the
   toolchain can drift between runs.

2. **PE timestamp determinism is much harder than ELF.** The PE /
   COFF format embeds timestamps in many places —
   `IMAGE_FILE_HEADER.TimeDateStamp`, debug directory timestamps,
   export directory timestamps, PDB GUIDs, resource section
   stamps. `SOURCE_DATE_EPOCH` alone doesn't cover all of them on
   MSVC, and `/Brepro` plus careful linker settings only gets part
   of the way. The mingw toolchain, through GNU ld or LLD, handles
   this cleanly with `-Wl,--no-insert-timestamp` and
   `-Wl,--build-id=none`, which is why our reproducibility
   wrappers (`/opt/wrappers/${triplet}-gcc`) work.

3. **The reproducibility workaround blows up CI time.** The only
   way to get reproducible MSVC output with input pinning is to
   download and install a pinned-version VS Build Tools bundle on
   every run. That costs roughly 20 minutes per job in install
   overhead alone, then still leaves the timestamp issues above
   unsolved. The net is: longer runs, worse determinism than the
   mingw path, and Microsoft licensing to worry about.

So: mingw on Linux. The apt snapshot pins
`gcc-mingw-w64-{i686,x86_64}` and `binutils-mingw-w64-*` by
version and SHA256, the compiler wrappers inject the determinism
flags, and a full Windows build takes about three minutes per
pass on a standard runner.

### What this means for adopters

- **Projects that ship a mingw cross-compile path** (e.g. xz's
  `windows/build.bash`, libsodium's `dist-build/msys2-*.sh`) are
  fully supported. Adoption is one file: either the project
  already has a suitable script, or a short
  `.slsa-autotools/windows-hook.sh` wraps whatever it does.
- **Projects that only ship MSVC-built Windows artefacts** are
  out of scope. The recommended path is to add a mingw build to
  your project, or to keep your existing MSVC pipeline separate
  and let slsa-autotools produce the mingw side. A C library
  consumer using Visual Studio still wants the MSVC-built
  artefact; slsa-autotools does not try to replace that, it
  stacks alongside.

## Signing: sigstore only, no Notary interop

slsa-autotools signs every released artefact with sigstore
(`cosign sign-blob`, plus `actions/attest-build-provenance` for
SLSA v1 attestation). It does **not** produce Notary v2 /
Notation signatures, and there are no plans to add a "convert
sigstore bundle to Notary signature" path.

Two reasons:

1. **Cert custody attack surface.** Notary binds trust to a
   long-lived X.509 cert held by the signer. That cert becomes a
   target — lose the key, rotate the key, leak the key, and the
   whole signing history is in question. Sigstore issues
   short-lived certs per-sign against an OIDC identity (GitHub
   Actions, in our case) and the cert expires in minutes. There
   is no long-lived secret for an attacker to steal, no rotation
   schedule to get wrong, no off-boarding story when a
   maintainer leaves.

2. **Transparency is a property of sigstore, not notary.**
   Every signature sigstore issues lands in Rekor, a public
   append-only transparency log. Anyone in the world can audit
   the full history of signatures issued for a given
   repository / workflow identity without asking the maintainer
   for anything. Notary signatures live in the registry and
   leave no independent trail — verifying a signature tells you
   it's valid, but not whether the signer ever issued a
   different one at the same version, or whether the cert was
   used to sign anything else suspicious. Rekor closes that gap
   by design.

Supporting Notary alongside would mean reintroducing the cert
custody model we spent effort removing, and would double the
signing surface without adding any verification property that
sigstore doesn't already provide to the same audience. If an
adopter needs Notary signatures for an enterprise registry
policy, that pipeline belongs upstream of or alongside
slsa-autotools, not inside it.

## Enterprise / air-gapped deployments

The default templates pin two external endpoints:

- `snapshot.ubuntu.com` — the apt snapshot mirror that makes
  package pinning reproducible. Every byte the builder image
  consumes is served from there.
- `fulcio.sigstore.dev` / `rekor.sigstore.dev` — the public
  sigstore services cosign talks to by default.

Most enterprise GitHub setups — GitHub Enterprise Cloud, or GHES
with internet egress — use the public endpoints as-is. Sigstore
is open source (Apache 2.0) and GHES issues OIDC tokens the same
way github.com does, so the workflow identity binding and the
transparency log story work identically.

Fully air-gapped deployments need internal equivalents:

- An internal mirror of the Ubuntu snapshot (the bigger lift,
  unrelated to signing).
- A private sigstore stack — Fulcio, Rekor, TUF root
  distribution — sitting inside the airgap.
- The enterprise's own OIDC identity provider (most setups
  already have one for GHES).

The scaffolder doesn't currently expose these endpoints as
first-class configuration — they're hardcoded to the public
defaults. Adding the overrides is a small templating change, not
an architectural one:

- `scan.yml` needs an input variable for the snapshot URL in
  place of the hardcoded `snapshot.ubuntu.com` path.
- `release.yml`'s `cosign sign-blob` steps (and the cosign
  install steps) need `FULCIO_URL` and `REKOR_URL` passed through
  the step's `env:` block, plus a TUF root distribution
  mechanism.
- `build-container.yml`'s `cosign sign` step needs the same
  overrides.

All three swaps preserve the cryptographic properties; they just
relocate the trust domain from the public internet to the
enterprise's own infrastructure. Deferring the implementation
until an adopter needs it — premature parameterisation when no
one is asking for it yet would add template complexity for
hypothetical users.
