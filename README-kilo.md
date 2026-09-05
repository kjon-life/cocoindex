# cocoindex fork — local dev runbook

Verified end-to-end 2026-09-05 on macOS arm64. mise 2026.7.5, uv 0.8.23 (00d3aa378), Docker Desktop.
Fork: `git@github.com:kjon-life/cocoindex.git` (cocoindex v1 — internal state is LMDB, not Postgres; see § Postgres).

## Fresh clone → working build

```bash
git clone git@github.com:kjon-life/cocoindex.git && cd cocoindex
mise trust               # ./mise.toml is committed; nothing runs until trusted
mise install             # python 3.11, uv 0.8.23, rust 1.89 — pinned in ./mise.toml
uv sync --locked         # deps only; does NOT install cocoindex itself
uv run maturin develop   # builds the Rust core, installs cocoindex into .venv (AGENTS.md; required)
```

`./mise.toml` also sets `UV_PYTHON_PREFERENCE=only-system` — uv must use mise's interpreter and may never download its own CPython. Global `python.uv_venv_auto = "source"` auto-activates `.venv` on cd.

## Verification (all ran 2026-09-05, all passed)

```bash
# 1. Toolchain resolves from ./mise.toml, not the ~/Dev cascade
mise ls --current | grep -E 'python|uv |rust'
#    → python 3.11.11 / uv 0.8.23 / rust 1.89.0, all sourced from ~/Dev/utils/cocoindex/mise.toml

# 2. venv is built on mise's interpreter, not a uv-downloaded CPython
uv run python -c 'import sys; print(sys.base_prefix)'
#    → /Users/trust/.local/share/mise/installs/python/3.11

# 3. Lockfile is honest (fails loudly on pyproject/lock drift)
uv sync --locked         # exit 0

# 4. Package importable (only true after maturin develop)
uv run python -c 'import cocoindex; print(cocoindex.__version__)'
#    → 999.0.0 (dev-build marker version)

# 5. Test suite
TESTCONTAINERS_RYUK_DISABLED=true uv run pytest python/ -q
#    → 1045 passed, 280 skipped (optional deps / platform), 0 failed, 95s
```

## Gotchas hit during setup — do not repeat

1. **`mise use` / `mise set` without `--path` write to the nearest EXISTING config up the tree.**
   With no `./mise.toml` yet, `mise use python@3.11` silently mutated `~/Dev/mise.toml` (the cascade file
   for every project under ~/Dev: python 3.14→3.11, added rust, added `[env]`). Restored by hand,
   then created `./mise.toml` explicitly. On any repo without its own mise.toml, always:
   `mise use --path ./mise.toml <tool@ver> …` and `edit the file directly for [env]`.
2. **`mise trust` is mandatory** after creating or editing a mise.toml with `[env]` — every mise
   invocation errors until then.
3. **`uv sync --locked` failed at first — correctly.** The fork's pyproject had added `aiohttp` and
   `pymysql` to the ci/dev groups without regenerating `uv.lock`. Resolution: `uv lock`, review the
   diff (12 lines, exactly those two packages), commit lock + mise.toml together (`2c30e90e`).
   Rule: a `--locked` failure means drift; never bypass it with a plain `uv sync`.
4. **`uv sync` does not install cocoindex.** The root pyproject is deps-only (hence the expected
   "Skipping installation of entry points… not packaged" warning). `import cocoindex` fails until
   `uv run maturin develop` has run. Re-run it after every Rust change.
5. **Postgres-target tests error with "container did not become running":** testcontainers' Ryuk
   reaper container cannot start on this Docker setup; the pgvector image itself runs fine (probed
   manually). Export `TESTCONTAINERS_RYUK_DISABLED=true` for test runs. Trade-off: no automatic
   container cleanup — check `docker ps` after aborted runs.
6. **Ordering matters:** with `UV_PYTHON_PREFERENCE=only-system`, uv cannot find python 3.11 until
   mise has put it on this directory's PATH. mise first, uv second — always.

## Internal state: LMDB (nothing to install)

cocoindex v1 keeps its incremental-processing bookkeeping in LMDB via the Rust `heed` crate,
compiled into the extension — no Python package, no system library, no service.
Configuration is a directory path plus two optional tunables:

- `COCOINDEX_DB` (or `db_path` on Environment/Settings) — where the LMDB env lives
- `COCOINDEX_LMDB_MAX_DBS` (default 1024), `COCOINDEX_LMDB_MAP_SIZE` (default 4 GiB *initial*;
  the engine doubles-and-retries, so it is not a cap)

This store holds component-run tracking, memoization, and **ownership of target states**. It is not
queryable and not the product data — but losing it means cocoindex can no longer clean up stale rows
it wrote to targets. Treat it as durable; keep it on a real disk and in backups.

## Postgres / pgvector (targets, not bookkeeping)

Postgres appears only as a connector: source (`PgTableSource`) and target (`declare_table_target`,
auto schema, ivfflat/hnsw index management, runs `CREATE EXTENSION IF NOT EXISTS vector` itself).
Product data — chunks, embeddings, anything the application queries — lives in a target, so a flow
doing vector search in SQL needs Postgres+pgvector regardless of LMDB.

Local server facts (verified): mise postgres 18.1 bundles pgvector 0.8.1 (`lib/vector.dylib` +
extension SQL present — no compile step). `vector` is **not** a trusted extension
(`vector.control`), so a superuser must create it once per database; the connector's
`IF NOT EXISTS` then no-ops for the unprivileged role. `asyncpg` is already in the dev group.

One-time setup, isolated from kilosynthesis on the same server:

```sql
-- as superuser:
CREATE ROLE cocoindex LOGIN PASSWORD '<pw>';
CREATE DATABASE cocoindex OWNER cocoindex;
\c cocoindex
CREATE EXTENSION vector;
```

DSN for flows: `postgresql://cocoindex:<pw>@localhost:5432/cocoindex`

## Routine commands (from AGENTS.md)

```bash
uv run maturin develop && cargo test    # after Rust changes
uv run mypy && uv run pytest python/    # after Python changes
uv run ruff format . && uv run ruff check .
prek run --all-files                    # before pushing (uv tool install prek, once)
```
