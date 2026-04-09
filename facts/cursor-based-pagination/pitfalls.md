## The Traps of Pagination

### 1. The Data Shift Anomaly (Offset Trap)

When using `OFFSET/LIMIT` on a highly active table, concurrent insertions or deletions will shift the dataset boundaries during pagination.

- **The Flaw:** User fetches Page 1 (Items 1-10). Before they fetch Page 2, two new items are inserted at the top. The database shifts all items down. When the user requests Page 2 (`OFFSET 10`), they will see Items 9 and 10 again (Duplicates). If items were deleted, they would permanently miss items (Skips).
- **The Fix:** Cursor pagination inherently solves this because it anchors to a specific item ID/Timestamp, rendering preceding shifts irrelevant.

### 2. The Deterministic Sort Trap

When sorting by a non-unique column (like `created_at` or `price`), standard cursors can break.

- **The Flaw:** If you sort only by `price`, and 30 items have a price of `$10.00`, a limit of 20 will cut the result set in the middle of these identical prices. The next query (`WHERE price > 10.00`) will skip the remaining 10 items entirely.
- **The Fix:** Always append a strict, unique identifier (like the primary key `id`) as a tie-breaker in your `ORDER BY` and your cursor payload (e.g., `ORDER BY price ASC, id ASC`).

### 3. Exposing Database Schema in the Cursor

Never pass `{ "id": 500 }` as a plain JSON or URL parameter.

- **The Flaw:** Clients will begin relying on that structure. If you later decide to migrate from Integer IDs to UUIDs, or change the sort logic to timestamps, you will break mobile apps or third-party integrations that hardcoded your cursor structure.
- **The Fix:** Always encode the cursor (e.g., Base64) to establish an API contract that the cursor is an opaque, un-parseable token.
