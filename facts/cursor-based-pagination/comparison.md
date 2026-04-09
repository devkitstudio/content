## Architectural Trade-offs: Pagination Strategies

Choosing a pagination method dictates your API's performance limits and the consistency of the data presented to the user during concurrent mutations.

| Architecture           | Time Complexity (Deep Pages)  | Data Stability (Concurrent Inserts)              | Navigation Paradigm            | Optimal Use Case                                            |
| :--------------------- | :---------------------------- | :----------------------------------------------- | :----------------------------- | :---------------------------------------------------------- |
| **Offset / Limit**     | **O(N)** (Linear Degradation) | **Poor** (Items shift, causing duplicates/skips) | Random Access (Jump to Page X) | Small datasets, Admin dashboards, SEO-indexed static pages. |
| **ID Cursor**          | **O(1)** (Constant Time)      | **Excellent** (No shifting)                      | Sequential (Next/Prev)         | Massive tables, batch processing, static dataset streaming. |
| **Timestamp Cursor**   | **O(1)** (Constant Time)      | **Excellent** (Chronological)                    | Sequential (Infinite Scroll)   | Social feeds, real-time activity logs.                      |
| **Keyset (Composite)** | **O(1)** (Constant Time)      | **Excellent**                                    | Sequential                     | Complex sorted lists (e.g., sort by Price, then by ID).     |

## Decision Matrix

```text
Do users absolutely need to jump to a specific random page (e.g., Page 45 of 100)?
├─ YES → Is the dataset guaranteed to remain small (< 10k rows)?
│   ├─ YES → Use Offset/Limit.
│   └─ NO → Redesign UX to infinite scroll/search, or combine Offset with deferred joins.
│
└─ NO → Does the data stream sequentially?
    ├─ YES, chronological feeds → Use Timestamp Cursor.
    └─ YES, ordered by complex criteria → Use Keyset Pagination (Composite Cursor).
```
