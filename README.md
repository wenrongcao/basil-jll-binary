# basil-jll-binary

Packaging notes and the [BinaryBuilder](https://github.com/JuliaPackaging/BinaryBuilder.jl)
recipe for shipping [basil](https://github.com/wenrongcao/basil) — a 2-D viscous-flow
finite-element solver — as the precompiled Julia artifact **`basil_jll`** via
[Yggdrasil](https://github.com/JuliaPackaging/Yggdrasil).

The JLL bundles 9 executables: the `basil` solver, the headless `sybilps`
PostScript post-processor, and the `xpoly` mesh/inversion helper tools. (The Motif
GUI `sybil` is excluded.)

## Contents

- **[`build_tarballs.jl`](build_tarballs.jl)** — the Yggdrasil recipe (lives at
  `B/basil/build_tarballs.jl` upstream).
- **[`SKILL.md`](SKILL.md)** — how to build, test, and repoint the recipe: design
  rationale, the GitHub Actions smoke test, the platform matrix, and known gotchas.

Tracking PR: [JuliaPackaging/Yggdrasil#14172](https://github.com/JuliaPackaging/Yggdrasil/pull/14172).
