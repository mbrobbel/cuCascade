# Multi-GPU test coverage

The CI test matrix runs on `linux-amd64-gpu-t4-latest-1`. A green run on that
single-GPU runner does not establish multi-GPU coverage. The inventory below
describes the existing source tests; it is not a record of a multi-GPU test run.

## Existing tests

| Executable / source | Coverage | Selection and hardware requirements |
| --- | --- | --- |
| `cucascade_cudf_tests`: [GPU-to-GPU converter](../test/cudf/test_gpu_to_gpu_converter.cpp) | Copies mixed table content from device 0 to device 1 while honoring the producer stream, with two concurrent readers. | `[cross-gpu-correctness]`; selected by the default run. Skips with fewer than two GPUs, unsupported pool peer access, or failed peer-DMA verification. |
| `cucascade_cudf_tests`: [table stream ownership](../test/cudf/test_release_table_stream.cpp) | Rejects streams belonging to another GPU and operates correctly when the representation's device is not current. | `[multi-device]`; two tests selected by the default run. Require CUDA devices 0 and 1; skip with fewer than two devices. The separate mocked device-mismatch test also runs on single-GPU hosts. |
| `cucascade_cudf_tests`: [cross-device representation conversion](../test/data/test_data_representation.cpp) | Copies patterned numeric columns in both directions, with and without pool peer grants; also checks device 1 identity in a section of the ordinary device-ID test. | Cross-device copying is hidden `[.multi-device]` and skips without two GPUs or, for the granted-pool sections, a usable grant. The visible device-ID test has no `multi-device` tag; it uses `SUCCEED` and returns from its device 1 section when hardware is absent. |
| `cucascade_cudf_tests`: [reader-event replacement](../test/data/test_data_batch.cpp) | Reader event pools remain device-local when a batch replaces a GPU 0 representation with one on GPU 1. | Hidden `[.multi-device]`; uses `SUCCEED` and returns without two available CUDA devices. |
| `cucascade_tests`: [reservation manager](../test/memory/test_memory_reservation_manager.cpp) | Falls back from a full reservation space on GPU 0 to GPU 1. | Hidden `[.multi-device]`; explicitly select `[multi-device]`. Configures two GPU spaces with 2 GiB budgets and host spaces; has no insufficient-device skip guard. |
| `cucascade_tests`: [pool peer access](../test/memory/test_pool_peer_access.cpp) | Grants are scoped to a pool, repeatable, and support self-access. | Hidden `[.multi-device]`; requires devices 0 and 1 with bidirectional peer capability. Skips when CUDA/devices/peer capability are unavailable. |
| `cucascade_tests`: [peer-access regressions](../test/memory/test_pool_peer_access_regression.cpp) | Private-pool probes, caller-device restoration, stream isolation, ordinary peer state, pool permissions, and actual GPU kernel copies through granted pools. | Hidden `[.multi-device]`; requires devices 0 and 1 with bidirectional peer capability. Skips without that capability. Working-route cases assert verification and byte-copy results. |
| `cucascade_topology_discovery_tests`: [CUDA visibility](../test/memory/test_topology_discovery.cpp) | Default discovery must not initialize CUDA while visibility is temporarily widened. | `[cuda-context]`; selected by the default run. Needs NVML and at least two physical GPUs. The child returns 77 without the hardware, and the parent reports `SUCCEED("Skipped: ...")`, not a Catch2 skip. |

The peer-regression file also has `[.peer_route_broken]`, a hidden test intended
only for a machine with a known corrupt direct-peer route. Do not include it in
the healthy-peer CI lane. Several ordinary peer tests use fake CUDA operations;
passing those tests does not demonstrate communication between real GPUs.

The hidden `[.bandwidth_matrix]` test in
[the bandwidth profiler](../test/data/test_bandwidth_profiler.cpp) can discover
extra GPUs for a diagnostic transfer matrix. It also touches host and disk tiers;
it is not a replacement for multi-GPU correctness assertions.

## Explicit selection

After building with cuDF enabled, run from the repository root on a host exposing
at least devices 0 and 1 with working bidirectional peer access:

```bash
build/release/test/cucascade_tests '[multi-device]'
build/release/test/cucascade_cudf_tests '[multi-device]'
build/release/test/cucascade_cudf_tests 'gpu_table_representation device_id'
build/release/test/cucascade_topology_discovery_tests '[cuda-context]'
```

Catch2's `[.multi-device]` hides cases from an unfiltered run. A positive
`[multi-device]` filter explicitly includes them. The
[CTest registration](../test/CMakeLists.txt) invokes each test executable without
a filter, so adding a second GPU to the current CTest job alone would still
leave the hidden cases unexecuted.

## Requirements for a CI lane

- Confirm a runner label with two GPUs, sufficient memory, and working
  bidirectional peer capability. GPU count alone does not certify peer DMA.
- Record GPU models, driver version, `CUDA_VISIBLE_DEVICES`, and topology. Check
  CUDA-visible devices in the test process's environment before running tests.
- Explicitly select `[multi-device]` in both `cucascade_tests` and
  `cucascade_cudf_tests` to include their hidden cases, plus the visible topology
  and device-ID cases. Keep the ordinary single-GPU matrix.
- Require at least one selected test and fail when required hardware or a
  required peer route is unavailable. Catch2 skips and the topology test's
  success-with-a-skip-message must not silently qualify the lane.
- Preserve per-case reports. Current CTest registration aggregates each entire
  executable into one result, so a passing CTest entry does not prove that every
  Catch2 case ran.

## Gaps and decisions for the CI implementation

The mixed-table converter test covers 0-to-1 copies; the numeric representation
test covers both directions. Neither enumerates every GPU pair on larger hosts.
The suite also lacks consistent tagging and
skip reporting across its hardware tests. GPU and peer-query failures can be
reported as skips in some paths, which makes a separate hardware preflight
necessary for a required lane.

Runner availability, the CUDA/RAPIDS configurations, and whether the lane runs on
PR updates, merge groups, or a schedule remain decisions for
[the CI implementation task](https://github.com/NVIDIA/cuCascade/issues/271).
This inventory does not enable a runner or change test behavior.
