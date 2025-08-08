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
{{#include ../../.clangd}}
```

## Debug

### Configure `.vscode/launch.json`

```json
{{#include ../../.vscode/launch.json}}
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
