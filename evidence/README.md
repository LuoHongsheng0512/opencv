# Farneback Gaussian flow update — raw measurement data

Raw data behind the numbers in the pull request that adds a scalable-SIMD (RVV) path to
`FarnebackUpdateFlow_GaussianBlur` in `modules/video/src/optflowgf.cpp`.

Everything in this directory was produced by the measurement harness described in
`validation/` of the local handoff package. Nothing here is hand-edited, with two
exceptions that are stated below (`receipt.json` and `farneback-submitted-include-red.log`
have absolute board paths replaced, and the kernel/output hash files are filtered to this
one kernel).

## Submitted code under test

| File | SHA-256 |
| --- | --- |
| `modules/video/src/optflowgf.cpp` | `5ea162e3a422d20e5d59be792e8976935c80a7c639b98a5b86d5b3fe34a947fb` |
| `modules/video/test/test_farneback_gaussian_simd.cpp` | `8fd33e0f032e5e232e2662646cb0dda97536dd1c04d1a4ad86baf8a7f32eb09b` |
| `modules/video/perf/perf_farneback_gaussian_simd.cpp` | `e392217b9136f1c1f633e277a58f63a896c05dc30cb88cbfbc4f2f878b67b6f9` |

Baseline (`baseline/modules/video/src/optflowgf.cpp` in the local package) is upstream
`opencv:5.x` at commit `f1c1be300d10b39834fd307f23e5c2d020313a31`, SHA-256
`5d53d2f5cbb49e31db7de51c364bb956c7088384164240285c0e1be894e8c98f`. The baseline file in
this package was verified byte-identical to the file that commit serves, so the diff is
`+77 / -2` against that exact file: two `#if CV_SIMD128` lines become
`#if CV_RVV && CV_SIMD_SCALABLE ... #elif CV_SIMD128`, and the rest is added lines only.

## Board environment

| Item | Value |
| --- | --- |
| Board | BPI-F3 (SpacemiT X60), riscv64 |
| Compiler | Ubuntu clang 17.0.2, `-O3`, C++17, RVV |
| Vector length | VLEN = 256 (8 `float` lanes, 32 `uint8` lanes) |
| Macros at runtime | `CV_RVV=1`, `CV_SIMD=0`, `CV_SIMD_SCALABLE=1` |
| Threading | OpenCV single-threaded, OpenCL off, CPU affinity pinned to one core |
| Link | newly compiled object files linked against existing identical support libraries |

The probe line (`probe-run.log`) is `CV_RVV=1 scalable=1 float_lanes=8 byte_lanes=32`.
The board run is *not* a freshly built whole-library RVV build; only the translation unit
under test is recompiled and everything else is linked unchanged from the same commit.

## Method

**Correctness.** The translation unit is compiled in three configurations — default RVV,
`-ffp-contract=off` ("strict"), and scalar (`CV_RVV=0`) — each both for the pinned baseline
source and for the submitted source. Both objects are linked into the same public-API
driver, which writes one record per (API call, output element) and the two record streams
are compared element by element.

- Float Farneback output uses the engineering tolerance `1e-4 + 2e-4*abs(reference)`.
- 240 API records and 1,016,064 compared values per configuration.
- Inputs are checked to be unchanged after the call; sentinels before, after and between
  rows are checked so that out-of-ROI writes would be caught.
- Coverage includes Gaussian and box flags, non-zero initial flow, several window sizes
  and pyramid level counts, vector-boundary widths (63/64/65/127/128/129/…) where the
  scalar tail length varies, and ROI inputs with row padding.

**Performance.** Seven alternating rounds (baseline, candidate, …), median of the seven
per-round values, batch calibrated so one measurement lasts about 10 ms (a single call
that is already longer is timed once). Transfer, compilation and SSH are excluded; timing
happens on the board with the process pinned. A ratio above 1.0 means the candidate is
faster. The `c=0` cases are the *non-Gaussian* flag value: they do not enter this function
at all and are kept as controls, so their ~1.006x is measurement noise by construction.

## Files

**Comparison summaries**

- `farneback-rvv-default-comparison.json`, `farneback-rvv-strict-comparison.json`,
  `farneback-scalar-comparison.json` — records / values / mismatches / max abs error /
  bit-exact flag for each configuration.
- `farneback-rvv-default-correct-{baseline,candidate}.log` and the `rvv-strict`, `scalar`
  pairs — the driver's one-line `PASS records=240 CV_RVV=… ` output.
- `receipt.json` — the board run receipt: base commit, supporting-library commit, compiler
  string, CPU affinity, temperature/frequency at start and end, source hashes for every
  direction, library hashes, `ldd` output.

**Performance, public API**

- `farneback-bench-0..6-{baseline,candidate}.csv` — the seven rounds, `ns_per_call` per
  case, columns `path,a,b,c,d` = frame width, window (block) size, `flags`
  (0 or 256 = `OPTFLOW_FARNEBACK_GAUSSIAN`), pyramid levels. The frame is a padded ROI
  (`height=240`, `width+13` storage, offset `(3,2)`) so `step != cols*elemSize`.
- `farneback-bench-N-*.log` — `PASS records=10` for each round.
- `farneback-performance.json` — per-case medians, all seven samples and the ratio.

**Performance, extracted kernel**

- `kernel-performance-gaussianupdate.json` / `.csv` — the two blur loops called directly
  through a harness, i.e. with the dispatch gate bypassed. These are kernel ratios, not
  end-to-end speedups, and they are measured with `update_matrices=false` so only the
  blur-and-solve body is timed. Filtered from the four-direction run; only
  `GaussianUpdate` cases are kept.
- `kernel-receipt.json`, `kernel-run-*.log` — kernel harness provenance
  (`PASS 10548 kernel checks; CV_RVV=1 scalable=1`).

**Provenance**

- `outputs-sha256-farneback.json` — sizes and SHA-256 of the compiled objects and output
  binaries (filtered to `farneback-*`).
- `probe-run.log` — runtime macro and lane probe.
- `farneback-regression-run.log` — the new gtest regression executed on the board.
- `farneback-submitted-include-red.log` — reproduction of the include failure in the
  earlier submission of this work
  (`'modules/video/src/precomp.hpp' file not found`). Kept as evidence that the
  module-relative include is required.

**2026-10-25 architecture-scope revision**

- `riscv-only-receipt-farneback.json` — full-translation-unit preprocessor output compared
  between baseline and candidate for the three configurations; byte-identical in all
  three. Also records that 26,496 fresh API output records are byte-identical to the
  previously verified candidate. This file is a slice of a receipt that covers four
  directions; only the Farneback entries are kept and the note inside says so.
- `riscv-only-farneback-{rvv-default,rvv-strict,scalar}-correct-candidate.log` — the
  re-run correctness driver lines.

**Windows x64 native build**

- `native-build.log`, `native-receipt.json` — normal CMake Release configuration and build
  of the official targets.
- `native-test-video.log` — the selected regression executed on Windows.
- `native-perf-smoke-video.log` — the new performance cases executed on Windows. The first
  line (`Can't open file: 'video.xml'`) is expected: the `opencv_extra` sanity reference
  data is not present, so `SANITY_CHECK` only records values instead of comparing them.

## Redactions and omissions

- Absolute board paths were replaced with `<board-workdir>` in `receipt.json` and
  `farneback-submitted-include-red.log`. No other content was changed.
- The command history of the harness (`commands.json`, ~470 KB) is not included: it is a
  shell transcript with absolute paths and adds nothing to the numbers above.
- Zero-byte logs from the original package are omitted.
- The other three directions of the same handoff package have their own branches and are
  not part of this PR, so their measurements are not included here.

## Scope

These numbers belong to this board, this compiler, VLEN=256 and the listed inputs. Nothing
is claimed for other VLENs, GCC's RVV backend, NEON/SVE or x86 hardware, and this is not a
full-repository regression.
