# Raw measurement data — opencv/opencv ORB orientation PR

Evidence for the RVV `ICAngles` (intensity-centroid) patch in
`modules/features/src/orb.cpp`.

## What was measured

| Item | Value |
| --- | --- |
| Baseline commit | `f1c1be300d10b39834fd307f23e5c2d020313a31` (5.x) |
| `modules/features/src/orb.cpp` SHA-256 | `1714ce04eb17aee22d1b80dcb8fdd6d6a3d6f5b489aa8d3520440de428b32acc` |
| Board | BPI-F3, SpacemiT X60, riscv64 |
| Compiler | Ubuntu clang 17.0.2, `-std=c++17 -O3`, target `riscv64-unknown-linux-gnu` |
| Vector config | `CV_RVV=1`, `CV_SIMD_SCALABLE=1`, 8 x 32-bit lanes, VLEN=256 |
| Threads / OpenCL | 1 / off, CPU affinity pinned to core 2 |

The SHA-256 above is the file in the PR. If that file changes, these numbers no longer
apply.

## Method

Three configurations were built from the same headers at the baseline commit:

1. `rvv-default` — the compiler's normal RVV settings.
2. `rvv-strict` — `-ffp-contract=off`, so no implicit float contraction; the explicit
   `v_fma` intrinsics are unaffected by this flag.
3. `scalar` — no vectorisation at all; used to prove the new path is not compiled in.

For each configuration the baseline and candidate objects were linked against the same
prebuilt support libraries (`libopencv_core/imgproc/features/flann/...`), then driven
through the public `cv::ORB` API and compared output record by record.

ORB angles are integers after `fastAtan2`, so the requirement is **bit-exact** output — no
tolerance is applied. Inputs are compared before and after the call, and sentinels outside
(and between rows of) the ROI are checked to catch out-of-bounds writes.

Coverage: several patch sizes, pyramid levels, WTA_K values, random / flat / checkerboard
images, and ROIs with padding. 216 API records and 1,147,497 compared values per
configuration.

## Performance method

Seven alternating rounds (baseline, candidate, baseline, ...). The reported number is the
median per-call time; batches are calibrated to about 10 ms after warm-up. Timing is on
the board only — transfer, compilation and SSH time are excluded. A ratio above 1.0 means
the candidate is faster.

Two different things are measured, and they must not be confused:

- **`orb-*` / `orb-performance.json`** — end-to-end `ORB::detectAndCompute` through the
  public API. This is the realistic number: FAST and descriptor generation dominate, so
  the gain is ~1.00x.
- **`kernel-*` / `kernel-performance.json`** — the centroid kernel extracted verbatim from
  the final sources and called directly. This bypasses the `2 * half_k + 1 >= vlanes`
  dispatch gate, so it reports what the vector helper can do, not what ships for small
  patches. Patches 3 and 32 show 0.97x–0.99x here and are reported as measured.

## Files

Correctness:

- `orb-rvv-default-comparison.json`, `orb-rvv-strict-comparison.json`,
  `orb-scalar-comparison.json` — records / values / mismatches / max abs error / bit-exact
  flag per configuration.
- `orb-*-correct-candidate.log` — one-line `PASS records=216 ...` from each run.
- `kernel-run-*.log` — `PASS 10548 kernel checks` for the extracted kernel, per mode.
- `orb-regression-run.log` — `Features2D_ORB.orientationCircularMoments` on the board.
- `outputs-sha256.json` — hashes of the per-output dumps kept on the board.

Performance:

- `orb-performance.json`, `orb-bench-{0..6}-{baseline,candidate}.csv` — end-to-end, seven
  rounds plus the per-round medians. Columns: `case, trial, variant, ns_per_call`.
- `kernel-performance.json`, `kernel-performance.csv` — extracted kernel, same layout.

Provenance:

- `receipt.json` — pass flag, base commit, compiler string, affinity, thermal/frequency
  snapshot, source and library hashes.
- `kernel-receipt.json` — hash of the extracted kernel source used for the kernel runs.
- `native-test-features.log`, `native-perf-smoke-features.log` — Windows x64 / MinGW
  (GCC 13.2) CMake Release build: the 4 selected `Features2D_ORB` regression cases
  (including the new `orientationCircularMoments`) and the 8 cases of the new
  `perf_orb_orientation_rvv.cpp`. The perf log starts with a harmless "Can't open
  features2d.xml" line: the sanity reference data from opencv_extra is not available here,
  so the strict sanity check is disabled and values are recorded instead of compared.

## What this does not cover

- The board runs reuse prebuilt support libraries; this is not a full-library RVV rebuild.
- One board, one compiler, VLEN=256, the listed inputs only. Nothing is claimed for GCC
  RVV, other VLENs, NEON/SVE, or a repository-wide regression run.
- Ratios within a few percent of 1.0 are measurement noise and should be read as
  "unchanged".
