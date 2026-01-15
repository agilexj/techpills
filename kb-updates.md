## What it does (one-liner)

**It converts a KB (a generic record containing multiple tables) into a map of tables where each table’s rows are keyed by their primary key, optionally evicting expired tombstone rows, while preserving the last-seen order for each primary key.**

***

## Inputs & Output

**Inputs**

*   `kb`: a `GenericRecord` acting like a container (by table name) ⇒ each entry is a `List<GenericRecord>` (rows).
*   `filteredTableNames`: the subset of table names to process.
*   `pkColumnsByTable`: table → list of column names that form the primary key.
*   `evictExpiredTombstones`: whether to drop tombstone rows that are out of retention.

**Output**

*   `Map<String, LinkedHashMap<GenericPK, GenericRecord>>`
    *   For each table name, you get a `LinkedHashMap` where:
        *   **key** = the computed `GenericPK` (from the PK columns).
        *   **value** = the last kept row for that PK.
    *   The whole map is wrapped with `Collections.unmodifiableMap(...)`.

***

## Step-by-step logic

1.  **Initialize accounting & result container**
    ```java
    evictedCount = 0
    keyedKb = new HashMap<>()
    ```

2.  **For each table in `filteredTableNames`:**
    *   Fetch the table’s PK definition: `kbTablePK = pkColumnsByTable.get(kbTableName)`.
    *   Fetch the table’s rows from `kb`: `kbTableRows = (List<GenericRecord>) kb.get(kbTableName)`.
    *   Create an **order-preserving** map: `keyedKbTableRows = new LinkedHashMap<>()`.

3.  **For each row in the table:**
    *   Compute its primary key:
        ```java
        GenericPK pk = extractPrimaryKey(kbTableRow, kbTablePK)
        ```
    *   If **tombstone eviction is enabled** AND the row is **marked deleted** (`IS_DELETED`) AND the tombstone is **outside retention** (`!isTombstoneInRetention(row)`):
        *   Increment `evictedCount`.
        *   **Remove** any existing entry for this PK from `keyedKbTableRows` (ensuring it won’t appear).
    *   Else (normal row or in-retention tombstone):
        *   First **remove** any existing entry for this PK (this is a trick to **refresh its insertion order**).
        *   Then **put** the row back (now it’s the **last** for that PK).

4.  **Store the table result** into the outer map:
    ```java
    keyedKb.put(kbTableName, keyedKbTableRows)
    ```

5.  **If any tombstones were evicted**, report metrics:
    ```java
    appReporter.reportEvictedTombstones((GenericRecord) kb.get("key"), evictedCount.get());
    ```

6.  **Return an unmodifiable view** of the outer map.

***

## Key behaviors and design choices

*   **Order preservation / “last write wins” per PK**
    *   Uses `LinkedHashMap` to preserve insertion order.
    *   The pattern `remove(pk)` then `put(pk, row)` forces the entry to move to the **end**, so the **last processed row for a PK** becomes the one represented in both value and order.

*   **Tombstone handling**
    
    _What is a [tombstone](tombstone.md)?_

    *   A row with `IS_DELETED=true` is a tombstone.
    *   If `evictExpiredTombstones=true` and `!isTombstoneInRetention(row)`, that tombstone is **dropped** (and any prior value under the same PK is removed), effectively **hiding** that key from the final map if nothing else reintroduces it later.
    *   If the tombstone **is within retention**, it’s **kept** (and will appear keyed by its PK).

*   **Metric reporting**
    *   Only reports if at least one tombstone was evicted.
    *   Uses `kb.get("key")` (likely an envelope key for the KB) for context in the report.

*   **Immutability**
    *   The returned outer map is unmodifiable to prevent accidental mutation downstream (note: the inner `LinkedHashMap`s themselves are **not** wrapped—if consumers get references, they can still mutate per-table maps unless you add another wrapper).

***

## Complexity

*   Let **N** be the total number of rows across all processed tables.
*   **Time:** O(N) on average (hash operations per row; remove+put per processed row).
*   **Space:** O(N) for the keyed maps.

***

## Small example

Suppose table `users` has PK = `[id]`, and the rows arrive as:

1.  `{id=1, name="Alice"}`
2.  `{id=2, name="Bob"}`
3.  `{id=1, name="Alice V2"}`

Result map for `users`:

*   Keys in order: `1, 2` becomes `2, 1` because step 3 “refreshes” `1`.
*   Values:
    *   For id=2: `Bob`
    *   For id=1: `Alice V2`

If in step 3 the row were a **tombstone** and **expired**, id=1 would be **removed** instead of refreshed, leaving only id=2.

***

## Potential pitfalls / things to watch

*   **Inner maps are mutable.** If you need full immutability, wrap the `LinkedHashMap` with `Collections.unmodifiableMap(...)` before putting it into `keyedKb`.
*   **Assumes `kb.get(kbTableName)` returns a `List<GenericRecord>`.** If `kb` can lack a table or return `null`, add null checks.
*   **Type casting safety.** The raw cast to `(List<GenericRecord>)` is unchecked—consider generics or validation.
*   **`IS_DELETED` field presence and type.** Assumes it exists and is a `boolean`. If not guaranteed, guard and/or coerce.
*   **Retention semantics.** The behavior heavily depends on `isTombstoneInRetention(kbTableRow)`. Make sure clock/timezone sources are consistent and deterministic.
*   **Duplicate rows for same PK.** Only the **last processed** one survives (unless evicted). This is intended.

***

## TL;DR

*   Builds per-table, PK‑keyed, **order‑preserving** maps.
*   Keeps the **last** row per PK by remove+put.
*   Optionally **drops expired tombstones** and reports how many were evicted.
*   Returns the result as an **unmodifiable outer map**.

***

Here’s what that method is doing in simple terms:

***

### **Purpose**

It merges **one update row** (from a KB update) into an existing **mutable table map** keyed by primary key (PK).  
The table map is **mutable for performance**, meaning it’s updated in place rather than creating a new copy.

***

### **Inputs**

*   `keyedKbUpdateTableRow`: a single `(PK → row)` pair from the update.
*   `keyedMutableKbTable`: the current table state, keyed by PK.

***

### **Step-by-step logic**

1.  **Extract metadata from the update row**
    *   `kbUpdateTableRowPK`: the primary key.
    *   `kbUpdateTableRow_opTs`: operation timestamp (microseconds).
    *   `kbUpdateTableRow_pos`: position string (used for ordering when timestamps tie).
    *   `kbUpdateTableRow_isDeleted`: whether this row is a tombstone (delete marker).

2.  **Check if there’s an existing row for the same PK**
    ```java
    maybeExistingKbTableRow = keyedMutableKbTable.get(kbUpdateTableRowPK)
    ```
    If present:
    *   Compare timestamps (`op_ts`) and positions (`pos`):
        *   If update has **greater timestamp**, it wins.
        *   If timestamps are equal, compare `pos` lexicographically; if update’s `pos` ≥ existing, update wins.
    *   If update wins → mark existing as stale and remove it.
    *   If update loses → **return early** (do nothing).

3.  **Merge the update row**
    *   If `isDeleted`:
        *   If tombstone is **within retention**, insert it (so consumers know the delete happened).
        *   If tombstone is **expired**, skip it (don’t store).
    *   Else (normal row):
        *   Insert or replace the existing row (UPSERT).

***

### **Key concepts**

*   **Conflict resolution**: Uses `op_ts` (timestamp) and `pos` to decide which row is newer.
*   **Tombstone handling**:
    *   Keeps tombstones only if they’re still in retention.
    *   Drops expired tombstones to save space.
*   **Mutable map**: Directly updates `keyedMutableKbTable` for efficiency.

***

### **Why timestamp + pos?**

*   `op_ts` ensures chronological ordering.
*   `pos` breaks ties when two operations have the same timestamp (e.g., ephemeral operations).

***

### **In short**

This method ensures:

*   The latest valid row per PK is kept.
*   Deletes are represented as tombstones (but only for a retention window).
*   Old/stale data is removed.

***
