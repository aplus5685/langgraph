# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository layout

This is a **monorepo**. Each publishable library lives in its own directory under `libs/` with its own `pyproject.toml` / `package.json`, `uv.lock`/`yarn.lock`, `Makefile`, and `tests/` directory. **Scope changes to a single library per PR** (see `CONTRIBUTING.md`).

Libraries (see `AGENTS.md` for the authoritative list):

- `libs/langgraph` — core framework (graph builder, Pregel runtime, channels, managed values)
- `libs/prebuilt` — higher-level agent helpers (e.g. `create_react_agent`)
- `libs/checkpoint` — base interfaces for checkpointers and stores
- `libs/checkpoint-sqlite`, `libs/checkpoint-postgres` — checkpointer implementations
- `libs/cli` — `langgraph` CLI (dev server, schema generation, Docker tooling)
- `libs/sdk-py` — Python SDK for the LangGraph Platform API
- `libs/sdk-js` — JS/TS SDK (standalone; uses yarn + vitest, not uv/pytest)

Other top-level dirs: `docs/` (MkDocs site + notebooks), `examples/` (user-facing example notebooks), `.github/workflows/` (CI).

### Dependency graph

`checkpoint` → `checkpoint-{postgres,sqlite}`, `prebuilt`, `langgraph`
`prebuilt` → `langgraph`
`sdk-py` → `langgraph`, `cli`
`sdk-js` is standalone.

Changes in a base library can break everything downstream — when editing `checkpoint` or `prebuilt`, consider running the `langgraph` test suite too.

## Commands

All Python work goes through each library's own `Makefile`, run from inside the library directory. The root `Makefile` just fans these out across every lib.

From inside a library under `libs/<name>/`:

```bash
make format      # ruff format + import sort
make lint        # ruff check + mypy
make test        # pytest for that library
```

**Run a single test or pass pytest args** (convention across libs, defined in `AGENTS.md`):

```bash
TEST=tests/test_foo.py make test
TEST="tests/test_foo.py::test_bar -x -vv" make test
```

### Library-specific notes

- **`libs/langgraph`**: `make test` auto-spins a Postgres container (`tests/compose-postgres.yml`) and a local LangGraph dev server (`make start-dev-server`) before running pytest, then tears them down. Set `NO_DOCKER=true` to skip Docker-dependent tests. Use `make test_parallel` for `pytest-xdist` parallel runs. Benchmarks: `make benchmark` / `make benchmark-fast` (writes `out/benchmark.json`). Profiling: `make profile GRAPH=bench/<file>.py`.
- **`libs/checkpoint-postgres`**: `make test` iterates over `POSTGRES_VERSIONS` (default `15 16`), bringing up and tearing down a Postgres container per version. Requires Docker.
- **`libs/cli`**: splits `make test` (unit) from `make test-integration`. `make update-schema` regenerates JSON schemas via `generate_schema.py`.
- **`libs/sdk-js`**: use `yarn install`, then `yarn build` / `yarn lint` / `yarn test` (vitest). Not integrated with the root `make`.

Install everything from the repo root with `make install` (creates a `uv` venv and `uv pip install -e` every Python lib).

### Docs

Docs live in `docs/` and are built with MkDocs from notebooks + markdown.

```bash
cd docs
make serve-docs     # local preview with live reload on ../libs/{langgraph,checkpoint,sdk-py}
make build-docs     # full build (runs build-typedoc + build-prebuilt first)
make clean-docs     # remove built site + deletes .ipynb from ./docs (careful)
make tests          # docs unit tests under docs/tests/unit_tests
```

Spell check is run from the repo root: `make spellcheck` (via `codespell`).

## Architecture notes

### Pregel runtime (`libs/langgraph/langgraph/pregel/`)

LangGraph is "Pregel-inspired" (per the README): execution is a loop of **supersteps**, each of which reads from channels, runs ready nodes, and writes back. The engine, not the graph object, is the unit to understand when debugging runtime behavior.

- `pregel/loop.py` — the main superstep loop (`PregelLoop`, `SyncPregelLoop`, `AsyncPregelLoop`); drives tick/checkpoint/interrupt handling.
- `pregel/algo.py` — planning: decides which tasks are ready given current channel versions.
- `pregel/runner.py`, `executor.py` — task execution (sync + async variants, retries, cancellation).
- `pregel/io.py` — input/output mapping between user payloads and channel updates.
- `pregel/checkpoint.py` — glue between the loop and the `BaseCheckpointSaver` interface.
- `pregel/remote.py` — `RemoteGraph` for calling graphs deployed on LangGraph Platform.
- `pregel/retry.py`, `pregel/types.py`, `pregel/protocol.py` — retry policies, `PregelTask`/`StateSnapshot`, and the `PregelProtocol` that `Pregel` and `RemoteGraph` both implement.

### Channels (`libs/langgraph/langgraph/channels/`)

Channels are the typed, versioned state slots that nodes read/write. Subclasses of `BaseChannel` define the update semantics:

- `LastValue` / `LastValueAfterFinish` — single value, overwrite on write.
- `BinaryOperatorAggregate` — reducer (e.g. `add_messages`, `operator.add`).
- `Topic` — append-only pub/sub.
- `EphemeralValue` — not persisted between supersteps.
- `NamedBarrierValue` — waits for all named writers.

`StateGraph` compiles user-declared `TypedDict`/`BaseModel`/`dataclass` state into channels automatically based on `Annotated[...]` reducers.

### Graph builder (`libs/langgraph/langgraph/graph/`)

- `graph/state.py` — `StateGraph`, the main user-facing builder; compiles to a `Pregel` instance.
- `graph/message.py` — `MessagesState`, `add_messages` reducer, legacy `MessageGraph`.
- `graph/branch.py` — conditional edges.
- `graph/ui.py` — UI message helpers.

### Managed values (`libs/langgraph/langgraph/managed/`)

Runtime-managed, non-persisted state like `RemainingSteps`, `IsLastStep` — injected into node args but not stored as regular channel values.

### Functional API (`libs/langgraph/langgraph/func/`)

`@entrypoint` / `@task` decorators — the imperative alternative to `StateGraph`. Built on the same Pregel runtime.

### Checkpointers and stores (`libs/checkpoint*`)

`BaseCheckpointSaver` (in `libs/checkpoint/langgraph/checkpoint/`) defines the persistence API the Pregel loop uses to save/resume state at superstep boundaries. `BaseStore` (in `libs/checkpoint/langgraph/store/`) defines the long-term memory key-value API. `checkpoint-sqlite` and `checkpoint-postgres` are the concrete implementations and are where schema/migration changes belong.

### Prebuilt (`libs/prebuilt`)

`create_react_agent` and tool-execution helpers. Depends on `langgraph` core — changes here should not need to reach into the Pregel runtime.

## Conventions

- **Backwards compatibility is a hard requirement** (per `CONTRIBUTING.md`). Public API changes that aren't bug/security fixes should be additive or deprecated-first. Changes ship as minor/patch releases per library, so preserve existing kwargs and return shapes.
- **Docstrings → public API reference.** The `docs/` site's API reference is autogenerated from docstrings; follow the Google Python Style Guide format shown in `CONTRIBUTING.md`.
- **Ruff + mypy** enforce style and types; run `make format` before `make lint`. Linting is mandatory in CI (`.github/workflows/_lint.yml`).
- **Tests accompany bug fixes.** For a bug fix, add a test that fails before the patch (per `CONTRIBUTING.md`).
- **Notebooks in `docs/` and `examples/`** are part of the docs build. `docs/codespell_notebooks.sh` and the `spell_check` targets run against them; keep cells runnable and use `make clean-docs` only intentionally (it deletes all `.ipynb` from `./docs`).
