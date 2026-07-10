---
name: package-basil-jll
description: Use when the user wants to build, update, test, or repoint the basil_jll BinaryBuilder/Yggdrasil recipe — the precompiled Julia artifact bundling the basil FEM solver, the headless sybilps post-processor, and the xpoly helper tools. Covers the recipe design (driving MakeSimple, the C++/libstdc++ linking fix, platform filters), how to smoke-test it on a GitHub Actions runner, the platform matrix and why i686-musl/Windows are excluded, the known BinaryBuilder auditor deadlock, repointing the source pin to upstream, and the follow-up basil.jl wrapper package.
---

# Packaging basil as `basil_jll` (BinaryBuilder + Yggdrasil)

Produce a precompiled, cross-platform Julia binary artifact (`basil_jll`) for basil
by adding a recipe to [Yggdrasil](https://github.com/JuliaPackaging/Yggdrasil). The
finished recipe lives at `B/basil/build_tarballs.jl`; a verbatim copy is in this
folder as [`build_tarballs.jl`](build_tarballs.jl) — read it alongside this file.

The recipe was accepted via Yggdrasil PR
[#14172](https://github.com/JuliaPackaging/Yggdrasil/pull/14172). On merge,
`basil_jll` auto-registers under JuliaBinaryWrappers.

## What ships

**One JLL, `basil_jll`, containing 9 executables:**

| binary | source dir | notes |
|---|---|---|
| `basil` | `basilsrc` | the FEM solver. F77 + C + one C++ file (`polyutils.cc`). Links libgfortran **and** a C++ runtime. |
| `sybilps` | `sybilsrc` | headless PostScript post-processor. F77 + C. **No X11.** |
| `xpoly`, `polyfix`, `selvect`, `mdcomp`, `basinv`, `circles`, `corotate` | `xpoly` | single-file Fortran tools |

- **`sybil` (the Motif GUI) is excluded** — there is no Motif/openmotif recipe in
  Yggdrasil, and packaging Motif is a project of its own. The headless
  `sybilps -> PostScript -> PNG` path (see the `basil-figures` skill) covers the only
  use a JLL can realistically serve.
- **`name = "basil"`** (lowercase — well-precedented: `x264_jll`, `libpng_jll`). The
  recipe lives at `B/basil/build_tarballs.jl` (Yggdrasil buckets by first letter).
- **`version = v"1.8.2"`.** Upstream calls itself `1.8.2g`; the trailing letter is not
  semver. Rebuilds of the same upstream version use JLL build numbers (`1.8.2+0`,
  `+1`, …), not a version change.

## Recipe design — why it looks the way it does

Read [`build_tarballs.jl`](build_tarballs.jl); the non-obvious decisions:

- **Drive the hand-written `MakeSimple` files, never imake.** Upstream's top-level
  Makefile is imake-generated and host-specific. All three `MakeSimple` files
  (`basilsrc`, `sybilsrc`, `xpoly`) accept `FOR`/`CC`/`CPP`/`FFLAGS`/`CFLAGS`/`LDFLAGS`
  as plain make variables, and command-line assignment overrides in-file assignment.
  They already pass `-DTRILIBRARY` to `triangle.c` and place `$(LDFLAGS)` after the
  objects. Re-listing ~40 compile commands inline, or patching via `DirectorySource`,
  would duplicate source lists and drift from upstream for zero gain.
- **`CPP="${CXX}"` — the one genuine defect fix.** `MakeSimple`'s `CPP` defaults to
  `gcc`, which cannot compile the C++ file `polyutils.cc`. Point it at the real C++
  compiler. No source patch needed.
- **`LDFLAGS=${CXXLIB}` picks the C++ runtime per platform.** `MakeSimple` hardcodes
  `-lstdc++`; Darwin and FreeBSD use clang with `libc++`. The recipe sets
  `-lc++` on `*-apple-darwin*` and `*freebsd*`, `-lstdc++` elsewhere. This is confined
  to `basilsrc` — `sybilps` links with an empty `LDFLAGS` and pulls in no C++ runtime.
- **`-DLINUX` only on `*-linux-gnu*`.** `triangle.c` clamps the x87 FPU control word
  via `fpu_control.h`, which is glibc-only. Guarded so macOS/FreeBSD/musl compile
  cleanly. (See the i686-musl note under the matrix.)
- **`FFLAGS="-O2 -std=legacy"`, and never `-fallow-argument-mismatch`.** The default
  compiler for libgfortran5 is **GCC 8.1**, and `-fallow-argument-mismatch` did not
  exist until GCC 10 — passing it hard-fails nearly every job with *"unrecognized
  command line option"*. The F77 needs no such flag (see Verified facts).
- **`-DGFORTRAN` is in the shared `FFLAGS`.** It is consumed by the preprocessed
  `basil.F` and is a harmless no-op on the plain `.f` compiles, so one `FFLAGS` covers
  both.
- **`expand_gfortran_versions` + `expand_cxxstring_abis` are nearly free.** Modern
  BinaryBuilderBase defaults to `old_abis=false`, so each emits only the *new* ABI
  (`libgfortran5`, `cxx11`) — the matrix count does not change. `expand_cxxstring_abis`
  is **required, not optional**: the audit flags `basil` as containing `std::string`
  values (from `polyutils.cc`), which cross the GCC 4/5 ABI boundary. Dropping it makes
  the audit warn and mis-tags the build dir. (This was independently confirmed by a
  Yggdrasil maintainer during review.)

## Source strategy and the standing constraint

The recipe pins a **maintained fork, `wenrongcao/basil`**, not `greg-houseman/basil`,
because the fork carries the regular-mesh **NOR regression fix** (`vsbcon.f`,
`crust.f`, `basil.F`) that upstream lacks. `GitSource` pins a full 40-char SHA.

> **STANDING CONSTRAINT — do not break this.** Because the recipe pins your own repo,
> `wenrongcao/basil` must **stay public** and its history must **never be rewritten**.
> A `filter-repo`/force-push would invalidate the pinned SHA and break every future
> rebuild of `basil_jll`, including rebuilds Yggdrasil triggers itself. Adding new
> commits is fine — only rewriting is fatal. GPL-3.0 independently obliges you to keep
> the corresponding source fetchable for as long as you distribute the binaries.

A JLL source must be **anonymously fetchable, forever** — Yggdrasil re-fetches on every
rebuild. (An early CI failure was exactly this: the repo was private and
`cached_git_clone` fell back to a username prompt. Fixed by making it public; verify
with an anonymous clone using an empty `HOME` and no credential helper.)

### Repointing the source to upstream later

Once the NOR fix is merged into `greg-houseman/basil`, you can point `basil_jll` back
at the original — it is a clean one-file Yggdrasil PR:

1. In `build_tarballs.jl`, change the `GitSource` URL to
   `https://github.com/greg-houseman/basil.git` and the SHA to the upstream merge
   commit that contains the fix.
2. Bump the JLL build number (`1.8.2+1`), or the version if upstream retags.
3. Open the PR; Yggdrasil CI verifies the upstream commit builds green.

This is low-risk because the build only needs *source + the NOR fix* — everything else
that lives only in the fork (the `-O2` default in `basil.tmpl`, skills, example
figures) is **not used by the recipe**: `-O2` comes from the recipe's own `FFLAGS`, and
`LICENSE` exists upstream too. Note: already-published builds (`1.8.2+0`) still record
the fork SHA as their GPL corresponding-source, so keep the fork public until those
binaries are no longer distributed.

## Verified facts (do not re-derive)

- **`basil_jll` / `Basil_jll` is a free name** — absent from JuliaRegistries/General
  and from Yggdrasil `B/`.
- **`sybilps` needs no X.** The `#include <X11/Xlib.h>` in `sybmain.c` is inside
  `#ifdef XSYB`; `defines.h` maps `fontdat`/`colourdat` to `int` in the non-X build;
  `elle.c`, `log.c`, `pref77.c` are likewise guarded. `MakeSimple` builds the PS object
  from the same `sybmain.c` without `-DXSYB`. `ldd bin/sybilps` shows no
  `libX11`/`libXm`/`libXt`. No patch needed.
- **No legacy-Fortran flags needed.** `gfortran -fsyntax-only -std=legacy` under
  gfortran 11.4 (GCC-10+ argument-mismatch-as-error) passes on every Fortran source
  actually built. The only two failures in the tree, `r4write.f` and `r8read.f`, are
  never compiled by `MakeSimple` (marked "unused ?"). Modern gfortran (GCC 13/14/15,
  used for riscv64 / aarch64-freebsd / aarch64-darwin) also compiles this F77 cleanly.
- **No Fortran modules in the build.** No `MODULE`/`USE` statements; the checked-in
  `basilsrc/modremesh.mod` is an orphan. `make -j` is safe (the recipe still `rm -f`s
  the stale `.mod`/`.o` defensively).
- **BinaryBuilder cannot run on aarch64 (e.g. a Jetson).** BB requires an x86_64 Linux
  or x86_64 macOS host to run its sandbox. `BinaryBuilderBase` (pure Julia) does load on
  aarch64, so you can compute the platform matrix there, but not build.

## The platform matrix

`supported_platforms()` returns 18; the recipe's filters leave **15**:

- **Drop Windows** (`filter!(!Sys.iswindows, …)`): basil writes gfortran unformatted
  sequential records through relative cwd paths, never validated on Windows.
- **Drop `i686-linux-musl`** (`filter!(p -> !(arch(p)=="i686" && libc(p)=="musl"), …)`):
  `triangle.c` clamps the x87 control word only under `-DLINUX` (glibc `fpu_control.h`)
  or `-DCPU86` (MSVC); on i686+musl neither applies, and i686 GCC defaults to
  `-mfpmath=387`, so Shewchuk's exact-arithmetic mesh predicates would run with
  unclamped 80-bit intermediates and can rarely misjudge a triangulation. This is a
  **correctness** argument, not a build failure — it holds even if the target builds
  green, which is the argument to give a reviewer. `i686-linux-gnu` gets `-DLINUX` and
  is fine. (Alternative if ever wanted: force `-msse2 -mfpmath=sse` on i686; untested.)

The build host is always x86_64 Linux; every target is cross-compiled. **A green
Yggdrasil CI proves compile + link + audit only — it never executes a target binary.**
Runtime correctness is entirely on you.

## Testing the recipe on a GitHub Actions runner

Since BB needs an x86_64 host, use a free Actions runner to get both a compile check
*and* a native run of the artifact on `x86_64-linux-gnu`. Put this on a **dedicated CI
branch of your Yggdrasil fork** (e.g. `basil-jll-ci`) and keep it out of the branch you
PR from — a workflow file must not appear in the Yggdrasil diff.

```yaml
name: basil JLL smoke test
on:
  push:
    branches: [basil-jll-ci]
  workflow_dispatch:
jobs:
  build-and-run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # Ubuntu 24.04 restricts unprivileged user namespaces via AppArmor, which
      # BinaryBuilder's sandbox needs; without this the build fails with an opaque
      # sandbox error.
      - name: Allow unprivileged user namespaces
        run: sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0 || true
      # basil links libgfortran.so.5 at runtime but the tarball does not bundle it.
      - name: Install runtime libs
        run: sudo apt-get update && sudo apt-get install -y libgfortran5
      - uses: julia-actions/setup-julia@v2
        with: { version: '1.10' }
      - name: Install BinaryBuilder
        run: julia -e 'using Pkg; Pkg.add("BinaryBuilder")'
      - name: Build x86_64-linux-gnu
        working-directory: B/basil
        run: julia --color=yes build_tarballs.jl --verbose x86_64-linux-gnu-libgfortran5
      - name: Unpack the product
        working-directory: B/basil
        run: |
          mkdir -p /tmp/bt
          # NOTE: select the product explicitly — see "two tarballs" gotcha below.
          tar -xzf products/basil.v*.tar.gz -C /tmp/bt
          ls -l /tmp/bt/bin
      - name: Smoke test (indenter INn3A0)
        run: |
          git clone --depth 1 https://github.com/wenrongcao/basil /tmp/basilsrc
          mkdir -p /tmp/run/FD.sols /tmp/run/FD.out
          cp /tmp/basilsrc/examples/indenter/INn3A0 /tmp/run/
          cd /tmp/run
          /tmp/bt/bin/basil INn3A0
          test -s FD.sols/INn3A0 || { echo "no solution file produced"; exit 1; }
```

**Gotchas:**

- **CLI triplets need the full tagged form.** With `expand_cxxstring_abis`, the Linux
  target is `x86_64-linux-gnu-libgfortran5-cxx11`. Passing a bare triplet or a wrong tag
  builds nothing.
- **`riscv64` can't be named as a CLI arg on Julia 1.10** (`ArgumentError: Key "arch"
  cannot have value "riscv64"` — Base.BinaryPlatforms learns it in 1.12). Yggdrasil's
  own CI builds it fine without naming triplets; just don't try to smoke-test it locally.
- **Two tarballs.** `products/` holds both `basil.vX.…tar.gz` (the product) and
  `basil-logs.vX.…tar.gz` (build logs), and `basil-logs` **sorts first**. A glob like
  `products/*.<triplet>.tar.gz` matches both and extracts the log archive, failing with
  a confusing "Not found in archive". Select the product explicitly:
  `products/basil.v*.tar.gz`.
- **Darwin** jobs need `BINARYBUILDER_AUTOMATIC_APPLE=true` in the environment to accept
  the macOS SDK licence non-interactively.

Local iteration on an x86_64 Mac with Docker Desktop instead of Actions:

```bash
julia --color=yes build_tarballs.jl --verbose x86_64-linux-gnu-libgfortran5
julia --color=yes build_tarballs.jl --verbose --debug x86_64-linux-gnu-libgfortran5  # sandbox shell on failure
```

Raise Docker Desktop's VM memory to 4–8 GB first (the 2 GB default causes failures that
look like source problems). `x86_64-apple-darwin` is the one artifact a Mac can execute
natively.

## Known gotcha: the BinaryBuilder auditor deadlock

**basil intermittently hangs one random target per matrix build, forever, inside
`audit()`** — not a basil bug, but you will hit it on rebuilds. Signature: the build
exits 0 (all 9 binaries compile and install in ~4 s), `[ Info: Beginning audit of …`
prints, then **zero further output** until the 4-hour job timeout (`exit_status: -1`).
It is a *different target on a different agent each build*, and it is not load —
the other 14 jobs finish in ~100–250 s on the same physical box.

**Mechanism (inferred from source, not a stack trace):** `BinaryBuilder/src/Auditor.jl`
loops over binaries with `Threads.@threads`, and Yggdrasil runs `julia --threads 16`.
`check_isa` spawns a nested sandbox running `${target}-objdump -d` and streams the
disassembly through a pipe into an `IOBuffer` under `AUDITOR_SANDBOX_LOCK`. basil ships
**9 ExecutableProducts** (vs 1–2 for a typical recipe), and `basil` embeds a
16k-line `triangle.c` whose disassembly is large enough to fill a pipe buffer; the
`@async` pipe-reader can be starved of a worker thread while `objdump` blocks on the
full pipe — a deadlock. Single-product recipes (e.g. Fastscapelib) never trip it because
the `@threads` loop has one uncontended iteration.

**You cannot fix it from the recipe.** Julia's thread count is fixed at startup, so a
recipe cannot force `JULIA_NUM_THREADS=1`, and the trigger (9 products) is inherent.
Shipping fewer binaries would mutilate the package to dodge someone else's bug — don't.

**What to do when it hits:**

- **Do not force-push to reroll.** ~1 job in 15 hangs; another matrix build costs 15
  jobs of volunteer CI at the same odds.
- Ask a Yggdrasil maintainer to **retry just the stuck job** (`retryable.allowed` is
  true for maintainers; an anonymous viewer cannot). A retry has ~14/15 odds of passing.
- Optionally file it upstream against `BinaryBuilder.jl` — as of last search nobody had
  reported this exact symptom, though the multithreaded auditor (BB #1364, merged
  "completely untested") has already produced one other concurrency bug (BB #1410).

Public Buildkite endpoints for diagnosing (no auth):
```
https://buildkite.com/julialang/yggdrasil/builds/<N>.json            # build state
https://buildkite.com/julialang/yggdrasil/builds/<N>/data/jobs.json  # per-job records
```

## Settled risks (recorded so they are not re-litigated)

- **C++ linking on Darwin/FreeBSD** — the `CXXLIB` conditional works; all BSD/macOS
  targets build green.
- **riscv64 & aarch64-freebsd** — build green with GCC 13/14/15; modern gfortran is not
  a problem.
- **`-O2` numerical drift** (low) — `-O2` was validated bit-identical to `-O0` on
  aarch64. Other targets differ in last-ulp ways that are physically irrelevant but
  defeat a byte-diff. Smoke tests should assert *"a solution file exists and the run
  completed"*, not byte equality (except on aarch64, where a reference exists).
- **Version string** (cosmetic) — the binary reports `1.8.2g`; the JLL says `1.8.2`.
- **Fortran unformatted I/O is portable enough.** gfortran uses 4-byte record markers on
  all shipped targets, all little-endian, so writer (`basil`) and reader (`sybilps`)
  always agree. The only caveat is reading *legacy big-endian* solutions from old
  SPARC/POWER machines — a `GFORTRAN_CONVERT_UNIT` note for the wrapper, not a build
  problem.

## Follow-up: the `basil.jl` wrapper package

The JLL alone is awkward — `basil` requires `FD.sols/` and `FD.out/` to exist in the
cwd and reads everything by relative path. Publish a thin wrapper *after* `basil_jll`
merges (the JLL name must exist first). It may be MIT-licensed even though basil is
GPL-3.0, because it only `run()`s the executable as a subprocess (arm's-length exec, not
linking) — GPL does not propagate across that boundary. Note in its README that
installing it pulls in GPL binaries.

```julia
module Basil
using basil_jll
"""
    solve(inputfile; dir=pwd())

Run `basil` on `inputfile` inside `dir`, creating `FD.sols/` and `FD.out/` first.
"""
function solve(inputfile::AbstractString; dir::AbstractString=pwd())
    mkpath(joinpath(dir, "FD.sols"))
    mkpath(joinpath(dir, "FD.out"))
    run(Cmd(`$(basil_jll.basil()) $inputfile`; dir))
end
function postscript(logfile::AbstractString; dir::AbstractString=pwd())
    run(Cmd(`$(basil_jll.sybilps()) -i $logfile`; dir))
end
using basil_jll: xpoly, polyfix, selvect, mdcomp, basinv, circles, corotate
end
```

`[compat] basil_jll = "1.8"`. Test end-to-end with the `INn3A0` case.

## Related

- `run-basil` skill — solving and post-processing basil cases.
- `basil-figures` skill — the headless `sybilps -> PostScript -> PNG` figure pipeline.
- `JLL_PACKAGING_GUIDE.md` (repo root) — the original blow-by-blow with dated CI build
  numbers, timings, and the PR-review narrative, if you need the historical record.
