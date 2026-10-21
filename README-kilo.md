# cocoindex fork — local dev runbook

Verified end-to-end 2026-09-05 on macOS arm64. mise 2026.7.5, uv 0.8.23 (00d3aa378), Docker Desktop.
Fork: `git@github.com:kjon-life/cocoindex.git` (cocoindex v1 — internal state is LMDB, not Postgres; see § Postgres).

## Fresh clone → working build

```bash
git clone git@github.com:kjon-life/cocoindex.git && cd cocoindex
mise trust               # ./mise.toml is committed; nothing runs until trusted
mise install             # python 3.11, uv 0.13.4, rust 1.89 — pinned in ./mise.toml
uv sync --locked         # deps only; does NOT install cocoindex itself
uv run maturin develop   # builds the Rust core, installs cocoindex into .venv (AGENTS.md; required)
```

`./mise.toml` also sets `UV_PYTHON_PREFERENCE=only-system` — uv must use mise's interpreter and may never download its own CPython. Global `python.uv_venv_auto = "source"` auto-activates `.venv` on cd.

## Verification (all ran 2026-09-05, all passed)

```bash
# 1. Toolchain resolves from ./mise.toml, not the ~/Dev cascade
mise ls --current | grep -E 'python|uv |rust'
#    → python 3.11.11 / uv 0.13.4 / rust 1.89.0, all sourced from ~/Dev/utils/cocoindex/mise.toml

# 2. venv is built on mise's interpreter, not a uv-downloaded CPython
uv run python -c 'import sys; print(sys.base_prefix)'
#    → /Users/trust/.local/share/mise/installs/python/3.11

# 3. Lockfile is honest (fails loudly on pyproject/lock drift)
uv sync --locked         # exit 0

# 4. Package importable (only true after maturin develop)
uv run python -c 'import cocoindex; print(cocoindex.__version__)'
#    → 999.0.0 (dev-build marker version)

# 5. Test suite (socket override comes from ./mise.toml [env]; see Gotcha 5)
uv run pytest python/ -q
#    → 1045 passed, 280 skipped (optional deps / platform), 0 failed, 95s

# 6. Type check + full pre-commit hook suite
uv run mypy                 # → Success: no issues found in 282 source files
prek run --all-files        # → 18/18 hooks pass (cargo test/check/fmt, pytest, mypy, maturin,
#    secrets scans, CLI-doc regen). Requires `protoc` (brew install protobuf, per AGENTS.md) —
#    the lance-encoding build script in the lancedb rust example needs it. libprotoc 36.1 installed.
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
5. **Postgres-target tests error with "container did not become running":** the daemon is colima,
   and testcontainers mounts the DOCKER_HOST socket path (`/Volumes/VRMini/colima/default/docker.sock`)
   into its Ryuk reaper container — a path that doesn't exist inside the colima VM, so Ryuk never
   starts. Fix (committed in `./mise.toml [env]`): `TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE=/var/run/docker.sock`
   — the socket path as the VM sees it. Verified: 16/16 postgres tests pass with the reaper alive,
   no orphaned containers. Do NOT use `TESTCONTAINERS_RYUK_DISABLED=true` — it kills crash-cleanup.
6. **Ordering matters:** with `UV_PYTHON_PREFERENCE=only-system`, uv cannot find python 3.11 until
   mise has put it on this directory's PATH. mise first, uv second — always.
7. **Rust pin ≠ MSRV.** Cargo.toml's `rust-version = "1.89"` is the library's minimum supported
   rustc, not the dev toolchain. The examples' aws-sdk crates need ≥1.94.1, so `./mise.toml` pins
   rust 1.98.1. Pinning the MSRV broke `cargo check` on the examples.
8. **`cargo test` dyld failure ("Library not loaded: /install/lib/libpython3.11.dylib"):**
   mise's relocatable python bakes `/install/...` as libpython's install name; pyo3 test binaries
   inherit it. Fixed via `DYLD_FALLBACK_LIBRARY_PATH` in `./mise.toml [env]` pointing at the mise
   python `lib/` dir. uv ≥0.13 drops `DYLD_*` from `uv run` children, so `./mise.toml` also sets
   `COCOINDEX_SKIP_UV=1`, which makes `dev/run_cargo_test.sh` call cargo directly.

## Runtime model — verified 2026-09-05

**cocoindex is a library, not a service.** No daemon, no background process, nothing to "start".
A flow runs as an ordinary process in the consuming project's venv and exits. Two modes:

- **Batch** (default): `app.update()` / `update_blocking()` — process changes, sync targets, exit.
- **Live**: `app.update(live=True)` — stays running, watches sources (watchdog), applies changes
  continuously; `watch()` yields progress snapshots. Shut down = stop the process / `Environment.stop()`.

Lifecycle: `Environment` start opens the LMDB state store; stop closes it. The CLI (`cocoindex ls`)
reads the same store.

**Consuming projects (tessera, ops-workspace) — editable path install.** Verified with two scratch
consumers against uv 0.8.23:

```toml
[tool.uv.sources]
cocoindex = { path = "/Users/trust/Dev/utils/cocoindex", editable = true }
```

- `[tool.uv] package = false` in the fork does NOT block consumers — the maturin build-system
  drives the PEP 517 build. Confirmed working.
- Compile cost is per-MACHINE, not per-project: first consumer sync 48s (warm cargo cache from
  the clone's `target/`), second consumer sync **0.15s** (uv cache hit).
- All editable consumers + the clone venv load the SAME compiled artifact:
  `python/cocoindex/_internal/core.abi3.so` in the clone. Consequences:
  - Python edits in the fork are live in every project immediately.
  - `uv run maturin develop` (in the clone) rebuilds Rust for every project at once.
  - The fork is a single mutable dependency shared across projects: a bad edit breaks all
    consumers simultaneously. Freeze via a built wheel if that ever matters.
- CLI entry points (`cocoindex` command) install correctly in consumers (never in the clone itself).

**Per-project isolation rule:** every project sets its own `COCOINDEX_DB` path (or `db_path`).
Never share an LMDB state dir across projects. Concurrent flows in different projects are fine.

**Doctor check** (run in any consumer or the clone; detects the one silent failure — stale Rust core):

```bash
uv run python -c 'import cocoindex; print(cocoindex.__version__)' \
&& { [ -n "$(find /Users/trust/Dev/utils/cocoindex/rust -name '*.rs' \
     -newer /Users/trust/Dev/utils/cocoindex/python/cocoindex/_internal/core.abi3.so 2>/dev/null | head -1)" ] \
     && echo 'STALE rust core — run in the clone: uv run maturin develop' \
     || echo 'rust core current'; }
```

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
