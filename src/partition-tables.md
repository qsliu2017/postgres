# Partition Tables

> A partitioned table is catalog metadata and has no heap of its own. Rows and ordinary indexes live in leaf partitions.

```sql
CREATE TABLE events (
  id bigint,
  happened_at date NOT NULL,
  payload text
) PARTITION BY RANGE (happened_at);

CREATE TABLE events_2024 PARTITION OF events
  FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
CREATE TABLE events_2025 PARTITION OF events
  FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

CREATE INDEX events_happened_at_idx ON events (happened_at);
```

```text
partitioned parent (`events`, no relation storage)
  -> partitioned index (`events_happened_at_idx`, no relation storage)
  -> leaf `events_2024` (heap + leaf index)
  -> leaf `events_2025` (heap + leaf index)
```

## Object and Storage

A PostgreSQL **object** is a logical database entity represented by one or more system-catalog rows.

<details>
<summary>An <code>ObjectAddress</code> identifies an object by <code>(classId, objectId, objectSubId)</code>.</summary>

```c
/* src/include/catalog/objectaddress.h */
typedef struct ObjectAddress
{
    Oid   classId;      /* Class Id from pg_class */
    Oid   objectId;     /* OID of the object */
    int32 objectSubId;  /* Subitem within object (eg column), or 0 */
} ObjectAddress;
```

</details>

`classId` is the `pg_class` OID of the catalog that defines the object. Common defining catalogs are:

- `pg_class`: relations such as tables, indexes, views, and sequences
- `pg_type`: data types
- `pg_proc`: functions and procedures
- `pg_namespace`: schemas

`objectId` identifies the object within that catalog. `objectSubId` is zero for a complete object and identifies a subobject, such as a table column, when nonzero.

```text
(pg_class's OID,     table OID,    0) -> table
(pg_proc's OID,      function OID, 0) -> function
(pg_namespace's OID, schema OID,   0) -> schema
(pg_class's OID,     table OID,    2) -> second column of the table
```

**Relation storage** is the set of physical relation files managed by PostgreSQL's storage manager (`smgr`). Only some relation kinds in `pg_class` own relation storage.

| Relation object   | `pg_class` row and OID | Own relation storage |
| ----------------- | ---------------------: | -------------------: |
| Ordinary table    |                    yes |                  yes |
| Leaf partition    |                    yes |                  yes |
| Partitioned table |                    yes |                   no |
| Ordinary index    |                    yes |                  yes |
| Partitioned index |                    yes |                   no |
| View              |                    yes |                   no |

<details>
<summary><code>pg_class.relfilenode</code> identifies the relation's current physical relation files.</summary>

```c
/* src/include/catalog/pg_class.h */
/* identifier of physical storage file */
/* relfilenode == 0 means it is a "mapped" relation, see relmapper.c */
Oid relfilenode BKI_DEFAULT(0);
```

</details>

A **relfilenumber** (`RelFileNumber`, historically called `relfilenode`) is the file-number component of a physical relation locator. It is not the relation's logical OID.

<details>
<summary>A physical relation locator contains a tablespace OID, database OID, and relfilenumber.</summary>

```text
relation OID
  -> pg_class row / relcache entry
  -> (tablespace OID, database OID, relfilenumber)
  -> physical relation files
```

</details>

A new ordinary relation commonly starts with a relfilenumber equal to its OID. The values can diverge after a relation rewrite. The OID remains the logical identity while PostgreSQL replaces the relation storage. A `relfilenode` value of zero identifies a mapped relation; PostgreSQL gets its relfilenumber from the relation mapper instead of `pg_class`.

A relation can use several physical files:

```text
12345       main fork
12345_fsm   free-space map
12345_vm    visibility map
12345_init  initialization fork for an unlogged relation
12345.1     a later segment after the main fork grows large
```

A table's TOAST table and indexes are separate relation objects. Each has its
own OID and, when applicable, its own relfilenumber and files.

<details>
<summary><code>heap_create()</code> disables <code>create_storage</code> for relation kinds without relation storage.</summary>

```c
/* src/backend/catalog/heap.c:heap_create */
if (!RELKIND_HAS_STORAGE(relkind))
    create_storage = false;
```

</details>

A partitioned table has no relation storage. Its OID identifies the hierarchy root and its partitioning metadata. Heap and index files belong to leaf partitions. DML and scans must therefore select a leaf.

## DDL

```text
CREATE TABLE grammar (`gram.y`)
  -> ProcessUtilitySlow (`utility.c`)
  -> transformCreateStmt (`parse_utilcmd.c`)
  -> DefineRelation (`tablecmds.c`)
  -> heap_create_with_catalog (`heap.c`)
  -> partition-specific catalog updates
```

`CREATE TABLE` accepts optional `PARTITION BY` and `PARTITION OF` clauses. In `CreateStmt`:

- `PARTITION BY` sets `partspec`.
- `PARTITION OF` adds the parent to `inhRelations` and sets `partbound`.
- A sub-partitioned partition sets both `partbound` and `partspec`.

<details>
<summary><code>ProcessUtilitySlow()</code> transforms and executes <code>CREATE TABLE</code> and its generated subcommands.</summary>

```c
/* src/backend/tcop/utility.c:ProcessUtilitySlow */
stmts = transformCreateStmt((CreateStmt *) parsetree, queryString);

while (stmts != NIL)
{
    Node *stmt = (Node *) linitial(stmts);
    /* ... */
    if (IsA(stmt, CreateStmt))
    {
        CreateStmt *cstmt = (CreateStmt *) stmt;

        address = DefineRelation(cstmt,
                                 RELKIND_RELATION,
                                 InvalidOid, NULL,
                                 queryString);
        /* ... */
        NewRelationCreateToastTable(address.objectId, toast_options);
    }
    else
        ProcessUtility(/* generated subsidiary command */);
}
```

</details>

`ProcessUtilitySlow()` passes `RELKIND_RELATION`. `DefineRelation()` changes it after detecting `partspec`. PostgreSQL transforms partition key and bound expressions after it creates and opens the relation.

### Creating the partitioned parent

```text
DefineRelation
  -> relkind = RELKIND_PARTITIONED_TABLE
  -> heap_create_with_catalog
       -> common pg_class / pg_attribute / pg_type / dependency rows
       -> no physical relation file
  -> ComputePartitionAttrs
  -> StorePartitionKey
       -> pg_partitioned_table
       -> dependencies for key columns, opclasses, collations, expressions
```

<details>
<summary><code>DefineRelation()</code> marks a statement with <code>partspec</code> as <code>RELKIND_PARTITIONED_TABLE</code>.</summary>

```c
/* src/backend/commands/tablecmds.c:DefineRelation */
if (stmt->partspec != NULL)
{
    if (relkind != RELKIND_RELATION)
        elog(ERROR, "unexpected relkind: %d", (int) relkind);

    relkind = RELKIND_PARTITIONED_TABLE;
    partitioned = true;
}
else
    partitioned = false;
```

</details>

`heap_create_with_catalog()` then runs the common relation-creation path. It creates `pg_class`, `pg_attribute`, and `pg_type` rows, plus any dependency, default, and constraint rows. The partition path differs in storage and partition-specific catalogs.

<details>
<summary>A partitioned parent has <code>relkind = 'p'</code>. This kind is absent from <code>RELKIND_HAS_STORAGE</code> and <code>RELKIND_HAS_TABLE_AM</code>.</summary>

```c
/* src/include/catalog/pg_class.h */
#define RELKIND_PARTITIONED_TABLE 'p'

#define RELKIND_HAS_STORAGE(relkind) \
    ((relkind) == RELKIND_RELATION || \
     (relkind) == RELKIND_INDEX || \
     /* ... */)

#define RELKIND_HAS_PARTITIONS(relkind) \
    ((relkind) == RELKIND_PARTITIONED_TABLE || \
     (relkind) == RELKIND_PARTITIONED_INDEX)

#define RELKIND_HAS_TABLE_AM(relkind) \
    ((relkind) == RELKIND_RELATION || \
     /* ... */)
```

</details>

<details>
<summary><code>heap_create()</code> suppresses file creation for the partitioned parent.</summary>

```c
/* src/backend/catalog/heap.c:heap_create */
if (!RELKIND_HAS_STORAGE(relkind))
    create_storage = false;
```

</details>

<details>
<summary><code>needs_toast_table()</code> rejects partitioned tables.</summary>

```c
/* src/backend/catalog/toasting.c:needs_toast_table */
if (rel->rd_rel->relkind == RELKIND_PARTITIONED_TABLE)
    return false;
```

</details>

The parent has no heap file, table-AM scan target, or TOAST table. Its tablespace and access-method settings provide defaults for partitions.

`pg_partitioned_table` stores one row for each partitioned table. It defines the table's partition key. It does not store child bounds; each child bound is stored in `pg_class.relpartbound`.

<details>
<summary><code>FormData_pg_partitioned_table</code> defines the <code>pg_partitioned_table</code> catalog schema.</summary>

```c
/* src/include/catalog/pg_partitioned_table.h:FormData_pg_partitioned_table */
CATALOG(pg_partitioned_table,3350,PartitionedRelationId)
{
    Oid         partrelid BKI_LOOKUP(pg_class);
    char        partstrat;
    int16       partnatts;
    Oid         partdefid BKI_LOOKUP_OPT(pg_class);
    int2vector  partattrs BKI_FORCE_NOT_NULL;

#ifdef CATALOG_VARLEN
    oidvector   partclass BKI_LOOKUP(pg_opclass) BKI_FORCE_NOT_NULL;
    oidvector   partcollation BKI_LOOKUP_OPT(pg_collation) BKI_FORCE_NOT_NULL;
    pg_node_tree partexprs;
#endif
} FormData_pg_partitioned_table;
```

</details>

The columns have these roles:

- `partrelid`: OID of the partitioned relation; also the catalog's primary key
- `partstrat`: strategy (`h` for hash, `l` for list, or `r` for range)
- `partnatts`: number of partition key fields
- `partdefid`: OID of the default partition, or zero
- `partattrs`: parent attribute number for each key field; zero denotes an expression
- `partclass`: operator class for each key field
- `partcollation`: collation for each key field
- `partexprs`: serialized expression trees for zero entries in `partattrs`

<details>
<summary><code>DefineRelation()</code> transforms and stores the partition key.</summary>

```c
/* src/backend/commands/tablecmds.c:DefineRelation */
stmt->partspec = transformPartitionSpec(rel, stmt->partspec);

ComputePartitionAttrs(pstate, rel, stmt->partspec->partParams,
                      partattrs, &partexprs, partopclass,
                      partcollation, stmt->partspec->strategy);

StorePartitionKey(rel, stmt->partspec->strategy, partnatts, partattrs,
                  partexprs, partopclass, partcollation);
```

</details>

<details>
<summary><code>StorePartitionKey()</code> inserts the partition key into <code>pg_partitioned_table</code>.</summary>

```c
/* src/backend/catalog/heap.c:StorePartitionKey */
values[Anum_pg_partitioned_table_partrelid - 1] =
    ObjectIdGetDatum(RelationGetRelid(rel));
values[Anum_pg_partitioned_table_partstrat - 1] = CharGetDatum(strategy);
values[Anum_pg_partitioned_table_partnatts - 1] = Int16GetDatum(partnatts);
values[Anum_pg_partitioned_table_partdefid - 1] =
    ObjectIdGetDatum(InvalidOid);
values[Anum_pg_partitioned_table_partattrs - 1] = PointerGetDatum(partattrs_vec);
values[Anum_pg_partitioned_table_partclass - 1] = PointerGetDatum(partopclass_vec);
values[Anum_pg_partitioned_table_partcollation - 1] =
    PointerGetDatum(partcollation_vec);
values[Anum_pg_partitioned_table_partexprs - 1] = partexprDatum;

CatalogTupleInsert(pg_partitioned_table, tuple);
```

</details>

### Creating a leaf with `PARTITION OF`

```text
DefineRelation
  -> MergeAttributes from parent
  -> heap_create_with_catalog
       -> ordinary relation catalogs
       -> leaf relation storage
  -> transformPartitionBound
  -> check_new_partition_bound
  -> StorePartitionBound
       -> pg_class.relpartbound
       -> pg_class.relispartition = true
  -> StoreCatalogInheritance
       -> pg_inherits
       -> child-to-parent dependency
```

A leaf uses `RELKIND_RELATION` unless it is also partitioned. In this example, `events_2024` owns heap relation storage and has `pg_class.relispartition = true`.

<details>
<summary><code>DefineRelation()</code> validates and stores the leaf's partition bound.</summary>

```c
/* src/backend/commands/tablecmds.c:DefineRelation */
bound = transformPartitionBound(pstate, parent, stmt->partbound);
check_new_partition_bound(relname, parent, bound, pstate);

/* Update the pg_class entry. */
StorePartitionBound(rel, parent, bound);

/* Store inheritance information for new rel. */
StoreCatalogInheritance(relationId, inheritOids, stmt->partbound != NULL);
```

</details>

<details>
<summary><code>StorePartitionBound()</code> stores the serialized bound in <code>pg_class.relpartbound</code> and marks the relation as a partition.</summary>

```c
/* src/backend/catalog/heap.c:StorePartitionBound */
new_val[Anum_pg_class_relpartbound - 1] =
    CStringGetTextDatum(nodeToString(bound));
new_repl[Anum_pg_class_relpartbound - 1] = true;
newtuple = heap_modify_tuple(tuple, RelationGetDescr(classRel),
                             new_val, new_null, new_repl);
((Form_pg_class) GETSTRUCT(newtuple))->relispartition = true;

CatalogTupleUpdate(classRel, &newtuple->t_self, newtuple);
```

</details>

<details>
<summary><code>StoreSingleInheritance()</code> stores the parent-child edge in <code>pg_inherits</code>.</summary>

```c
/* src/backend/catalog/pg_inherits.c:StoreSingleInheritance */
values[Anum_pg_inherits_inhrelid - 1] = ObjectIdGetDatum(relationId);
values[Anum_pg_inherits_inhparent - 1] = ObjectIdGetDatum(parentOid);
values[Anum_pg_inherits_inhseqno - 1] = Int32GetDatum(seqNumber);
values[Anum_pg_inherits_inhdetachpending - 1] = BoolGetDatum(false);

CatalogTupleInsert(inhRelation, tuple);
```

</details>

<details>
<summary>A partition uses an <code>AUTO</code> dependency; ordinary inheritance uses a <code>NORMAL</code> dependency.</summary>

```c
/* src/backend/commands/tablecmds.c */
#define child_dependency_type(child_is_partition) \
    ((child_is_partition) ? DEPENDENCY_AUTO : DEPENDENCY_NORMAL)
```

</details>

Ordinary `INHERITS` also uses `pg_inherits`, but it creates a `NORMAL` dependency and has no partition metadata. For a default partition, `StorePartitionBound()` also stores the leaf OID in `pg_partitioned_table.partdefid`.

The catalogs distinguish each relation type as follows:

| Object                | `pg_class`                                                                | `pg_partitioned_table`     | `pg_inherits`                    | Relation storage |
| --------------------- | ------------------------------------------------------------------------- | -------------------------- | -------------------------------- | ---------------- |
| Normal heap           | `relkind = 'r'`, `relispartition = false`                                 | none                       | none normally                    | yes              |
| Partitioned parent    | `relkind = 'p'`                                                           | partition strategy and key | row only if it is itself a child | no               |
| Leaf partition        | usually `relkind = 'r'`, `relispartition = true`, bound in `relpartbound` | none                       | child-to-parent row              | yes              |
| Sub-partitioned child | `relkind = 'p'`, `relispartition = true`, bound                           | its own key                | child-to-parent row              | no               |

A parent index follows the same model. It has `relkind = 'I'` and no relation storage. Its leaf indexes are ordinary index relations.

### Leaf column definitions

```text
PARTITION OF
  -> MergeAttributes(parent)
  -> BuildDescForRelation
  -> heap_create_with_catalog
       -> independent pg_class row
       -> independent pg_attribute rows
       -> independent pg_type row

ATTACH PARTITION
  -> MergeAttributesIntoExisting
       -> match columns by name
       -> validate type, collation, and constraints
```

The parent and each leaf have separate column definitions. The parent defines the logical row type. Each leaf has its own `TupleDesc` and `pg_attribute` rows.

<details>
<summary><code>DefineRelation()</code> builds a new partition's descriptor from its parent before creating the leaf relation.</summary>

```c
/* src/backend/commands/tablecmds.c:DefineRelation */
stmt->tableElts =
    MergeAttributes(stmt->tableElts, inheritOids,
                    stmt->relation->relpersistence,
                    stmt->partbound != NULL,
                    &old_constraints);

descriptor = BuildDescForRelation(stmt->tableElts);

relationId = heap_create_with_catalog(/* ... */, descriptor, /* ... */);
```

</details>

An attached table must expose the same logical columns, but its physical attribute numbers can differ.

<details>
<summary><code>MergeAttributesIntoExisting()</code> matches an attached table's columns by name.</summary>

```c
/* src/backend/commands/tablecmds.c:MergeAttributesIntoExisting */
for (AttrNumber parent_attno = 1;
     parent_attno <= parent_desc->natts;
     parent_attno++)
{
    Form_pg_attribute parent_att =
        TupleDescAttr(parent_desc, parent_attno - 1);
    char *parent_attname = NameStr(parent_att->attname);

    tuple = SearchSysCacheCopyAttName(RelationGetRelid(child_rel),
                                      parent_attname);

    /* Validate type, typmod, collation, NOT NULL, and generation state. */
}
```

</details>

Tuple-conversion maps translate between parent and leaf attribute numbers. Structural changes on the parent recurse through the hierarchy and update each leaf's catalog rows. Leaves can still have distinct defaults, indexes, additional constraints, statistics, and storage settings.

## DML

### `INSERT`: route one row to a leaf

```text
ExecModifyTable
  -> ExecInsert
       -> ExecPrepareTupleRouting
            -> ExecFindPartition
       -> table_tuple_insert(leaf relation)
       -> ExecInsertIndexTuples(leaf indexes)
```

`ExecModifyTable()` reads one tuple from its subplan and calls `ExecInsert()`. A partitioned target has a `PartitionTupleRouting` structure.

<details>
<summary><code>ExecInsert()</code> replaces the root <code>ResultRelInfo</code> with the selected leaf before writing the row.</summary>

```c
/* src/backend/executor/nodeModifyTable.c:ExecInsert */
if (proute)
{
    ResultRelInfo *partRelInfo;

    slot = ExecPrepareTupleRouting(mtstate, estate, proute,
                                   resultRelInfo, slot,
                                   &partRelInfo);
    resultRelInfo = partRelInfo;
}

resultRelationDesc = resultRelInfo->ri_RelationDesc;
```

</details>

<details>
<summary><code>ExecFindPartition()</code> evaluates the partition key and finds a matching child at each partitioning level.</summary>

```c
/* src/backend/executor/execPartition.c:ExecFindPartition */
while (dispatch != NULL)
{
    /* ... */
    rel = dispatch->reldesc;
    partdesc = dispatch->partdesc;

    ecxt->ecxt_scantuple = slot;
    FormPartitionKeyDatum(dispatch, slot, estate, values, isnull);

    if (partdesc->nparts == 0 ||
        (partidx = get_partition_for_tuple(dispatch, values, isnull)) < 0)
        ereport(ERROR,
                (errmsg("no partition of relation \"%s\" found for row",
                        RelationGetRelationName(rel))));

    is_leaf = partdesc->is_leaf[partidx];
    /* Return the leaf, or descend into a sub-partitioned child. */
}
```

</details>

<details>
<summary><code>ExecPrepareTupleRouting()</code> converts the root tuple to the selected leaf's physical attribute layout when needed.</summary>

```c
/* src/backend/executor/nodeModifyTable.c:ExecPrepareTupleRouting */
partrel = ExecFindPartition(mtstate, targetRelInfo, proute, slot, estate);

map = ExecGetRootToChildMap(partrel, estate);
if (map != NULL)
{
    TupleTableSlot *new_slot = partrel->ri_PartitionTupleSlot;

    slot = execute_attr_map_slot(map->attrMap, slot, new_slot);
}
```

</details>

PostgreSQL initializes each leaf's `ResultRelInfo` on first use. A single-row insert does not open every partition. After routing, `table_tuple_insert()` and `ExecInsertIndexTuples()` enter the normal leaf heap and index paths.

An insert through the partitioned parent has additional cost:

- Evaluate and route each row.
- Search again at each sub-partitioning level.
- Convert the tuple when the leaf layout differs.

When traffic is distributed across leaves, partitioning may:

- Update smaller leaf indexes.
- Improve cache and data locality.
- Separate hot heap and index pages across leaves.

### Multiple rows and batch insertion

```text
multi-row SQL INSERT
  -> ExecModifyTable
       -> route and insert one row at a time

COPY FROM
  -> ExecFindPartition for each row
  -> buffer rows by leaf
  -> table_multi_insert for each leaf buffer
```

<details>
<summary><code>ExecModifyTable()</code> processes a multi-row SQL insert one row at a time.</summary>

```c
/* src/backend/executor/nodeModifyTable.c:ExecModifyTable */
for (;;)
{
    /* fetch the next row from the subplan */
    context.planSlot = ExecProcNode(subplanstate);
    if (TupIsNull(context.planSlot))
        break;

    /* ... */
    slot = ExecInsert(&context, resultRelInfo, slot,
                      node->canSetTag, NULL, NULL);
}
```

</details>

Each row may route to a different leaf and reaches `table_tuple_insert()` separately. PostgreSQL reuses routing state and initialized leaf state. `ExecBatchInsert()` handles FDWs that implement `ExecForeignBatchInsert`; it does not batch local heap inserts.

<details>
<summary><code>COPY FROM</code> routes each row and maintains a buffer for each destination leaf.</summary>

```c
/* src/backend/commands/copyfrom.c:CopyFrom */
resultRelInfo = ExecFindPartition(mtstate, target_resultRelInfo,
                                  proute, myslot, estate);

/* ... determine whether this leaf supports multi-insert ... */
if (leafpart_use_multi_insert)
{
    if (resultRelInfo->ri_CopyMultiInsertBuffer == NULL)
        CopyMultiInsertInfoSetupBuffer(&multiInsertInfo, resultRelInfo);
}
```

</details>

<details>
<summary><code>CopyMultiInsertBufferFlush()</code> writes one leaf buffer with a single table-AM call.</summary>

```c
/* src/backend/commands/copyfrom.c:CopyMultiInsertBufferFlush */
table_multi_insert(resultRelInfo->ri_RelationDesc,
                   slots,
                   nused,
                   mycid,
                   ti_options,
                   buffer->bistate);
```

</details>

Triggers, volatile expressions, and an FDW without batch support can force `COPY` to use single-row insertion:

```text
multi-row SQL INSERT: route row -> insert row -> repeat
COPY FROM:            route rows -> group by leaf -> table_multi_insert per leaf
```

### `DELETE`

```text
partition expansion and pruning
  -> leaf scan
       -> leaf ResultRelInfo + ctid
  -> ExecDelete
       -> ExecDeleteAct
       -> table_tuple_delete(leaf relation)
```

`DELETE` does not route rows by value. The planner expands and prunes the hierarchy. Each scan row carries the source leaf OID in a junk `tableoid` column and the row identity in `ctid`.

<details>
<summary><code>ExecModifyTable()</code> uses <code>tableoid</code> to select the leaf's <code>ResultRelInfo</code>.</summary>

```c
/* src/backend/executor/nodeModifyTable.c:ExecModifyTable */
if (AttributeNumberIsValid(node->mt_resultOidAttno))
{
    datum = ExecGetJunkAttribute(context.planSlot,
                                 node->mt_resultOidAttno,
                                 &isNull);
    resultoid = DatumGetObjectId(datum);

    if (resultoid != node->mt_lastResultOid)
        resultRelInfo = ExecLookupResultRelByOid(node, resultoid,
                                                 false, true);
}
```

</details>

`ExecDelete()` then calls `table_tuple_delete()` on that leaf. Tuple deletion and later index cleanup follow the normal heap path.

### `UPDATE`

```text
leaf scan
  -> ExecUpdate
       -> ExecUpdateAct
            -> partition constraint passes
                 -> table_tuple_update(current leaf)
            -> partition constraint fails
                 -> ExecCrossPartitionUpdate
                      -> ExecDelete(old leaf)
                      -> ExecInsert(root)
                           -> route to new leaf
```

An update starts on the leaf selected by the scan:

1. If the new row satisfies the leaf's partition constraint, `table_tuple_update()` updates the leaf and its indexes.
2. If the new row violates the constraint, PostgreSQL deletes it from the old leaf and routes an insert to the new leaf.

<details>
<summary><code>ExecUpdateAct()</code> chooses an in-leaf update or a cross-partition update.</summary>

```c
/* src/backend/executor/nodeModifyTable.c:ExecUpdateAct */
partition_constraint_failed =
    resultRelationDesc->rd_rel->relispartition &&
    !ExecPartitionCheck(resultRelInfo, slot, estate, false);

if (partition_constraint_failed)
{
    /* DELETE from source, then route INSERT from the root. */
    if (ExecCrossPartitionUpdate(context, resultRelInfo,
                                 tupleid, oldtuple, slot,
                                 canSetTag, updateCxt,
                                 &result, &retry_slot,
                                 &inserted_tuple, &insert_destrel))
    {
        updateCxt->crossPartUpdate = true;
        return TM_Ok;
    }
    /* concurrent-update retry omitted */
}

/* If the row remains in this leaf, perform an ordinary update. */
result = table_tuple_update(resultRelationDesc, tupleid, slot,
                            estate->es_output_cid,
                            estate->es_snapshot,
                            estate->es_crosscheck_snapshot,
                            true, &context->tmfd,
                            &updateCxt->lockmode,
                            &updateCxt->updateIndexes);
```

</details>

<details>
<summary><code>ExecCrossPartitionUpdate()</code> deletes from the old leaf and inserts through the root.</summary>

```c
/* src/backend/executor/nodeModifyTable.c:ExecCrossPartitionUpdate */
ExecDelete(context, resultRelInfo,
           tupleid, oldtuple,
           false,  /* processReturning */
           true,   /* changingPart */
           false,  /* canSetTag */
           tmresult, &tuple_deleted, &epqslot);

/* Convert the old leaf layout back to the root layout if necessary. */
if (tupconv_map != NULL)
    slot = execute_attr_map_slot(tupconv_map->attrMap,
                                 slot, mtstate->mt_root_tuple_slot);

/* ExecInsert starts routing at the root and finds the new leaf. */
context->cpUpdateReturningSlot =
    ExecInsert(context, mtstate->rootResultRelInfo, slot, canSetTag,
               inserted_tuple, insert_destrel);
```

</details>

The delete and insert form one transactional SQL update, but modify two leaf heaps and their indexes. An update that directly targets a leaf cannot move the row outside that leaf; PostgreSQL reports a partition-constraint violation.

## DQL

### The shared planning path: expand, prune, append

```text
expand_partitioned_rtentry
  -> PartitionDirectoryLookup
  -> prune_append_rel_partitions
       -> gen_partprune_steps
       -> get_matching_partitions
  -> build child RelOptInfo objects
  -> set_append_rel_pathlist
       -> choose an access path for each leaf
       -> build Append paths
  -> make_partition_pruneinfo for runtime pruning
  -> ExecInitAppend
       -> ExecInitPartitionPruning
  -> ExecAppend
```

<details>
<summary><code>expand_partitioned_rtentry()</code> prunes the hierarchy and creates planner relations for surviving children.</summary>

```c
/* src/backend/optimizer/util/inherit.c:expand_partitioned_rtentry */
partdesc = PartitionDirectoryLookup(root->glob->partition_directory,
                                    parentrel);

relinfo->live_parts = live_parts =
    prune_append_rel_partitions(relinfo);

/* ... */
while ((i = bms_next_member(live_parts, i)) >= 0)
{
    Oid childOID = partdesc->oids[i];
    Relation childrel = try_table_open(childOID, lockmode);

    expand_single_inheritance_child(root, parentrte, parentRTindex,
                                    parentrel, top_parentrc, childrel,
                                    &childrte, &childRTindex);
    childrelinfo = build_simple_rel(root, childRTindex, relinfo);
    /* Recurse if this child is also partitioned. */
}
```

</details>

<details>
<summary><code>set_append_rel_pathlist()</code> builds access paths for each surviving child and combines them into append paths.</summary>

```c
/* src/backend/optimizer/path/allpaths.c:set_append_rel_pathlist */
foreach(l, root->append_rel_list)
{
    /* ... locate childRTE and childrel ... */
    set_rel_pathlist(root, childrel, childRTindex, childRTE);

    if (!IS_DUMMY_REL(childrel))
        live_childrels = lappend(live_childrels, childrel);
}

add_paths_to_append_rel(root, rel, live_childrels);
```

</details>

<details>
<summary><code>prune_append_rel_partitions()</code> performs plan-time pruning.</summary>

```c
/* src/backend/partitioning/partprune.c:prune_append_rel_partitions */
if (!enable_partition_pruning || clauses == NIL)
    return bms_add_range(NULL, 0, rel->nparts - 1);

gen_partprune_steps(rel, clauses, PARTTARGET_PLANNER, &gcontext);
if (gcontext.contradictory)
    return NULL;

return get_matching_partitions(&context, gcontext.steps);
```

</details>

`Append` concatenates child results without sorting or duplicate removal. It is similar to `UNION ALL`, but also represents partition and inheritance scans. `MergeAppend` merges ordered child results. `make_partition_pruneinfo()` records pruning steps for values available only during executor startup or rescan.

<details>
<summary><code>ExecInitAppend()</code> initializes only the matching subplans.</summary>

```c
/* src/backend/executor/nodeAppend.c:ExecInitAppend */
if (node->part_prune_info != NULL)
{
    prunestate = ExecInitPartitionPruning(&appendstate->ps,
                                          list_length(node->appendplans),
                                          node->part_prune_info,
                                          &validsubplans);
    appendstate->as_prune_state = prunestate;
    nplans = bms_num_members(validsubplans);
}
```

</details>

<details>
<summary><code>ExecAppend()</code> reads tuples from each surviving leaf subplan.</summary>

```c
/* src/backend/executor/nodeAppend.c:ExecAppend */
subnode = node->appendplans[node->as_whichplan];
result = ExecProcNode(subnode);

if (!TupIsNull(result))
    return result;

/* Current leaf is exhausted; choose the next surviving subplan. */
if (!node->choose_next_subplan(node) && node->as_nasyncremain == 0)
    return ExecClearTuple(node->ps.ps_ResultTupleSlot);
```

</details>

For a parameterized plan, `ExecFindMatchingSubPlans()` runs again when a `PARAM_EXEC` value changes. The same expansion, pruning, and append path supports table and index scans. PostgreSQL chooses a scan method for each leaf, so one `Append` can contain both sequential and index scans.

### Table scan

```text
surviving leaf RelOptInfo
  -> set_rel_pathlist
  -> create_seqscan_plan
       -> scan_relid = leaf RT index
  -> SeqNext
       -> normal table-AM scan
```

For example:

```sql
SELECT * FROM events
WHERE happened_at >= DATE '2025-02-01';
```

pruning may remove `events_2024`. A typical shape is:

```text
Append
  -> Seq Scan on events_2025
```

<details>
<summary><code>create_seqscan_plan()</code> creates a scan for the selected leaf relation.</summary>

```c
/* src/backend/optimizer/plan/createplan.c:create_seqscan_plan */
Index scan_relid = best_path->parent->relid;

scan_plan = make_seqscan(tlist,
                         scan_clauses,
                         scan_relid);
```

</details>

The partition layer ends after selecting the leaf RT index. `SeqNext()` then runs the normal table-AM scan.

### Index scan

```text
surviving leaf RelOptInfo
  -> create_index_paths for leaf indexes
  -> create_indexscan_plan
       -> baserelid = leaf RT index
       -> indexoid = leaf index OID
  -> IndexNext
       -> normal index scan
```

An index declared on the parent is a partitioned index. It defines an index hierarchy but is not scanned. The planner considers each leaf index and may produce:

```text
Append
  -> Index Scan using events_2024_happened_at_idx on events_2024
  -> Index Scan using events_2025_happened_at_idx on events_2025
```

<details>
<summary><code>create_indexscan_plan()</code> records the selected leaf table and leaf index in the scan plan.</summary>

```c
/* src/backend/optimizer/plan/createplan.c:create_indexscan_plan */
Index baserelid = best_path->path.parent->relid;
IndexOptInfo *indexinfo = best_path->indexinfo;
Oid indexoid = indexinfo->indexoid;

/* ... */
scan_plan = (Scan *) make_indexscan(/* ... */,
                                    baserelid,
                                    indexoid,
                                    /* ... */);
```

</details>

Partitioning determines which leaf scan plans exist and execute. `IndexNext()` and the index AM then run an ordinary scan on the selected leaf index.
