# B-Tree Node Structure and On-Disk Storage

## 1. Introduction

A B-Tree is a balanced search tree designed for efficient storage and retrieval of data from secondary storage such as HDDs and SSDs.

Unlike an ordinary Binary Search Tree, a B-Tree node can contain multiple keys and child pointers. This allows each node to have a high fanout, reducing the tree height and the number of page accesses required during searches.

In disk-based databases, B-Tree nodes are commonly organized into fixed-size pages. For simplicity, assume that each B-Tree node occupies exactly one page.

### Core Idea
* A B-Tree consists of multiple nodes.
* Each node is represented by a page on disk.
* Each page contains multiple cells.
* Cells store keys and associated pointers.
* Page identifiers connect different nodes.
* Cell offsets locate individual cells within a page.

---

## 2. Logical Structure of a B-Tree

Consider the following example:

```text
                  [20 | 40]
                 /    |    \
                /     |     \
          [5 | 10] [25 | 30] [50 | 60]
The root node contains two keys: 20 and 40. These keys divide the search space into three ranges:

First child: Keys smaller than 20.

Second child: Keys between 20 and 40.

Third child: Keys greater than 40.

Each node contains multiple keys and references to its child nodes. Unlike a Binary Search Tree, where a node generally contains one key and at most two children, a B-Tree node can contain many keys and child references.

3. How a B-Tree Node Is Organized Inside a Page
A B-Tree node is stored as a page containing three major components:

Page Header

Cell Pointer Block

Cells

3.1 Page Header
The page header contains metadata required to manage the page. Typical information includes:

Page type (internal node or leaf node).

Number of cells in the page.

Free-space boundaries.

Page-related metadata.

(The exact header fields depend on the storage-engine implementation.)

3.2 Cell Pointer Block
The cell pointer block contains offsets pointing to individual cells within the page. It does not store the complete cell data. Instead, it maintains the locations of cells.

For example:

Cell Pointer	Offset
Slot 0	400
Slot 1	300
Slot 2	200
These offsets are relative to the beginning of the page.

3.3 Cells
Cells contain the actual B-Tree entries:

For an internal node, a cell can contain:

Search key.

Child page identifier.

For a leaf node, a cell can contain:

Search key.

Associated record or record reference.

The precise representation depends on the B-Tree variant and storage-engine design.

4. Physical Page Layout
Consider a page size of 4096 bytes (4 KB). A conceptual page layout looks like this:

Plaintext
+----------------------------------------+
|               Page Header              |
|----------------------------------------|
|              Cell Pointer 0            |
|----------------------------------------|
|              Cell Pointer 1            |
|----------------------------------------|
|              Cell Pointer 2            |
|----------------------------------------|
|                                        |
|               Free Space               |
|                                        |
|----------------------------------------|
|                Cell 2                  |
|----------------------------------------|
|                Cell 1                  |
|----------------------------------------|
|                Cell 0                  |
+----------------------------------------+
The header and cell pointer block occupy space near the beginning of the page. Cell data is allocated from the opposite end, generally growing toward the beginning. This arrangement allows the page to accommodate cells of different sizes while maintaining a separate directory of their locations.

5. Example: Storing a B-Tree on Disk
Consider the following B-Tree:

Plaintext
                    [20 | 40]
                   /    |    \
                  /     |     \
            [5 | 10] [25 | 30] [50 | 60]
Assume the database uses a 4 KB page size, and each node occupies one page. For illustration, assign page identifiers:

Node	Page ID	Node Type
Root [20, 40]	10	Internal
[5, 10]	20	Leaf
[25, 30]	30	Leaf
[50, 60]	40	Leaf
5.1 Root Page (Page ID: 10)
The root page stores the separator keys and references to its child pages.

Logical representation:

Plaintext
Page ID: 10
Node Type: Internal

Keys: [20, 40]

Child References:
P0 → Page 20
P1 → Page 30
P2 → Page 40
Internal node logical structure:

Plaintext
          [20 | 40]
          /   |   \
         /    |    \
       P0     P1    P2
Its cells contain separator keys and child references, while the first child reference is represented separately in this simplified example.

5.2 Leaf Page (Page ID: 20)
This page stores the keys [5, 10].

Plaintext
Page ID: 20
Node Type: Leaf

Cells:
  Key 5  → Record Reference A
  Key 10 → Record Reference B
5.3 Leaf Page (Page ID: 30)
Plaintext
Page ID: 30
Node Type: Leaf

Cells:
  Key 25 → Record Reference C
  Key 30 → Record Reference D
5.4 Leaf Page (Page ID: 40)
Plaintext
Page ID: 40
Node Type: Leaf

Cells:
  Key 50 → Record Reference E
  Key 60 → Record Reference F
6. Understanding the Two Levels of Pointers
A B-Tree uses references to navigate between pages and locate entries inside pages.

6.1 Page Identifiers
A page identifier identifies another page in the B-Tree file.

Plaintext
Root Page 10
     |
     | Child Page ID = 30
     |
     v
Leaf Page 30
The database uses the page identifier to locate and read the child page. Page IDs are logical identifiers within the storage structure; they are not necessarily raw physical disk addresses.

6.2 Cell Offsets
Cell offsets identify where a particular cell starts within the current page.

Suppose Page 10 has the following cell pointer block:

Cell Pointer	Offset	Cell Content
Slot 0	400	Key 20
Slot 1	300	Key 40
To access the cell containing key 40:

Plaintext
Page 10
   |
   v
Cell Pointer Slot 1
   |
   v
Offset 300
   |
   v
Cell containing Key 40
The offset is relative to the beginning of Page 10.

Important Distinction:

Page ID locates a page within the tree file.

Cell offset locates a cell within that page.

7. Complete Search Example: Searching for Key 30
Suppose we want to find key 30.

Step 1: Read the Root Page
The database starts with the root page:Page 10 
→
→ [20 | 40]

It examines the keys and identifies the correct child range. Since 
20
<
30
<
40
20<30<40, it follows the second child reference:Page ID = 30

Step 2: Read the Child Page
The database reads Page 30:Page 30 
→
→ [25 | 30]

Step 3: Locate the Cell
The database examines the cell pointer block to locate the cell containing key 30:

Plaintext
Page 30
   |
   v
Cell Pointer
   |
   v
Cell Offset
   |
   v
Key 30
The search is successful.

Complete Navigation
Plaintext
          Root Page 10
             [20 | 40]
                 |
                 | Page ID = 30
                 v
          Leaf Page 30
             [25 | 30]
                 |
                 | Cell Offset
                 v
              Key 30
The search involves reading the root page and the appropriate leaf page, assuming neither page is already cached in memory.

8. How the Database Reads and Writes These Pages
The database does not generally access each key as an independent disk operation. Instead, it works with pages.

Reading
The database receives a search request.

It identifies the root page.

It reads the root page into memory if it is not already cached.

It examines keys and child page identifiers.

It follows the appropriate child page.

It repeats the process until the target leaf page is reached.

It uses cell offsets to locate the required entry.

Writing
When a B-Tree node is modified:

The database modifies the relevant page in memory.

It updates keys, cell pointers, or child references.

If required, it performs a split, merge, or redistribution.

It marks the page as modified.

The storage engine persists the modified page using its recovery and write-management mechanisms.

(The exact write process depends on the database, including its buffer manager, write-ahead logging, and recovery strategy.)

9. Why This Layout Is Efficient
Design Feature	Benefit
Multiple keys per page	High fanout
Fixed-size pages	Efficient page-oriented I/O
Cell pointer block	Locates entries without requiring sequential cell storage
Page identifiers	Connect nodes across the tree
Cell offsets	Locate cells within a page
Variable-size cells	Support different key and record sizes
Balanced tree structure	Keeps search depth low
The main performance advantage comes from reducing the number of page accesses required to locate a record.

10. Key Takeaways
A B-Tree is a hierarchy of nodes, commonly stored as disk pages.

Each page contains a header, cell pointer block, and cells.

Internal-node cells contain separator keys and child-page references.

Leaf-node cells contain keys and associated records or record references.

Page identifiers locate child pages in the tree file.

Cell offsets locate individual cells inside a page.

The database reads pages rather than performing independent disk reads for every key.

High fanout reduces tree height and therefore reduces the number of page accesses during search.

Core Idea: A disk-based B-Tree is not merely a logical tree of keys and pointers. It is a hierarchy of physical pages, where page identifiers connect nodes and cell offsets locate individual entries within each page.