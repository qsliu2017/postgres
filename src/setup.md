# Set up Development Environment

## Clone, Build

```sh
git clone https://github.com/postgres/postgres
cd postgres

./configure \
  --prefix=$(pwd)/.install \
  --enable-debug
bear - make
make install
```

## Configure Clangd

`compile_commands.json`: make with [`bear`](https://github.com/rizsotto/Bear)

`.clangd`: 
```yaml
{{#include ../../.clangd}}
```

## Configure `.vscode/launch.json`

```json
{{#include ../../.vscode/launch.json}}
```
