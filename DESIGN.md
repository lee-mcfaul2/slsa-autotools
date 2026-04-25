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

## What the two-pass diff surfaced

Every source and Windows build runs twice in independent build
trees. The `Verify reproducibility` step byte-compares the
outputs; any difference fails the job. The following are real
non-determinism issues the diff caught during development
against xz — not bugs in our scaffolder, but latent
non-determinism in standard build tooling that only shows up
when you compare bytes across runs.

### 1. Modern gzip silently ignores the `GZIP` environment variable

`automake`'s `dist-gzip` recipe compresses via `GZIP=$(GZIP_ENV)
gzip -c`. For years the documented way to make gzip's output
reproducible was `export GZIP=-n` (suppress the filename +
mtime fields in the gzip header). On Ubuntu 24.04, `gzip`
deprecated reading flags from the `GZIP` env var for security
reasons and silently ignores it. Setting `GZIP=-n` in the
workflow's environment, or even inline on the make command, had
no effect — the header embedded the current wall-clock time,
and the two passes produced different bytes a few seconds
apart.

**Fix:** abandon `dist-gzip` entirely. The pipeline now runs
`make distdir` → builds `.tar` manually with explicit
`TAR_OPTIONS` → pipes into `gzip -c --no-name --best` where the
`--no-name` flag is passed as a direct argument rather than via
the environment. `.tar.xz` is built the same way.

### 2. Silent empty-tarball SLSA attestation

`make -s dist-gzip dist-xz` resolves the shared `distdir`
prerequisite once, runs `dist-gzip`'s recipe (which tars the
distdir and then removes it), then runs `dist-xz`'s recipe:
`tardir=$(distdir) && $(am__tar) | xz -c > file.tar.xz`. By the
time `dist-xz` fires, the distdir is gone. `tar` prints
`Cannot stat: No such file or directory`, emits an empty
archive, and exits 0. Every prior release shipped 108-byte
empty `.tar.xz` files and 45-byte empty `.tar.gz` files with
cryptographically valid SLSA attestations — valid attestations
for no content.

**Fix, two parts:** (a) stop relying on automake's intertwined
dist-* recipes; build the distdir once, tar it ourselves. (b)
Add a size-floor liar-detector after the build: if either
tarball falls below 32 KiB, fail the job. The bit-compare alone
can't distinguish "two identical empty tarballs" from "two
identical real tarballs" — the size check does.

### 3. `dist-hook` side effects on a warm source tree

`xz`'s `dist-hook` runs `git log`, `manconv` (groff → .txt), a
license-check script, and a couple of other normalisation steps
during `make dist`. On a cold source tree, the hook produced
deterministic output; invoked a second time against the same
tree, several of those generated files differed by a line or
two. The first bit-compare after switching to single-format
`make dist-xz` passed on identical runs but failed on the
two-pass verification because pass 1 left state pass 2 was
affected by.

**Fix:** do each pass in its own out-of-tree build directory
(`build1/`, `build2/`), both configured from the same source.
Each pass is a genuinely cold invocation; the dist-hook runs
once per pass, produces the same output, and the bit-compare
holds.

### 4. `po4a` / `xgettext` stamps wall-clock time into translation files

The dist-hook runs `xgettext` to refresh `.pot` files and
`po4a` to regenerate translated man pages. Both write a
`POT-Creation-Date:` header field set to `strftime`-of-now. The
header ends up in `po/xz.pot`, every `po/*.po`, every
`po4a/po/*.po` — dozens of files per distdir, each differing by
seconds between the two passes.

**Fix:** after `make distdir` completes, `find ... -name '*.pot'
-o -name '*.po'` and `sed` every `POT-Creation-Date:` line to
`date -u -d @${SOURCE_DATE_EPOCH}`-derived fixed string. No-op
for projects without `po/`; unconditional for projects that
have one.

### 5. `tar` file ordering and mtimes

`tar` walks the filesystem in whatever order the underlying FS
returns entries, which on ext4 can vary between runs. Mtimes
are whatever the filesystem has at tar-time — every file in the
distdir carries the `make distdir` wall-clock timestamp, which
differs across passes.

**Fix:** `TAR_OPTIONS='--owner=0 --group=0 --numeric-owner
--mode=u+rw,go+r-w --sort=name --clamp-mtime --mtime=@${SDE}'`.
`--sort=name` forces deterministic entry order, `--clamp-mtime
--mtime` pins every mtime to `SOURCE_DATE_EPOCH`. Set
`LC_COLLATE=C` so `--sort=name` produces the same ordering
regardless of the runner's locale.

### 6. Multi-threaded xz is non-deterministic

`xz`'s multi-threaded mode splits input into blocks and assigns
them to worker threads for parallel LZMA encoding. Block
assignment is scheduling-dependent, so the output bytes differ
across runs even with identical input.

**Fix:** `xz -c --threads=1`. Single-threaded xz is
byte-deterministic; the throughput cost is negligible on
source tarballs.

### 7. PE `IMAGE_FILE_HEADER.TimeDateStamp`

`gcc` / `ld` on the mingw-w64 cross-toolchain default to
embedding a wall-clock timestamp in every PE output's
`IMAGE_FILE_HEADER.TimeDateStamp`, plus a GUID in the
`.build-id` section derived from the build process. Both differ
across runs.

**Fix:** a PATH-shadowed `${triplet}-gcc` wrapper at
`/opt/wrappers/` that injects `-Wl,--no-insert-timestamp` and
`-Wl,--build-id=none` into every compile line. The upstream
build script (xz's `windows/build.bash`, libsodium's
`dist-build/msys2-*.sh`, etc.) stays untouched.

### 8. Absolute paths in DWARF debug info

`gcc` bakes the full build-directory path into DWARF debug
info. The build directory is `${GITHUB_WORKSPACE}/build1/xz-
5.8.3/` in pass 1 and `${GITHUB_WORKSPACE}/build2/xz-5.8.3/` in
pass 2 — different strings, different bytes.

**Fix:** the same wrapper passes `-ffile-prefix-map=${DISTDIR_
TOP}=.` so the absolute distdir path is rewritten to `.` in
debug info. `DISTDIR_TOP` is exported by the job before the
build fires.

### 9. `.zip` / `.7z` central directory mtimes

When `windows/build.bash` calls `7z a -tzip` / `7z a -t7z` to
package the final Windows distribution, the archive's central
directory records each entry's filesystem mtime. Those mtimes
are whatever the filesystem has at archive-time — different
between passes.

**Fix:** a PATH-shadowed `7z` wrapper that clamps every file's
mtime to `SOURCE_DATE_EPOCH` via `touch` before forwarding the
`a` / `u` subcommand to the real `7z`.

---

Each of these is the kind of issue a release pipeline without a
bit-compare step would ship without ever noticing. That's the
argument for the two-pass diff: not "we trust our build system
more when it's deterministic" (fine), but "we literally cannot
ship attestations that mean anything until this many subtle
timestamp / ordering / locale dependencies are pinned down."
The diff is cheap; each fix is mechanical once identified. The
expensive part is knowing they exist.

## Pending refactor: per-pass container isolation

> **Status:** queued for a fresh session. Read this section on
> resume; the architectural decision is made, only the
> implementation remains.

### The problem

The current `build_source` and `build_windows` jobs run both
reproducibility passes inside the same container, in sibling
build directories (`build1/` and `build2/`). The two passes
therefore have *different absolute build paths* — `pwd` in pass
1 is `.../build1/${PACKAGE}-${VERSION}`, in pass 2
`.../build2/${PACKAGE}-${VERSION}`. Most autotools-derived
artefacts are oblivious to that difference, but a few install-
time outputs bake the configure-time `--prefix` into their
content verbatim:

- `lib/pkgconfig/*.pc` — `prefix=...` line
- `lib/lib*.la` libtool archives — `libdir=...` line
- Some projects' generated headers that embed install paths

For libsodium specifically, the build_windows hook configures
with `--prefix="$(pwd)/libsodium-win64"`. `$(pwd)` differs
across passes, so the resulting `libsodium.pc` differs, the
tarball differs, the bit-compare fails. The same class of bug
will hit any project whose `make dist` happens to also bake
absolute paths in.

A `DESTDIR` workaround inside the hook (configure with
`--prefix=/`, install with `make install
DESTDIR=$(pwd)/libsodium-winN`) fixes libsodium specifically
but is per-target whack-a-mole.

### The chosen architecture

Each pass runs in **its own container**, both at the same `pwd`
inside the container (e.g. `/work/build/${PACKAGE}-${VERSION}`).
Identical absolute paths in both passes means anything that
embeds the build path embeds the *same* build path. The whole
class of "absolute-path leakage broke reproducibility" goes
away by construction, with no per-target hooks needed.

GitHub Actions natively supports this: each top-level `job:` is
its own runner with its own container, and jobs run in parallel
by default. Artifact upload/download is the standard way jobs
hand bytes to one another.

### Target shape (release.yml jobs)

```
test
  ↓ (provides outputs.has_windows)
  ├─ build_source_pass1   ─┐
  ├─ build_source_pass2   ─┤  → verify_source   ─┐
  ├─ build_windows_pass1  ─┐                     ├─→ publish
  └─ build_windows_pass2  ─┘  → verify_windows  ─┘
                              (only if has_windows == 'true')
```

- `build_source_pass{1,2}`: identical jobs. Each does
  checkout, bootstrap, configure, distdir, tar, gzip, xz,
  optional bz2. Uploads its raw `${DISTDIR}.tar.*` set as an
  artifact named `source-pass1` / `source-pass2`. No
  attestation, no signing.
- `verify_source`: needs both pass jobs. Downloads both
  artifacts, runs `diff -q` on every file with the same name,
  fails if any byte differs. Then assembles the final
  `release/` directory from pass1's bytes, runs
  `actions/attest-build-provenance`, runs `cosign sign-blob`,
  writes `SHA256SUMS`, uploads `source-artifacts` artifact.
- `build_windows_pass{1,2}`: gated on
  `needs.test.outputs.has_windows == 'true'`. Identical to the
  current `build_windows` step body but without the second
  pass and without the verify+attest+sign steps.
- `verify_windows`: gated the same way; needs both pass jobs.
  Same shape as `verify_source`.
- `publish`: needs `[verify_source, verify_windows]`. Uses the
  existing skipped-allowed gate for `verify_windows` so
  Windows-less projects publish cleanly.

### What carries over unchanged

- Hooks themselves (`windows-hook.sh`, `build-hook.sh`).
- Reproducibility wrappers in `/opt/wrappers/`.
- The pinned builder image — both pass jobs use the same
  `container: image:` digest.
- Liar-detector size floors and manifest checks (move to
  `verify_*` jobs).
- SLSA attestation and cosign signing semantics. The bytes
  being signed are the bit-verified pass1 bytes — same trust
  property, just attested in the verify step rather than the
  build step.

### Net effect

- Reproducibility becomes a property of the architecture,
  not of each target's build system happening not to leak
  state. The `DESTDIR` workaround in libsodium's hook can be
  reverted; no equivalent workaround is needed for any future
  project's hook either.
- Wall-clock time drops: pass1 and pass2 run concurrently, so
  total ≈ max(pass1, pass2) + verify overhead, instead of
  pass1 + pass2 sequentially. ~6 min instead of ~12 min for
  libsodium-scale Windows builds.
- Runner-minute usage roughly doubles (two parallel runners
  per pair instead of one sequential). Acceptable trade.

### Implementation notes for a fresh session

- The body of `build_*_pass{1,2}` is mechanically identical
  per pair. Either inline-duplicate (simplest, slightly bigger
  YAML) or extract a reusable workflow (`workflow_call`).
  Inline is fine for v1 — the duplication is bounded.
- Move the `Diagnose` step (currently `if: failure()` in the
  monolithic build job) into the `verify_*` job. It only runs
  on diff failure, which now happens there.
- After landing in the template, propagate to xz and
  libsodium's `release.yml`. xz currently passes its bit-
  compare so the refactor is functional; libsodium's hook
  already accommodates the contained pwd (no DESTDIR needed
  once each pass has identical pwd).
- Update the relevant DESIGN.md sections — the `dist-hook`
  side-effects section (#3) becomes weaker because each pass
  is a cold container, not a sibling build dir; the underlying
  fixes still apply but the framing becomes "each pass is
  cold by construction" rather than "each pass uses a fresh
  build dir within a shared container."
