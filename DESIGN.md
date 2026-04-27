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

## Linux binaries: AppImage only, no .deb / .rpm / bare-ELF

slsa-autotools produces Linux binaries as AppImages via the same
two-pass per-container architecture used for source tarballs and
Windows binaries. It does **not** emit `.deb`, `.rpm`, Snap, Flatpak,
or bare-ELF tarballs.

This is a deliberate trade-off, parallel in shape to the Windows
decision but with the answer flipped — Linux binary determinism is
tractable, just only at one specific format level.

1. **One build needs to work everywhere modern.** Bare ELF would
   force adopters to ship one tarball per glibc / libstdc++ ABI
   window — at minimum an "old enough" build for Debian-stable users
   and a "new" build for rolling-release users. AppImage bundles its
   library closure, so a single `*.AppImage` runs across most distros
   from a build cut on a stable old base. One binary set, one
   provenance attestation, one cosign signature.

2. **Distro packaging means reproducing distro tooling.** A
   reproducible `.deb` requires running `dpkg-deb` against a
   particular Debian-policy-compliant tree, with that distro's
   `dh_*` helpers in scope; `.rpm` requires `rpmbuild` and a spec
   file. Each ties the pipeline to a specific distro's release
   cadence and packaging conventions, which then have to be kept
   in sync as the distro evolves. Distros already package
   well-known autotools projects via their own infrastructure;
   slsa-autotools does not try to compete with that — it stacks
   alongside, the same way it does with MSVC for Windows.

3. **The reproducibility surface is small and well-trodden.** An
   AppImage is a fixed-bytes runtime stub concatenated with a
   SquashFS payload. SquashFS has documented reproducible mode
   (`-mkfs-time 0 -all-time 0 -no-exports -no-xattrs -all-root`,
   plus `-reproducible` on newer versions). The ELF binaries inside
   need `-Wl,--build-id=none` (or `=sha1` for content-derived) and
   `-ffile-prefix-map=$DISTDIR_TOP=.` for DWARF stability — same
   `SOURCE_DATE_EPOCH` story the source-tarball path already
   handles. No equivalent of the PE `IMAGE_FILE_HEADER.TimeDateStamp`
   / debug-directory / PDB-GUID maze that made MSVC untenable.

### Wrappers

Two PATH-shadowed wrappers, both constructed inline in the
`build_appimage_pass*` jobs (mirroring how
`build_windows_pass*` stages the mingw + 7z wrappers):

- `/opt/wrappers/{gcc,g++,cc,c++}` — `exec` the real compiler with
  `-ffile-prefix-map=${DISTDIR_TOP}=.` and `-Wl,--build-id=none`
  prepended to the argument list. Project build script stays
  unchanged upstream.
- `/opt/wrappers/mksquashfs` — `exec /usr/bin/mksquashfs "$@"` with
  the determinism flags **appended**. mksquashfs resolves duplicate
  options last-wins, so appending lets our flags override anything
  the calling AppImage builder passed. Means: any AppImage builder
  that calls `mksquashfs` from PATH gets reproducible payloads
  transparently — linuxdeploy, linuxdeployqt, appimagetool, and
  appimage-builder all qualify.

### What slsa-autotools does not pin

The specific AppImage builder (linuxdeploy, linuxdeployqt,
appimagetool standalone, appimage-builder, electron-builder, …) is
the project's choice. The `.slsa-autotools/linux-hook.sh` adapter is
where the project downloads its preferred builder by SHA256 and
invokes it. Same model as `cosign` in the existing release.yml — the
universal flow doesn't lock in a specific tool, but the hook is
expected to pin whatever it pulls.

This is also where the AppImage-builder tooling's own determinism
holes get patched. linuxdeployqt's "continuous" tag floats; a
pinning hook downloads it at a known SHA256 and rejects any other
bytes. Same pattern for any tool that ships rolling releases.

### What this means for adopters

- **Projects that already ship an AppImage** (ImageMagick, GIMP,
  KDE apps via linuxdeploy plugins) need a `linux-hook.sh` that
  re-implements the existing AppImage build with SHA-pinned tooling.
  Usually a 30–60 line script.
- **Projects that ship bare ELF tarballs or distro packages** are
  out of scope. Producing an AppImage from an autotools project that
  has never had one is feasible but project-specific work; the
  scaffolder doesn't generate one for you.
- **Projects that don't ship Linux binaries at all** (most C
  libraries — libsodium, libpng, libgcrypt) skip this path entirely.
  No `linux-hook.sh`, the `build_appimage_pass*` jobs see the
  `has_linux` gate as false and report skipped.

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

**Fix:** each pass runs in its own container against a freshly
checked-out source tree, so the dist-hook never fires against
warm state. (Originally fixed by sibling out-of-tree build
directories — `build1/`, `build2/` — within a shared container;
the per-pass-container split that came later subsumes that
mechanism. The architectural property is the same: every pass
is genuinely cold, every dist-hook run produces the same
output, the bit-compare holds.)

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

## Per-pass container isolation

> **Status:** landed in `release.yml` and verified against
> libsodium. The Windows bit-compare clears without any DESTDIR
> workaround in `windows-hook.sh`, which is the simplest
> available demonstration that the architectural property holds
> in practice.

### The problem this solves

Originally `build_source` and `build_windows` ran both
reproducibility passes inside the same container, in sibling
build directories (`build1/` and `build2/`). The two passes
therefore had *different absolute build paths* — `pwd` in pass
1 was `.../build1/${PACKAGE}-${VERSION}`, in pass 2
`.../build2/${PACKAGE}-${VERSION}`. Most autotools-derived
artefacts were oblivious to that difference, but a handful of
install-time outputs bake the configure-time `--prefix` into
their content verbatim:

- `lib/pkgconfig/*.pc` — `prefix=...` line
- `lib/lib*.la` libtool archives — `libdir=...` line
- Some projects' generated headers that embed install paths

For libsodium specifically, `windows-hook.sh` (via
`dist-build/msys2-win64.sh`) configures with
`--prefix="$(pwd)/libsodium-win64"`. `$(pwd)` differed across
passes, so the resulting `libsodium.pc` differed, the tarball
differed, the bit-compare failed. The same class of bug would
hit any project whose `make dist` happened to bake an absolute
path in.

A `DESTDIR` workaround inside the hook (configure with
`--prefix=/`, install with
`make install DESTDIR=$(pwd)/libsodium-winN`) fixed libsodium
specifically but was per-target whack-a-mole — every new
project that leaked a path needed its own hook tweak.

### The architecture

Each pass runs in **its own container**, both at the same `pwd`
inside the container
(`${GITHUB_WORKSPACE}/build/${PACKAGE}-${VERSION}`). Identical
absolute paths in both passes means anything that embeds the
build path embeds the *same* build path. The whole class of
"absolute-path leakage broke reproducibility" goes away by
construction, with no per-target hooks needed.

GitHub Actions natively supports this: each top-level `job:` is
its own runner with its own container, and jobs run in parallel
by default. Artifact upload/download is the standard way jobs
hand bytes to one another.

### Job graph

```
test
  ↓ (provides outputs.has_windows)
  ├─ build_source_pass1   ─┐
  ├─ build_source_pass2   ─┤  → verify_source   ─┐
  ├─ build_windows_pass1  ─┐                     ├─→ publish
  └─ build_windows_pass2  ─┘  → verify_windows  ─┘
                              (skipped on projects with
                              no Windows entry point)
```

- `build_*_pass{1,2}`: identical jobs in parallel. Each does
  checkout, bootstrap, configure, distdir/build, per-pass
  liar-detector (size floor + manifest match), and uploads its
  raw output as `source-pass{1,2}` / `windows-pass{1,2}`. Pass
  jobs are read-only (`contents: read`) — no attestation, no
  signing.
- `verify_*`: needs both pass jobs. Downloads both artifacts,
  file-set match, `diff -q` byte-compare, then assembles
  `release/` from pass1's bytes (identical to pass2 by the
  bit-compare), runs `actions/attest-build-provenance`, runs
  `cosign sign-blob`, writes `SHA256SUMS`, uploads
  `source-artifacts` / `windows-artifacts`. The verify jobs
  hold `attestations: write` and `id-token: write`.
- `publish`: needs `[verify_source, verify_windows]`. The
  `if:` permits `verify_windows` to be skipped (Windows-less
  projects) but blocks on failure or cancellation.

### Net effect

- Reproducibility becomes a property of the architecture, not
  of each target's build system happening not to leak state.
  The `DESTDIR` workaround in libsodium's hook is gone, and no
  equivalent workaround is needed for any future project's
  hook either.
- Wall-clock time drops: pass1 and pass2 run concurrently, so
  total ≈ max(pass1, pass2) + verify overhead, instead of
  pass1 + pass2 sequentially. Roughly halves the Windows-side
  release latency on libsodium-scale builds.
- Runner-minute usage roughly doubles (two parallel runners
  per pair instead of one sequential). Acceptable trade for
  the structural guarantee.
