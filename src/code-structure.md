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
The former is responsible for managing pages on disk and its buffer in shared memory.
The later is about formats in one page.
