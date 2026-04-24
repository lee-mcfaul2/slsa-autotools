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
