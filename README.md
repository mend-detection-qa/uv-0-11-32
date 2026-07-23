# Probe: workspace-canonical-lockfile-v0-11-32

## Probe metadata

| Field                 | Value                                           |
|-----------------------|-------------------------------------------------|
| Pattern               | `workspace-canonical-lockfile-v0-11-32`         |
| PM                    | uv                                              |
| pm_version_under_test | 0.11.32                                         |
| Schema version        | 1.2                                             |
| Categories            | `lockfile_format`, `tree_command`               |
| Generated at          | 2026-07-23T23:31:32Z                            |
| Generator             | dispatcher → python-uv:project-creator          |

## What this probe exercises

uv 0.11.32 introduced two behavioral changes relevant to Mend SCA:

### 1. Lockfile canonicalization enforcement (`lockfile_format`)

`uv lock --check` and `--locked` now reject non-canonical `uv.lock`
formats. The `--refresh` flag regenerates the lockfile in canonical
form. Canonical format enforces:

- `[[package]]` blocks sorted **alphabetically by name**.
- Consistent field ordering within each block:
  `name`, `version`, `source`, `dependencies`, `sdist`, `wheels`.
- Hash arrays on separate lines per wheel entry.
- Workspace member entries use `source = { editable = "..." }`.
- Virtual root has no `[[package]]` entry.

This probe's `uv.lock` is written in exactly this canonical form.
Mend's lockfile parser must accept it without error.

### 2. Workspace metadata output format change (`tree_command`)

`uv workspace metadata` output format was modified in 0.11.32.
Tools that consume `uv workspace metadata` JSON (including
Mend's dependency-tree command parser) must handle the updated
shape. This workspace structure exercises that path: a virtual
root with two named members where one member (`worker`) depends
on another (`api`) via a workspace cross-reference.

## Project structure

```
workspace-canonical-lockfile-v0-11-32-20260723-233132/
├── pyproject.toml              # Virtual workspace root (no [project])
├── uv.lock                     # Canonical lockfile (uv 0.11.32 format)
├── .python-version             # 3.11 (PIP chain: higher precedence)
├── packages/
│   ├── api/
│   │   ├── pyproject.toml      # httpx, structlog
│   │   └── src/api/__init__.py
│   └── worker/
│       ├── pyproject.toml      # celery + api (workspace dep)
│       └── src/worker/__init__.py
├── README.md
└── expected-tree.json
```

## Dependency graph

```
(virtual root)
├── api (local/editable)
│   ├── httpx 0.27.2
│   │   ├── anyio 4.4.0
│   │   │   ├── idna 3.7
│   │   │   └── sniffio 1.3.1
│   │   ├── certifi 2024.7.4
│   │   ├── httpcore 1.0.5
│   │   │   ├── certifi 2024.7.4
│   │   │   └── h11 0.14.0
│   │   ├── idna 3.7
│   │   └── sniffio 1.3.1
│   └── structlog 24.4.0
└── worker (local/editable)
    ├── api (workspace, local/editable)
    └── celery 5.4.0
        ├── billiard 4.2.0
        ├── click 8.1.7
        ├── click-didyoumean 0.3.1
        ├── click-plugins 1.1.1
        ├── click-repl 0.3.0
        │   ├── click 8.1.7
        │   └── prompt-toolkit 3.0.47
        │       └── wcwidth 0.2.13
        ├── kombu 5.3.7
        │   ├── amqp 5.2.0
        │   │   └── vine 5.1.0
        │   └── vine 5.1.0
        ├── python-dateutil 2.9.0.post0
        │   └── six 1.16.0
        ├── tzdata 2024.1
        └── vine 5.1.0
```

## Python version detection

Mend SCA follows the PIP precedence chain for Python version
detection. **`.python-version` has higher precedence than
`[project] requires-python` in `pyproject.toml`.**

This probe emits:
- `.python-version` → `3.11` (Mend will use this)
- Member `pyproject.toml` files → `requires-python = ">=3.11"`

Both declare compatible versions. Mend will read `3.11` from
`.python-version`.

## Mend config

**Bucket B — no `.whitesource` (dynamic Python detection covers it;
uv tool itself not pinnable).**

uv is not in the `install-tool` list, so the exact uv version
cannot be pinned via `scanSettings.versioning`. Only `python`
can be pinned via versioning if needed. For this probe, dynamic
detection from `.python-version` (Python 3.11) is sufficient
— no `.whitesource` is emitted.

## Expected Mend behavior

Mend's UA runs through the UV Project Filtering path
(`MEND_SCA_UV_PROJECTS`) and the Pip resolver (uv projects are
resolved via the pip-compatible path, reading `uv.lock` as a
lockfile). Expected outputs:

- `api` and `worker` detected as `source: local` (editable workspace
  members, not PyPI packages).
- All remote registry deps correctly attributed to PyPI.
- Cross-member dep `worker → api` correctly recorded as local.
- Full transitive closure from both members included.

## Known Mend failure modes (probe targets)

1. **Canonical lockfile parse failure**: Mend's lockfile parser
   may reject the new `uv.lock` canonical format if it hard-codes
   assumptions about field ordering or hash layout from pre-0.11.32
   lockfiles.

2. **workspace metadata output drop**: If Mend calls `uv workspace
   metadata` and the output format changed, the parser may silently
   return an empty dep list for workspace members.

3. **Member misclassification**: `api` and `worker` reported as
   registry packages (PyPI) instead of `source: local`.

4. **Virtual root handling**: Virtual root (no `[project]` in root
   `pyproject.toml`) may cause a parse failure or cause Mend to
   scan only the root and miss both members.

5. **Cross-member dep duplication**: `api` appearing twice — once
   under its own member scan and once as a dep of `worker`.
