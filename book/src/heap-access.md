# Heap Access Method

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

Extension `pageinspect` can explore the physical page layout.

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

A page consists of a header and multiple tuples (also refered as *line*s or *item*s).

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

<details>
<summary><code>PageHeaderData</code> is space management information generic to any page.</summary>

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

- `pd_lsn`: identifies xlog record for last change to this page.
- `pd_checksum`: page checksum, if set.
- `pd_flags`: flag bits.
- `pd_lower`: offset to start of free space.
- `pd_upper`: offset to end of free space.
- `pd_special`: offset to start of special space.
- `pd_pagesize_version`: size in bytes and page layout version number.
- `pd_prune_xid`: oldest XID among potentially prunable tuples on page.

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

## DQL

### Heap Scan

```sql
EXPLAIN (COSTS OFF)
SELECT ctid, id, a, b, c
FROM heap_test
WHERE a >= 95;
```

```text
Seq Scan on heap_test
  Filter: (a >= 95)
```

```text
ExecSeqScan -> ExecScan -> SeqNext
  -> table_beginscan -> heap_beginscan
  -> table_scan_getnextslot -> heap_getnextslot
    -> heapgettup_pagemode
      -> heap_prepare_pagescan
        -> page_collect_tuples
```

The call path has two branches: `SeqNext()` starts the scan once through
`table_beginscan()`, then requests each tuple through
`table_scan_getnextslot()`. The table-AM wrappers dispatch both operations to
the heap implementation. The remaining heap functions fetch pages, collect
visible offsets, and return one tuple.

These layers reject tuples for different reasons. `page_collect_tuples()`
checks MVCC visibility. `heapgettup_pagemode()` can apply table-AM scan keys,
but this sequential scan passes zero scan keys. Finally, `ExecScan()` evaluates
`a >= 95` and projects `ctid, id, a, b, c` only after the tuple passes that
filter.

<details>
<summary><code>ExecSeqScan()</code> delegates to <code>ExecScan()</code>, which filters and projects tuples from <code>SeqNext()</code>.</summary>

```c
/* src/backend/executor/nodeSeqscan.c:ExecSeqScan, SeqNext */
static TupleTableSlot *
ExecSeqScan(PlanState *pstate)
{
    SeqScanState *node = castNode(SeqScanState, pstate);

    return ExecScan(&node->ss,
                    (ExecScanAccessMtd) SeqNext,
                    (ExecScanRecheckMtd) SeqRecheck);
}

/* src/backend/executor/execScan.c:ExecScan */
TupleTableSlot *
ExecScan(ScanState *node,
         ExecScanAccessMtd accessMtd,
         ExecScanRecheckMtd recheckMtd)
{
    ExprContext *econtext = node->ps.ps_ExprContext;
    ExprState *qual = node->ps.qual;
    ProjectionInfo *projInfo = node->ps.ps_ProjInfo;

    if (!qual && !projInfo)
    {
        ResetExprContext(econtext);
        return ExecScanFetch(node, accessMtd, recheckMtd);
    }

    ResetExprContext(econtext);

    for (;;)
    {
        TupleTableSlot *slot;

        slot = ExecScanFetch(node, accessMtd, recheckMtd);
        if (TupIsNull(slot))
        {
            if (projInfo)
                return ExecClearTuple(projInfo->pi_state.resultslot);
            return slot;
        }

        econtext->ecxt_scantuple = slot;

        if (qual == NULL || ExecQual(qual, econtext))
        {
            if (projInfo)
                return ExecProject(projInfo);
            return slot;
        }

        InstrCountFiltered1(node, 1);
        ResetExprContext(econtext);
    }
}

/* src/backend/executor/nodeSeqscan.c:SeqNext */
static TupleTableSlot *
SeqNext(SeqScanState *node)
{
    TableScanDesc scandesc = node->ss.ss_currentScanDesc;
    EState *estate = node->ss.ps.state;
    TupleTableSlot *slot = node->ss.ss_ScanTupleSlot;

    if (scandesc == NULL)
    {
        scandesc = table_beginscan(node->ss.ss_currentRelation,
                                   estate->es_snapshot, 0, NULL);
        node->ss.ss_currentScanDesc = scandesc;
    }

    if (table_scan_getnextslot(scandesc, estate->es_direction, slot))
        return slot;
    return NULL;
}
```

</details>

<details>
<summary><code>table_beginscan()</code> dispatches <code>scan_begin</code>; <code>heap_beginscan()</code> initializes the heap scan and its read stream.</summary>

```c
/* src/include/access/tableam.h:table_beginscan */
static inline TableScanDesc
table_beginscan(Relation rel, Snapshot snapshot,
                int nkeys, struct ScanKeyData *key)
{
    uint32 flags = SO_TYPE_SEQSCAN |
        SO_ALLOW_STRAT | SO_ALLOW_SYNC | SO_ALLOW_PAGEMODE;

    return rel->rd_tableam->scan_begin(rel, snapshot,
                                       nkeys, key, NULL, flags);
}

/* src/backend/access/heap/heapam.c:heap_beginscan */
TableScanDesc
heap_beginscan(Relation relation, Snapshot snapshot,
               int nkeys, ScanKey key,
               ParallelTableScanDesc parallel_scan,
               uint32 flags)
{
    HeapScanDesc scan;

    RelationIncrementReferenceCount(relation);
    scan = (HeapScanDesc) palloc(sizeof(HeapScanDescData));
    scan->rs_base.rs_rd = relation;
    scan->rs_base.rs_snapshot = snapshot;
    scan->rs_base.rs_nkeys = nkeys;
    scan->rs_base.rs_flags = flags;
    scan->rs_base.rs_parallel = parallel_scan;
    /* ... */

    initscan(scan, key, false);

    if (scan->rs_base.rs_flags & SO_TYPE_SEQSCAN ||
        scan->rs_base.rs_flags & SO_TYPE_TIDRANGESCAN)
    {
        ReadStreamBlockNumberCB cb;

        if (scan->rs_base.rs_parallel)
            cb = heap_scan_stream_read_next_parallel;
        else
            cb = heap_scan_stream_read_next_serial;

        scan->rs_read_stream =
            read_stream_begin_relation(READ_STREAM_SEQUENTIAL,
                                       scan->rs_strategy,
                                       scan->rs_base.rs_rd,
                                       MAIN_FORKNUM,
                                       cb, scan, 0);
    }

    return (TableScanDesc) scan;
}
```

</details>

<details>
<summary><code>table_scan_getnextslot()</code> dispatches <code>scan_getnextslot</code>; <code>heap_getnextslot()</code> runs the heap scan and stores its result.</summary>

```c
/* src/include/access/tableam.h:table_scan_getnextslot */
static inline bool
table_scan_getnextslot(TableScanDesc sscan, ScanDirection direction,
                       TupleTableSlot *slot)
{
    slot->tts_tableOid = RelationGetRelid(sscan->rs_rd);
    /* ... */
    return sscan->rs_rd->rd_tableam->scan_getnextslot(sscan,
                                                       direction, slot);
}

/* src/backend/access/heap/heapam.c:heap_getnextslot */
bool
heap_getnextslot(TableScanDesc sscan, ScanDirection direction,
                 TupleTableSlot *slot)
{
    HeapScanDesc scan = (HeapScanDesc) sscan;

    if (sscan->rs_flags & SO_ALLOW_PAGEMODE)
        heapgettup_pagemode(scan, direction,
                            sscan->rs_nkeys, sscan->rs_key);
    else
        heapgettup(scan, direction,
                   sscan->rs_nkeys, sscan->rs_key);

    if (scan->rs_ctup.t_data == NULL)
    {
        ExecClearTuple(slot);
        return false;
    }

    pgstat_count_heap_getnext(scan->rs_base.rs_rd);
    ExecStoreBufferHeapTuple(&scan->rs_ctup, slot, scan->rs_cbuf);
    return true;
}
```

</details>

<details>
<summary><code>heapgettup_pagemode()</code> fetches pages and points the current tuple at each selected page item.</summary>

```c
/* src/backend/access/heap/heapam.c:heapgettup_pagemode */
static void
heapgettup_pagemode(HeapScanDesc scan, ScanDirection dir,
                    int nkeys, ScanKey key)
{
    HeapTuple tuple = &(scan->rs_ctup);
    Page page;
    int lineindex;
    int linesleft;

    if (likely(scan->rs_inited))
    {
        page = BufferGetPage(scan->rs_cbuf);
        lineindex = scan->rs_cindex + dir;
        linesleft = ScanDirectionIsForward(dir)
            ? scan->rs_ntuples - lineindex
            : scan->rs_cindex;
        goto continue_page;
    }

    while (true)
    {
        heap_fetch_next_buffer(scan, dir);
        if (!BufferIsValid(scan->rs_cbuf))
            break;

        heap_prepare_pagescan((TableScanDesc) scan);
        page = BufferGetPage(scan->rs_cbuf);
        linesleft = scan->rs_ntuples;
        lineindex = ScanDirectionIsForward(dir) ? 0 : linesleft - 1;

continue_page:
        for (; linesleft > 0; linesleft--, lineindex += dir)
        {
            OffsetNumber lineoff = scan->rs_vistuples[lineindex];
            ItemId lpp = PageGetItemId(page, lineoff);

            tuple->t_data = (HeapTupleHeader) PageGetItem(page, lpp);
            tuple->t_len = ItemIdGetLength(lpp);
            ItemPointerSet(&tuple->t_self, scan->rs_cblock, lineoff);

            if (key != NULL &&
                !HeapKeyTest(tuple, RelationGetDescr(scan->rs_base.rs_rd),
                             nkeys, key))
                continue;

            scan->rs_cindex = lineindex;
            return;
        }
    }

    if (BufferIsValid(scan->rs_cbuf))
        ReleaseBuffer(scan->rs_cbuf);
    scan->rs_cbuf = InvalidBuffer;
    scan->rs_cblock = InvalidBlockNumber;
    tuple->t_data = NULL;
    scan->rs_inited = false;
}
```

</details>

<details>
<summary><code>heap_prepare_pagescan()</code> prunes and locks the page, then delegates visibility collection to <code>page_collect_tuples()</code>.</summary>

```c
/* src/backend/access/heap/heapam.c:heap_prepare_pagescan */
void
heap_prepare_pagescan(TableScanDesc sscan)
{
    HeapScanDesc scan = (HeapScanDesc) sscan;
    Buffer buffer = scan->rs_cbuf;
    BlockNumber block = scan->rs_cblock;
    Snapshot snapshot = scan->rs_base.rs_snapshot;
    Page page;
    int lines;
    bool all_visible;
    bool check_serializable;

    heap_page_prune_opt(scan->rs_base.rs_rd, buffer);
    LockBuffer(buffer, BUFFER_LOCK_SHARE);

    page = BufferGetPage(buffer);
    lines = PageGetMaxOffsetNumber(page);
    all_visible = PageIsAllVisible(page) && !snapshot->takenDuringRecovery;
    check_serializable =
        CheckForSerializableConflictOutNeeded(scan->rs_base.rs_rd, snapshot);

    if (likely(all_visible))
    {
        if (likely(!check_serializable))
            scan->rs_ntuples = page_collect_tuples(scan, snapshot, page, buffer,
                                                   block, lines, true, false);
        else
            scan->rs_ntuples = page_collect_tuples(scan, snapshot, page, buffer,
                                                   block, lines, true, true);
    }
    else
    {
        if (likely(!check_serializable))
            scan->rs_ntuples = page_collect_tuples(scan, snapshot, page, buffer,
                                                   block, lines, false, false);
        else
            scan->rs_ntuples = page_collect_tuples(scan, snapshot, page, buffer,
                                                   block, lines, false, true);
    }

    LockBuffer(buffer, BUFFER_LOCK_UNLOCK);
}
```

</details>

<details>
<summary><code>page_collect_tuples()</code> records the offset of every normal tuple visible to the scan snapshot.</summary>

```c
/* src/backend/access/heap/heapam.c:page_collect_tuples */
static int
page_collect_tuples(HeapScanDesc scan, Snapshot snapshot,
                    Page page, Buffer buffer,
                    BlockNumber block, int lines,
                    bool all_visible, bool check_serializable)
{
    int ntup = 0;

    for (OffsetNumber lineoff = FirstOffsetNumber;
         lineoff <= lines;
         lineoff++)
    {
        ItemId lpp = PageGetItemId(page, lineoff);
        HeapTupleData loctup;
        bool valid;

        if (!ItemIdIsNormal(lpp))
            continue;

        loctup.t_data = (HeapTupleHeader) PageGetItem(page, lpp);
        loctup.t_len = ItemIdGetLength(lpp);
        loctup.t_tableOid = RelationGetRelid(scan->rs_base.rs_rd);
        ItemPointerSet(&loctup.t_self, block, lineoff);

        valid = all_visible ||
            HeapTupleSatisfiesVisibility(&loctup, snapshot, buffer);

        if (check_serializable)
            HeapCheckForSerializableConflictOut(valid, scan->rs_base.rs_rd,
                                                &loctup, buffer, snapshot);

        if (valid)
            scan->rs_vistuples[ntup++] = lineoff;
    }

    return ntup;
}
```

</details>

### Tuple Passing

PostgreSQL keeps tuples in shared buffers and passes references whenever the
consumer's lifetime allows it.

| Method or stage | What passes to the next stage | Tuple copy |
| --- | --- | --- |
| Buffer manager | A page in a shared-buffer frame | Page transfer on a miss; not a per-tuple copy |
| `heapgettup_pagemode()` | `HeapTupleData` whose `t_data` points into the page | No |
| `ExecStoreBufferHeapTuple()` | The tuple pointer plus a buffer pin | No |
| Predicate in `ExecScan()` | The same scan slot | No |
| Projection in `ExecProject()` | `Datum` values in a virtual slot | No full tuple copy; expressions may create values |
| Parent executor node | Usually a `TupleTableSlot *` | No |
| `ExecMaterializeSlot()` or an owning plan node | Independently owned tuple data | Yes, when independent storage is required |

Tuple references appear throughout a simple sequential scan:

1. `heapgettup_pagemode()` stores the pointer from `PageGetItem()` in
   `scan->rs_ctup.t_data`.
2. `ExecStoreBufferHeapTuple()` stores that tuple reference in the scan slot
   and takes another pin on the page.
3. `ExecProcNode()` and ordinary parent/child calls pass a `TupleTableSlot *`
   between executor nodes.
4. A simple projection creates a virtual result slot. It copies pass-by-value
   attributes, such as an `int`, into `Datum` values, but pass-by-reference
   values usually continue to point into the pinned page.

<details>
<summary><code>tts_buffer_heap_store_tuple()</code> stores the tuple pointer and pins its page instead of copying the tuple body.</summary>

```c
/* src/backend/executor/execTuples.c:tts_buffer_heap_store_tuple */
static inline void
tts_buffer_heap_store_tuple(TupleTableSlot *slot, HeapTuple tuple,
                            Buffer buffer, bool transfer_pin)
{
    BufferHeapTupleTableSlot *bslot = (BufferHeapTupleTableSlot *) slot;

    /* Cleanup of a previously materialized tuple omitted. */
    slot->tts_flags &= ~TTS_FLAG_EMPTY;
    slot->tts_nvalid = 0;
    bslot->base.tuple = tuple;
    bslot->base.off = 0;
    slot->tts_tid = tuple->t_self;

    if (bslot->buffer != buffer)
    {
        if (BufferIsValid(bslot->buffer))
            ReleaseBuffer(bslot->buffer);

        bslot->buffer = buffer;
        if (!transfer_pin && BufferIsValid(buffer))
            IncrBufferRefCount(buffer);
    }
    else if (transfer_pin && BufferIsValid(buffer))
        ReleaseBuffer(buffer);
}
```

</details>

For a simple sequential scan, the scan path makes **zero full heap tuple
copies** after the page enters shared buffers.

A data copy happens when PostgreSQL needs to move data or own it independently:

1. On a buffer miss, the storage and buffer managers transfer the whole page
   into a shared-buffer frame. This is a page transfer, not an additional copy
   of each tuple.
2. `ExecMaterializeSlot()` copies a buffer-backed tuple before it releases the
   buffer pin.
3. `ExecCopySlot()` can copy tuple data when the source and destination slot
   formats cannot share the same buffer-backed representation. Compatible
   buffer slots can instead share the page with separate pins.
4. Sort, hash, materialization, and tuplestore nodes copy or serialize the data
   that they must retain beyond the input slot's lifetime.
5. Projection expressions can allocate new values or detoast existing values,
   although a simple `Var` projection normally reuses the input `Datum`.

<details>
<summary><code>tts_buffer_heap_materialize()</code> copies a buffer-backed tuple when the slot needs independent storage.</summary>

```c
/* src/backend/executor/execTuples.c:tts_buffer_heap_materialize */
static void
tts_buffer_heap_materialize(TupleTableSlot *slot)
{
    BufferHeapTupleTableSlot *bslot = (BufferHeapTupleTableSlot *) slot;
    MemoryContext oldContext;

    if (TTS_SHOULDFREE(slot))
        return;

    oldContext = MemoryContextSwitchTo(slot->tts_mcxt);
    bslot->base.off = 0;
    slot->tts_nvalid = 0;

    if (!bslot->base.tuple)
        bslot->base.tuple = heap_form_tuple(slot->tts_tupleDescriptor,
                                            slot->tts_values,
                                            slot->tts_isnull);
    else
    {
        bslot->base.tuple = heap_copytuple(bslot->base.tuple);
        if (BufferIsValid(bslot->buffer))
            ReleaseBuffer(bslot->buffer);
        bslot->buffer = InvalidBuffer;
    }

    slot->tts_flags |= TTS_FLAG_SHOULDFREE;
    MemoryContextSwitchTo(oldContext);
}
```

</details>

## DML

### Insert

```sql
INSERT INTO heap_test (a, b, c) VALUES (1, 'test', true);
```

```text
ExecModifyTable
  -> ExecInsert
    -> table_tuple_insert
      -> heapam_tuple_insert
        -> heap_insert
```

<details>
<summary><code>table_tuple_insert()</code> dispatches through the relation's table AM, which maps heap insertion to <code>heapam_tuple_insert()</code>.</summary>

```c
/* src/include/access/tableam.h:table_tuple_insert */
static inline void
table_tuple_insert(Relation rel, TupleTableSlot *slot, CommandId cid,
                   int options, struct BulkInsertStateData *bistate)
{
    rel->rd_tableam->tuple_insert(rel, slot, cid, options, bistate);
}

/* src/backend/access/heap/heapam_handler.c:heapam_methods */
static const TableAmRoutine heapam_methods = {
    /* ... */
    .tuple_insert = heapam_tuple_insert,
    /* ... */
};
```

</details>

<details>
<summary><code>heapam_tuple_insert()</code> converts the slot to a heap tuple and copies the assigned TID back to the slot.</summary>

```c
/* src/backend/access/heap/heapam_handler.c:heapam_tuple_insert */
static void
heapam_tuple_insert(Relation relation, TupleTableSlot *slot, CommandId cid,
                    int options, BulkInsertState bistate)
{
    bool shouldFree = true;
    HeapTuple tuple = ExecFetchSlotHeapTuple(slot, true, &shouldFree);

    slot->tts_tableOid = RelationGetRelid(relation);
    tuple->t_tableOid = slot->tts_tableOid;

    heap_insert(relation, tuple, cid, options, bistate);
    ItemPointerCopy(&tuple->t_self, &slot->tts_tid);

    if (shouldFree)
        pfree(tuple);
}
```

</details>

The table-AM interface lets PostgreSQL support the built-in heap access method
and extension-provided table access methods. See [Table Access Method Interface
Definition](https://www.postgresql.org/docs/17/tableam.html).

`heap_insert()` prepares the tuple, chooses a page, updates the shared buffer,
and records the change in WAL.

<details>
<summary><code>heap_insert()</code> prepares and places the tuple, marks the buffer dirty, writes WAL, and releases the buffer.</summary>

```c
/* src/backend/access/heap/heapam.c:heap_insert */
void
heap_insert(Relation relation, HeapTuple tup, CommandId cid,
            int options, BulkInsertState bistate)
{
    TransactionId xid = GetCurrentTransactionId();
    HeapTuple heaptup;
    Buffer buffer;
    Buffer vmbuffer = InvalidBuffer;

    heaptup = heap_prepare_insert(relation, tup, xid, cid, options);

    buffer = RelationGetBufferForTuple(relation, heaptup->t_len,
                                       InvalidBuffer, options, bistate,
                                       &vmbuffer, NULL, 0);

    START_CRIT_SECTION();

    RelationPutHeapTuple(relation, buffer, heaptup,
                         (options & HEAP_INSERT_SPECULATIVE) != 0);
    /* Clear all-visible state when necessary. */
    MarkBufferDirty(buffer);

    if (RelationNeedsWAL(relation))
    {
        /* Build and insert the XLOG_HEAP_INSERT record. */
        /* ... */
    }

    END_CRIT_SECTION();

    UnlockReleaseBuffer(buffer);
    if (vmbuffer != InvalidBuffer)
        ReleaseBuffer(vmbuffer);

    /* Cache invalidation, statistics, and tuple cleanup omitted. */
}
```

</details>

#### Multi-Insert

```sql
COPY heap_test (a, b, c) FROM STDIN WITH (FORMAT csv);
101,multi-1,true
102,multi-2,false
103,multi-3,true
\.
```

```text
CopyFrom
  -> CopyMultiInsertInfoFlush
    -> CopyMultiInsertBufferFlush
      -> table_multi_insert
        -> heap_multi_insert
```

<details>
<summary><code>CopyFrom()</code> flushes buffered tuple slots through <code>table_multi_insert()</code>.</summary>

```c
/* src/include/access/tableam.h:table_multi_insert */
static inline void
table_multi_insert(Relation rel, TupleTableSlot **slots, int nslots,
                   CommandId cid, int options,
                   struct BulkInsertStateData *bistate)
{
    rel->rd_tableam->multi_insert(rel, slots, nslots,
                                  cid, options, bistate);
}
```

</details>

<details>
<summary>The heap table AM maps <code>multi_insert</code> directly to <code>heap_multi_insert()</code>.</summary>

```c
/* src/backend/access/heap/heapam_handler.c:heapam_methods */
static const TableAmRoutine heapam_methods = {
    /* ... */
    .multi_insert = heap_multi_insert,
    /* ... */
};
```

</details>

`heap_multi_insert()` follows the same basic process as `heap_insert()`, but it
groups tuple placement, page locking, and WAL by page.

<details>
<summary><code>heap_multi_insert()</code> prepares all tuples, fills each page, emits one multi-insert WAL record per page, and returns the assigned TIDs.</summary>

```c
/* src/backend/access/heap/heapam.c:heap_multi_insert */
void
heap_multi_insert(Relation relation, TupleTableSlot **slots, int ntuples,
                  CommandId cid, int options, BulkInsertState bistate)
{
    TransactionId xid = GetCurrentTransactionId();
    HeapTuple *heaptuples;
    int ndone = 0;

    heaptuples = palloc(ntuples * sizeof(HeapTuple));
    for (int i = 0; i < ntuples; i++)
    {
        HeapTuple tuple = ExecFetchSlotHeapTuple(slots[i], true, NULL);

        slots[i]->tts_tableOid = RelationGetRelid(relation);
        tuple->t_tableOid = slots[i]->tts_tableOid;
        heaptuples[i] = heap_prepare_insert(relation, tuple, xid, cid,
                                            options);
    }

    while (ndone < ntuples)
    {
        Buffer buffer;
        Page page;
        int nthispage;

        buffer = RelationGetBufferForTuple(relation,
                                           heaptuples[ndone]->t_len,
                                           InvalidBuffer, options, bistate,
                                           &vmbuffer, NULL,
                                           npages - npages_used);
        page = BufferGetPage(buffer);

        START_CRIT_SECTION();
        RelationPutHeapTuple(relation, buffer, heaptuples[ndone], false);

        for (nthispage = 1; ndone + nthispage < ntuples; nthispage++)
        {
            HeapTuple heaptup = heaptuples[ndone + nthispage];

            if (PageGetHeapFreeSpace(page) <
                MAXALIGN(heaptup->t_len) + saveFreeSpace)
                break;
            RelationPutHeapTuple(relation, buffer, heaptup, false);
        }

        MarkBufferDirty(buffer);
        if (needwal)
        {
            /* Emit XLOG_HEAP2_MULTI_INSERT for nthispage tuples. */
            /* ... */
        }
        END_CRIT_SECTION();

        UnlockReleaseBuffer(buffer);
        ndone += nthispage;
    }

    for (int i = 0; i < ntuples; i++)
        slots[i]->tts_tid = heaptuples[i]->t_self;
}
```

</details>

Compared with repeatedly calling `heap_insert`, multi-insert locks each heap
page once and normally emits one WAL record per page rather than per tuple. A
batch that spans several pages still performs one buffer and WAL cycle for each
page. Afterward, `COPY FROM` creates index entries and runs row-level triggers
one tuple at a time. An ordinary multi-row `INSERT` does not use this path; it
calls `table_tuple_insert` for each row.

### Delete

```sql
DELETE FROM heap_test WHERE id = 1;
```

```text
ExecModifyTable
  -> ExecDelete
    -> ExecDeleteAct
      -> table_tuple_delete
        -> heapam_tuple_delete
          -> heap_delete
```

A heap delete is an MVCC operation. It does not immediately remove the tuple or
its index entries. Instead, it records the deleting transaction in the tuple
header; `VACUUM` can reclaim the storage after no snapshot can see the old
version. The scan that finds the row supplies its TID to the modify-table
executor.

<details>
<summary><code>ExecDeleteAct()</code> passes the TID and the command's snapshots to the table access method.</summary>

```c
/* src/backend/executor/nodeModifyTable.c:ExecDeleteAct */
static TM_Result
ExecDeleteAct(ModifyTableContext *context, ResultRelInfo *resultRelInfo,
              ItemPointer tupleid, bool changingPart)
{
    EState *estate = context->estate;

    return table_tuple_delete(resultRelInfo->ri_RelationDesc, tupleid,
                              estate->es_output_cid,
                              estate->es_snapshot,
                              estate->es_crosscheck_snapshot,
                              true /* wait for commit */,
                              &context->tmfd,
                              changingPart);
}
```

</details>

<details>
<summary><code>table_tuple_delete()</code> dispatches through <code>rd_tableam</code>.</summary>

```c
/* src/include/access/tableam.h:table_tuple_delete */
static inline TM_Result
table_tuple_delete(Relation rel, ItemPointer tid, CommandId cid,
                   Snapshot snapshot, Snapshot crosscheck, bool wait,
                   TM_FailureData *tmfd, bool changingPart)
{
    return rel->rd_tableam->tuple_delete(rel, tid, cid,
                                         snapshot, crosscheck,
                                         wait, tmfd, changingPart);
}
```

</details>

<details>
<summary>The heap callback is a small wrapper around <code>heap_delete()</code>.</summary>

```c
/* src/backend/access/heap/heapam_handler.c:heapam_tuple_delete */
static TM_Result
heapam_tuple_delete(Relation relation, ItemPointer tid, CommandId cid,
                    Snapshot snapshot, Snapshot crosscheck, bool wait,
                    TM_FailureData *tmfd, bool changingPart)
{
    return heap_delete(relation, tid, cid, crosscheck, wait,
                       tmfd, changingPart);
}
```

</details>

`heap_delete()` locks the target page, checks concurrent tuple state, computes
the new `xmax`, and marks the old tuple version as deleted.

<details>
<summary><code>heap_delete()</code> checks the target tuple and records the deleting transaction without removing the tuple bytes.</summary>

```c
/* src/backend/access/heap/heapam.c:heap_delete */
TM_Result
heap_delete(Relation relation, ItemPointer tid,
            CommandId cid, Snapshot crosscheck, bool wait,
            TM_FailureData *tmfd, bool changingPart)
{
    TransactionId xid = GetCurrentTransactionId();
    Buffer vmbuffer = InvalidBuffer;
    TransactionId new_xmax;
    uint16 new_infomask;
    uint16 new_infomask2;
    bool iscombo;
    BlockNumber block = ItemPointerGetBlockNumber(tid);
    Buffer buffer = ReadBuffer(relation, block);
    Page page = BufferGetPage(buffer);
    HeapTupleData tp;

    if (PageIsAllVisible(page))
        visibilitymap_pin(relation, block, &vmbuffer);
    LockBuffer(buffer, BUFFER_LOCK_EXCLUSIVE);

    ItemId lp = PageGetItemId(page, ItemPointerGetOffsetNumber(tid));
    tp.t_data = (HeapTupleHeader) PageGetItem(page, lp);
    tp.t_len = ItemIdGetLength(lp);
    tp.t_self = *tid;

    TM_Result result = HeapTupleSatisfiesUpdate(&tp, cid, buffer);
    /* Wait for a concurrent writer and recheck when necessary. */
    if (result != TM_Ok)
    {
        /* Fill tmfd, release locks and pins, and return result. */
        /* ... */
    }

    HeapTupleHeaderAdjustCmax(tp.t_data, &cid, &iscombo);
    compute_new_xmax_infomask(HeapTupleHeaderGetRawXmax(tp.t_data),
                              tp.t_data->t_infomask,
                              tp.t_data->t_infomask2,
                              xid, LockTupleExclusive, true,
                              &new_xmax, &new_infomask,
                              &new_infomask2);

    START_CRIT_SECTION();
    PageSetPrunable(page, xid);

    if (PageIsAllVisible(page))
    {
        PageClearAllVisible(page);
        visibilitymap_clear(relation, block, vmbuffer,
                            VISIBILITYMAP_VALID_BITS);
    }

    tp.t_data->t_infomask &= ~(HEAP_XMAX_BITS | HEAP_MOVED);
    tp.t_data->t_infomask2 &= ~HEAP_KEYS_UPDATED;
    tp.t_data->t_infomask |= new_infomask;
    tp.t_data->t_infomask2 |= new_infomask2;
    HeapTupleHeaderSetXmax(tp.t_data, new_xmax);
    HeapTupleHeaderSetCmax(tp.t_data, cid, iscombo);
    tp.t_data->t_ctid = tp.t_self;
    MarkBufferDirty(buffer);

    /* Write XLOG_HEAP_DELETE when the relation needs WAL. */
    /* ... */
    END_CRIT_SECTION();

    /* Release locks and pins, remove external TOAST values, and count it. */
    return TM_Ok;
}
```

</details>

If another transaction is modifying the tuple, `heap_delete()` may release the
buffer lock, acquire a heavyweight tuple lock, wait, and then recheck. It
returns statuses such as `TM_Updated`, `TM_Deleted`, `TM_SelfModified`, or
`TM_BeingModified` to the executor.

When WAL is required, `heap_delete` writes an `XLOG_HEAP_DELETE` record with the
tuple offset, new `xmax`, relevant header flags, and replica identity data
needed by logical decoding. It then releases the buffer lock, deletes external
TOAST values using MVCC, releases the pins and tuple lock, and updates
statistics.

Index entries are intentionally left in place because an older snapshot may
still use them to find the tuple. A later `VACUUM` removes dead index entries
and reclaims the heap space.

### Update

```text
ExecModifyTable
  -> ExecUpdate
    -> ExecUpdateAct
      -> table_tuple_update
        -> heapam_tuple_update
          -> heap_update
```

An update creates a new tuple version and marks the old version with `xmax`. It
therefore resembles a delete followed by an insert, but PostgreSQL normally
performs both parts inside `heap_update()` rather than calling `heap_delete()`
and `heap_insert()` separately.

`heap_update()` links the old tuple's `t_ctid` to the new version. When no
indexed column changes and the new tuple fits on the same page, it can create a
HOT update and avoid new index entries. A cross-partition update is the literal
exception: `ExecCrossPartitionUpdate()` deletes the tuple from the old leaf and
inserts it into the new leaf.

## Concurrency

PostgreSQL combines relation locks, buffer synchronization, tuple locks, and
MVCC. Each mechanism protects a different thing:

| Mechanism | DDL | `UPDATE` / `DELETE` | `INSERT` | Plain `SELECT` |
| --- | --- | --- | --- | --- |
| Relation lock | Varies; often `AccessExclusiveLock` | `RowExclusiveLock` | `RowExclusiveLock` | `AccessShareLock` |
| Buffer pin | Pins buffers as needed | Pins old and new heap pages | Pins the destination page | The scan and slot pin the current page |
| Buffer content lock | Varies by operation | Exclusive while changing a page | Exclusive while adding tuples | Shared while collecting visible offsets |
| Tuple lock | Usually none | Stores ownership in `xmax`; may use a heavyweight lock while waiting | None for the new tuple | None; locking clauses are exceptions |
| MVCC metadata | Not the primary mechanism | Sets `xmax`; an update creates a new version | Sets `xmin` | Tests `xmin`, `xmax`, and infomask against the snapshot |

`AccessShareLock` and `RowExclusiveLock` are compatible, so ordinary reads and
writes can run together. `AccessShareLock` conflicts with `AccessExclusiveLock`,
so a scan protects its relation from `DROP TABLE`, `TRUNCATE`, and DDL that
requires exclusive access.

A page-at-a-time heap scan uses the page synchronization mechanisms in this
order:

```text
reader
  -> pin the buffer
  -> take a shared content lock
    -> inspect line pointers and evaluate MVCC visibility
  -> release the content lock
    -> follow tuple pointers and deform attributes without the content lock
  -> release the pins when the scan and slot leave the page

writer
  -> pin the buffer
  -> take an exclusive content lock
    -> add a tuple or change tuple metadata
  -> release the content lock and pin

VACUUM or page pruning
  -> check that the visibility horizon permits removal
  -> acquire a cleanup lock after competing pins leave
    -> physically remove or move dead tuples
```

The buffer pin keeps the page in the buffer pool and keeps tuple addresses
stable. It does not prevent ordinary changes to that page. For example, a
concurrent `DELETE` can take the exclusive content lock and set the tuple's
`xmax` while a reader keeps the page pinned. The delete does not remove the
tuple bytes, so the reader can continue using the version selected by its
snapshot.

The shared content lock protects the reader only while
`heap_prepare_pagescan()` examines line pointers and visibility. After that
function collects the visible offsets, the reader releases the content lock.
The retained pin prevents `VACUUM` and page pruning from obtaining the cleanup
lock that they need to move or remove those tuples. Tuple-at-a-time mode instead
takes and releases the shared content lock for each selected tuple.

Normal updates also preserve user data in the old tuple version. They create a
new tuple version and connect the versions through tuple metadata. A reader's
snapshot chooses the appropriate version, while tuple locks coordinate writers
that target the same row. Plain `SELECT` never waits on those tuple locks.

Finally, the all-visible page flag lets a heap scan skip per-tuple visibility
checks outside recovery. A writer clears that flag and the corresponding
visibility-map bits before it makes the page no longer all-visible, so the fast
path preserves the same snapshot semantics.
