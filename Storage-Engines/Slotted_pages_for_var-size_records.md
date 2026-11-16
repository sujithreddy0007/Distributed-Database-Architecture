# Slotted Pages

## 1. Introduction

A Slotted Page is a page organization technique used by database storage engines to efficiently manage records of variable sizes within a fixed-size page.

Instead of storing records and their physical locations together, a slotted page maintains a separate slot directory containing references to the actual records.

This separation allows records to be inserted, deleted, moved, and reorganized without requiring external references to change, provided those references use stable page-and-slot identifiers.

Slotted pages are commonly used in database systems such as PostgreSQL.

### Why are Slotted Pages Needed?
Traditional page layouts that store records sequentially work well when all records have fixed sizes. However, variable-size records introduce several problems:
* **Insertion overhead:** Inserting a record in the middle of a sequential layout may require shifting existing records.
* **Space fragmentation:** Deleting a record can leave unused gaps within the page.
* **Variable-size records:** Fixed-size segments can waste space when record sizes do not align with segment boundaries.
* **Reference management:** Moving records can invalidate references that point directly to their physical offsets.

Slotted pages address these problems by separating record storage from record references.

---

## 2. Internal Page Structure

A slotted page is generally divided into three major regions:
1. **Page Header:** Stores metadata required to manage the page.
2. **Slot Directory:** Maintains entries containing references to records.
3. **Record Storage Area:** Stores the actual records, which can have different sizes.

The slot directory and record storage area generally grow from opposite ends of the page, leaving free space between them.

### Conceptual Layout

```text
+----------------------------------+
|            Page Header           |
|----------------------------------|
|             Slot 0               |
|----------------------------------|
|             Slot 1               |
|----------------------------------|
|             Slot 2               |
|----------------------------------|
|                                  |
|             Free Space           |
|                                  |
|----------------------------------|
|           Record C               |
|----------------------------------|
|           Record B               |
|----------------------------------|
|           Record A               |
+----------------------------------+
Page Header
The page header contains metadata required for page management, such as:

Page identifier or page-related metadata.

Number of slots or item identifiers.

Starting position of the slot directory.

Boundary of free space.

Position where record storage begins.

The exact metadata fields depend on the database implementation.

Slot Directory
The slot directory is an array of entries that reference records stored within the page. A slot entry commonly contains:

Offset: Starting position of the record within the page.

Length: Size of the record.

Status or flags: Information about whether the slot is valid or available.

The slot directory provides an additional level of indirection between a record reference and its physical location.

Record Storage Area
This region contains the actual record data.

Records can have different sizes.

Records do not need to be stored in slot order.

Records can be moved during page reorganization.

Their corresponding slot entries are updated when their locations change.

3. How Slot References Work
Consider a page containing three records.

Slot ID	Record	Offset	Size
0	Record A	4000	40 bytes
1	Record B	3900	60 bytes
2	Record C	3800	30 bytes
These offsets are illustrative positions within a page.

The slot directory maintains the relationship:

Slot 0 → Offset 4000 → Record A

Slot 1 → Offset 3900 → Record B

Slot 2 → Offset 3800 → Record C

To retrieve Record B:Page ID → Slot ID 1 → Offset 3900 → Record B

Instead of directly storing the physical location of a record in every reference, the database uses its slot identifier. This makes record references more stable when records move within the page.

4. Record Insertion
When a new record is inserted:

The database checks whether the page has sufficient free space.

It allocates space for the record in the record storage region.

It writes the record into the allocated space.

It creates or reuses a slot directory entry.

The slot entry is updated with the record's offset and length.

Page metadata is updated to reflect the new free-space boundaries.

The slot directory and record storage area grow toward each other. If sufficient contiguous free space is unavailable, the database may need to compact the page or select another page.

5. Record Deletion and Space Reclamation
When a record is deleted, its slot entry can be marked invalid or made available for reuse, depending on the implementation.

For example, initially:

Slot 0 → Record A (40 bytes)

Slot 1 → Record B (60 bytes)

Slot 2 → Record C (30 bytes)

After deleting Record B:

Slot 0 → Record A (40 bytes)

Slot 1 → Invalid / Available

Slot 2 → Record C (30 bytes)

The space previously occupied by Record B becomes reclaimable. However, deleting the record does not necessarily mean that all free space immediately becomes one contiguous region. This can lead to page fragmentation.

6. Page Compaction
Page Compaction is the process of reorganizing records within a page to consolidate scattered free space into a contiguous region.

Consider the following conceptual example.

Before Compaction
Plaintext
+----------------------------------+
| Record A                         |
|----------------------------------|
| Free Space                       |
|----------------------------------|
| Record C                         |
|----------------------------------|
| Free Space                       |
|----------------------------------|
| Record D                         |
+----------------------------------+
After Compaction
Plaintext
+----------------------------------+
| Record A                         |
|----------------------------------|
| Record C                         |
|----------------------------------|
| Record D                         |
|----------------------------------|
|                                  |
|        Contiguous Free Space     |
|                                  |
+----------------------------------+
Compaction Process
Identify all valid records in the page.

Move or copy records into a compact region.

Update the corresponding slot offsets.

Consolidate scattered free space.

Update page metadata.

The slot identifiers can remain unchanged even when the physical positions of records change. This is one of the major advantages of slotted-page organization.

7. Indirection and Stable Record References
One of the most important properties of slotted pages is indirection. Indirection means accessing a record through an intermediate reference rather than directly using its physical location.

Consider a record initially stored at offset 4000. An external reference identifies it using:Page 5 → Slot 0

After page compaction, the record moves to offset 3500.

Before compaction: Page 5 → Slot 0 → Offset 4000 → Record A

After compaction: Page 5 → Slot 0 → Offset 3500 → Record A

The external reference remains unchanged:Page 5 → Slot 0

Only the offset maintained by the slot directory changes. This avoids updating every external reference when a record moves within the page.

Important: This stability applies when references use page-and-slot identifiers. Direct references to physical offsets would need to be updated.

8. Advantages of Slotted Pages
Advantage	Explanation
Variable-size records	Supports records with different sizes without fixed-size allocation units.
Efficient space utilization	Reduces space wasted by fixed-size segments.
Flexible insertion	Records can be placed wherever suitable space is available.
Simplified deletion	Slots can be invalidated or reused.
Stable references	Record locations can change without changing page-and-slot references.
Space reclamation	Compaction consolidates fragmented free space.
9. Limitations and Trade-offs
Although slotted pages are efficient, they introduce some additional overhead.

Slot directory overhead: Every record requires a corresponding slot entry.

Indirect access: Accessing a record requires resolving its slot to obtain the actual offset.

Compaction cost: Moving records and updating slot offsets requires additional processing and page writes.

Limited page capacity: A page can still become full, requiring records to be stored in another page or additional storage structures.

These trade-offs are generally worthwhile because they simplify variable-size record management.

10. Real-World Implementation: PostgreSQL
PostgreSQL uses a slotted-page organization for its heap pages. Its page structure contains:

Page Header: Stores metadata about the page.

Item Identifier Array: Contains entries that reference tuple locations.

Free Space: Available space between item identifiers and tuple storage.

Tuple Storage: Contains the actual row versions (tuples).

Conceptually:
Plaintext
+----------------------------------+
|            Page Header           |
|----------------------------------|
|          Item Identifier 1       |
|----------------------------------|
|          Item Identifier 2       |
|----------------------------------|
|          Item Identifier 3       |
|----------------------------------|
|                                  |
|             Free Space           |
|                                  |
|----------------------------------|
|             Tuple 3              |
|----------------------------------|
|             Tuple 2              |
|----------------------------------|
|             Tuple 1              |
+----------------------------------+
PostgreSQL uses item identifiers, commonly called line pointers, to locate tuples within a page. A tuple can move during page compaction while its corresponding item identifier continues to reference it. This is particularly useful for maintaining tuple references used by indexes.

11. Related Concepts
Database Pages: Fixed-size storage units used to organize database data.

B-Trees: Index structures whose nodes are commonly organized into pages.

Page Compaction: Reorganizing records to consolidate free space.

Fragmentation: Scattered unused space within a page or storage structure.

Indirection: Accessing records through intermediate references.

MVCC: A concurrency-control technique that maintains multiple row versions, as used by PostgreSQL.

Tuple Identifier (TID): PostgreSQL's physical row identifier, consisting of a block number and an offset number.

12. Key Takeaways
Slotted pages are designed to manage variable-size records efficiently.

A page consists of a header, slot directory, free space, and record storage area.

The slot directory stores references to records rather than the actual records.

Records can move within a page without changing stable page-and-slot references.

Page compaction consolidates fragmented free space.

PostgreSQL uses item identifiers (line pointers) to locate tuples within heap pages.

Core Idea: Slotted pages separate the logical reference to a record from its physical location, making variable-size record management, deletion, insertion, and page reorganization more efficient.