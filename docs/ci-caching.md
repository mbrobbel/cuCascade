# CI cache baseline

## Existing compiler cache

The build job sets `SCCACHE_GHA_ENABLED=true` and installs
[sccache-action v0.0.10](https://github.com/mozilla-actions/sccache-action/blob/9e7fa8a12102821edf02ca5dbea1acd0f89a2696/src/show_stats.ts).
Its post step already publishes human-readable statistics, JSON, and a job
summary with hits, misses, errors, requests, and cache timing. Additional
instrumentation is not needed to collect a first baseline.

The following samples come from the completed `main` runs
[43197090829](https://github.com/NVIDIA/cuCascade/actions/runs/43197090829)
(2026-11-12) and
[43051524862](https://github.com/NVIDIA/cuCascade/actions/runs/43051524862)
(2026-11-11). Counts are from each job's sccache post-step log. Durations are
GitHub job-step timestamps: the setup-pixi step and the `pixi run ... build`
step, excluding runner queue time and artifact packaging.

| Run | CUDA / RAPIDS | Compile requests | Cache hits / misses / errors | Pixi setup (s) | Build (s) |
| --- | --- | --- | --- | --- | --- |
| 43197090829 | 13 / nightly | 249 | 144 / 0 / 0 | 8 | 243 |
| 43197090829 | 13 / stable | 249 | 144 / 0 / 0 | 8 | 242 |
| 43197090829 | 12 / nightly | 249 | 140 / 0 / 0 | 12 | 254 |
| 43197090829 | 12 / stable | 249 | 140 / 0 / 0 | 15 | 248 |
| 43051524862 | 12 / stable | 249 | 140 / 0 / 0 | 15 | 267 |
| 43051524862 | 13 / stable | 249 | 144 / 0 / 0 | 8 | 244 |
| 43051524862 | 13 / nightly | 249 | 144 / 0 / 0 | 10 | 257 |
| 43051524862 | 12 / nightly | 249 | 140 / 0 / 0 | 14 | 295 |

All eight jobs report 100% hits among cache hits plus misses. Compile requests
include requests that are not cacheable, so dividing hits by all compile
requests would measure something different. Build time still includes
configuration, linking, and other uncached work.

These are cache-warm observations, not a controlled cold/warm experiment. They
do not establish behavior after compiler, dependency, or lockfile changes, or
on PR and merge-group refs. Do not extrapolate a general hit rate from them.

## Pixi environment caching

The workflow explicitly sets `cache: false` in both setup-pixi steps. At the
[pinned v0.9.3 implementation](https://github.com/prefix-dev/setup-pixi/blob/82d477f15f3a381dbcc8adc1206ce643fe110fb7/src/cache.ts),
project caching saves `.pixi`; its key includes the lockfile contents,
environment list, Pixi binary, lockfile path, working directory, and conda
architecture. It restores exact keys rather than a broad fallback. The
[action inputs](https://github.com/prefix-dev/setup-pixi/blob/82d477f15f3a381dbcc8adc1206ce643fe110fb7/action.yml)
also expose `cache-write` for read-only restores.

Measured setup takes 8-15 seconds, compared with 242-295 seconds for the build
step. Environment cache restore/save overhead could erase that small saving.
The sample contains build jobs only; GPU test-job setup must also be measured.

Keep Pixi environment caching disabled until a controlled comparison measures:

- Cold and warm setup time, cache download/upload time, and cache size for each
  CUDA/RAPIDS environment on both CPU and GPU runners.
- Total job time and runner queue time, rather than setup time alone.
- A lockfile change and a Pixi version change, to confirm invalidation.
- Cache reuse and write permissions across `main`, copied-PR, and merge-group
  refs. Preserve the existing contribution trust boundary.

For sccache, collect the existing post-step statistics after a source edit and
after a dependency/compiler change before changing cache keys or compiler
settings. This baseline leaves all cache configuration unchanged.
