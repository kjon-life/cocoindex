# ccc (cocoindex-code) on the kjon-life/cocoindex fork — install & invocation

Status: installed, indexed, and verified on this Mac 2026-09-05. This document is the portable
instruction set: § MACHINE is already done here (repeat only on a new machine or after fork
changes); § PER-REPO is what an agent runs inside a target repo (tessera next).

**Agent quickstart (paste this to the in-repo agent, nothing more):**

> Read /Users/trust/Dev/utils/cocoindex/CCC-SETUP.md and execute § SKILL and § PER-REPO for
> this repo. § MACHINE is already done on this machine — install nothing, and ignore any
> install instructions inside the skill's references (they say pipx; that would fetch PyPI
> cocoindex instead of our fork).

## What this is

`ccc` is the cocoindex-code CLI: AST-aware semantic code search over a repo, for any coding
agent (Claude, Grok, Codex, Gemini — it's a plain CLI). Entirely local: SQLite index per repo,
local embedding model, no API keys, no network. It is a MACHINE-LEVEL uv tool, like prek.
**It is never a project dependency — nothing enters any repo's pyproject, lockfile, or images.**

Ours runs on the fork at `/Users/trust/Dev/utils/cocoindex` (which carries the AST layer:
41 tree-sitter grammars in `rust/code_ast`), not on PyPI cocoindex.

Verified stack: ccc 0.2.41 · cocoindex 999.0.0 built from the fork · python 3.12.13 ·
sentence-transformers 6.0.1 · model Snowflake/snowflake-arctic-embed-xs on `mps` (Apple GPU).

## MACHINE — install (done on this Mac)

Two config files, one command.

**1. `~/.cocoindex_code/fork-overrides.txt`** (exists):

```
cocoindex[litellm,sentence-transformers] @ file:///Users/trust/Dev/utils/cocoindex
```

Why every piece is load-bearing (all failure modes hit empirically):
- cocoindex-code 0.2.41 pins `cocoindex>=1.0.17,<1.1.0`. The fork's dev version is `999.0.0`,
  which violates the upper bound → without an override, uv **silently backtracks to
  cocoindex-code 0.1.3** (no `ccc` binary, no `full` extra).
- A bare-name override (`cocoindex`) is not enough: the registry's 1.0.21 then satisfies it
  (fork loses, even with `--with-editable`) **and** the override strips the extras, so
  sentence-transformers never installs. The override must carry **both** the extras and the
  file URL.
- Editable install into the tool env is not achievable with uv 0.8.23's resolver (editables
  lose to a satisfiable registry candidate). The frozen-wheel posture is deliberate anyway:
  a resident daemon shouldn't have its library mutate underneath it. Fork updates are an
  explicit refresh (below).

**2. `~/.cocoindex_code/global_settings.yml`** (exists):

```yaml
embedding:
  provider: sentence-transformers
  model: Snowflake/snowflake-arctic-embed-xs
  device: mps

daemon:
  idle_timeout_minutes: 180
  keep_alive_with_mcp: true
```

Writing this file BEFORE first init also skips ccc's interactive provider prompt (there is no
non-interactive flag for the local provider — only `--litellm-model` for cloud, which we don't use).

**3. Install** (pin python 3.12 explicitly — otherwise the tool env grabs the newest python on
the machine, and torch wheel coverage lags new pythons):

```bash
uv tool install 'cocoindex-code[full]==0.2.41' \
  --python /Users/trust/.local/share/mise/installs/python/3.12/bin/python3.12 \
  --overrides ~/.cocoindex_code/fork-overrides.txt
```

Success looks like: `+ cocoindex==999.0.0 (from file:///Users/trust/Dev/utils/cocoindex)` and
`Installed 2 executables: ccc, cocoindex-code`.

**Refresh after fork changes** (rebuilds the fork wheel from the tree, ~release-build minutes):

```bash
uv tool install 'cocoindex-code[full]==0.2.41' --reinstall \
  --python /Users/trust/.local/share/mise/installs/python/3.12/bin/python3.12 \
  --overrides ~/.cocoindex_code/fork-overrides.txt
ccc daemon restart
```

**Verify the fork is the provider** (do this after any install/refresh):

```bash
~/.local/share/uv/tools/cocoindex-code/bin/python -c \
  'import cocoindex; print(cocoindex.__version__, cocoindex.__file__)'
# → 999.0.0  …/uv/tools/cocoindex-code/lib/python3.12/site-packages/cocoindex/__init__.py
```

## PER-REPO — setup (what the tessera agent runs, from the repo root)

```bash
cd <repo-root>
ccc init -f          # -f skips the parent-directory warning; REQUIRED if any parent dir is
                     # also a ccc project, else init refuses and searches silently use the
                     # parent's index. Afterwards `ccc status` MUST show this repo's root.
# init creates .cocoindex_code/{settings.yml,…} and adds /.cocoindex_code/ to .gitignore —
# verify the .gitignore line landed; repo policy owns that file.

# Tune .cocoindex_code/settings.yml BEFORE first index. For tessera at minimum exclude:
#   **/node_modules, **/.venv, frontend/dist, _preview/**, **/.*
# and keep max_file_size modest (default 500KB). Include only source/docs languages you want
# searchable. This file is the per-repo taxonomy knob — everything else is global.

ccc index            # first build; benchmark from the fork repo: 916 files → 13,188 chunks
                     # in 94s on this Mac. Scales roughly with file count.
ccc status           # confirm root, chunk/file counts, language breakdown
ccc doctor           # full diagnostics if anything looks off
```

## SKILL — install /ccc per repo, without npx

The upstream docs say `npx skills add cocoindex-io/cocoindex-code`; that command only copies the
repo's `skills/ccc/` directory into the agent's skills dir. The canonical copy already lives on
this machine, fetched at tag **v0.2.41** (exactly matching the installed binary, so skill
instructions can't drift ahead of the CLI):

    ~/.cocoindex_code/skills/ccc/   (SKILL.md + references/{management,settings}.md)

Per-repo install (Claude Code project scope — do this as part of § PER-REPO):

```bash
mkdir -p <repo-root>/.claude/skills
cp -R ~/.cocoindex_code/skills/ccc <repo-root>/.claude/skills/ccc
```

Then `/ccc` works in that repo, and the skill's description auto-triggers on "search the
codebase" / "find code related to". The skill is generic (no repo-specific content), so the
same copy goes into every repo — index isolation still comes from each repo's own
`.cocoindex_code/`, not from the skill. Whether `.claude/skills/ccc/` gets committed or
gitignored is each repo's taxonomy call. Refresh path if ccc is ever upgraded: re-fetch
`skills/ccc` from the cocoindex-code repo at the tag matching the new version, replace the
canonical copy, re-copy into repos.

## INVOCATION — for agents (any brand)

```bash
ccc search "natural language question about the code"     # semantic, ~0.8s warm
ccc search --refresh "…"       # re-index changed files first, then search (see Daemon truth)
ccc search --lang python "…"   # filter by language
ccc search --path 'src/graph/**' "…"                      # filter by path glob
ccc search --limit 5 --offset 10 "…"                      # pagination
ccc grep <pattern>             # structural search by example; needs no index at all
ccc status | ccc doctor        # health
ccc daemon status|restart|stop # daemon control
ccc reset [--all] [-f]         # delete index (and settings with --all)
```

MCP is available (`ccc mcp`; `claude mcp add cocoindex-code -- ccc mcp`) but the fork's stance
is deliberate: **prefer the plain CLI** — it costs agents zero standing context and behaves
identically across agent brands.

## Daemon — the truth, code-verified

- Spawns automatically on the first `ccc` command; keeps the embedding model resident so
  index/search never pay model-load time.
- **It does NOT watch the filesystem.** There is no watcher in cocoindex-code 0.2.41 (verified
  in the installed package source). The index updates only when something asks: `--refresh` on
  a search, or `ccc index`. Both are incremental — only changed files re-chunk/re-embed —
  and cheap with the model resident. Practical rule for agents: append `--refresh` whenever
  freshness matters; it is one flag, not a tree-sitter walk in tokens.
  If zero-touch freshness is ever wanted, schedule `ccc index` (cron/launchd) per repo —
  decide per repo, it is not on by default.
- Measured footprint at idle (this Mac): ~81 MB daemon + ~207 MB model worker + ~6 MB tracker
  ≈ **295 MB RSS, 0.0% CPU**. Auto-exits after 180 idle minutes (`daemon.idle_timeout_minutes`),
  so it is not permanently resident.
- The daemon inherits the environment of whichever shell first invokes it (`ccc doctor` prints
  the full list — API keys included). If that matters, first-invoke from a clean shell.
- Logs: `~/.cocoindex_code/daemon.log`. Index DBs per repo: `<repo>/.cocoindex_code/*.db`.

## UNINSTALL — complete exit (everything is local files; nothing was ever deployed)

Per repo (repeat for each initialized repo):

```bash
ccc daemon stop                      # once, before removing things it has open
rm -rf <repo>/.cocoindex_code        # index + project settings (or: ccc reset --all -f)
rm -rf <repo>/.claude/skills/ccc     # the skill copy
# remove the /.cocoindex_code/ line from the repo's .gitignore
# remove the SessionStart hook from .claude/settings.json if one was added
```

Machine level:

```bash
uv tool uninstall cocoindex-code     # removes ccc + the tool venv (incl. the fork wheel copy)
rm -rf ~/.cocoindex_code             # global settings, fork-overrides, skill canonical copy, daemon.log
rm -rf ~/.cache/huggingface/hub/models--Snowflake--snowflake-arctic-embed-xs   # embedding model cache
```

The fork clone at `~/Dev/utils/cocoindex` is independent (it's the library source, not part of
ccc's install) — keep or remove it on its own merits. No repo's pyproject, lockfile, image, or
database ever referenced any of this.

## Trap list (all hit during this install — do not repeat)

1. `uv tool install 'cocoindex-code[full]'` with no override → silent backtrack to 0.1.3.
2. Bare-name override → registry cocoindex wins AND extras stripped (no sentence-transformers).
3. `--with-editable` into a tool env → loses to the registry; use the file-URL override.
4. No `--python` pin → tool env takes the newest python on the machine (3.14 here).
5. `ccc init` without pre-written global settings → interactive provider prompt.
6. `ccc init` without `-f` under a parent ccc project → refuses, or searches silently hit the
   parent's index. Always confirm with `ccc status`.
