# HashCSR

A nine-slide animated explanation of HashCSR joins and groupby. The example constructs a hash directory and CSR segments from six input rows. Join emits matching row pairs; groupby aggregates values within each group.

## Usage

View the [presentation on GitHub Pages](https://bdice.github.io/hashcsr/).

For offline use, download [`index.html`](index.html) and open it in a browser. No dependencies, server, or network access are needed.

- Use **Back / Next** or **← / →** to change slides.
- Select a labeled timeline step to display its slide. Clicking the slide itself does not navigate.
- **Home / End** display the first / last slide.
- Each scene’s **source citation** opens the relevant quotation and explanation.
- **Notes & sources** in the footer includes the bibliography, example, and implementation details.

Motion runs only when you navigate. System reduced-motion preferences are respected. A desktop or landscape display gives the diagrams the most room.

## Implementation

The join sequence follows the packed HashCSR design in [cuDF PR #24231](https://github.com/NVIDIA/cudf/pull/24231), revision `c118ec760e6d`. Fixed illustrative 32-bit hashes preserve the example’s probe collision. The diagrams abbreviate each fingerprint to its first and last three bits; the hidden middle bits may contain both zeros and ones. They are not cuDF’s actual hash outputs. GPU insertion winners and within-group ordering are not deterministic.

## Research reference

Oded Green. 2021. *HashGraph—Scalable Hash Tables Using a Sparse Graph Data Structure.* ACM Transactions on Parallel Computing 8(2), 1–17. [doi:10.1145/3460872](https://dl.acm.org/doi/10.1145/3460872).

The presentation includes short quotations supporting CSR ranges, counting, prefix sums, contiguous placement, and probing. Apart from the published abstract excerpt, quotations and section/page locators are explicitly from the [author’s open 2019 preprint, arXiv:1907.02900v1](https://arxiv.org/abs/1907.02900v1). The quotations are embedded for offline reading.

HashGraph groups by hash bucket, including collisions; the illustrated HashCSR variant separates equal-key groups using an open-addressed directory. Citations distinguish the paper’s count–scan–place principle from cuDF-specific packing, cursor reuse, and kernel fusion. The groupby reduction is a worked use of the CSR ranges, with SUM as the example aggregation, not a groupby kernel quoted from the paper.

## Capacity comes first

The host sizes the directory using the build-row count `N` (including duplicates) and requested load factor `α`, which defaults to `0.5`:

- `R = max(N + 1, ceil(N / α))`.
- For `α ≤ 0.5`, capacity is `R`; for `α > 0.5`, round `R` up to the next power of two (leaving it unchanged if already a power of two).
- Six rows at `α = 0.5` therefore get **12 slots**, not 16. The `N + 1` safeguard leaves an empty slot even at `α = 1`.

This uses the row count, not a distinct-count pass. Our three groups ultimately occupy 25% of those slots. Capacity controls the hash modulus and allocation; the **3 row bits** still come from `bit_width(6)`, not the 12-slot capacity. The Size slide cites the pinned cuDF sizing implementation.

## Kernel boundaries

Presentation slides are not kernel launches. Hash insertion and counting share **one fused build kernel**. They are shown on separate slides and grouped together in the timeline. Both operations share this construction:

**Build kernel (hash + insert + count) → device scan → fill kernel (find representative + decrement + scatter)**

The completed CSR is consumed independently by:

- **Join:** probe kernel (hash + lookup + count) → device scan → retrieve kernel (emit pairs).
- **Groupby:** reduction kernel (indirect payload loads + reduction), with no separate gather kernel. The worked example uses SUM.

Host setup is unboxed and is not a GPU kernel. Outlined boxes show GPU stages. Kernels are labeled explicitly; a device scan may launch multiple kernels. Initialization and allocation plumbing are omitted. Notes & sources includes the corresponding join kernel names.

All SVG artwork, typography, styles, and JavaScript are embedded in the HTML. Copy or share that file on its own.
