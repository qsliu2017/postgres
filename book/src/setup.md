# Set up Development Environment

## Clone, Build

```sh
git clone https://github.com/postgres/postgres
cd postgres

./configure \
  --prefix=$(pwd)/.install \
  --enable-debug \
  --enable-cassert \
  --without-icu
bear - make
make install
export PATH=$(pwd)/.install/bin:$PATH
```

## Initial database cluster

```sh
initdb -D .data
```

## Configure Clangd

`compile_commands.json`: make with [`bear`](https://github.com/rizsotto/Bear)

`.clangd`: 
```yaml
If:
  PathMatch: .*\.h$
CompileFlags:
  Add:
    - -include postgres.h
```

## Debug

### Configure `.vscode/launch.json`

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug PostgreSQL Backend",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/.install/bin/postgres",
            "args": [
                "-D", "${workspaceFolder}/.data",
                "-p", "5432",
                "-c", "log_statement=all",
                "-c", "log_min_messages=debug1"
            ],
            "cwd": "${workspaceFolder}",
            "environment": [
                {
                    "name": "PGDATA",
                    "value": "${workspaceFolder}/.data"
                },
                {
                    "name": "PATH",
                    "value": "${workspaceFolder}/.install/bin:${env:PATH}"
                }
            ],
            "MIMode": "lldb"
        },
        {
            "name": "Attach to PostgreSQL Backend",
            "type": "cppdbg",
            "request": "attach",
            "program": "${workspaceFolder}/.install/bin/postgres",
            "processId": "${command:pickProcess}",
            "MIMode": "lldb"
        }
    ]
}
```

### Debug PostMaster

### Debug Worker

```sh
pg_ctl -D .data -l .data/logfile start
psql postgres
```

Then check which process is serving this session.

```sql
postgres=# select pg_backend_pid();
 pg_backend_pid 
----------------
          59407
(1 row)

```

Attach to this pid.
