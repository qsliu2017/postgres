# Code Structure

The following is the main directories of Postgres codebase.

```
src/backend
├── access
│   ├── heap
│   ├── index
│   └── transam
├── catalog
├── executor
├── main
├── optimizer
├── parser
├── postmaster
└── storage
    ├── buffer
    ├── file
    ├── ipc
    ├── lmgr
    └── smgr
```

We will first look at `storage` and `access`.
The former is responsible for managing pages on disk and their buffers in shared memory.
The latter concerns the formats within a page, whether it is a heap table, B-tree index, or WAL log.
The separation of page management and page format is convenient for extending access methods or storage.
The following are examples:
- `pgvector` is an extension that provides support for storing vector indexes inside Postgres.
- `neon` is a distributed database built on top of Postgres. It replaces local disk management with S3-based page servers.
