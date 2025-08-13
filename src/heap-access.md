# Heap Access Method

Prepare:

```sql
CREATE TABLE heap_test (
  id SERIAL PRIMARY KEY,
  a INT NOT NULL,
  b VARCHAR(255) NOT NULL,
  c BOOLEAN NOT NULL
);
INSERT INTO heap_test (a, b, c)
SELECT
  generate_series(1, 100),
  md5(random()::text),
  random() < 0.5;
```

## Page Layout

First we use extension `pageinspect` to explore the physical page layout.

<details>
<summary>Build and install pageinspect</summary>

```sh
pushd contri/pageinspect
make
make install
popd
psql -c 'CREATE EXTENSION pageinspect;'
```
</details>

As described in [docs](https://www.postgresql.org/docs/17/storage-page-layout.html), a page consists of a header and multiple tuples (also refered as *line*s or *item*s).

```plaintext
+----------------+---------------------------------+
| PageHeaderData | linp1 linp2 linp3 ...           |
+-----------+----+---------------------------------+
| ... linpN |                                      |
+-----------+--------------------------------------+
|           ^ pd_lower                             |
|                                                  |
|             v pd_upper                           |
+-------------+------------------------------------+
|             | tupleN ...                         |
+-------------+------------------+-----------------+
|       ... tuple3 tuple2 tuple1 | "special space" |
+--------------------------------+-----------------+
                                 ^ pd_special
```

`PageHeaderData` is space management information generic to any page.
<details>
<summary>Fields in PageHeader</summary>

```c
typedef struct PageHeaderData
{
  /* XXX LSN is member of *any* block, not only page-organized ones */
  PageXLogRecPtr pd_lsn;                     /* LSN: next byte after last byte of xlog
                                              * record for last change to this page */
  uint16 pd_checksum;                        /* checksum */
  uint16 pd_flags;                           /* flag bits, see below */
  LocationIndex pd_lower;                    /* offset to start of free space */
  LocationIndex pd_upper;                    /* offset to end of free space */
  LocationIndex pd_special;                  /* offset to start of special space */
  uint16 pd_pagesize_version;
  TransactionId pd_prune_xid;                /* oldest prunable XID, or zero if none */
  ItemIdData pd_linp[FLEXIBLE_ARRAY_MEMBER]; /* line pointer array */
} PageHeaderData;

typedef PageHeaderData *PageHeader;
```

* `pd_lsn`: identifies xlog record for last change to this page.
* `pd_checksum`: page checksum, if set.
* `pd_flags`: flag bits.
* `pd_lower`: offset to start of free space.
* `pd_upper`: offset to end of free space.
* `pd_special`: offset to start of special space.
* `pd_pagesize_version`: size in bytes and page layout version number.
* `pd_prune_xid`: oldest XID among potentially prunable tuples on page.

</details>

To inspect the page in table we created above:

```sql
postgres=# select * from page_header(get_raw_page('heap_test', 0));
    lsn    | checksum | flags | lower | upper | special | pagesize | version | prune_xid 
-----------+----------+-------+-------+-------+---------+----------+---------+-----------
 0/14E84B0 |        0 |     0 |   428 |   952 |    8192 |     8192 |       4 |         0
(1 row)
```

Line pointer (`linp`) is a fixed-size fat pointer to real tuple.

```c
typedef struct ItemIdData
{
 unsigned lp_off:15,  /* offset to tuple (from start of page) */
          lp_flags:2, /* state of line pointer, see below */
          lp_len:15;  /* byte length of tuple */
} ItemIdData;

typedef ItemIdData *ItemId;
```

Each heap tuple consists of HeapTupleHeader and actual data.

```c
typedef struct HeapTupleFields
{
  TransactionId t_xmin;   /* inserting xact ID */
  TransactionId t_xmax;   /* deleting or locking xact ID */

  union
  {
    CommandId t_cid;      /* inserting or deleting command ID, or both */
    TransactionId t_xvac; /* old-style VACUUM FULL xact ID */
  } t_field3;
} HeapTupleFields;

struct HeapTupleHeaderData
{
  union
  {
    HeapTupleFields t_heap;
   DatumTupleFields t_datum;
  } t_choice;

  ItemPointerData t_ctid;  /* current TID of this or newer tuple (or a
                            * speculative insertion token) */

  /* Fields below here must match MinimalTupleData! */

#define FIELDNO_HEAPTUPLEHEADERDATA_INFOMASK2 2
  uint16 t_infomask2;      /* number of attributes + various flags */

#define FIELDNO_HEAPTUPLEHEADERDATA_INFOMASK 3
  uint16 t_infomask;       /* various flag bits, see below */

#define FIELDNO_HEAPTUPLEHEADERDATA_HOFF 4
  uint8 t_hoff;            /* sizeof header incl. bitmap, padding */

  /* ^ - 23 bytes - ^ */

#define FIELDNO_HEAPTUPLEHEADERDATA_BITS 5
  bits8 t_bits[FLEXIBLE_ARRAY_MEMBER]; /* bitmap of NULLs */

  /* MORE DATA FOLLOWS AT END OF STRUCT */
};

```

To inspect the first 5 tuples in the table:

```sql
postgres=# select * from heap_page_items(get_raw_page('heap_test', 0)) limit 5;
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |                                         t_data                                         
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------------------------------------------------------------------------------
  1 |   8120 |        1 |     66 |    746 |      0 |        0 | (0,1)  |           4 |       2306 |     24 |        |       | \x010000000100000043633736316331636439653330383035373531623465323266666437613432633901
  2 |   8048 |        1 |     66 |    746 |      0 |        0 | (0,2)  |           4 |       2306 |     24 |        |       | \x020000000200000043633736353635636135393064663437373330373534643664616631326236326101
  3 |   7976 |        1 |     66 |    746 |      0 |        0 | (0,3)  |           4 |       2306 |     24 |        |       | \x030000000300000043376132346439643734333731643563356163333966616230636566373034353700
  4 |   7904 |        1 |     66 |    746 |      0 |        0 | (0,4)  |           4 |       2306 |     24 |        |       | \x040000000400000043663562626233633534376435643162373465623034633362333063316637333100
  5 |   7832 |        1 |     66 |    746 |      0 |        0 | (0,5)  |           4 |       2306 |     24 |        |       | \x050000000500000043373035653238393330366232383664306434366137633534373035633361633800
(5 rows)

```

## Heap Insert

Start a new session and then attach to the backend process. Breakpoint at `heapam.c:heap_insert`. Then insert a new tuple.

```sql
INSERT INTO heap_test (a, b, c) VALUES (1, 'test', true);
```

At breakpoint, the call stack is:

```text
heap_insert (src/backend/access/heap/heapam.c:2008)
heapam_tuple_insert (src/backend/access/heap/heapam_handler.c:253)
table_tuple_insert (src/include/access/tableam.h:1406)
ExecInsert (src/backend/executor/nodeModifyTable.c:1159)
ExecModifyTable (src/backend/executor/nodeModifyTable.c:4150)
ExecProcNode (src/include/executor/executor.h:274)
ExecutePlan (src/backend/executor/execMain.c:1649)
standard_ExecutorRun (src/backend/executor/execMain.c:361)
ExecutorRun (src/backend/executor/execMain.c:307)
ProcessQuery (src/backend/tcop/pquery.c:160)
PortalRunMulti (Unknown Source:0)
PortalRun (src/backend/tcop/pquery.c:789)
exec_simple_query (src/backend/tcop/postgres.c:1278)
PostgresMain (Unknown Source:0)
BackendMain (src/backend/tcop/backend_startup.c:105)
postmaster_child_launch (src/backend/postmaster/launch_backend.c:277)
BackendStartup (src/backend/postmaster/postmaster.c:3594)
ServerLoop (src/backend/postmaster/postmaster.c:1676)
PostmasterMain (src/backend/postmaster/postmaster.c:1374)
main (src/backend/main/main.c:199)
```

In this chapter, we focus on storage layer, which is the top 3 frames.

`table_tuple_insert` wraps call to relation's table access method (`tableam`). By this way, PostgreSQL can support different table access methods, such as heap, hash, and B-tree. See [Table Access Method Interface Definition](https://www.postgresql.org/docs/17/tableam.html).

```c
{{#include ../../src/include/access/tableam.h:1402:1408}}
```

```c
{{#include ../../src/backend/access/heap/heapam_handler.c:2592}}
    /* ... */
{{#include ../../src/backend/access/heap/heapam_handler.c:2614}}
    /* ... */
{{#include ../../src/backend/access/heap/heapam_handler.c:2649}}
```

`heapam_tuple_insert` do the preparation and cleanup around the insertion.

```c
{{#include ../../src/backend/access/heap/heapam_handler.c:241:258}}
```

`heap_insert` is the dirty work of the insertion. Since it has ~200 LoC (`src/backend/access/heap/heapam.c:2004-2185`), the whole function is not shown here for saving network traffic :). We pick the most important part of the function here.

1. prepares the tuple for insertion. We would go back to TOAST later.

    ```c
{{#include ../../src/backend/access/heap/heapam.c:2018:2024}}
    ```

2. finds the buffer to insert the tuple into.

    ```c
{{#include ../../src/backend/access/heap/heapam.c:2026:2033}}
    ```

3. puts the tuple into the buffer.

    ```c
{{#include ../../src/backend/access/heap/heapam.c:2055:2056}}
    ```

4. marks the buffer as modified so that the buffer would be flushed to disk by background worker.

    ```c
{{#include ../../src/backend/access/heap/heapam.c:2078}}
    ```

5. WAL part, skip for now.

    ```c
{{#include ../../src/backend/access/heap/heapam.c:2080:2081}}
    ```

6. release the buffer.

    ```c
{{#include ../../src/backend/access/heap/heapam.c:2161:2163}}
    ```

[[TODO]]
