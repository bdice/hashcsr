# HashCSR

A fifteen-slide animated explanation of cuDF’s HashCSR joins and groupby. Follow six input rows through two implementations of the count–scan–place technique. Join emits matching row pairs; groupby aggregates values within each group.

## Usage

View the [presentation on GitHub Pages](https://bdice.github.io/hashcsr/).

For offline use, download [`index.html`](index.html) and open it in a browser. No dependencies, server, or network access are needed.

- Use **Back / Next** or **← / →** to change slides.
- Select a labeled timeline step to display its slide. Clicking the slide itself does not navigate.
- **Home / End** display the first / last slide.
- Each scene’s **source citation** opens its supporting source and explanation.
- **Notes & sources** in the footer includes the bibliography, example, and implementation details.

Motion runs only when you navigate. System reduced-motion preferences are respected. Thrust-style pseudocode and device-buffer sizes appear beside the diagrams on wide displays and below the captions on narrower ones. Shorter displays can scroll vertically.

## Implementation

The join sequence follows the packed HashCSR design in [cuDF PR #24231](https://github.com/NVIDIA/cudf/pull/24231), revision `c118ec760e6d`, with capacity sizing and starting slots from [cuDF PR #24331](https://github.com/NVIDIA/cudf/pull/24331). Fixed illustrative 32-bit hashes preserve the example’s probe collision. The diagrams abbreviate each fingerprint to its first and last three bits; the hidden middle bits may contain both zeros and ones. They are not cuDF’s actual hash outputs. GPU insertion winners and within-group ordering are not deterministic.

Groupby has its own build: plain representative-row slots, saved per-row ranks, compact group IDs, and dense CSR offsets. The example uses the small-input, representative-counting path and the small-group SUM reduction.

## Research reference

Oded Green. 2021. *HashGraph—Scalable Hash Tables Using a Sparse Graph Data Structure.* ACM Transactions on Parallel Computing 8(2), 1–17. [doi:10.1145/3460872](https://dl.acm.org/doi/10.1145/3460872).

The presentation includes short quotations supporting CSR ranges, counting, prefix sums, contiguous placement, and probing. Apart from the published abstract excerpt, quotations and section/page locators are explicitly from the [author’s open 2019 preprint, arXiv:1907.02900v1](https://arxiv.org/abs/1907.02900v1). The quotations are embedded for offline reading.

HashGraph groups by hash bucket, including collisions; the illustrated HashCSR variant separates equal-key groups using an open-addressed directory. Citations distinguish the paper’s count–scan–place principle from cuDF-specific packing, cursor reuse, and kernel fusion. The groupby build and SUM reduction use cuDF-specific kernels; Green supplies the underlying CSR connection.

## Capacity comes first

For join, the host sizes the directory using the build-row count `N` (including duplicates) and requested load factor `α`, which defaults to `0.5`:

- `R = max(N + 1, ceil(N / α))`.
- Capacity is `R`. The starting slot is `__umulhi(__brev(hash), capacity)`: reverse the hash’s bits, multiply by capacity, and keep the high 32 bits, giving a slot in `[0, capacity)`.
- Six rows at `α = 0.5` therefore get **12 slots**. The `N + 1` safeguard leaves an empty slot even at `α = 1`.

This uses the row count, not a distinct-count pass. Our three groups ultimately occupy 25% of those slots. Capacity controls the range of starting slots and the allocation; the **3 row bits** still come from `bit_width(6)`, not the 12-slot capacity. The Size slide cites the pinned cuDF sizing implementation.

Groupby also reserves 12 slots for this six-row input. At `n ≥ 2²¹`, groupby samples distinct keys to size the table; bounded probing triggers a full-capacity retry if the estimate is insufficient. Notes & sources describes the alternative counting and singleton paths.

## Kernel boundaries

The timeline has separate join and groupby routes. Slides 2–6 explain the join build; slides 7–8 separate match counting/output sizing from output-slot retrieval. Slides 9–13 follow groupby through build/rank, compaction, scan, fill, and reduction. Slide 14 uses a separate illustrative workload to show size-based reduction dispatch.

- **Join build:** build kernel (hash + insert + count) → device scan → fill kernel (find representative + decrement + scatter). Hash and Count show different parts of the same build kernel.
- **Join consumption:** probe kernel (hash + lookup + count) → device scan → retrieve kernel (one pair per output slot).
- **Groupby build:** build kernel (insert + count + rank) → device algorithms (compact + scan + scatter starts) → fill kernel (group start + saved rank).
- **Groupby consumption:** segment-size scheduling → reduction (indirect payload loads + SUM) → gather representative keys. The six-row example takes the one-thread-per-group path; the alternate workload shows a one-warp group and a long group with two block-reduction passes. Int32 payloads produce int64 sums.

Host setup is unboxed. Outlined boxes show GPU stages. Kernels are labeled explicitly; device algorithms may launch several kernels. Notes & sources maps the pseudocode to pinned implementation sources and explains shorthand and initialization.

## Device memory

Each algorithm slide defines its variables above symbolic buffer sizes and sums the listed live **temporary** storage at the bottom. Here temporary means any allocation that is **not returned to the caller**: join slots, offsets, and grouped row IDs count even while they remain live across slides. The returned pair columns or groupby keys and sums do not. `n` denotes build rows, `m` probe rows, `J` join outputs, `G` groups, and `cap` directory slots.

The example uses non-null int32 keys and payloads. Join counts, ends, and starts reuse one offsets allocation; its scan workspace is released before allocating grouped row IDs. Groupby’s directory is released after its build on this small-input path. Its `group_slots` holds `G` selected entries but retains capacity for `n` until released before fill; counts are reused as group starts. Each saved `position` stores a uint32 index and int32 rank. The panels show the storage live at each labeled stage rather than adding allocations from different moments together. The dispatch slide instead uses group sizes 3, 64, and 2050 (n=2117, G=3) and counts scheduling arrays and int64 chunk partials live during its reduction. Transient sort scratch has already been released.

Library workspaces are shown symbolically because their sizes depend on the device and library version. These figures cover the listed buffers rather than total GPU memory: input storage, preprocessing/descriptors, allocator overhead and reserved pool memory are outside the accounting. Notes & sources includes lifetime details and the extra allocations used by larger groups or nullable inputs.

All SVG artwork, typography, styles, and JavaScript are embedded in the HTML. Copy or share that file on its own.
