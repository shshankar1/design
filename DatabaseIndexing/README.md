📘 Database Storage & Indexing — Deep Dive

This document explains how databases (with PostgreSQL as the main reference) store data on disk, organize indexes, and achieve sub-millisecond query performance.

1. Storage Fundamentals
   1.1 Tables as Files

Each table in PostgreSQL is a physical file (or set of segment files) on disk:

$PGDATA/base/<db_oid>/<relfilenode>


A table = heap file, which is just an unordered collection of pages.

1.2 Data Blocks / Pages

Page (a.k.a. Block) = fundamental I/O unit.

Default page size in PostgreSQL = 8 KB (BLCKSZ, fixed at build time).

Database always reads/writes entire pages, never partial bytes.

1.3 Page Layout
### PostgreSQL Page Layout (8 KB)
```
+------------------------------+
| Page Header (~24B) | ← metadata about the page
+------------------------------+
| Line Pointer Array (4B each) | ← slot directory of rows
+------------------------------+
| Free Space |
+------------------------------+
| Heap Tuples (rows) |
+------------------------------+
| Special Space (indexes only) |
+------------------------------+
```

Tuple (row) = heap tuple header (~23B) + column data.

Each row is addressed via a TID = (block#, offset#).

2. Caching Layers

Databases achieve microsecond latencies with multi-level caching:

OS Page Cache

Recently accessed disk blocks are cached by the OS.

DB Buffer Pool

Postgres maintains its own cache of 8KB pages in shared memory.

Hot pages (root index, frequently queried rows) almost always live here.

Query Execution Caches

Execution plans and visibility info are cached for faster re-runs.

👉 Result: Even if data is stored on disk, most queries never touch the physical disk.

3. Indexing Basics
   3.1 Why Indexes?

Without an index, fetching WHERE id = 123 = full table scan.
With an index, DB jumps directly to the right page & slot.

3.2 B+-Tree Index

PostgreSQL’s default index structure.

Internal nodes: keys only (guide traversal).

Leaf nodes: (key → TID) mappings.

Leaves linked: efficient range queries.

           [20 | 40]             ← root
          /     |    \
[10,15]   [25,30]   [45,50]     ← leaves
↔          ↔         ↔        (linked list)


Lookup = O(log n).

Height is shallow: billions of rows = height ~3–4.

Very cache-friendly.

3.3 B-Tree vs B+-Tree <br>
### B-Tree vs B+-Tree

| Feature        | **B-Tree**       | **B+-Tree** (Postgres)    |
|----------------|------------------|---------------------------|
| Data storage   | In all nodes     | Only in leaves            |
| Search ends at | Internal or leaf | Always leaf               |
| Range queries  | Inefficient      | Efficient (linked leaves) |
| Fan-out        | Smaller          | Larger (better)           |


👉 That’s why databases prefer B+-trees.

3.4 Why Not Hash Indexes?

Hash index = equality only (id=123), no range/order support.

Historically unsafe (not WAL-logged until PG10).

Limited advantage over B+-trees (thanks to caching).

DBs default to B+-tree for versatility.

4. Cold Reads — How DB Fetches Data

Even if a page is not cached (cold read):

Index lookup → TID (block, offset).

DB computes file offset = block * 8192.

Issues read() syscall → OS loads page from disk.

Page goes into buffer pool; row at offset is returned.

👉 Only one 8 KB page read, not the whole table.

5. 2D & Spatial Indexing
   5.1 Problem with B+-Trees

B+-tree works for 1D ordered data.

For (lat, lon) there is no natural total ordering.

5.2 Spatial Index Types in Postgres

GiST (Generalized Search Tree) → R-tree–like.

Nodes store bounding boxes (MBRs).

Search prunes entire regions.

SP-GiST → supports kd-trees, quadtrees.

BRIN → compact block-range index for clustered data.

Root <br/>
├── Box A: lat 10–20, lon 70–80 <br/>
└── Box B: lat 20–30, lon 70–80

Box A <br/>
├── Box A1: lat 12–15, lon 75–78 <br/>
└── Box A2: lat 15–18, lon 76–79


Query “find points near Bangalore” → check only relevant boxes.

5.3 Geohashing Index

Converts (lat, lon) → 1D geohash string (like "tdr3n7").

Prefix search = spatial proximity.

Works with normal B+-tree.

Used in distributed DBs (Elasticsearch, MongoDB).

In Postgres, you can use ST_GeoHash() + B-tree, but GiST is preferred for accuracy.

6. Why DB Reads Are Fast

Even with large data:

Indexes reduce search space (O(log n)).

Shallow trees → 3–4 page accesses at most.

Caching layers keep hot pages in RAM.

Disk read (cold) → only 1 page (8 KB).

Prefetching & sequential I/O reduce latency further.

👉 Net effect: queries complete in microseconds–milliseconds.

📊 Key Visual Recap
Heap Page Layout
+ Page Header
+ Line Pointers
+ Free Space
+ Tuples (rows)

B+-Tree vs B-Tree
B-Tree:   Data at all nodes
B+-Tree:  Data only at leaves (linked)

Spatial Index (R-tree style)
Root → Bounding Boxes → Refine → Actual Rows

✅ Summary

Heap file: table stored as unordered 8 KB pages.

B+-tree index: default in Postgres (versatile, efficient).

Hash index: limited, niche use.

Cold read: direct page fetch via TID.

Caching: buffer pool + OS cache = most queries served from RAM.

Spatial data: GiST / R-tree, SP-GiST, or geohashing depending on workload.

This layered design explains why databases scale to billions of rows while still offering microsecond lookup times.