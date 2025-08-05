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

`table_tuple_insert` wraps call to relation's table access method (`tableam`). By this way, PostgreSQL can support different table access methods, such as heap, hash, and B-tree.

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

In this chapter, we focus on the physical storage format, so we dive into `RelationPutHeapTuple`.

[[TODO]]
